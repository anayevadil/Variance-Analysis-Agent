# Variance Analysis Agent

An AI agent for finance teams that explains month-over-month variances. Give it a file of
transactions and two periods; it flags the accounts whose balance moved materially, drills into
the transactions to find what actually drove each move, and writes an Excel report with a short,
evidence-backed explanation for every material account. It keeps a memory of past findings, so
running it again on the same business produces sharper explanations.

Built by Adil Anayev and Heidi Tam for the Maximor hackathon (Money Ops track).

## At a glance

| | |
|---|---|
| **What it does** | Compares two periods, applies a materiality rule (change over 10% and above a minimum amount), then runs a Claude tool-use loop per material account to find which category, tags and transactions drove the change, and how unusual it is against that category's own history. |
| **Data** | One CSV of transactions with `date_time, type, category, account, amount, currency, tags` (expenses and income together). The bundled sample (`v2/data/sample_transactions.csv`, about 1,300 rows, 2025, BYN) comes from the Kaggle dataset [Financial transactions: expenses and income](https://www.kaggle.com/datasets/artemkabseu/financial-transactions-dataset-expenses-and-income). |
| **Stack** | Python 3.11, pandas, Claude API (`anthropic` SDK tool runner with `@beta_tool` functions), openpyxl for the report, a JSON memory store, pytest (30 tests). |
| **Output** | An Excel workbook, `reports/variance_report_<dataset>_<period_a>_vs_<period_b>.xlsx`, with a **Summary** sheet (one row per account: both period totals, $ and % change, a Material flag, and the explanation, with material accounts highlighted and linked) and a **Drill-Down** sheet (driving category, how unusual its move is, the tags that concentrate the change and their share, and the supporting transactions). |

Example explanation from a run on the sample data (October vs November 2025, expenses):

> **acct_1 changed by −53.2% (−879 BYN), driven by "Loan given" dropping −969 BYN (992 → 23),
> with 1 contributor (tag_1) accounting for 100% of that change, mainly the non-recurrence of
> October's single 854 BYN loan.** A new 299 BYN "Clothes" charge (also tag_1) partially offset
> the decline.

## How it works

```
transactions CSV ─► monthly summary ─► materiality rule ─► for each material account:
                                       (plain Python)       Claude tool-use loop
                                                              ├─ get_business_context   (memory)
                                                              ├─ compare_categories     (dollar impact + z-score)
                                                              ├─ analyze_concentration  (which tags drove it)
                                                              ├─ get_transactions       (evidence)
                                                              └─ record_insight         (writes memory)
                                                                   │
                                     Excel report  ◄── structured findings
```

- Which accounts get analyzed is decided by a transparent business rule, not by the model.
- Every number in the report is computed in Python; the model explains them but never invents them.
- `v2/memory/context.json` stores past findings and notes about accounts (for example "acct_2
  behaves like a shared account"), and every run reads it first.

## Run it

```bash
cd v2
python -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt
export ANTHROPIC_API_KEY=...

python agent.py --data data/sample_transactions.csv --dataset expenses
python agent.py --data data/sample_transactions.csv --dataset both \
    --period-a 2025-10 --period-b 2025-11 "focus on seasonal categories"

pytest   # offline tests, no API key needed
```

`--dataset` is `expenses`, `income` or `both`; the periods default to the two most recent in the
file. Without `--data` the agent looks for `sample_transactions.csv` on your Desktop.
[`v2/MANUAL.md`](v2/MANUAL.md) walks through the code step by step.

## Repository layout

| Path | What it is |
|---|---|
| `v2/` | Current version: the tool-use agent, analysis modules, Excel report, memory, tests |
| `*.py`, `data/` at the root | First hackathon version: a staged pipeline (variance engine, drill-down, narrative, memory). See [docs/v1-hackathon.md](docs/v1-hackathon.md) |
| `legacy/` | The original single-file prototype and notebook |
