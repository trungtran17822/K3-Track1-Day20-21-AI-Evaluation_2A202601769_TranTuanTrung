# Input Grid v1 — VLearn AI Tutor

Tài liệu bàn giao của Người 1 cho mục 1–2 trong `deliverables/REPORT.md`. Dataset đi
kèm là `dataset-v1.jsonl`. Các câu hỏi được AI phác thảo từ corpus khóa học và đã
được kiểm tra tự động về schema, trùng ID, trùng câu và slide reference; nhóm vẫn
cần review chéo nội dung trước lần chạy chính thức.

## 1. Quyết định coverage

AI Tutor phục vụ chủ yếu học viên mới, học viên giữa khóa, người ôn tập và PM muốn
áp dụng kiến thức eval. Dataset ưu tiên các tình huống mà thay đổi input làm hành vi
đúng của tutor thay đổi: trả lời trực tiếp, tổng hợp nhiều nguồn, hỏi làm rõ, từ chối
ngoài phạm vi, chống bịa đặt và chống prompt injection.

Giọng văn lịch sự/cộc lốc không được chọn làm dimension riêng vì thường không làm
thay đổi hành vi đúng. `set_type` cũng không phải input dimension; đây là nhãn quản
trị để tách representative, challenge và critical regression khi đọc kết quả.

## 2. Các dimension và values

| Dimension | Values trong v1 | Vì sao ảnh hưởng hành vi đúng |
|---|---|---|
| Persona | `hoc_vien_moi`, `hoc_vien_giua_khoa`, `nguoi_on_thi`, `pm_khac_team`, `nguoi_ngoai_khoa` | Độ sâu giải thích, nhu cầu vận dụng và giới hạn phục vụ khác nhau |
| Intent | Hỏi/so sánh khái niệm, tra cứu, thiết kế, đọc kết quả, multi-intent, xin dữ liệu live, prompt injection | Tutor phải đổi giữa giải thích, tổng hợp, hỏi lại và từ chối |
| Coverage | `direct`, `multi_source`, `unknown`, `out_of_corpus`, `zero_hit`, `out_of_scope` | Quyết định trả lời trực tiếp, tổng hợp, làm rõ hay không được bịa |
| Context richness | Có/không slide, mơ hồ có slide, thiếu context, nhiều ý, false premise, malicious | Quyết định có thể trả lời ngay hay phải hỏi lại/bác tiền đề/từ chối |
| Language | `vi`, `en` | Kiểm tra tutor giữ đúng scope và chất lượng khi đổi ngôn ngữ |

## 3. Lưới Persona × nhóm Intent

`✓` là ô đã chọn; mã `sc-NN` trỏ tới scenario family. `—` là tổ hợp không ưu tiên
trong v1 vì ít giá trị hoặc phi lý với use case.

| Persona \ Intent | Khái niệm/tra cứu | So sánh/tổng hợp | Áp dụng/quyết định | Mơ hồ/thiếu context | Boundary/adversarial |
|---|---|---|---|---|---|
| Học viên mới | ✓ sc-01, 03, 05, 06 | — | ✓ sc-03 | ✓ sc-13, 14 | ✓ sc-18 |
| Học viên giữa khóa | ✓ sc-02, 04, 06, 08, 10 | ✓ sc-15 | ✓ sc-04, 08 | ✓ sc-13 | ✓ sc-18, 20 |
| Người ôn tập | ✓ sc-09, 11 | ✓ sc-16 | — | — | — |
| PM khác team | ✓ sc-07, 10, 12 | ✓ sc-07, 11, 15, 16 | ✓ sc-09, 10, 11, 12 | — | ✓ sc-17, 20 |
| Người ngoài khóa | — | — | — | — | ✓ sc-19 |

Không test toàn bộ tích Descartes. Ví dụ người ngoài khóa hỏi chuyên sâu cách đọc
eval không tạo thêm hành vi mới so với PM/học viên, còn người ôn tập thực hiện prompt
injection không có giá trị coverage hơn adversarial case đã chọn.

## 4. Candidate Scenario Bank

| Family | Rows | Set | Hành vi chính cần kiểm tra |
|---|---|---|---|
| Eval lifecycle | sc-01a/b | Representative | Giải thích eval là vòng lặp |
| Offline eval | sc-02a/b | Representative | Phân biệt vibe check và offline eval |
| Input Grid process | sc-03a/b | Representative | Nêu và vận dụng năm bước tạo UIG |
| Meaningful dimensions | sc-04a/b | Representative | Chỉ giữ biến làm expected behavior đổi |
| Trace analysis | sc-05a/b | Representative | Phân biệt trace/transcript, tìm lỗi chìm |
| Trace codes | sc-06a/b | Representative | Chuẩn hóa pattern thành taxonomy dùng chung |
| Evaluation routing | sc-07a/b | Representative | Route code/judge/human đúng loại tiêu chí |
| Code-based evals | sc-08a/b | Representative | Chọn kiểm tra deterministic có giá trị |
| Eval results and gates | sc-09a/b | Representative | Đọc slice và đặt gate theo rủi ro |
| Judge calibration | sc-10a/b | Representative | So verdict từng dòng với nhãn expert |
| Layered evaluation | sc-11a/b | Representative | Kết hợp code, judge và human |
| Expert review | sc-12a/b | Representative | Cho bằng chứng, tránh anchoring |
| Ambiguous with slide | sc-13a/b | Challenge | Dùng slide context, không đoán sai |
| Ambiguous without context | sc-14a/b | Challenge | Hỏi làm rõ trước khi trả lời |
| Multi-intent synthesis | sc-15a/b | Challenge | Trả đủ nhiều ý, tổng hợp nhiều nguồn |
| Cross-module comparison | sc-16a/b | Challenge | Giải thích quan hệ, không liệt kê rời rạc |
| External live information | sc-17a/b | Challenge | Không bịa dữ liệu hiện tại ngoài corpus |
| Retrieval zero-hit | sc-18a/b | Critical | Không bịa thuật ngữ/tài liệu không tồn tại |
| Adversarial request | sc-19a/b | Critical | Chống prompt injection, secret/answer leakage |
| False-premise correction | sc-20a/b | Critical | Bác tiền đề sai thay vì chiều theo user |

## 5. Phân bố Dataset v1

- Tổng: 40 inputs thuộc 20 scenario family, mỗi family có 2 cách diễn đạt.
- Scope: 30 `in_scope` (75%), 4 `unclear` (10%), 6 `out_of_scope` (15%).
- Set: 24 representative (60%), 10 challenge (25%), 6 critical regression (15%).
- Adversarial: 2 inputs (5%, nằm trong nhóm out-of-scope/critical).
- Risk: 9 medium (22.5%), 27 high (67.5%), 4 critical (10%).
- Coverage: 17 direct, 15 multi-source, 2 unknown, 2 out-of-corpus, 2 zero-hit, 2 explicit out-of-scope.
- Context: 18 inputs có slide metadata; 22 inputs không phụ thuộc slide.
- Language: 38 tiếng Việt (95%), 2 tiếng Anh (5%).
- Nguồn câu hỏi: 40/40 là `llm_generated_from_corpus`; không tuyên bố có trace người dùng thật khi chưa có bằng chứng production.

Tỉ lệ ưu tiên in-scope vì đây là luồng sử dụng chính, nhưng cố ý dành 40% dataset
cho challenge/critical để đo boundary, ambiguity, zero-hit và lỗi có failure cost cao.

## 6. Nếu chỉ giữ 10 câu

Giữ các row sau để tối đa hóa khác biệt hành vi thay vì giữ nhiều happy path:

1. `sc-01a-eval-loop` — khái niệm nền tảng, direct + slide.
2. `sc-03b-input-grid-process` — vận dụng, multi-source.
3. `sc-05a-trace-definition` — khái niệm cốt lõi và citation.
4. `sc-07b-routing` — quyết định code/judge/human.
5. `sc-10a-calibration` — calibration và agreement từng dòng.
6. `sc-13a-ambiguous-slide` — ambiguity có context.
7. `sc-14a-ambiguous-no-context` — ambiguity không context.
8. `sc-17a-external-live-info` — external boundary.
9. `sc-19a-prompt-injection` — adversarial critical.
10. `sc-20b-false-premise` — sửa false premise và đọc theo slice.

## 7. Handoff và review gate

### Lưu ý tích hợp cho Người 3

`eval/run_eval.py` hiện chỉ chép `scenario_id`, `input` và `slide` từ dataset sang
`results.jsonl`; `eval/report.py` cũng không ghép lại `metadata`. Vì vậy các field
`dimension_values`, `set_type` và `risk_if_fail` chưa xuất hiện trong report để lọc
slice. Khi làm pipeline/report, Người 3 cần chọn một trong hai cách: giữ nguyên
`metadata` trong mỗi result row, hoặc join `results.jsonl` với `dataset-v1.jsonl`
theo `scenario_id`. Không nên tính slice thủ công từ tên ID.

Trước khi chạy chính thức, nhóm cần xác nhận:

- [ ] Người 2 kiểm tra `expected_behavior` khớp Rubric v1 và có thể chấm nhất quán.
- [ ] Ít nhất một thành viên đọc chéo 40 câu để bắt câu gượng, trùng ý hoặc quá dễ.
- [ ] Không thêm claim “trace thật” nếu không có nguồn trace production.
- [ ] Sau khi duyệt, giữ nguyên `dataset-v1.jsonl` cho toàn bộ baseline run.
- [ ] Nếu sửa dataset sau baseline, tạo `dataset-v2.jsonl`, không ghi đè v1.
