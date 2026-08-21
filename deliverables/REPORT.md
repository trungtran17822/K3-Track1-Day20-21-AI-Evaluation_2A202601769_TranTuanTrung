# REPORT — Eval loop A→Z: VLearn AI Tutor

Report A→Z của eval loop — mỗi mục ứng một phase của bài lab. Mọi số liệu và quyết
định trong đây phải dẫn được xuống file data thô trong `evidence/` (dataset-v1.jsonl,
results-vN.jsonl, labels.csv, judge-prompt-vN.md, verdicts-vN.jsonl, braintrust-link.md).


---

## 1. Input Grid

> Lưới input = trục "ai hỏi" × "hỏi kiểu gì". LLM giúp sinh input, con người kiểm soát
> coverage. Trả lời các câu hỏi sau rồi vẽ lưới của bạn.

- AI Tutor của bạn phục vụ những **nhóm người dùng** nào? (học viên mới, học viên đang
  làm bài, học viên ôn lại, PM khác team...?)
- Mỗi nhóm có những **ý định (intent)** hỏi nào? (hỏi khái niệm, xin ví dụ, hỏi ngoài
  lề, xin đáp án, hỏi mơ hồ...?)
- Ô nào trong lưới là **rủi ro cao** nhất (trả lời sai thì hại người học)? Ô nào **tần
  suất cao** nhất?

### Lưới của bạn

| Nhóm user \ Intent | ... | ... | ... |
|---|---|---|---|
| ... | | | |

---

## 2. Dataset v1

> Dataset là "bộ đề thi" của tutor. Nêu rõ nó phủ những ô nào trong input-grid.

- `dataset.jsonl` của bạn có **bao nhiêu câu**? Mỗi câu thuộc ô nào trong lưới input?
- Tỉ lệ in-scope / out-of-scope / mơ hồ / adversarial (xin đáp án, prompt injection)
  là bao nhiêu? Vì sao chọn tỉ lệ đó?
- Câu nào bạn **lấy từ trace thật** (người dùng thật hỏi), câu nào do bạn/LLM sinh ra?
- Ai đã **review** dataset? Phát hiện gì khi review (câu trùng ý, câu quá dễ, thiếu ô
  rủi ro cao)?
- Nếu chỉ được giữ 10 câu, bạn giữ 10 câu nào? Vì sao?

### Danh sách scenario (bảng tóm tắt)

| scenario_id | ô trong lưới | expected | nguồn câu hỏi |
|---|---|---|---|
| | | | |

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
