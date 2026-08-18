# Individual Reflection — Lab 18

**Tên:** Nguyễn Cao Quang Anh
**Module phụ trách:** Cá nhân — cả 5 module (M1–M5)

---

## 1. Đóng góp kỹ thuật

- Module đã implement: M1 (Chunking), M2 (Hybrid Search), M3 (Rerank), M4 (RAGAS Eval), M5 (Enrichment)
- Các hàm/class chính đã viết:
  - M1: `chunk_semantic()`, `chunk_hierarchical()`, `chunk_structure_aware()`
  - M2: `segment_vietnamese()`, `BM25Search.index/search()`, `DenseSearch.index/search()`, `reciprocal_rank_fusion()`
  - M3: `CrossEncoderReranker._load_model()`, `.rerank()`
  - M4: `evaluate_ragas()`, `failure_analysis()`
  - M5: `summarize_chunk()`, `generate_hypothesis_questions()`, `contextual_prepend()`, `extract_metadata()`, `_enrich_single_call()`
- Số tests pass: 37/37

## 2. Kiến thức học được

- Khái niệm mới nhất: Reciprocal Rank Fusion (RRF) — cách kết hợp 2 ranking list khác thang điểm (BM25 score vs cosine similarity) mà không cần chuẩn hoá, chỉ dựa vào rank.
- Điều bất ngờ nhất: production pipeline (hierarchical chunk + rerank + enrichment) có faithfulness và answer_relevancy cao hơn naive baseline, nhưng context_recall lại thấp hơn (0.9250 → 0.8167) — chunk nhỏ hơn giúp precision nhưng dễ bỏ sót thông tin khi câu hỏi multi-hop hoặc liên quan đến bảng dài.
- Kết nối với bài giảng: mapping cụ thể ở bảng phần 1 dưới đây.

| Lecture Concept | Module | Hàm cụ thể | Observation |
|----------------|--------|-------------|-------------|
| Semantic chunking | M1 | `chunk_semantic()` | Threshold 0.85 tách theo cosine similarity giữa câu liền kề, không theo giới hạn ký tự cố định như hierarchical. |
| Parent-child hierarchy | M1 | `chunk_hierarchical()` | 100 child chunk (≤256 ký tự) từ 26 document, mỗi child có `parent_id`. Retrieve child chính xác hơn nhưng dễ cắt đứt bảng/danh sách dài. |
| BM25 + Dense fusion | M2 | `reciprocal_rank_fusion()` | BM25 tốt cho exact match ("mật khẩu 90 ngày"), dense tốt cho paraphrase. RRF kết hợp 2 nguồn cho recall cao hơn từng loại riêng. |
| Cross-encoder reranking | M3 | `CrossEncoderReranker.rerank()` | bge-reranker-v2-m3 tải model lần đầu ~5 phút (mạng chậm), sau đó cache nhanh. Rerank đẩy đúng đoạn "nghỉ phép" lên top so với "VPN" không liên quan. |
| RAGAS 4 metrics | M4 | `evaluate_ragas()` | Faithfulness thấp nhất ở câu numeric/multi-hop — model suy diễn thay vì bám sát context khi cần tính toán hoặc ghép nhiều nguồn. |
| Contextual embeddings | M5 | `contextual_prepend()` / `_enrich_single_call()` | Enrichment giúp answer_relevancy tăng +0.10 so với naive — summary + hypothesis questions bridge khoảng cách từ vựng giữa câu hỏi và chunk gốc. |

## 3. Khó khăn & Cách giải quyết

- Khó khăn lớn nhất: `WinError 206: The filename or extension is too long` khi `pip install` torch — do đường dẫn project quá dài (tên thư mục gốc dài, lồng nhiều cấp), file license nested của torch vượt giới hạn 260 ký tự của Windows.
- Cách giải quyết: bật Windows Long Path support (`HKLM\SYSTEM\CurrentControlSet\Control\FileSystem\LongPathsEnabled = 1`) và đổi tên thư mục project ngắn lại (`Day18-NguyenCaoQuangAnh`). Ngoài ra còn gặp `UnicodeEncodeError` khi in tiếng Việt ra console Windows (cp1252) — set `PYTHONUTF8=1` trước khi chạy script; và `ReadTimeoutError` khi tải package nặng do mạng chậm — tăng `--timeout`/`--retries` cho pip.
- Thời gian debug: khoảng 30–40 phút cho phần setup môi trường (long path + encoding + pip timeout) trước khi vào code.

## 4. Nếu làm lại

- Sẽ làm khác điều gì: đặt tên project ngắn ngay từ đầu, bật Long Path support trước khi setup thay vì gặp lỗi mới sửa; dùng structure-aware chunking cho các file có bảng (lương, mua sắm) thay vì áp hierarchical cho toàn bộ corpus.
- Module nào muốn thử tiếp: query decomposition cho câu hỏi multi-hop (M2), vì đây là nguyên nhân chính khiến context_recall giảm so với baseline.

## 5. Tự đánh giá

| Tiêu chí | Tự chấm (1-5) |
|----------|---------------|
| Hiểu bài giảng | 5 |
| Code quality | 5 |
| Teamwork | (bài cá nhân) |
| Problem solving | 5 |

---

## Action Plan cho project

### Hiện tại
- RAG pipeline hiện tại: chưa có, hoặc đang dùng basic paragraph chunking + dense-only.
- Known issues: context recall giảm khi câu hỏi multi-hop hoặc bảng bị chunk cắt đứt.

### Plan áp dụng
1. [x] Chunking strategy: hierarchical (parent 2048/child 256) làm mặc định; chuyển sang structure-aware cho file có bảng/markdown table.
2. [x] Search: Hybrid (BM25 + Dense qua RRF) — BM25 xử lý exact-match tiếng Việt, dense xử lý paraphrase.
3. [x] Reranking: có, dùng CrossEncoder bge-reranker-v2-m3, top-20 → top-3.
4. [x] Evaluation: RAGAS 4 metric, kèm failure analysis bottom-N để tìm root cause theo diagnostic tree.
5. [x] Enrichment: combined single-call (1 API call/chunk) để tiết kiệm chi phí, ưu tiên contextual prepend vì tác động lớn nhất tới answer relevancy.

### Timeline
- Tuần 1: Áp dụng M1+M2 vào corpus thật, đo baseline.
- Tuần 2: Thêm M3+M5, chạy RAGAS, so sánh Δ.
- Tuần 3: Xử lý multi-hop/numeric failure (query decomposition), fix chunking bảng.
