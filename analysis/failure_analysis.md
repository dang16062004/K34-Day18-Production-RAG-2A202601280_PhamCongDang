# Failure Analysis — Lab 18: Production RAG

**Thực hiện:** Phạm Công Đăng (cá nhân — tự làm cả 5 module M1–M5)
**Ngày chạy:** 2026-08-18 · `python main.py` · Total: 875.3s

---

## RAGAS Scores

| Metric | Naive Baseline | Production | Δ |
|--------|---------------|------------|---|
| Faithfulness | 0.7875 | 0.8000 | +0.0125 |
| Answer Relevancy | 0.6771 | 0.7692 | +0.0922 |
| Context Precision | 0.9250 | 0.9292 | +0.0042 |
| Context Recall | 0.9250 | 0.7917 | **-0.1333** |

**Nhận xét chung:** 3/4 metric cải thiện đúng như kỳ vọng lý thuyết — hybrid search + rerank giảm noise nên Precision nhích lên, enrichment giúp Answer Relevancy tăng mạnh nhất (+0.0922, do câu hỏi/summary sinh thêm giúp bridge vocabulary gap). Nhưng **Context Recall giảm 0.1333** — đây là điểm bất ngờ, phân tích ở phần Case Study.

## Bottom-5 Failures

### #1
- **Question:** Bao lâu phải đổi mật khẩu một lần?
- **Expected:** Theo chính sách hiện hành (v2.0), mật khẩu phải được thay đổi mỗi 120 ngày. Chính sách cũ yêu cầu 90 ngày nhưng đã bị thay thế.
- **Got:** Không lưu trong `ragas_report.json` ở lần chạy này (bug trong scaffold: `failure_analysis()` gốc không giữ `EvalResult.answer`). Đã sửa `src/m4_eval.py` để lưu `answer`/`contexts` cho lần chạy sau.
- **Worst metric:** faithfulness = 0.0000 (avg 0.3958)
- **Error Tree:** Output sai → Context đúng? Có khả năng đúng (corpus có cả `mat_khau_v1.md` 90 ngày và `mat_khau_v2.md` 120 ngày) → Query OK? Có → **Root cause: retrieval trả về cả 2 version, LLM không phân biệt được version nào "hiện hành" → trả lời lẫn lộn/hallucinate giữa 90 và 120 ngày.**
- **Suggested fix:** Thêm metadata `version`/`effective_date` khi chunk, ưu tiên version mới nhất trong prompt hoặc lọc version cũ ra khỏi context; tighten system prompt: "Nếu có nhiều phiên bản, chỉ dùng bản mới nhất."

### #2
- **Question:** Một nhân viên Senior có 9 năm thâm niên được nghỉ bao nhiêu ngày phép năm và lương trong khoảng nào?
- **Expected:** Theo chính sách v2024: 15 ngày cơ bản + 3 ngày thâm niên (9÷3=3) = 18 ngày phép. Lương Senior (P3-P4): 20-35 triệu VNĐ/tháng.
- **Got:** Chưa lưu (như #1).
- **Worst metric:** answer_relevancy = 0.0000 (avg 0.5417)
- **Error Tree:** Output sai → Context đúng? Cần 2 nguồn khác nhau (`nghi_phep_nam_v2024.md` + `bang_luong_2024.md`) → Query OK? Câu hỏi multi-hop (2 câu hỏi con) → **Root cause: retrieval/rerank top-3 chỉ ưu tiên 1 trong 2 chủ đề (nghỉ phép HOẶC lương), không đủ context cho cả 2 → answer thiếu 1 vế nên bị chấm relevancy thấp.**
- **Suggested fix:** Query decomposition (tách câu hỏi multi-hop thành 2 sub-query trước khi retrieve) hoặc tăng `RERANK_TOP_K` khi phát hiện câu hỏi multi-hop.

### #3
- **Question:** Muốn mua thiết bị trị giá 55 triệu cần ai phê duyệt?
- **Expected:** Đơn hàng trên 50.000.000 VNĐ cần Tổng Giám đốc (CEO) phê duyệt.
- **Got:** Chưa lưu (như #1).
- **Worst metric:** context_recall = 0.0000 (avg 0.5976)
- **Error Tree:** Output sai → Context đúng? **Không** — dòng ngưỡng ">50 triệu → CEO" trong `mua_sam.md` (approval matrix dạng bảng) không nằm trong top-3 sau rerank → Query OK? Có, query rõ ràng → **Root cause: chunk_hierarchical cắt theo 256 ký tự (child_size) làm vỡ bảng approval matrix thành nhiều chunk nhỏ, mỗi dòng ngưỡng tách rời khỏi ngữ cảnh "cần ai phê duyệt"; rerank_top_k=3 quá hẹp để đảm bảo dòng đúng lọt vào.**
- **Suggested fix:** Đây chính là case đáng phân tích sâu nhất — xem Case Study bên dưới.

### #4
- **Question:** Nhân viên tạm ứng 15 triệu, sau 20 ngày mới thanh toán. Bị phạt bao nhiêu?
- **Expected:** Thời hạn thanh toán là 15 ngày. Quá hạn 5 ngày, bị tính phí 2%/tháng trên 15.000.000 VNĐ = 300.000 VNĐ/tháng (tính pro-rata khoảng 50.000 VNĐ cho 5 ngày).
- **Got:** Chưa lưu (như #1).
- **Worst metric:** faithfulness = 0.1667 (avg 0.6169)
- **Error Tree:** Output sai → Context đúng? Có thể đúng (có trong `tam_ung.md`) → Query OK? Có, nhưng cần suy luận số học (20-15=5 ngày trễ → tính phí pro-rata) → **Root cause: LLM giỏi trích xuất nhưng yếu tính toán nhiều bước; context đúng nhưng answer generation tự suy diễn công thức sai → faithfulness thấp vì câu trả lời không bám sát số liệu gốc.**
- **Suggested fix:** Thêm ví dụ tính toán mẫu (few-shot) vào system prompt, hoặc tách bước "trích số liệu" và "tính toán" thành 2 lần gọi LLM riêng.

### #5
- **Question:** Lương thử việc của nhân viên Junior mức cao nhất là bao nhiêu?
- **Expected:** Junior cao nhất là 20.000.000 VNĐ/tháng. Lương thử việc = 85% x 20.000.000 = 17.000.000 VNĐ/tháng.
- **Got:** Chưa lưu (như #1).
- **Worst metric:** faithfulness = 0.0000 (avg 0.706)
- **Error Tree:** Output sai → Context đúng? Có (bảng lương + tỷ lệ thử việc 85% đều trong `bang_luong_2024.md`/`thu_viec.md`) → Query OK? Có, nhưng cần nhân 2 số từ 2 chunk khác nhau → **Root cause: giống #4 — multi-hop numeric reasoning (20tr × 85% = 17tr) vượt khả năng "chỉ trích xuất" của prompt hiện tại.**
- **Suggested fix:** Giống #4 — few-shot ví dụ tính toán, hoặc chain-of-thought có kiểm soát trong system prompt.

## Case Study (cho presentation)

**Question chọn phân tích:** #3 — "Muốn mua thiết bị trị giá 55 triệu cần ai phê duyệt?" (context_recall = 0.0)

**Error Tree walkthrough:**
1. Output đúng? → Không — model không nêu được "CEO phê duyệt" (thông tin quan trọng nhất).
2. Context đúng? → Không — dòng approval-matrix chứa ngưỡng ">50 triệu → CEO" không có mặt trong 3 context được đưa vào prompt.
3. Query rewrite OK? → Có — BM25 + Dense đều tìm ra `mua_sam.md` (search chỉ mất ~90-100ms, đúng file), nhưng **sau khi hierarchical chunking cắt file thành nhiều child 256 ký tự, dòng ngưỡng phê duyệt bị tách khỏi phần diễn giải**, và **rerank top-3 (RERANK_TOP_K=3) không đủ rộng** để đảm bảo đúng child chứa ngưỡng lọt vào.
4. Fix ở bước: **M1 (chunking) + M3 (rerank top-k)**. Nguyên nhân gốc: `chunk_hierarchical()` được thiết kế đúng chuẩn parent-child (retrieve bằng child nhỏ, chính xác — trả về parent lớn, đủ ngữ cảnh), nhưng `src/pipeline.py::run_query()` hiện **dùng thẳng text của child đã rerank làm context cho LLM, không map ngược về parent** (`child.parent_id` bị bỏ qua ở bước cuối). Đây là gap giữa thiết kế lý thuyết (đã học trong lecture) và cách pipeline nối dây thực tế.

**Nếu có thêm 1 giờ, sẽ optimize:**
- Sửa `run_query()` để sau khi rerank chọn được child, lookup `parent_id` → dùng text của **parent** (2048 ký tự, đủ nguyên vẹn bảng approval matrix) làm context đưa cho LLM, thay vì dùng thẳng child 256 ký tự.
- Tăng `RERANK_TOP_K` từ 3 → 5 cho riêng nhóm câu hỏi có từ khóa "phê duyệt / ngưỡng / bao nhiêu triệu" (numeric/threshold pattern).
- Áp dụng `chunk_structure_aware()` cho các file có bảng (markdown table) thay vì `chunk_hierarchical()` theo ký tự, để không cắt giữa dòng bảng.
