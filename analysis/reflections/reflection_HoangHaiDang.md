# Individual Reflection — Lab 18: Production RAG Pipeline

**Tên:** Hoàng Hải Đăng  
**MSSV:** 2A202600916

---

## 1. Phần 1: Mapping bài giảng → Code

| Lecture Concept | Module | Hàm cụ thể | Observation |
|----------------|--------|-------------|-------------|
| Semantic chunking | M1 | `chunk_semantic()` | Dùng cosine similarity threshold 0.85 để nhóm câu cùng chủ đề. Tạo ít chunks hơn basic paragraph splitting. |
| Hierarchical chunking | M1 | `chunk_hierarchical()` | Parent (2048 chars) + Child (256 chars). Retrieve child → return parent giúp precision cao hơn mà vẫn giữ context. |
| Structure-aware chunking | M1 | `chunk_structure_aware()` | Parse markdown header (#, ##, ###) → chunk theo section. Giữ nguyên tables, lists, code blocks. |
| Vietnamese segmentation | M2 | `segment_vietnamese()` | Dùng underthesea word_tokenize + replace "_" bằng space để BM25 tokenize đúng từ ghép tiếng Việt (VD: "nghỉ phép" → "nghỉ_phép" → "nghỉ phép"). |
| BM25 + Dense fusion | M2 | `reciprocal_rank_fusion()` | RRF (k=60) gộp rank từ BM25 và dense search. score(d) = Σ 1/(k + rank(d) + 1). Giải quyết vocabulary gap giữa query và document. |
| Cross-encoder reranking | M3 | `CrossEncoderReranker.rerank()` | CrossEncoder (bge-reranker-v2-m3) predict pairs (query, doc) → score relevance. Tốn latency hơn bi-encoder nhưng precision cao hơn. |
| RAGAS 4 metrics | M4 | `evaluate_ragas()` | Faithfulness (hallucination), Answer Relevancy, Context Precision, Context Recall. Faithfulness là metric quan trọng nhất. |
| Text enrichment | M5 | `summarize_chunk()`, `generate_hypothesis_questions()`, `contextual_prepend()`, `extract_metadata()`, `_enrich_single_call()` | Combined mode: 1 API call/chunk cho cả summary + questions + context + metadata. Giảm 75% API cost so với 4 calls riêng lẻ. |

## 2. Phần 2: Khó khăn & giải quyết

| Khó khăn | Cách giải quyết |
|----------|----------------|
| underthesea word_tokenize nối từ ghép bằng "_" -> BM25 không match query "nghỉ phép" với token "nghỉ_phép" | `replace("_", " ")` sau khi segment để BM25 split đúng |
| Qdrant client >= 2.0 dùng API `query_points()` thay vì `search()` | Đọc docs và dùng `query_points()` đúng version |
| FlagReranker crash với transformers>=5.0 (XLMRobertaTokenizer lỗi) | Dùng `sentence_transformers.CrossEncoder` thay vì FlagEmbedding |
| RAGAS cần OPENAI_API_KEY + Groq API không có `set_run_config` + OpenAIEmbeddings | Dùng ChatOpenAI với base_url=groq + LangchainLLMWrapper + HuggingFaceEmbeddings. Groq `n=1 only` → override generate_text. |
| PDF scan ảnh không có text layer | pypdf trả về "" -> bỏ qua kèm warning, cần OCR pipeline riêng |
| Model download từ HuggingFace rất chậm (bge-reranker 1.4GB) | Chạy pre-download scripts trước, hoặc chờ network tốt hơn |
| UnicodeEncodeError khi in tiếng Việt ra console Windows | Set `$env:PYTHONIOENCODING='utf-8'` |

## 3. Phần 3: Action Plan cho Project

## Project: Hệ thống HR Chatbot cho doanh nghiệp

### Hiện tại
- RAG pipeline hiện tại: Chưa có, đang thiết kế MVP
- Known issues: Dữ liệu HR phân tán nhiều version (v2023/v2024), cần xử lý versioning

### Plan áp dụng
1. [x] Chunking strategy: Hierarchical (parent 2048, child 256) + Structure-aware cho markdown policies. Lý do: HR documents có cấu trúc rõ ràng, cần giữ section context.
2. [x] Search: Hybrid (BM25 + Dense + RRF). Lý do: Tiếng Việt cần BM25 cho keyword matching, dense search cho semantic understanding.
3. [x] Reranking: Cross-encoder (bge-reranker-v2-m3). Lý do: Giảm noise từ top-20 xuống top-3, cải thiện context precision.
4. [x] Evaluation: RAGAS 4 metrics (faithfulness, answer_relevancy, context_precision, context_recall). Lý do: Đánh giá toàn diện quality pipeline. Score: F=0.83, AR=0.10, CP=0.40, CR=0.93 (Groq free tier, 5 questions).
5. [x] Enrichment: Combined mode (1 call/chunk). Lý do: Tiết kiệm cost, tăng retrieval quality nhờ contextual prepend + HyQA.

### Timeline
- Tuần 1: Setup chunking strategies cho HR documents + indexing
- Tuần 2: Implement hybrid search + reranking
- Tuần 3: RAGAS evaluation + failure analysis
- Tuần 4: Enrichment pipeline + optimization
- Tuần 5: Deployment + monitoring

### Self-assessment

| Tiêu chí | Tự chấm (1-5) |
|----------|---------------|
| Hiểu bài giảng | 4 |
| Code quality | 4 |
| Problem solving | 4 |
| Testing | 4 |

**Total effort:** ~2.5h implementation + 30m reflection
