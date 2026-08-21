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

V1 đã chạy `eval/judge.py` trên 40 rows bằng `openai/gpt-4o-mini`; output lưu tại
[`evidence/verdicts-v1.jsonl`](evidence/verdicts-v1.jsonl), prompt lưu tại
[`evidence/judge-prompt-v1.md`](evidence/judge-prompt-v1.md).

Chưa có `labels.csv` do người chấm export từ `report.html`, nên **chưa thể tính
agreement/confusion matrix với chuẩn vàng của con người**. Vì vậy judge v1 chỉ được xem
là tín hiệu tham khảo, chưa đủ điều kiện làm release gate độc lập.

Kết quả judge v1:

| Verdict | Số row | Tỉ lệ |
|---|---:|---:|
| pass | 38 | 95% |
| fail | 2 | 5% |
| uncertain | 0 | 0% |

Hai fail của judge đều nằm ở nhóm zero-hit:

- `sc-18a-zero-hit`: tutor không nên mở rộng sang các phương pháp khác khi câu hỏi hỏi một thuật ngữ không có trong corpus.
- `sc-18b-zero-hit`: tutor nhắc sang mô hình "Swiss cheese" không liên quan trực tiếp tới thuật toán bịa đặt "Falcon-RAG Gate 7 lớp".

Trace logging: `BRAINTRUST_API_KEY` hiện tại trả `401 Invalid API Key`, nên link trace bị
blocked; ghi lại tại [`evidence/braintrust-link.md`](evidence/braintrust-link.md).

### Confusion matrix (dán output judge.py)

```
labels.csv chưa có nhãn nào trùng scenario_id -> chưa tính được agreement.
Mở report.html, gán nhãn rồi bấm 'Export labels.csv' để có nhãn người.
```

---

## 6. Scorecard & Gate

V1 pipeline đã chạy xong trên Dataset v1 40 rows:

- Input: [`evidence/dataset-v1.jsonl`](evidence/dataset-v1.jsonl)
- Tutor output: [`evidence/results-v1.jsonl`](evidence/results-v1.jsonl)
- Code checks: [`evidence/code-checks-v1.txt`](evidence/code-checks-v1.txt)
- Judge output: [`evidence/verdicts-v1.jsonl`](evidence/verdicts-v1.jsonl)
- Review UI: `report.html` ở root repo, đã sinh 40 dòng dữ liệu

Run summary:

| Metric | Value |
|---|---:|
| Dataset rows | 40 |
| Tutor run errors | 0 |
| Total tokens | 194,004 |
| Average latency | 11.57s / row |
| Max latency | 23.01s |
| Estimated cost | Not available: `PRICING` chưa có Gemini model |

### Scorecard

| Tiêu chí | Pass | Fail | Uncertain | Pass rate |
|---|---|---|---|---|
| Schema & output contract | 40 | 0 | 0 | 100% |
| Citation exists | 40 | 0 | 0 | 100% |
| Quote verbatim | 14 | 26 | 0 | 35% |
| Scope contract | 40 | 0 | 0 | 100% |
| LLM judge tổng hợp | 38 | 2 | 0 | 95% |

Quan sát chính: model thường cite đúng `doc_id#section_id`, nhưng quote trong `sources`
không khớp nguyên văn section đã cite. Vì citation integrity là blocker trong rubric,
`quote_verbatim` 35% là lỗi release-blocking dù LLM judge tổng hợp đạt 95%.

### Quyết định gate

**Ngưỡng đề xuất trước khi chạy:** schema & citation integrity = 100%; groundedness
≥ 90%; scope handling ≥ 95%; không có fail blocker ở các scenario rủi ro cao
(out-of-scope, xin đáp án, prompt injection nếu có); latency trung bình và chi phí
được ghi nhận đầy đủ. Judge chỉ được dùng cho gate sau khi có agreement với nhãn người.

**CHƯA SHIP** — vì `quote_verbatim` chỉ đạt 14/40, fail 26/40 ở tiêu chí blocker.
Ngoài ra chưa có human labels để calibrate judge, và trace link Braintrust đang blocked
do API key invalid.

Ba việc cần fix trước vòng v2:

1. Sửa prompt/output contract để `sources[].quote` phải là đoạn trích nguyên văn từ section, không paraphrase.
2. Rerun `eval/run_eval.py` và `eval/code_checks.py`; gate tối thiểu là `quote_verbatim = 40/40`.
3. Gán nhãn tay trong `report.html`, export `labels.csv`, rồi chạy lại judge để có agreement/confusion matrix.

---

## 7. Verdict + Report cuối

> Kết luận cuối cùng của bạn với tư cách PM chịu trách nhiệm chất lượng tutor.
> Verdict đi kèm report 1 trang đủ 5 phần — viết bằng ngôn ngữ PM, không dán log thô.

### Report

#### 1. Dataset đã đánh giá

Đã đánh giá Dataset v1 gồm 40 traces / 20 scenario family. Coverage chính: khái niệm,
so sánh, áp dụng vào lab, mơ hồ có/không context, zero-hit, prompt injection, dữ liệu
hiện tại ngoài corpus, và false premise. Blind spot còn lại: chưa có trace người dùng
thật và chưa có nhãn người độc lập.

#### 2. Quá trình đồng thuận của con người

- Agreement vòng độc lập: chưa có, vì chưa export `labels.csv`.
- Mâu thuẫn lớn nhất dự kiến cần review: câu có citation đúng section nhưng quote không nguyên văn.
- Việc cần làm: mỗi người gán nhãn trong `report.html`, export labels, chạy `eval/agreement.py`.

#### 3. LLM judge

- Model judge: `openai/gpt-4o-mini`
- Số vòng calibration: 0 — v1 mới chạy judge, chưa calibrate với nhãn người.
- Kết quả thô: 38 pass / 2 fail / 0 uncertain. Không dùng làm gate độc lập cho tới khi có agreement.

#### 4. Bảng quyết định routing (kèm lý giải)

| Tiêu chí | Ngưỡng pass | Giao cho | Vì sao (dựa trên số liệu) |
|---|---|---|---|
| Schema & output contract | 100% | Code | V1 đạt 40/40, kiểm deterministic được. |
| Citation exists | 100% | Code | V1 đạt 40/40, manifest kiểm được. |
| Quote verbatim | 100% | Code + human audit near-miss | V1 chỉ đạt 14/40, đây là blocker lớn nhất. |
| Groundedness/scope | ≥90-95% sau calibration | LLM judge + human labels | Judge v1 đạt 38/40 nhưng chưa có agreement với người. |

#### 5. Verdict + bước tiếp theo

**Hold / CHƯA SHIP** — vì citation quote đang fail blocker 26/40, judge chưa calibrate,
và trace logging chưa có link hợp lệ.

- Đòn bẩy tiếp theo: sửa prompt hoặc post-process source quotes để ép quote nguyên văn.
- Metric chứng minh sẵn sàng: `schema_valid`, `citation_exists`, `quote_verbatim`, `scope_contract` đều 40/40; judge agreement với nhãn người đạt ngưỡng nhóm chốt.
- Sau khi có Braintrust/LangSmith key hợp lệ, rerun để bổ sung trace link vào evidence.

### Câu hỏi tự soi

- Tin cậy nhất ở đâu, đáng lo nhất ở đâu? (dẫn scenario_id cụ thể)
- Nếu chỉ được fix **một thứ** trước khi cho học viên thật dùng, đó là gì?
- Eval loop này sẽ chạy lại **khi nào** (mỗi lần đổi prompt? mỗi tuần? khi corpus đổi?) và ai nhìn kết quả?
- Điều gì trong bài này bạn sẽ **mang về áp dụng** vào sản phẩm thật của mình?
