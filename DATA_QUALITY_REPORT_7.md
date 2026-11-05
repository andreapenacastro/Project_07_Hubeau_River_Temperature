# Data-Quality Report — <project / dataset name>

**Group:** <names> · **Date:** <YYYY-MM-DD> · **Notebook:** `<path/to/notebook.ipynb>` · **Commit:** `<git short hash>`
**Clean-data fingerprint:** `<12-char hash printed at the end of the notebook>`

> Max **2 pages**. Write for a modelling teammate who has never seen your data. Every claim should point to a section, table or figure in the notebook.

---

## 1. Source & provenance
| | |
|---|---|
| Business question | |
| Source(s) & endpoint / URL | |
| Collection method (API / scraping / file) & date range | |
| Licence, robots.txt / ToS constraints | |
| Unit of observation (one row = …) | |
| Final size (rows × columns) | |

## 2. Schema (the contract)
Link to the pandera schema in the notebook. List the **key checks** and what each protects against:
- `<column>` — `<check>` — protects against `<defect>`
- …

Validation on raw data: **<n> violations** (summarise by column/check). Validation on clean data: **passes**.

## 3. Missingness
| Column | % missing | Mechanism (MCAR / MAR / MNAR) | Evidence | Decision |
|---|---|---|---|---|
| | | | | |

## 4. Duplicates & outliers — decision table
Paste the `decision_table` (one row per decision). Each row: column · issue · n rows · decision · why.

| Column | Issue | n rows | Decision | Why |
|---|---|---|---|---|
| | | | | |

## 5. Key findings (max 3)
1. **<finding>** — evidence: Fig. <n> — so what for modelling: <…>
2. …
3. …

## 6. What cannot be trusted ⚠️
*The most important section. Be specific: column, subset, and why.*
- …
- …

## 7. Hypotheses for modelling (≥ 5)
Each names a variable, a direction, a population and a test.
- **H1** — …
- **H2** — …
- **H3** — …
- **H4** — …
- **H5** — …

## 8. How to reproduce
```bash
python -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt
jupyter nbconvert --to notebook --execute <notebook.ipynb>
```
Expected fingerprint: `<hash>`
