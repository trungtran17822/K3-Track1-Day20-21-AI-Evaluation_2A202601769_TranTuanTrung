# AI Support Log

| # | Bước | AI hỗ trợ | Tôi/nhóm kiểm chứng và quyết định |
|---|---|---|---|
| 1 | Input Grid và Dataset v1 | Gợi ý nhóm intent và câu challenge | Nhóm chốt 5 scenario, kiểm ID/slide context và chạy 44 test offline |
| 2 | Rubric và Routing Map | Gợi ý tách code check, LLM judge và human review | Đối chiếu output contract/corpus; giữ citation, groundedness và scope là blocker |
| 3 | Code checks | Đề xuất `scope_contract` và xử lý skip khi output lỗi | Chạy trên 5 output; xác nhận schema/citation/scope 100%, quote 60% |
| 4 | Human baseline | Hỗ trợ chạy script agreement | Ba thành viên tự chấm độc lập; AI không tạo nhãn người; nhóm chốt gold theo đa số 2/3 |
| 5 | Judge calibration | Phân tích confusion matrix và đề xuất sửa một điểm ở prompt | Lưu prompt/verdict cả hai vòng; agreement tăng 60% → 80%, không ép thành 100% |
| 6 | Scorecard/report | Tổng hợp token, latency, cost và evidence | Số liệu tính trực tiếp từ JSONL; verdict Hold vì quote blocker và dataset nhỏ |
| 7 | Troubleshooting | Chẩn đoán 401/429/503, JSON truncate và Braintrust config | Kiểm bằng HTTP status, test offline và SDK project ID; không commit API key |

## AI sai hoặc đề xuất chưa phù hợp

- Có lần xóa nhầm scratch `results.jsonl`; kết quả sau đó được chạy/ghép lại, xác minh
  đủ 5 ID, 0 API error và 0 parse error. Bài học: luôn snapshot evidence trước thao tác xóa.
- Judge v1 pass cả 5 câu và bỏ qua chất lượng follow-up out-of-scope. Nhóm không dùng
  verdict đó làm chuẩn; prompt v2 chỉ sửa đúng failure mode đã quan sát.
- Không coi agreement 100% trên 1 case chung là hợp lệ; chỉ báo cáo sau khi cả ba file
  có đủ 5 nhãn.

## Phần con người giữ quyền quyết định

- Ba thành viên tự gán nhãn, thảo luận `sc-04`/`sc-05` và chốt nhãn vàng.
- Nhóm đặt release gate và quyết định **Hold**; AI chỉ hỗ trợ tính số liệu và kiểm tra
  tính nhất quán giữa REPORT với evidence.
