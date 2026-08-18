# Group Report — Lab 18: Production RAG

**Nhóm:** Cá nhân — Phạm Công Đăng - 2A202601280 (bài tập cá nhân, tự làm toàn bộ 5 module)
**Ngày:** 2026-08-18

## Thành viên & Phân công

| Tên               | Module            | Hoàn thành | Tests pass |
| ------------------ | ----------------- | ------------ | ---------- |
| Phạm Công Đăng | M1: Chunking      | ☑           | 13/13      |
| Phạm Công Đăng | M2: Hybrid Search | ☑           | 5/5        |
| Phạm Công Đăng | M3: Reranking     | ☑           | 5/5        |
| Phạm Công Đăng | M4: Evaluation    | ☑           | 4/4        |
| Phạm Công Đăng | M5: Enrichment    | ☑           | 10/10      |

**Tổng:** 37/37 test pass · 0 TODO còn lại · Pipeline chạy end-to-end thành công (`python main.py`, 875.3s)

## Kết quả RAGAS

| Metric            | Naive  | Production | Δ      |
| ----------------- | ------ | ---------- | ------- |
| Faithfulness      | 0.7875 | 0.8000     | +0.0125 |
| Answer Relevancy  | 0.6771 | 0.7692     | +0.0922 |
| Context Precision | 0.9250 | 0.9292     | +0.0042 |
| Context Recall    | 0.9250 | 0.7917     | -0.1333 |

## Key Findings

1. **Biggest improvement:** Answer Relevancy (+0.0922) — nhờ M5 Enrichment (combined mode, 1 API call/chunk) sinh hypothesis questions + context giúp thu hẹp khoảng cách từ vựng giữa câu hỏi người dùng và văn bản chính sách gốc.
2. **Biggest challenge:** Latency của M3 Reranking — avg 5316ms/query (max 15034ms lần đầu, gồm cả model load), chiếm phần lớn tổng thời gian mỗi query so với search (98ms) và LLM answer (2589ms). Nguyên nhân: `bge-reranker-v2-m3` (~568M tham số) chạy CPU-only.
3. **Surprise finding:** Context Recall giảm (0.9250 → 0.7917) dù pipeline "production" phức tạp và tốn nhiều tài nguyên hơn hẳn baseline. Root cause: `run_query()` dùng thẳng text của **child chunk** (256 ký tự, sau rerank) làm context, thay vì map ngược `child.parent_id` để lấy **parent** (2048 ký tự, đủ ngữ cảnh hơn) — đúng thiết kế "retrieve child (precision) → return parent (context)" đã học trên lớp nhưng chưa được nối dây đủ trong `pipeline.py`. Chi tiết ở `analysis/failure_analysis.md` — Case Study #3.

## Presentation Notes (5 phút)

1. **RAGAS scores (naive vs production):** 3/4 metric tăng (Faithfulness, Answer Relevancy, Context Precision), 1/4 metric giảm rõ rệt (Context Recall -0.1333) — minh chứng "production pipeline" không mặc định tốt hơn baseline trên mọi mặt, phải đọc từng metric riêng.
2. **Biggest win — module nào, tại sao:** M5 Enrichment — Answer Relevancy tăng mạnh nhất, chi phí chỉ 1 API call/chunk (combined mode) thay vì 4 call riêng lẻ, đúng khuyến nghị bonus của đề bài.
3. **Case study — 1 failure, Error Tree walkthrough:** Câu hỏi "Muốn mua thiết bị trị giá 55 triệu cần ai phê duyệt?" (context_recall = 0.0) — Output sai → Context đúng? Không (dòng ngưỡng ">50 triệu → CEO" trong approval matrix bị hierarchical chunking (256 ký tự) cắt rời khỏi phần diễn giải) → Query OK? Có → Fix ở bước M1 (chunking) + M3 (rerank top-k) + pipeline wiring (return parent thay vì child).
4. **Next optimization nếu có thêm 1 giờ:** Sửa `run_query()` để lookup `parent_id` sau khi rerank, trả về text của parent (đủ ngữ cảnh) thay vì child; áp dụng `chunk_structure_aware()` riêng cho các file có bảng (approval matrix, bậc lương) để không cắt giữa dòng bảng.
