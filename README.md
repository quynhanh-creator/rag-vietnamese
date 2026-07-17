# Vietnamese RAG Pipeline

RAG cho tài liệu tiếng Việt: hybrid retrieval (dense + sparse), 
parent-child chunking, rerank.

## Kiến trúc

**Indexing:** semantic chunk → parent 2000 → child 400 → BGE-M3 → Qdrant  
**Query:** multi-query → hybrid search → RRF → rerank → LLM

## Điểm đáng chú ý

- **Không dùng BM25.** BGE-M3 sinh dense + sparse trong 1 forward, 
  nhánh sparse gần như miễn phí. Thêm BM25 phải thêm bộ tách từ tiếng Việt.
- **Parent-child chunking.** Child 400 ký tự để search chính xác, 
  parent 2000 ký tự để LLM đủ ngữ cảnh.
- **PyMuPDF** thay PyPDF vì trích text tiếng Việt tốt hơn.

## Tham số chính

| Tham số | Giá trị | Ý nghĩa |
|---|---|---|
| SEMANTIC_THRESHOLD_AMOUNT | 95 | Ngưỡng cắt semantic chunk |
| MAX_PARENT_SIZE | 2000 | Trần ngữ cảnh trả cho LLM |
| CHILD_CHUNK_SIZE | 400 | Đơn vị được embed và search |
| CHILD_CHUNK_OVERLAP | 50 | Overlap giữa các child |
| VECTOR_DIM | 1024 | Chiều vector dense |

## Trạng thái

| Hạng mục | |
|---|---|
| Hybrid search (dense + sparse) | ✅ |
| Parent-child chunking | ✅ |
| Rerank | ✅ |
| Multi-query + RRF | ✅ |
| Incremental index | ❌ đang full rebuild |
| Eval | 🚧 |

## License

Apache 2.0
