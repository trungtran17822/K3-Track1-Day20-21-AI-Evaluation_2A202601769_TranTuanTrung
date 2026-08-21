# evidence/ — data thô của từng bước eval loop

Thư mục này chứa **data thô** minh chứng cho mọi quyết định trong các file
`deliverables/REPORT.md`. File làm việc sinh ra ở **root repo**
(`dataset.jsonl`, `results.jsonl`, `verdicts.jsonl`, `labels.csv`) — chốt một vòng
là copy vào đây ngay, đặt tên theo version, KHÔNG ghi đè vòng cũ.

Cần có đủ:

| File | Lấy từ đâu | Là gì |
|---|---|---|
| `dataset-v1.jsonl` | `dataset.jsonl` (root) | Dataset nhóm chốt — đầu vào mọi lần chạy |
| `results-v1.jsonl` (v2, v3...) | `results.jsonl` (root) | Output tutor thật: input, output JSON, `tool_calls`, tokens, cost từng câu |
| `labels.csv` | Export từ `report.html` | Nhãn người của các thành viên (vòng chấm độc lập) |
| `judge-prompt-v1.md` (v2...) | `eval/judge_prompt.md` | Prompt judge TỪNG VÒNG — copy trước mỗi lần sửa |
| `verdicts-v1.jsonl` (v2...) | `verdicts.jsonl` (root) | Output judge từng vòng calibration |
| `braintrust-link.md` | tự tạo | Link project Braintrust/LangSmith — trace mọi run |

Artefact canonical của bài nộp này dùng Dataset v1 **5 scenario**:

- `dataset-v1.jsonl`, `results-v1.jsonl`, `labels.csv`.
- `labels_TranTuanTrung.csv`, `labels-NguyenTrongDuc.csv`, `labels_Khanh.csv` là ba
  vòng chấm độc lập.
- `agreement-v1.txt` là agreement người–người.
- `judge-prompt-v1/v2.md`, `verdicts-v1/v2.jsonl`, `calibration-v1/v2.txt` là hai
  vòng calibration judge.
- Các file hậu tố `-40-legacy` chỉ giữ lịch sử, không dùng trong REPORT hiện tại.

Số liệu trong mục 5 (Calibration Report) của `deliverables/REPORT.md` phải đối chiếu được với các
file ở đây (confusion matrix, % agreement in ra từ `eval/judge.py`).

Nhớ: chạy xong một vòng là copy ngay — cuối buổi mới gom là mất dấu các vòng trước.
