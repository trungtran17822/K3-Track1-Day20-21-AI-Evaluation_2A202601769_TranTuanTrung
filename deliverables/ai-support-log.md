# AI Support Log

> Ghi lại bạn đã dùng AI (ChatGPT/Claude/Kimi...) ở những bước nào khi làm deliverables.
> Trung thực là một phần của bài nộp — không ai làm một mình, quan trọng là bạn giữ
> quyền kiểm soát chất lượng.

| # | Bước | AI dùng để làm gì | Bạn kiểm chứng kết quả thế nào |
|---|------|-------------------|-------------------------------|
| 1 | Input Grid v1 | Đề xuất 5 dimensions, 20 scenario family và phân nhóm representative/challenge/critical cho VLearn AI Tutor | Áp dụng phép thử “đổi value thì expected behavior có đổi không”, loại giọng văn khỏi dimension; đối chiếu với slide s26–s30 và ghi rõ các ô không ưu tiên |
| 2 | Dataset v1 | Viết 40 câu hỏi tự nhiên và metadata gồm dimension values, expected behavior, risk, set type, source type và slide context | Parse toàn bộ JSONL, kiểm tra ID duy nhất, dò near-duplicate, đối chiếu mọi slide ID/title với corpus và chạy 44 test offline |
| 3 | Handoff/report evidence | Tổng hợp phân bố scope, set, risk, language; lập candidate scenario bank và danh sách 10 câu ưu tiên | Số liệu được tính trực tiếp từ `dataset.jsonl`; `dataset-v1.jsonl` được so byte với file làm việc trước khi bàn giao |

- Phần nào AI gợi ý mà bạn **bác bỏ**? Vì sao?
- Bác bỏ việc dùng nguyên metadata của dataset mẫu: `sc-01` gắn title calibration vào
  `s51` và `sc-02` gắn trace codes vào `s29`, trong khi corpus cho thấy nội dung phù
  hợp lần lượt là `s53` và `s35`.
- Không nhận các câu AI sinh là “trace thật”; toàn bộ row v1 được ghi rõ
  `source_type=llm_generated_from_corpus` để tránh tạo provenance giả.

- Phần nào bạn **hoàn toàn tự làm**?
- Chưa khai báo phần nào là hoàn toàn tự làm trong bước tạo Dataset v1. Phần review
  chéo, quyết định chấp nhận dataset và nhãn human phải do các thành viên tự thực hiện,
  độc lập với nhãn của AI/judge.
