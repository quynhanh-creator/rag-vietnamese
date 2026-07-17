# Kiến trúc

Hệ thống chia làm hai pha độc lập: **indexing** (chạy nền, một lần) 
và **query** (real-time, mỗi câu hỏi).

---

## Pha 1 — Indexing
file (.pdf .docx .txt .md)
→ load_documents
→ semantic_chunking      (percentile 95)
→ cap parent             (MAX_PARENT_SIZE = 2000)
→ child_chunking         (400 / overlap 50)
→ encode_batch           (BGE-M3 → dense + sparse)
→ upsert                 (Qdrant)

### 1. Load documents

| Định dạng | Loader |
|---|---|
| `.pdf` | PyMuPDFLoader |
| `.docx` `.doc` | Docx2txtLoader |
| `.txt` `.md` | TextLoader (UTF-8) |

**PyMuPDF thay vì PyPDF** — trích text tiếng Việt tốt hơn đáng kể.

> **Đây là nơi dữ liệu thật sự bị mất.** File load lỗi chỉ ghi 
> warning rồi bỏ qua — job vẫn báo "done". Chunking không làm mất 
> chữ, nó chỉ cắt. Nếu thiếu nội dung, điều tra loader trước.

### 2. Semantic chunking

Cắt tại điểm ngữ nghĩa đứt gãy thay vì cắt cứng theo ký tự. 
SemanticChunker embed từng câu (chỉ dense), đo khoảng cách giữa 
các câu liền kề, cắt ở nơi khoảng cách vượt ngưỡng.

| Tham số | Giá trị |
|---|---|
| breakpoint_threshold_type | percentile |
| breakpoint_threshold_amount | 95 |

Ngưỡng 95 = chỉ cắt ở 5% điểm đứt gãy mạnh nhất → chunk to hơn, 
ý trọn vẹn hơn.

### 2b. Cap parent

Semantic chunk có thể dài tùy ý. Cắt tiếp bằng 
RecursiveCharacterTextSplitter để không vượt `MAX_PARENT_SIZE = 2000` 
(overlap 200). Đây là thứ chặn "context dài không kiểm soát".

### 3. Child chunking

Mỗi parent nhận một `parent_id` (uuid), rồi tách thành child 
400 ký tự / overlap 50. **Child là đơn vị được embed và search.**

Mỗi child mang 2 metadata:
- `parent_id` — để nhóm và dedupe
- `parent_text` — toàn bộ nội dung parent, nhân bản vào từng child

Nhờ vậy search ra child là có ngay context đầy đủ, không cần join.

### 4. Encoding

BGE-M3 (`BAAI/bge-m3`), `use_fp16=True`, batch 32. Model load 
một lần mỗi worker qua singleton có lock.

Sinh **đồng thời** trong một forward pass:
- `dense` — 1024 chiều, ngữ nghĩa
- `sparse` — vector thưa, trọng số theo token

### 5. Upsert Qdrant

| Vector | Cấu hình |
|---|---|
| dense | size 1024 · COSINE · name `"dense"` |
| sparse | SparseVectorParams · name `"sparse"` |

Mỗi child = 1 point. Payload gồm `page_content` + metadata. 
Upsert theo batch 100.

---

## Pha 2 — Query
câu hỏi
→ sinh query variants     (LLM paraphrase → 4 query)
→ encode 4 query          (BGE-M3, 1 forward)
→ hybrid search           (Qdrant query_points, RRF nội bộ)
→ RRF gộp chéo            (k = 60)
→ rerank                  (bge-reranker-v2-m3 → top 5)
→ dedupe parent_id → lấy parent_text
→ LLM sinh câu trả lời    (stream SSE)

**Dedupe là bước dễ sót.** Nhiều child có thể cùng một parent. 
Không dedupe thì cùng một đoạn 2000 ký tự bị nhét vào prompt 
nhiều lần — tốn token và làm LLM tưởng thông tin đó quan trọng 
gấp đôi.

---

## Quyết định thiết kế

### Hybrid = hai nhánh embedding, không có BM25

Tài liệu IR kinh điển định nghĩa hybrid = term-based (BM25) + 
embedding-based. Repo này **không** dùng BM25.

**Lý do:**

1. **"BM25 rẻ hơn" không áp dụng khi đã chạy BGE-M3.** Dense và 
   sparse ra từ cùng một forward pass — sparse gần như miễn phí. 
   Thêm BM25 nghĩa là thêm một hệ thống song song để index và 
   vận hành.

2. **Tiếng Việt làm BM25 đắt hơn nhiều.** BM25 cần tokenize trước. 
   "học máy" là một từ hai âm tiết — cắt theo khoảng trắng là hỏng. 
   Phải thêm bộ tách từ (VnCoreNLP/underthesea). BGE-M3 dùng subword 
   tokenizer đa ngôn ngữ, né được cả lớp vấn đề này.

3. **Learned sparse chịu biến thể tốt hơn.** "E-4092" vs "E4092" 
   vs "lỗi 4092" — BM25 khớp mặt chữ nên có thể trượt.

**Hệ quả về thuật ngữ:** nhánh sparse ở đây **không phải** 
term-based. Nó thuộc họ SPLADE — sparse embedding, bản chất gần 
dense hơn gần BM25. Đừng gọi nó là "keyword search".

**Cái giá:** mất khả năng giải thích. BM25 chỉ được chính xác từ 
nào khớp; learned sparse thì không.

**Khi nào nên xét lại:** corpus toàn part number / mã hàng cần 
chính xác từng ký tự; hoặc đã có sẵn Elasticsearch trong hạ tầng.

### Parent-child

Child nhỏ (400) → embedding tập trung, match chính xác.  
Parent lớn (2000) → LLM đủ ngữ cảnh để trả lời.

Tách bài toán *tìm* khỏi bài toán *trả lời*.

**Cái giá:** `parent_text` nhân bản vào mọi child → payload phình. 
Parent 2000 ký tự chia ~5 child = lưu 5 lần. Đổi lại: lấy context 
không cần join.

---

## Hạn chế đã biết

| Vấn đề | Trạng thái |
|---|---|
| `delete_collection` mỗi lần index → index 1 file xóa hết file cũ | Chưa sửa. Cần chuyển sang upsert theo uuid + delete có filter |
| Point ID tuần tự (`i + j`) | Chỉ an toàn vì collection bị recreate. Bỏ recreate là ID đè nhau |
| Encode 2 lượt — semantic chunking encode câu, rồi encode lại child | Tốn GPU nhất với corpus lớn |
| Chưa có eval | Mọi tham số (95, 2000, 400, k=60) hiện là mặc định, chưa đo |
| Docstring ghi "BGE-M3 via HTTP" nhưng thực tế load model local | Comment sai, dễ gây nhầm khi debug |
