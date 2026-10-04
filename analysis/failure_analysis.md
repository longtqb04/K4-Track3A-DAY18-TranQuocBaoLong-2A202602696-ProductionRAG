# Failure Analysis — Lab 18: Production RAG

**Họ và tên học viên:** Tran Quoc Bao Long
**Khóa:** K4 - Track 3A

---

## RAGAS Scores

| Metric | Naive Baseline | Production | Δ |
|--------|---------------|------------|---|
| Faithfulness | 0.8500 | 0.6750 | -0.1750 |
| Answer Relevancy | 0.6675 | 0.5963 | -0.0711 |
| Context Precision | 0.9250 | 0.9667 | +0.0417 |
| Context Recall | 0.9250 | 0.8667 | -0.0583 |

Production đạt context precision cao nhất (0.9667), cho thấy các context được lấy nhìn chung liên quan; tuy nhiên context recall giảm 0.0583, nên một số thông tin cần thiết có thể bị bỏ sót. Faithfulness giảm 0.1750 và answer relevancy giảm 0.0711, nghĩa là câu trả lời production kém bám sát context và kém đúng trọng tâm hơn baseline theo hai metric này. Các điểm tổng thể không đủ để kết luận nguyên nhân của từng câu; cần xem câu trả lời và context cụ thể.

## Bottom-5 Failures

Điểm trong danh sách failure là điểm trung bình của bốn metric cho mỗi câu (trường `score` trong báo cáo); `worst_metric` cho biết metric thấp nhất. `ragas_report.json` không lưu câu trả lời thực tế hoặc context theo từng câu, nên mục **Got** không thể trích xuất từ report và được ghi rõ thay vì suy đoán. Expected answers được đối chiếu từ `test_set.json`.

### #1 — Điểm tổng 0.3750
- **Question:** Nhân viên tạm ứng 15 triệu, sau 20 ngày mới thanh toán. Bị phạt bao nhiêu?
- **Expected:** Thời hạn thanh toán là 15 ngày. Quá hạn 5 ngày, bị tính phí 2%/tháng trên 15.000.000 VNĐ = 300.000 VNĐ/tháng (tính pro-rata khoảng 50.000 VNĐ cho 5 ngày).
- **Got:** Nhân viên thanh toán trễ 5 ngày sẽ bị phạt tiền chậm nộp theo quy định chung là 0.05%/ngày trên tổng số tiền tạm ứng. Với số tiền 15 triệu đồng, phạt chậm nộp 5 ngày là 15.000.000 × 0.05% × 5 = 37.500 VNĐ. Ngoài ra, nhân viên sẽ bị trừ 1 điểm thi đua của tháng.
- **Worst metric:** Faithfulness.
- **Error Tree:** Faithfulness thấp → câu trả lời có thể chứa thông tin không được hỗ trợ bởi context. Context đúng và đủ cho câu này? Chưa xác minh được nếu thiếu context theo câu.
- **Root cause:** RAGAS chẩn đoán “LLM hallucinating”. Có thể mô hình suy diễn số tiền/phí khi context không cung cấp đủ quy tắc tính; cần đối chiếu retrieval trước khi kết luận.
- **Suggested fix:** Ràng buộc prompt chỉ trả lời theo context, yêu cầu nói không tìm thấy khi thiếu công thức hoặc dữ kiện; truy vết các chunk tài chính đã retrieve và bổ sung quy tắc tính phí nếu bị thiếu.

### #2 — Điểm tổng 0.3750
- **Question:** Một nhân viên Senior có 9 năm thâm niên được nghỉ bao nhiêu ngày phép năm và lương trong khoảng nào?
- **Expected:** Theo chính sách v2024: 15 ngày cơ bản + 3 ngày thâm niên (9÷3=3) = 18 ngày phép. Lương Senior (P3-P4): 20-35 triệu VNĐ/tháng.
- **Got:** Nhân viên Senior có 9 năm thâm niên sẽ được hưởng 12 ngày phép năm cơ bản cộng thêm 9 ngày phép thâm niên (mỗi năm thâm niên cộng 1 ngày), tổng cộng là 21 ngày phép/năm. Mức lương của cấp Senior dao động từ 15.000.000 VNĐ đến 25.000.000 VNĐ/tháng.
- **Worst metric:** Faithfulness.
- **Error Tree:** Faithfulness thấp → câu trả lời có thể trộn chính sách cũ với chính sách hiện hành hoặc suy diễn phép tính. Cần kiểm tra các chunk về phép năm v2024 và dải lương Senior.
- **Root cause:** Báo cáo gắn chẩn đoán “LLM hallucinating”. Câu hỏi kết hợp hai dữ kiện và phép tính; retrieval có thể tìm thiếu một trong hai chính sách, hoặc generation không phân biệt phiên bản.
- **Suggested fix:** Ưu tiên metadata/phiên bản hiện hành khi retrieve, trả lời riêng từng phần kèm căn cứ, và chỉ tính 15 + (9/3) khi context có cả quy tắc phép năm lẫn thâm niên.

### #3 — Điểm tổng 0.5000
- **Question:** Bao lâu phải đổi mật khẩu một lần?
- **Expected:** Theo chính sách hiện hành (v2.0), mật khẩu phải được thay đổi mỗi 120 ngày. Chính sách cũ yêu cầu 90 ngày nhưng đã bị thay thế.
- **Got:** Theo quy định An toàn thông tin của công ty, tất cả nhân viên phải thực hiện thay đổi mật khẩu tài khoản định kỳ 90 ngày một lần để đảm bảo tính bảo mật.
- **Worst metric:** Faithfulness.
- **Error Tree:** Faithfulness thấp → có thể câu trả lời nhắc quy định 90 ngày từ chính sách cũ. Cần kiểm tra context có cả v1.0 và v2.0 hay không.
- **Root cause:** Báo cáo chẩn đoán “LLM hallucinating”; tuy nhiên dữ liệu có phiên bản cũ/mới nên nguyên nhân cũng có thể là retrieval ưu tiên nhầm tài liệu cũ.
- **Suggested fix:** Đưa version/effective-date vào metadata và ưu tiên chính sách hiện hành; yêu cầu câu trả lời nêu phiên bản/ngày hiệu lực khi có nhiều quy định.

### #4 — Điểm tổng 0.5000
- **Question:** Lương thử việc của nhân viên Junior mức cao nhất là bao nhiêu?
- **Expected:** Junior cao nhất là 20.000.000 VNĐ/tháng. Lương thử việc = 85% × 20.000.000 = 17.000.000 VNĐ/tháng.
- **Got:** Mức lương Junior cao nhất là 20.000.000 VNĐ/tháng. Trong thời gian thử việc, nhân viên nhận 80% lương chính thức, do đó lương thử việc cao nhất của Junior là 16.000.000 VNĐ/tháng.
- **Worst metric:** Faithfulness.
- **Error Tree:** Faithfulness thấp → cần xác định câu trả lời sai ở mức lương tối đa, tỷ lệ thử việc 85%, hay phép tính. Context được retrieve cho câu này chưa có trong report.
- **Root cause:** Báo cáo chẩn đoán “LLM hallucinating”. Câu hỏi đòi hỏi ghép bảng lương với quy tắc thử việc và tính toán; thiếu một trong hai chunk có thể khiến mô hình tự điền dữ liệu.
- **Suggested fix:** Đảm bảo retrieval lấy được cả dải lương Junior và tỷ lệ thử việc; yêu cầu câu trả lời trình bày phép tính 0.85 × 20.000.000.

### #5 — Điểm tổng 0.5833
- **Question:** Nhân viên thử việc có được nghỉ phép năm không?
- **Expected:** KHÔNG. Nhân viên thử việc KHÔNG được nghỉ phép năm. Nếu cần nghỉ, phải xin nghỉ không lương và được trưởng phòng phê duyệt.
- **Got:** Chính sách nghỉ phép của công ty quy định nhân viên chính thức được hưởng 12 ngày phép/năm. Đối với trường hợp đang trong thời gian thử việc, nếu nhân viên có việc phát sinh cần nghỉ thì phải làm đơn xin nghỉ không lương trên hệ thống HR và nhận được sự phê duyệt của Trưởng phòng trực tiếp.
- **Worst metric:** Answer relevancy.
- **Error Tree:** Answer relevancy thấp → câu trả lời có thể dài dòng hoặc không trả lời trực tiếp “có/không”. Context precision/recall riêng câu này không có nên chưa kết luận retrieval đúng hay sai.
- **Root cause:** Báo cáo chẩn đoán “Answer doesn't match question”. Nhiều khả năng prompt/format câu trả lời chưa ưu tiên kết luận trực tiếp.
- **Suggested fix:** Bắt đầu bằng “Không”, sau đó nêu trường hợp nghỉ không lương và người phê duyệt; giữ câu trả lời gọn, bám sát chính sách.

## Case Study (cho presentation)

**Question chọn phân tích:** Nhân viên tạm ứng 15 triệu, sau 20 ngày mới thanh toán. Bị phạt bao nhiêu?

**Error Tree walkthrough:**
1. Output đúng? → Chưa thể đọc output thực tế từ báo cáo. Điểm tổng 0.3750 và faithfulness là metric thấp nhất.
2. Context đúng? → Chưa xác minh được vì report chỉ giữ aggregate và danh sách chẩn đoán, không lưu context theo câu. Ground truth cần thời hạn 15 ngày, lãi suất 2%/tháng và phép tính pro-rata.
3. Query rewrite OK? → Chưa có log query rewrite; truy vấn nêu rõ số tiền và số ngày, nên kiểm tra xem retrieval có đưa chunk về quy định tạm ứng/quyết toán vào top results.
4. Fix ở bước: Trước tiên kiểm tra retrieved chunks. Nếu thiếu quy tắc thì sửa retrieval/chunking; nếu context đủ nhưng answer suy diễn thì giới hạn generation theo context và yêu cầu abstain khi thiếu căn cứ.

**Nếu có thêm 1 giờ, sẽ optimize:**
- Lưu `per_question` gồm question, answer, contexts, ground truth và metric scores vào report để failure analysis có thể xác định lỗi retrieval hay generation thay vì chỉ dựa vào nhãn chẩn đoán.
- Kiểm tra top retrieved chunks cho 5 câu dưới điểm thấp nhất; tập trung phiên bản chính sách hiện hành, các câu hỏi ghép nhiều dữ kiện và quy tắc tính toán.
- Chạy lại evaluator sau khi chỉnh prompt/retrieval và so sánh bốn metric trên cùng 20 câu hỏi.
