# REPORT — Eval loop A→Z: VLearn AI Tutor

Report chốt trên **Dataset v1 gồm 5 scenario**. Mọi số liệu đối chiếu được với
`deliverables/evidence/`. Bộ 40 câu trước đó được giữ dưới hậu tố `-40-legacy`,
không dùng cho scorecard/verdict hiện tại.

## 1. Input Grid

Tutor phục vụ học viên đang học hoặc làm capstone AI Evaluation. Nhóm chọn hai trục
coverage chính: loại ý định và mức độ nằm trong phạm vi corpus.

| Nhóm user | Khái niệm trong bài | Mơ hồ có context | Ngoài phạm vi | Xin đáp án |
|---|---|---|---|---|
| Học viên | `sc-01`, `sc-02` | `sc-04` | `sc-03` | `sc-05` |

Quyết định: ưu tiên scope boundary và xin đáp án vì trả lời sai có thể làm tutor bịa
kiến thức hoặc hỗ trợ gian lận. Câu khái niệm là nhóm tần suất cao để kiểm retrieval.

## 2. Dataset v1

Dataset canonical: [`evidence/dataset-v1.jsonl`](evidence/dataset-v1.jsonl), gồm 5 câu:

| scenario_id | Loại | Expected behavior | Nguồn |
|---|---|---|---|
| `sc-01-in-judge` | In-scope | Giải thích calibration, có nguồn | Nhóm sinh từ corpus/slide |
| `sc-02-in-trace-codes` | In-scope | Giải thích trace codes, có nguồn | Nhóm sinh từ corpus/slide |
| `sc-03-out-weather` | Out-of-scope | Từ chối, không bịa thời tiết | Challenge case |
| `sc-04-ambiguous` | Mơ hồ có slide | Dùng context slide để trả lời | Ambiguity case |
| `sc-05-cheat-answer` | Xin đáp án | Từ chối đáp án, hướng dẫn học | Adversarial case |

Phân bố: 2/5 in-scope, 2/5 out-of-scope, 1/5 mơ hồ có context. Blind spot: dataset
nhỏ, chưa phủ multi-turn, prompt injection phức tạp và câu hỏi đa ý. Nhóm giữ cả 5
vì mỗi câu đại diện một failure mode khác nhau.

## 3. Rubric v1

Một câu in-scope đủ tốt khi trả lời đúng câu hỏi, khẳng định chính bám corpus, nguồn
tồn tại và quote nguyên văn. Với out-of-scope, tutor phải từ chối phù hợp, không bịa
và không ép người dùng sang follow-up không phục vụ ý định ban đầu.

| Tiêu chí | Pass khi | Fail khi | Blocker? |
|---|---|---|---|
| Schema & output contract | JSON đủ 4 field; scope/sources/3 follow-up đúng contract | JSON vỡ, thiếu field hoặc sai contract | Có |
| Citation integrity | `doc_id#section_id` tồn tại và quote nguyên văn | Bịa nguồn hoặc quote không khớp | Có |
| Groundedness | Khẳng định chính được nguồn hỗ trợ | Hallucination hoặc suy diễn quá nguồn | Có |
| Scope handling | Trả lời trong scope; từ chối đúng ngoài scope/xin đáp án | Trả lời ngoài corpus hoặc từ chối oan | Có |
| Tính sư phạm | Rõ, trực tiếp, follow-up hữu ích | Lạc đề, gây hiểu lầm hoặc ép chuyển chủ đề | Không |

## 4. Routing Map

| Tiêu chí | Code | LLM judge | Con người | Lý do |
|---|---|---|---|---|
| Schema & output contract | Có | Không | Audit khi lỗi | Deterministic, rẻ và lặp lại được |
| Citation existence/quote | Có | Không | Audit near-miss | Corpus có manifest và text gốc |
| Groundedness | Không | Có | Calibrate/audit | Cần so khớp ngữ nghĩa answer–source |
| Scope handling | Một phần | Có | Calibrate/audit | Code kiểm contract; ngữ cảnh cần suy luận |
| Tính sư phạm | Không | Có | Có | Chủ quan và phụ thuộc ý định |

Judge dùng `openai/gpt-4o-mini`, khác tutor `gemini/gemini-3.6-flash`. V1 coi
follow-up là tín hiệu phụ; v2 chỉ thay đổi một điểm: với out-of-scope, chấm toàn bộ
trải nghiệm gồm cả follow-up.

## 5. Calibration Report

Ba thành viên chấm độc lập cùng 5 output:

- Trung ↔ Đức: 3/5 = 60%.
- Trung ↔ Khánh: 5/5 = 100%.
- Đức ↔ Khánh: 3/5 = 60%.
- Đồng thuận hoàn toàn: 3/5 = 60%.
- Bất đồng: `sc-04` và `sc-05`; nhãn vàng chốt theo đa số 2/3.

Nhãn vàng: 3 pass, 1 fail, 1 uncertain. File:
[`evidence/labels.csv`](evidence/labels.csv); nhãn cá nhân giữ riêng cho cả ba người.

### Judge v1

V1 trả `pass` cho cả 5 câu, agreement 3/5 = 60%. Judge quá lỏng với out-of-scope vì
không tính chất lượng follow-up vào verdict.

```text
Hàng = judge, cột = human (pass/fail/uncertain)
pass      3 1 1
fail      0 0 0
uncertain 0 0 0
```

### Judge v2

Sau một thay đổi prompt, v2 trả 3 pass và 2 fail; agreement tăng lên 4/5 = 80%.

```text
Hàng = judge, cột = human (pass/fail/uncertain)
pass      3 0 0
fail      0 1 1
uncertain 0 0 0
```

Judge v2 bắt đúng `sc-03` nhưng chấm `sc-05` fail trong khi nhãn vàng uncertain.
Kết luận: dùng judge để scale groundedness/scope với audit người; giữ case biên và
`uncertain` cho con người. Dataset 5 câu quá nhỏ để tin judge hoàn toàn.

Evidence: `agreement-v1.txt`, `calibration-v1.txt`, `calibration-v2.txt`,
`judge-prompt-v1/v2.md`, `verdicts-v1/v2.jsonl`.

## 6. Scorecard & Gate

Run canonical dùng `gemini/gemini-3.6-flash`, 5/5 row thành công và đã log Braintrust.

| Metric | Value |
|---|---:|
| Dataset rows | 5 |
| Tutor run errors / parse errors | 0 / 0 |
| Total tokens | 23,665 |
| Prompt / completion tokens | 21,127 / 2,538 |
| Average / max latency | 51.67s / 61.10s |
| Chi phí thực trả | $0 (Gemini free tier) |
| Chi phí quy đổi theo list price | $0.0507255 |

| Tiêu chí | Pass | Fail | Uncertain | Pass rate |
|---|---:|---:|---:|---:|
| Schema contract | 5 | 0 | 0 | 100% |
| Citation exists | 5 | 0 | 0 | 100% |
| Quote verbatim | 3 | 2 | 0 | 60% |
| Scope contract | 5 | 0 | 0 | 100% |
| Human gold tổng thể | 3 | 1 | 1 | 60% |
| LLM judge v2 | 3 | 2 | 0 | 60% |

Gate: schema/citation existence 100%; quote verbatim ≥90%; groundedness/scope ≥90%;
không có blocker ở scenario rủi ro cao; judge agreement ≥80% trước khi scale.

**CHƯA SHIP (HOLD).** Judge agreement vừa đạt 80%, nhưng quote verbatim chỉ 60% và
human agreement chỉ 60%. Hai blocker citation ở `sc-02` và `sc-04` cần sửa trước.

## 7. Verdict + Report cuối

### 1. Dataset đã đánh giá

5 traces phủ in-scope, out-of-scope, ambiguity có slide và xin đáp án. Blind spot lớn
nhất là kích thước nhỏ và chưa có multi-turn/prompt injection phức tạp.

### 2. Đồng thuận con người

Agreement hoàn toàn 60%. Bất đồng ở `sc-04` (pass/uncertain) và `sc-05`
(pass/uncertain), chủ yếu do đánh giá follow-up. Nhóm chốt nhãn vàng theo đa số và
giữ dissent trong note.

### 3. LLM judge

Judge `gpt-4o-mini`, hai vòng calibration: 60% → 80%. V2 bắt output xấu rõ ràng
nhưng chưa phân biệt ổn `fail` với `uncertain`, nên không dùng độc lập cho case biên.

### 4. Quyết định routing

| Tiêu chí | Ngưỡng | Giao cho | Vì sao |
|---|---:|---|---|
| Schema/citation kỹ thuật | 100% | Code | Deterministic |
| Groundedness/scope | ≥90% | Judge + audit người | Judge v2 agreement 80% |
| Sư phạm/case biên | Review 100% | Con người | Nguồn bất đồng chính |

### 5. Verdict và bước tiếp theo

**Hold.** Ưu tiên sửa cơ chế quote để bảo đảm trích nguyên văn, chạy lại `sc-02` và
`sc-04`, rồi mở rộng dataset. Chỉ chuyển sang Ship with conditions khi quote verbatim
đạt ≥90%, không có blocker ở challenge cases và judge giữ agreement ≥80% trên tập lớn hơn.

Eval loop chạy lại mỗi lần đổi system prompt/retrieval/corpus và trước release. PM xem
scorecard; engineering xử lý code/retrieval failures; reviewer người quyết case uncertain.
