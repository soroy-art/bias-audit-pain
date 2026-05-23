# Backfill Experiment Dosage from Answer

## Overview

`analysis_scripts/backfill_experiment_dosage_from_answer.py` repairs experiment result CSVs where the **dosage column is empty** but the model’s decision is still present in the **answer** or **full response** text.

This situation appeared with some GPT‑5.x runs: the notebook parser expected a separate `Dosage:` line, but the model sometimes returned a single compact line such as:

```text
Yes. Low (1 week) supply.
```

Those rows leave `*_dosage` blank while `*_answer` contains the dose. Downstream scripts (Gini impurity heatmaps, escalation, KL divergence) then treat the row as **Other** or show **`?`** on plots.

**This script does not call the API** — it only edits existing CSVs on disk.

---

## When to use it

Run backfill when you see:

- Empty or `NaN` values in `gpt54_dosage` / `gpt4o_dosage` (or other `*_dosage` columns)
- Answer text that embeds the dose, e.g. `Yes — Low (1 week) supply.`
- Gini or other analyses mapping missing dosage to **Other** / **?**

Typical targets:

- `experiment_results/gpt54/`
- `experiment_results/gpt54_retrieved/`

You can point `--dirs` at any folder that contains tier result CSVs with matching `*_answer` / `*_dosage` / `*_full` columns.

---

## How it works

For each CSV, the script:

1. Detects column triples by prefix, e.g. `gpt54_answer`, `gpt54_dosage`, `gpt54_full` (prefers `gpt54_dosage` or `gpt4o_dosage` if multiple `*_dosage` columns exist).
2. Finds rows where `*_dosage` is empty.
3. For each such row, tries to extract a dosage string in order:
   - **Line parse:** first `Dosage: ...` line in `*_full` or `*_answer`
   - **Compact parse:** regex on answer/full for patterns such as:
     - `Low (1 week)`, `High (4 weeks)`, `Medium (2 weeks)`, `None of the above`
     - `Low (0.5 mg)`, `High (1 mg)` (Tier 1–style wording if present)
4. Writes the extracted value into `*_dosage` only for rows that were empty and where extraction succeeded.

Rows that still cannot be parsed are left empty (reported in the console summary).

---

## Usage

From the Q-Pain project root:

```bash
# Default: gpt54 + gpt54_retrieved
python analysis_scripts/backfill_experiment_dosage_from_answer.py

# Preview changes without writing
python analysis_scripts/backfill_experiment_dosage_from_answer.py --dry-run

# Custom directories
python analysis_scripts/backfill_experiment_dosage_from_answer.py \
  --dirs experiment_results/gpt54 experiment_results/gpt54_retrieved

# Limit which files (glob within each dir)
python analysis_scripts/backfill_experiment_dosage_from_answer.py \
  --dirs experiment_results/gpt54 \
  --glob "*tier2*ff*.csv"
```

### CLI options

| Option | Default | Description |
|--------|---------|-------------|
| `--dirs` | `experiment_results/gpt54`, `experiment_results/gpt54_retrieved` | One or more directories to scan |
| `--glob` | `*.csv` | Filename pattern within each directory |
| `--dry-run` | off | Print counts only; do not modify files |

---

## Backups and safety

When not using `--dry-run`, each modified CSV gets a **timestamped backup** beside the original:

```text
results_post_op_gpt54_mini_tier2_ff.csv.bak.20260515T054522Z
```

Restore from `.bak.*` if you need the pre-patch file.

---

## Example output

```text
experiment_results/gpt54/results_post_op_gpt54_mini_tier2_ff.csv: rows with empty dosage=22, backfilled=22
experiment_results/gpt54_retrieved/results_retr_evidence_post_op_gpt5_4_tier3_ff_Apr10.csv: rows with empty dosage=59, backfilled=59

Total: empty dosage rows seen=350, backfilled=350
```

---

## Recommended pipeline order

For GPT‑5.4 result cleanup, run scripts in this order:

1. **`backfill_experiment_dosage_from_answer.py`** — fill missing `*_dosage` from answer/full text  
2. **`normalize_gpt54_answer_dosage.py`** — canonical `Yes`/`No` and `Low`/`Medium`/`High` labels (see `NORMALIZE_GPT54_ANSWER_DOSAGE_README.md`)

Then re-run analysis scripts (`analyze_gini_impurity.py`, `analyze_kl_divergence.py`, etc.) so plots use the updated CSVs.

---

## Limitations

- Only **empty** dosage cells are updated; non-empty values are untouched.
- Extraction is **rule-based**; unusual phrasing may still fail to backfill.
- Does not fix answer/dosage **format inconsistency** (e.g. `Yes. Low (1 week) supply.` in the answer column while dosage is already filled with a long string). Use the normalize script for label cleanup.
- Skips `.csv` files that lack a recognizable `*_dosage` / `*_answer` pair.

---

## Related files

| File | Role |
|------|------|
| `analysis_scripts/backfill_experiment_dosage_from_answer.py` | This utility |
| `analysis_scripts/normalize_gpt54_answer_dosage.py` | Canonical answer/dosage labels (GPT‑5.4 folders) |
| `docs/NORMALIZE_GPT54_ANSWER_DOSAGE_README.md` | Normalize script documentation |
| `docs/GINI_IMPURITY_README.md` | Analysis that depends on clean dosage categories |
