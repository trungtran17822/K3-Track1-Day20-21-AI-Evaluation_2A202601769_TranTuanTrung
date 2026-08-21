# AI Support Log — Mẫu giả định cá nhân

> **Lưu ý:** Đây là log giả định được tạo để phản ánh đúng vai trò của **Trần Tuấn Trung (2A202601769)** trong bài AI Evaluation. Hãy điều chỉnh ngày, lệnh đã chạy và kết quả nếu khác với quá trình thực tế trước khi nộp.

| Mốc | Công việc tôi phụ trách | AI hỗ trợ | Việc tôi kiểm chứng và quyết định |
|---|---|---|---|
| 1 | Hoàn thiện rubric và routing map | Gợi ý tách tiêu chí thành code checks, LLM judge và human review; đề xuất các failure mode cần chặn. | Tôi đối chiếu output contract và corpus, rồi chốt citation, groundedness và scope là các release blocker. |
| 2 | Viết và rà soát code checks | Gợi ý cấu trúc kiểm tra `schema_valid`, `citation_exists`, `quote_verbatim` và `scope_contract`. | Tôi đọc logic kiểm tra, chạy trên bộ kết quả v1 và chỉ giữ các rule phù hợp với rubric đã chốt. |
| 3 | Chấm human baseline | Hỗ trợ tạo checklist chấm nhất quán và giải thích cách đọc các trường trong output. | Tôi tự đọc 5 output, gán nhãn độc lập và ghi note cho các case out-of-scope hoặc yêu cầu đáp án trực tiếp. AI không tạo nhãn thay tôi. |
| 4 | Calibrate LLM judge, vòng 1 | Hỗ trợ so sánh verdict judge với nhãn người và chỉ ra các case bất đồng. | Tôi kiểm tra từng bất đồng, xác định judge v1 chưa phạt đầy đủ follow-up out-of-scope và lưu prompt/verdict v1 làm evidence. |
| 5 | Calibrate LLM judge, vòng 2 | Đề xuất sửa một điểm cụ thể trong prompt: đánh giá follow-up theo scope của câu hỏi. | Tôi tự sửa prompt, chạy lại judge, đối chiếu với human baseline và chỉ chấp nhận thay đổi khi agreement cải thiện mà không làm nới rubric. |
| 6 | Tổng hợp scorecard và report | Hỗ trợ tính pass rate, latency, token, cost và nhóm kết quả theo tiêu chí. | Tôi lấy số trực tiếp từ các file JSONL/evidence, rà lại các con số trong REPORT và ghi rõ giới hạn của dataset nhỏ. |
| 7 | Đề xuất release verdict | Hỗ trợ diễn đạt trade-off và tóm tắt các blocker còn lại. | Tôi quyết định **Hold / chưa ship** vì quote verbatim chưa đạt ngưỡng mong muốn; không dùng AI để tự phê duyệt release. |

## Ví dụ một đề xuất AI không được tôi áp dụng

- AI từng gợi ý coi tất cả case có citation hợp lệ là pass. Tôi không áp dụng vì citation tồn tại không chứng minh được quote là nguyên văn hoặc câu trả lời bám đúng nguồn.
- AI từng gợi ý tăng agreement bằng cách rút gọn rubric. Tôi giữ rubric đầy đủ vì mục tiêu là phát hiện failure mode, không phải tối đa hoá một chỉ số.

## Phần việc tôi giữ quyền quyết định

- Chọn và chốt rubric, routing map, ngưỡng release và verdict cuối.
- Tự gán nhãn human baseline, xem các case bất đồng và quyết định nội dung sửa prompt.
- Xác nhận số liệu trong report khớp evidence trước khi đưa vào bài nộp.
