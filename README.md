# LLM Evals: Choose the Best Model for Your Task

A hands-on case study in picking **one** LLM for a real application using a three-step framework: requirements, leaderboard filtering, and a custom evaluation pipeline.

The running example is a **Text-to-SQL engine for ESPN Cricinfo**, evaluated on an IPL (2021-2024) SQLite database. Candidate models are called through [OpenRouter](https://openrouter.ai) with LangChain's `ChatOpenRouter`, so swapping in a new model is a one-line change.

---

## Why this exists

Public leaderboards tell you which models are good in general, not which one is best (and affordable) for *your* task. This repo shows how to close that gap:

> **Requirements -> Leaderboard shortlist -> Custom eval on your own data -> Pick the winner**

---

## The three-step framework

### Step 1: Requirement gathering
Define the constraints before looking at any model. Full write-up in [`1.0.use_case.txt`](1.0.use_case.txt).

| Requirement | This case study |
|---|---|
| Task | Text-to-SQL |
| Budget | 3 lakh INR / month |
| Traffic estimate | ~50,000 queries/day, ~400 input + ~100 output tokens each |
| Latency | 2-3 seconds |
| Context window | Not a constraint |
| Deployment | Open-source is an option (data-privacy friendly) |
| Correctness | High accuracy, no hallucinated answers on cricket questions |

**Cost check (worked example):** a premium model priced at $10 / 1M input and $50 / 1M output tokens works out to about $0.009 per query, roughly $13,500 per month, or about 12.8 lakh INR. That is far above the 3 lakh budget, so it is ruled out and cheaper models are shortlisted. The same file also covers prompt caching as a cost and latency lever.

### Step 2: Leaderboard filtering
Shortlist 5-10 models by balancing performance, cost, and speed, and discard benchmarks that do not resemble real database work.

### Step 3: Custom evaluation pipeline
Build a golden dataset of 30-50 hard questions, each with a human-validated SQL query. For every model:

```
Question --> LLM --> generated SQL --> run on DB --> Table A
Question --> golden SQL (human-written) --> run on DB --> Table B
                                   Table A == Table B ?  -> correct / incorrect

accuracy = correct answers / total questions
```

This is **execution accuracy**: SQL is judged by the rows it returns, not by string-matching the query text. Rows are compared with an order-sensitive or order-insensitive check depending on each question's `order_sensitive` flag.

---

## Repository structure

| File | Purpose |
|---|---|
| `1.0.use_case.txt` | Case study: requirements, cost math, framework |
| `2.0.db.py` | Builds the IPL SQLite database |
| `3.0.schema_extractor.py` | Extracts the DB schema used in prompts |
| `4.0.make_golden_dataset.py` | Builds the golden dataset |
| `5.0.first_test.py` | First smoke test of the pipeline |
| `7.0.main.py` | **Main entry point**: runs every model over the golden set and scores it |
| `evaluator.py` | Result-comparison logic (`evaluate_one`) |
| `golden_dataset_generator.py` | Helper for generating golden questions |
| `model_openrouter_slug.py` | Models under test, as `(display name, OpenRouter slug)` |
| `schema.sql` | Database schema injected into the prompt |
| `golden_dataset.csv` | Golden questions with gold SQL and stored gold results |
| `golden_hard.csv` | Harder question set |
| `ipl_2021_2024.db` | SQLite database of IPL 2021-2024 data |
| `eval_results.csv` | Per-question output from the latest run |
| `data/` | Source data |
| `requirements.txt` | Python dependencies |

---

## Getting started

### 1. Clone and install

```bash
git clone https://github.com/rohankumarsoni/LLM-EVALS-Choose-Best-Model-for-Task.git
cd LLM-EVALS-Choose-Best-Model-for-Task

python -m venv .venv
source .venv/bin/activate        # Windows: .venv\Scripts\activate
pip install -r requirements.txt
```

### 2. Add your OpenRouter key

Create a `.env` file in the project root:

```env
OPENROUTER_API_KEY=sk-or-...
```

### 3. Choose the models to test

Edit `model_openrouter_slug.py`. Any slug from [openrouter.ai/models](https://openrouter.ai/models) works:

```python
MODELS = [
    ("GPT-5.6 Terra",   "openai/gpt-5.6-terra-pro"),
    ("Kimi K3",         "moonshotai/kimi-k3"),
    ("Grok 4.5",        "x-ai/grok-4.5"),
    ("Claude Sonnet 5", "anthropic/claude-sonnet-5"),
    ("MiniMax M3",      "minimax/minimax-m3"),
]
```

### 4. Run the evaluation

```bash
python 7.0.main.py
```

For each model, the script:

1. Loads the model through `ChatOpenRouter` (`temperature=0`, `max_tokens=800`)
2. Generates SQL for every golden question from the schema plus the question
3. Cleans the output (strips markdown fences and prose) and runs it on the SQLite DB
4. Compares the result with the gold result via `evaluator.py`
5. Prints a final scoreboard and writes per-question details to `eval_results.csv`

---

## Output

`eval_results.csv` has one row per (model, question):

| Column | Meaning |
|---|---|
| `model` | Display name from `MODELS` |
| `id` | Golden question id |
| `difficulty` | Question difficulty label |
| `correct` | `True` if the result matched the gold result |
| `reason` | Why it passed or failed (for example `gen_error`) |
| `sql` | The SQL the model generated |

### Results

Fill this in after your run (accuracy = correct / total):

| Model | Correct | Accuracy |
|---|---|---|
| ... | ... | ... |

---

## Design notes

- **Execution-based scoring.** Two different SQL queries can both be right. Comparing result tables avoids penalizing valid alternatives.
- **Order sensitivity per question.** Only questions that ask for a ranking or ordering are compared with row order enforced.
- **`max_tokens` is capped.** SQL is short, and a low cap keeps OpenRouter's up-front credit reservation small. Without it you can hit `GEN-ERROR: This request would exceed your available credits` even with a healthy balance.
- **One prompt for all models.** Every model gets the identical system prompt and schema so the comparison is fair.

---

## Troubleshooting

| Problem | Fix |
|---|---|
| `Missing OPENROUTER_API_KEY` | Add the key to `.env` and run from the project root |
| `GEN-ERROR ... exceed your available credits` | Add credits, lower `max_tokens`, or test fewer models or questions at a time |
| `ImportError: langchain_openrouter` | `pip install -U langchain-openrouter` (note the hyphen when installing, underscore when importing) |
| Model returns prose instead of SQL | `clean_sql()` extracts the first `SELECT` or `WITH`; check the `RAW (uncleaned)` log line |

---

## Extending this

- Swap in your own domain: replace `ipl_2021_2024.db`, `schema.sql`, and the golden dataset
- Add cost and latency columns to `eval_results.csv` and rank on accuracy per rupee
- Add a third answer-quality check (for example an LLM judge) for non-SQL tasks
- Re-run whenever a new model ships; the harness makes this a matter of minutes

---

## Author

**Rohan Kumar Soni**
GitHub: [@rohankumarsoni](https://github.com/rohankumarsoni)
