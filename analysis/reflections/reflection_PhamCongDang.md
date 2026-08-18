# Individual Reflection — Lab 18

**Tên:** Phạm Công Đăng
**Module phụ trách:** M1 + M2 + M3 + M4 + M5 (bài cá nhân, tự làm toàn bộ 5 module)

---

## Phần 1: Mapping bài giảng → code

| Lecture Concept | Module | Hàm cụ thể | Observation |
|----------------|--------|-------------|--------------|
| Hierarchical (parent-child) chunking | M1 | `chunk_hierarchical()` | Corpus 26 doc (đã lọc 2 PDF scan) → **100 child chunks** (256 ký tự) khi chạy production, so với **57 chunk** của basic paragraph-split (baseline). Nhiều chunk nhỏ hơn = precision retrieval tốt hơn nhưng dễ vỡ ngữ cảnh dạng bảng (xem Case Study #3 trong failure_analysis.md). |
| BM25 + Dense fusion (RRF) | M2 | `reciprocal_rank_fusion()` | Search latency trung bình chỉ **98.2ms/query** (min 73ms, max 195ms) — rẻ hơn rerank/LLM rất nhiều lần. RRF giải quyết vấn đề BM25 khớp từ khóa chính xác (số liệu, tên riêng) trong khi Dense khớp ngữ nghĩa — hai điểm mạnh bù nhau, không cần chọn 1 trong 2. |
| Cross-encoder reranking | M3 | `CrossEncoderReranker.rerank()` | Latency **rất cao**: avg 5316ms/query, max 15034ms (lần đầu, gồm cả model load). Đây là bottleneck lớn nhất pipeline (so với search 98ms, LLM answer 2589ms) — mô hình `bge-reranker-v2-m3` (~568M params) chạy CPU-only nên chậm. Đổi lại, Context Precision tăng nhẹ (0.9250→0.9292). |
| RAGAS 4 metrics | M4 | `evaluate_ragas()` + `failure_analysis()` | Context Recall **giảm** 0.9250→0.7917 (metric duy nhất tệ đi) — cho thấy "production tốt hơn baseline" không phải lúc nào cũng đúng trên toàn bộ metric; phải nhìn từng metric riêng, không chỉ nhìn trung bình. |
| Contextual embeddings / Enrichment | M5 | `_enrich_single_call()` (combined mode) | 100 chunks enrich trong 452.8s (~4.5s/chunk, 1 API call/chunk theo đúng combined mode để tối ưu chi phí thay vì 4 call riêng). Answer Relevancy tăng mạnh nhất trong 4 metric (+0.0922) — enrichment (hypothesis questions + context prepend) giúp bridge vocabulary gap giữa câu hỏi người dùng và văn bản chính sách gốc, đúng như lý thuyết Anthropic contextual retrieval. |

## Phần 2: Khó khăn & giải quyết

**Khó khăn 1 — `docker compose up -d` báo lỗi:**
```
no configuration file provided: not found
```
- Nguyên nhân: chạy lệnh ở `C:\Users\phamc` thay vì thư mục project (không có `docker-compose.yml`).
- Cách giải quyết: `cd` đúng thư mục project trước khi chạy. Bài học: luôn kiểm tra `pwd`/thư mục hiện tại trước khi chạy lệnh phụ thuộc file cấu hình local.

**Khó khăn 2 — RAGAS trả về toàn 0.0000 ở lần chạy `naive_baseline.py` đầu tiên:**
- Nguyên nhân: chạy `cp .env.example .env` **sau khi** đã điền API key thật, vô tình ghi đè `.env` bằng placeholder `sk-...`. RAGAS gọi OpenAI thất bại, rơi vào nhánh `except` trả về toàn 0 — không crash nên dễ bị bỏ qua ban đầu, tưởng là lỗi logic code chứ không phải lỗi config.
- Cách giải quyết: kiểm tra lại nội dung `.env`, khôi phục key thật, chạy lại. Bài học: khi metric = 0 đồng loạt (không phải thấp mà là *đúng 0 tuyệt đối*), nghi ngờ đầu tiên nên là **exception bị nuốt** (`try/except` trả default), không phải logic sai.

**Khó khăn 3 — `pytest` not found:**
```
error: Failed to spawn: `pytest` — Caused by: program not found
```
- Nguyên nhân: `pytest` không nằm trong `requirements.txt` của scaffold, `uv pip install -r requirements.txt` không cài nó.
- Cách giải quyết: `uv pip install pytest` riêng. Bài học: `requirements.txt` production không nhất thiết gồm dev-dependencies (test tooling) — cần phân biệt 2 nhóm dependency này khi setup môi trường thật (thường tách `requirements-dev.txt`).

**Khó khăn 4 (phát hiện khi phân tích, không phải lỗi runtime) — Context Recall giảm dù pipeline "production" đáng lẽ phải tốt hơn:**
- Thời gian debug: ~15 phút đọc lại `src/pipeline.py::run_query()`.
- Nguyên nhân: `chunk_hierarchical()` trả về đúng `(parents, children)` theo thiết kế, nhưng `run_query()` dùng thẳng text của **child** đã rerank làm context cho LLM/RAGAS, không map ngược `child.parent_id` để lấy **parent** (đủ ngữ cảnh hơn). Đây là gap giữa lý thuyết "retrieve child (precision) → return parent (context)" học trên lớp và cách scaffold nối dây thực tế trong `pipeline.py` (phần "đã implement sẵn", không phải TODO của tôi).
- Kiến thức thiếu → bổ sung: hiểu rằng "implement đúng function" (M1 pass test) chưa đảm bảo "pipeline dùng đúng function đó theo đúng thiết kế" — cần đọc lại toàn bộ luồng dữ liệu end-to-end, không chỉ unit test từng module riêng lẻ.

## Phần 3: Action Plan cho project

```markdown
## Project: RAG nội bộ cho tài liệu chính sách công ty (dạng tương tự corpus Lab 18)

### Hiện tại
- RAG pipeline hiện tại: chưa có, đang ở giai đoạn khảo sát kiến trúc sau lab này.
- Known issues (rút ra từ lab): (1) chunking theo ký tự cố định dễ vỡ bảng/danh sách có cấu trúc,
  (2) retrieve-child-return-parent phải wiring đúng tới tận bước cuối (LLM context), không chỉ đúng ở M1,
  (3) đa phiên bản tài liệu (v1/v2) cần metadata version + lọc, nếu không LLM sẽ trộn lẫn.

### Plan áp dụng
1. [x] Chunking strategy: Hierarchical (parent 2048 / child 256) làm mặc định, nhưng dùng
       Structure-Aware cho các file có bảng/markdown table (approval matrix, bậc lương) —
       tránh lặp lại lỗi Case Study #3.
2. [x] Search: Hybrid BM25 + Dense (RRF) — vì corpus tiếng Việt có nhiều số liệu/tên riêng
       (BM25 mạnh) lẫn câu hỏi diễn giải tự nhiên (Dense mạnh).
3. [x] Reranking: Có — CrossEncoder, nhưng cần đánh giá bản nhẹ hơn (`FlashrankReranker`,
       <5ms) nếu latency 5s/query không chấp nhận được cho sản phẩm thời gian thực (đã có sẵn
       class trong `src/m3_rerank.py`, chỉ cần A/B latency vs precision).
4. [x] Evaluation: RAGAS 4 metrics làm chuẩn, nhưng bổ sung custom metric "numeric accuracy"
       riêng cho câu hỏi cần tính toán (2/5 bottom-failure của lab là lỗi tính toán, RAGAS
       chuẩn không đo riêng khả năng này).
5. [x] Enrichment: Combined single-call mode (`_enrich_single_call`) — hiệu quả chi phí đã
       chứng minh trong lab (Answer Relevancy +0.0922, chỉ 1 API call/chunk).

### Timeline
- Tuần 1: Sửa lỗi return-parent trong pipeline (bài học từ Case Study #3), thêm metadata
  version cho tài liệu đa phiên bản, viết lại prompt xử lý version conflict.
- Tuần 2: Thêm Structure-Aware chunking cho file có bảng, benchmark FlashrankReranker vs
  CrossEncoder (latency vs precision trade-off) để quyết định dùng cái nào cho production.
- Tuần 3: Xây custom eval set có nhãn riêng cho câu hỏi multi-hop/numeric, đo baseline trước
  khi thử few-shot prompt cho tính toán.
```
