# Failure Analysis — Lab 18

**Người thực hiện:** Nguyễn Cao Quang Anh (cá nhân, 5 module)

## RAGAS Scores

| Metric | Naive Baseline | Production | Δ |
|--------|---------------|------------|---|
| Faithfulness | 0.7917 | 0.8250 | +0.0333 |
| Answer Relevancy | 0.6669 | 0.7692 | +0.1023 |
| Context Precision | 0.9417 | 0.9250 | -0.0167 |
| Context Recall | 0.9250 | 0.8167 | -0.1083 |

Production tốt hơn rõ ở faithfulness và answer relevancy — nhờ rerank + LLM answer generation với prompt ràng buộc "chỉ dựa context". Nhưng context precision và đặc biệt context recall giảm. Nguyên nhân: naive dùng toàn bộ 57 chunk theo paragraph, đưa cả block lớn vào context nên recall dễ đạt cao; production dùng hierarchical child (256 ký tự) + rerank top-3, chunk nhỏ hơn nên khi câu hỏi cần multi-hop (nhiều điều kiện, nhiều nguồn) dễ thiếu chunk liên quan.

## Bottom-5 Failures

### #1
- **Question:** Nhân viên thử việc có được nghỉ phép năm không?
- **Worst metric:** faithfulness (0.0)
- **Error Tree:** Output sai → Context có nhưng answer không bám sát → Query OK → Root cause: LLM trả lời vượt quá thông tin trong context (hallucination), hoặc context chứa cả 2 phiên bản policy (thử việc + chính thức) gây nhiễu.
- **Suggested fix:** Tighten prompt, giảm temperature, thêm rule "nếu context có nhiều phiên bản mâu thuẫn thì nêu rõ".

### #2
- **Question:** Muốn mua thiết bị trị giá 55 triệu cần ai phê duyệt?
- **Worst metric:** context_recall (0.0)
- **Error Tree:** Output sai → Context thiếu → Query OK → Root cause: chunk quy định mua sắm bị cắt nhỏ (child 256 ký tự), bảng phân cấp phê duyệt theo mức tiền không nằm trọn trong 1 chunk được retrieve.
- **Suggested fix:** Tăng child_size cho các bảng/quy định phân cấp, hoặc dùng structure-aware chunking cho file mua_sam.md thay vì hierarchical.

### #3
- **Question:** Một nhân viên Senior có 9 năm thâm niên được nghỉ bao nhiêu ngày phép năm và lương trong khoảng nào?
- **Worst metric:** answer_relevancy (0.0)
- **Error Tree:** Output sai → Context đúng (có cả 2 thông tin) → Query là multi-hop 2 phần → Root cause: câu hỏi ghép 2 domain khác nhau (nghỉ phép + lương), retrieval/rerank ưu tiên 1 phía, answer chỉ trả lời được nửa câu hỏi.
- **Suggested fix:** Query decomposition — tách câu hỏi multi-hop thành 2 sub-query trước khi retrieve.

### #4
- **Question:** Nhân viên tạm ứng 15 triệu, sau 20 ngày mới thanh toán. Bị phạt bao nhiêu?
- **Worst metric:** faithfulness (0.25)
- **Error Tree:** Output sai → Context đúng → Query OK → Root cause: câu hỏi cần tính toán số học dựa trên công thức trong context (numeric reasoning), LLM suy diễn sai công thức hoặc áp dụng nhầm mốc thời gian.
- **Suggested fix:** Prompt thêm chain-of-thought cho câu hỏi numeric, hoặc validate số bằng post-processing.

### #5
- **Question:** Lương thử việc của nhân viên Junior mức cao nhất là bao nhiêu?
- **Worst metric:** faithfulness (0.0)
- **Error Tree:** Output sai → Context có bảng lương → Query OK → Root cause: bảng lương bị chunking cắt mất cột/dòng liên quan (256 ký tự quá nhỏ cho bảng markdown), LLM đoán số thay vì đọc bảng.
- **Suggested fix:** Structure-aware chunking giữ nguyên table thay vì hierarchical cắt theo độ dài ký tự.

## Case Study (presentation)

**Question:** Muốn mua thiết bị trị giá 55 triệu cần ai phê duyệt?

**Error Tree walkthrough:**
1. Output đúng? → Không, model trả lời "Không tìm thấy" hoặc phê duyệt sai mức.
2. Context đúng? → Không, chunk retrieve về không chứa đúng ngưỡng phê duyệt 55 triệu.
3. Query rewrite OK? → Không áp dụng (không có rewrite step trong pipeline hiện tại).
4. Fix ở bước: chunking (M1) — child chunk 256 ký tự cắt đứt bảng phân cấp phê duyệt theo mức tiền trong mua_sam.md.

**Nếu có thêm 1 giờ:**
- Đổi mua_sam.md sang structure-aware chunking, giữ nguyên bảng.
- Thêm query rewrite/decomposition cho câu hỏi multi-hop.
- Test lại context_recall trên các câu numeric/multi-hop để xác nhận cải thiện.
