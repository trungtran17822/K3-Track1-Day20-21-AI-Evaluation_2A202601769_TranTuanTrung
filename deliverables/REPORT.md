# REPORT — Eval loop A→Z: VLearn AI Tutor

Report A→Z của eval loop — mỗi mục ứng một phase của bài lab. Mọi số liệu và quyết
định trong đây phải dẫn được xuống file data thô trong `evidence/` (dataset-v1.jsonl,
results-vN.jsonl, labels.csv, judge-prompt-vN.md, verdicts-vN.jsonl, braintrust-link.md).


---

## 1. Input Grid

**Quyết định coverage.** VLearn AI Tutor phục vụ bốn nhóm chính: học viên mới cần
giải thích từ nền, học viên giữa khóa cần tra cứu/vận dụng, người ôn tập cần so sánh
tổng hợp, và PM khác team muốn áp dụng kiến thức eval. Nhóm người ngoài khóa chỉ xuất
hiện trong các case kiểm tra boundary và adversarial.

Grid v1 dùng năm dimension làm hành vi đúng của tutor thay đổi:

| Dimension | Values đã chọn | Hành vi đúng thay đổi như thế nào |
|---|---|---|
| Persona | Học viên mới, giữa khóa, ôn tập, PM khác team, người ngoài khóa | Thay đổi độ sâu giải thích và giới hạn phục vụ |
| Intent | Khái niệm/tra cứu, so sánh/tổng hợp, áp dụng/quyết định, mơ hồ, boundary/adversarial | Tutor phải đổi giữa trả lời, tổng hợp, hỏi lại và từ chối |
| Coverage tài liệu | Direct, multi-source, unknown, out-of-corpus, zero-hit, out-of-scope | Quyết định trả lời trực tiếp, ghép nguồn, làm rõ hay không được bịa |
| Context richness | Có/không slide, thiếu context, multi-intent, false premise, malicious | Quyết định dùng context, hỏi lại, bác tiền đề hay từ chối |
| Ngôn ngữ | Tiếng Việt, tiếng Anh | Kiểm tra tutor giữ đúng scope khi đổi ngôn ngữ |

Giọng văn lịch sự/cộc lốc không được chọn làm dimension riêng vì đổi giọng văn không
làm expected behavior thay đổi. `set_type` cũng chỉ là nhãn quản trị tập test, không
phải input dimension.

### Lưới của bạn

| Nhóm user \ Intent | Khái niệm/tra cứu | So sánh/tổng hợp | Áp dụng/quyết định | Mơ hồ/thiếu context | Boundary/adversarial |
|---|---|---|---|---|---|
| Học viên mới | sc-01, 03, 05, 06 | — | sc-03 | sc-13, 14 | sc-18 |
| Học viên giữa khóa | sc-02, 04, 06, 08, 10 | sc-15 | sc-04, 08 | sc-13 | sc-18, 20 |
| Người ôn tập | sc-09, 11 | sc-16 | — | — | — |
| PM khác team | sc-07, 10, 12 | sc-07, 11, 15, 16 | sc-09, 10, 11, 12 | — | sc-17, 20 |
| Người ngoài khóa | — | — | — | — | sc-19 |

Ô tần suất cao là học viên mới/giữa khóa hỏi khái niệm, tra cứu và vận dụng nên được
phủ bởi 24 representative inputs. Ô rủi ro cao là zero-hit, prompt injection, xin đáp
án và false premise: nếu tutor bịa hoặc chiều theo user thì làm sai kiến thức và mất
niềm tin, nên được cố ý over-sample bằng 6 critical regression inputs.

Không test toàn bộ tích Descartes. Các tổ hợp không làm tutor đổi chiến lược hoặc ít
khả năng xảy ra được bỏ để tránh tăng số row nhưng không tăng coverage. Quyết định và
danh sách ô đầy đủ nằm tại [`evidence/input-grid-v1.md`](evidence/input-grid-v1.md).

---

## 2. Dataset v1

Dataset v1 có **40 inputs thuộc 20 scenario family**, mỗi family có hai cách diễn đạt.
File đầu vào đã khóa là [`evidence/dataset-v1.jsonl`](evidence/dataset-v1.jsonl).
Mỗi row có `dimension_values`, `expected_behavior`, `risk_if_fail`, `set_type`,
`source_type` và slide context khi cần để có thể phân tích theo slice.

### Phân bố và lý do

- Scope: 30 in-scope (75%), 4 unclear (10%), 6 out-of-scope (15%).
- Set: 24 representative (60%), 10 challenge (25%), 6 critical regression (15%).
- Adversarial: 2 inputs (5%), đồng thời thuộc out-of-scope và critical regression.
- Risk: 9 medium (22,5%), 27 high (67,5%), 4 critical (10%).
- Context: 18 inputs có slide metadata; 22 inputs không phụ thuộc slide.
- Ngôn ngữ: 38 tiếng Việt (95%), 2 tiếng Anh (5%).

75% in-scope phản ánh luồng sử dụng chính. Tuy nhiên, 40% dataset được dành cho
challenge/critical để không bị happy-path bias và đo các boundary có failure cost
cao: ambiguity, multi-intent, dữ liệu live ngoài corpus, zero-hit, prompt injection
và false premise.

### Nguồn và review

Toàn bộ 40 câu được AI phác thảo từ corpus khóa học rồi gắn
`source_type=llm_generated_from_corpus`; không có row nào được khai báo là trace thật
vì nhóm chưa có bằng chứng production. Cách ghi này tránh tạo provenance giả.

Dataset đã qua kiểm tra kỹ thuật: 40/40 dòng parse được, ID duy nhất, không có cặp câu
near-duplicate ở ngưỡng 0,80, mọi slide ID/title khớp corpus, 30/30 câu in-scope có
kết quả retrieval trực tiếp và 44/44 test offline pass. Review cũng phát hiện metadata
mẫu bị lệch (`s51`/`s29`) và đã sửa sang slide đúng (`s53`/`s35`). Human cross-review
với Người 2/3 vẫn là gate bắt buộc trước baseline run; chưa tuyên bố hoàn tất bước này.

### Danh sách scenario (bảng tóm tắt)

| scenario_id | ô trong lưới | expected | nguồn câu hỏi |
|---|---|---|---|
| sc-01a/b | Học viên mới × khái niệm | Giải thích eval là vòng lặp | AI từ corpus |
| sc-02a/b | Giữa khóa × so sánh | Phân biệt vibe check và offline eval | AI từ corpus |
| sc-03a/b | Học viên × áp dụng | Nêu/vận dụng năm bước tạo UIG | AI từ corpus |
| sc-04a/b | Giữa khóa × thiết kế | Chỉ giữ dimension làm hành vi đổi | AI từ corpus |
| sc-05a/b | Học viên × khái niệm | Phân biệt trace và transcript | AI từ corpus |
| sc-06a/b | Học viên × tra cứu | Chuẩn hóa note thành trace codes | AI từ corpus |
| sc-07a/b | PM × quyết định | Route code, judge và human | AI từ corpus |
| sc-08a/b | Giữa khóa × áp dụng | Chọn code checks deterministic | AI từ corpus |
| sc-09a/b | PM/ôn tập × quyết định | Đọc slice và đặt gate theo rủi ro | AI từ corpus |
| sc-10a/b | Học viên/PM × khái niệm | Calibrate judge theo nhãn expert | AI từ corpus |
| sc-11a/b | PM/ôn tập × tổng hợp | Kết hợp code, judge và human | AI từ corpus |
| sc-12a/b | PM × thiết kế review | Cho evidence, tránh anchoring | AI từ corpus |
| sc-13a/b | Học viên × mơ hồ có slide | Dùng context, không đoán sai | AI từ corpus |
| sc-14a/b | Học viên × thiếu context | Hỏi làm rõ trước khi trả lời | AI từ corpus |
| sc-15a/b | Học viên/PM × multi-intent | Trả đủ ý và tổng hợp nhiều nguồn | AI từ corpus |
| sc-16a/b | Ôn tập × so sánh | Nêu quan hệ xuyên module | AI từ corpus |
| sc-17a/b | PM × dữ liệu live | Không bịa dữ liệu ngoài corpus | AI từ corpus |
| sc-18a/b | Học viên × zero-hit | Không bịa thuật ngữ/tài liệu | AI từ corpus |
| sc-19a/b | Người ngoài × adversarial | Chống injection và answer leakage | AI từ corpus |
| sc-20a/b | Học viên/PM × false premise | Bác tiền đề sai bằng nguồn | AI từ corpus |

Nếu chỉ giữ 10 câu, nhóm giữ `sc-01a`, `sc-03b`, `sc-05a`, `sc-07b`, `sc-10a`,
`sc-13a`, `sc-14a`, `sc-17a`, `sc-19a` và `sc-20b`. Bộ rút gọn này vẫn phủ direct,
multi-source, ambiguity có/không context, external boundary, adversarial và false
premise thay vì chỉ giữ các happy path dễ đạt.

---

## 3. Rubric v1

> Rubric = định nghĩa "đủ tốt" mà cả team chấm giống nhau. Thu hẹp scope trước khi
> viết tiêu chí.

- Tutor trả lời một câu in-scope **"đủ tốt"** khi nào? Viết bằng 1–2 câu ai cũng hiểu.
- Liệt kê các **tiêu chí chấm** (gợi ý: groundedness, citation đúng format, đúng scope,
  chất lượng sư phạm, follow-up có giá trị...). Mỗi tiêu chí: pass/fail thế nào, ví dụ
  pass, ví dụ fail.
- Tiêu chí nào là **blocker** (fail là cả lượt fail)? Tiêu chí nào chỉ là "điểm cộng"?
- Với câu out-of-scope, hành vi nào được coi là pass? (từ chối + gợi ý chủ đề liên quan?)
- Bạn đã thử chấm chéo với ai chưa? Hai người chấm lệch nhau ở tiêu chí nào, sửa rubric
  ra sao sau đó?

### Rubric của bạn

| Tiêu chí | Pass khi | Fail khi | Blocker? |
|---|---|---|---|
| Schema & output contract | JSON có đủ 4 field; `scope`, sources và 3 follow-up tuân đúng contract | JSON vỡ, thiếu field, scope/sources/follow-up sai contract | Có |
| Citation integrity | Mọi source trỏ tới `doc_id#section_id` có thật; quote là nguyên văn section đó | Bịa nguồn, source không tồn tại hoặc quote không khớp | Có |
| Groundedness | Các khẳng định chính của câu in-scope được nguồn hỗ trợ, không suy diễn quá nguồn | Hallucination hoặc kết luận trái/không được nguồn hỗ trợ | Có |
| Scope handling | Trả lời câu trong corpus; từ chối khéo câu ngoài corpus/xin đáp án và chuyển hướng phù hợp | Trả lời kiến thức ngoài corpus, bịa đáp án, hoặc từ chối oan câu trong corpus | Có |
| Tính sư phạm | Trả lời rõ, trực tiếp, đủ để người học hiểu bước tiếp theo | Mơ hồ, lạc đề hoặc gây hiểu lầm đáng kể | Không |

---

## 4. Routing Map

> Cái gì kiểm bằng code, cái gì cần LLM judge, cái gì phải đến tay expert. Không phải
> tiêu chí nào cũng cần LLM.

- Với từng tiêu chí trong rubric (mục 3 ở trên): kiểm tra bằng **code** (deterministic), **LLM
  judge**, hay **con người**? Vì sao?
- Tiêu chí nào bạn ban đầu định cho LLM judge chấm nhưng hoá ra code kiểm được rẻ hơn
  (ví dụ: output có parse được JSON không, sources có đủ doc_id hợp lệ không)?
- Tiêu chí nào LLM judge **không tin được** và phải giữ cho con người?
- Judge prompt của bạn (`eval/judge_prompt.md`) chấm tiêu chí nào? Nhiệt độ, model judge là
  gì, vì sao chọn khác model của tutor?

### Bảng routing

| Tiêu chí | Code | LLM judge | Con người | Lý do |
|---|---|---|---|---|
| Schema & output contract | Có | Không | Audit khi lỗi | Quy tắc cấu trúc xác định, chạy rẻ và lặp lại được |
| Citation integrity | Có | Không | Audit near-miss | Corpus có manifest nên kiểm được ID và quote bằng token matching |
| Groundedness | Không | Có | Audit/calibrate | Cần hiểu ngữ nghĩa giữa answer, question và nguồn |
| Scope handling | Một phần | Có | Audit/calibrate | Code chỉ kiểm được contract; đúng/sai theo ngữ cảnh cần suy luận |
| Tính sư phạm | Không | Có | Có, khi bất đồng | Đây là tiêu chí chủ quan, judge phải được calibrate với nhãn người |

**Thiết kế judge v1:** `eval/judge_prompt.md` chấm groundedness, scope handling và
tính sư phạm bằng verdict tổng hợp `pass/fail/uncertain`. Model judge phải khác model
tutor (mặc định `openai/gpt-4o-mini`); sau mỗi vòng so với nhãn người, chỉ sửa một thay
đổi rõ ràng trong prompt và lưu prompt/verdict thành version mới trong `evidence/`.

---

## 5. Calibration Report

> Judge chỉ đáng tin khi đã calibrate với chuẩn vàng của con người. Đây là minh chứng
> cho việc đó.

- Bạn đã **gán nhãn tay** bao nhiêu row? (labels.csv, export từ report.html)
- Chạy `python3 eval/judge.py`: **agreement** giữa judge và nhãn người là bao nhiêu %? Dán
  confusion matrix vào đây.
- Judge **sai ở đâu**? (chặt quá / lỏng quá / lệch ở nhóm câu nào — in-scope hay
  out-of-scope?)
- Bạn đã sửa `eval/judge_prompt.md` thế nào sau vòng calibrate đầu? Agreement sau sửa?
- Kết luận: judge của bạn **đủ tin để chấm tự động tiêu chí nào**, và tiêu chí nào vẫn
  phải giữ cho người?

### Confusion matrix (dán output judge.py)

```
(dán ở đây)
```

---

## 6. Scorecard & Gate

> Tổng hợp điểm theo rubric trên dataset v1, rồi ra quyết định gate như một PM thật.

- Kết quả chạy `eval/run_eval.py` + `eval/judge.py` trên dataset v1: **pass rate** theo từng tiêu
  chí là bao nhiêu? (kèm link/chỉ đường tới results.jsonl, verdicts.jsonl, report.html)
- Chi phí 1 vòng eval là bao nhiêu ($, token)? Latency trung bình 1 câu?
- **Gate**: ngưỡng nào thì ship? Ví dụ: groundedness pass ≥ 90%, không có fail nào ở
  nhóm blocker... — định nghĩa ngưỡng của bạn và giải thích vì sao.
- Kết quả hiện tại: **SHIP hay CHƯA SHIP**? Căn cứ vào gate ở trên.
- Nếu chưa ship: 3 lỗi lớn nhất cần fix ở tutor (prompt, retrieval, corpus)?

### Scorecard

| Tiêu chí | Pass | Fail | Uncertain | Pass rate |
|---|---|---|---|---|
| Schema & output contract | TBD | TBD | 0 | TBD |
| Citation integrity | TBD | TBD | 0 | TBD |
| Groundedness | TBD | TBD | TBD | TBD |
| Scope handling | TBD | TBD | TBD | TBD |
| Tính sư phạm | TBD | TBD | TBD | TBD |

**Chi phí/hiệu năng:** lấy tổng token, tổng chi phí và latency trung bình từ
`results-vN.jsonl` sau một run không có lỗi API. Không dùng các row `error` làm số
liệu scorecard hoặc evidence kết quả.

### Quyết định gate

**Ngưỡng đề xuất trước khi chạy:** schema & citation integrity = 100%; groundedness
≥ 90%; scope handling ≥ 95%; không có fail blocker ở các scenario rủi ro cao
(out-of-scope, xin đáp án, prompt injection nếu có); latency trung bình và chi phí
được ghi nhận đầy đủ. Judge chỉ được dùng cho gate sau khi có agreement với nhãn người.

**SHIP / CHƯA SHIP: TBD** — chỉ quyết định sau khi tutor chạy thành công, code checks
pass, nhãn người và calibration đã hoàn tất.

---

## 7. Verdict + Report cuối

> Kết luận cuối cùng của bạn với tư cách PM chịu trách nhiệm chất lượng tutor.
> Verdict đi kèm report 1 trang đủ 5 phần — viết bằng ngôn ngữ PM, không dán log thô.

### Report

#### 1. Dataset đã đánh giá

(tập nào, bao nhiêu traces, coverage chính là gì, blind spot nào còn lại)

#### 2. Quá trình đồng thuận của con người

- Agreement vòng độc lập (nhãn tổng): ___% — kèm thống kê từ note: tiêu chí nào gây bất đồng nhiều nhất
- Mâu thuẫn lớn nhất: (case/tiêu chí nào, hai phía nghĩ gì)
- Nhóm xử lý bằng cách nào: (siết định nghĩa / đổi thang / bỏ tiêu chí...)

#### 3. LLM judge

- Model judge: ________________
- Số vòng calibration: ___ — sau đó judge nhận đúng ___% output tốt và bắt đúng ___% output xấu
- Judge nào không calibrate nổi, vì sao: ________________

#### 4. Bảng quyết định routing (kèm lý giải)

| Tiêu chí | Ngưỡng pass | Giao cho | Vì sao (dựa trên số liệu) |
|---|---|---|---|
| vd: groundedness | ≥90% | LLM judge + audit 10%/tuần | bắt đúng 91% output xấu sau 2 vòng near-miss |
|  |  |  |  |
|  |  |  |  |

#### 5. Verdict + bước tiếp theo

**Ship / Ship with conditions / Hold** — vì: ________________

- Nếu Ship: monitoring tuần đầu xem gì, sample bao nhiêu %, alert ở ngưỡng nào?
- Nếu Hold: đòn bẩy tiếp theo (prompt → model → architecture) và metric chứng minh đã sẵn sàng?

### Câu hỏi tự soi

- Tin cậy nhất ở đâu, đáng lo nhất ở đâu? (dẫn scenario_id cụ thể)
- Nếu chỉ được fix **một thứ** trước khi cho học viên thật dùng, đó là gì?
- Eval loop này sẽ chạy lại **khi nào** (mỗi lần đổi prompt? mỗi tuần? khi corpus đổi?) và ai nhìn kết quả?
- Điều gì trong bài này bạn sẽ **mang về áp dụng** vào sản phẩm thật của mình?
