# Braintrust / LangSmith trace link

Status: blocked for v1 run.

Reason: `BRAINTRUST_API_KEY` in `.env` returned `401 Invalid API Key` from Braintrust during `eval/run_eval.py`.

Action taken: Braintrust was temporarily disabled so the eval pipeline could complete and produce `results-v1.jsonl`, `code-checks-v1.txt`, `verdicts-v1.jsonl`, and `report.html`.

Next action: replace `BRAINTRUST_API_KEY` or configure `LANGSMITH_API_KEY`, rerun `eval/run_eval.py` and `eval/judge.py`, then paste the project URL here.
