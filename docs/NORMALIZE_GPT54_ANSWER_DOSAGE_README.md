# Normalize GPT‑5.4 Answer and Dosage Columns

## Overview

`analysis_scripts/normalize_gpt54_answer_dosage.py` standardizes **answer** and **dosage** columns in GPT‑5.4 experiment CSVs so downstream analyses see **consistent categorical labels** instead of free-text variants.

Default directories:

- `experiment_results/gpt54/`
- `experiment_results/gpt54_retrieved/`

**This script does not call the API** — it only rewrites CSVs on disk.

---

## Why this exists

After experiments, GPT‑5.4 outputs often mixed formats in the same column, for example:

| Column | Examples before normalize |
|--------|---------------------------|
| `*_answer` | `Yes.`, `Yes`, `Yes. Low (1 week) supply.`, `Yes — Low (1 week) supply.`, `**Answer: Yes**` |
| `*_dosage` | `Low (1 week)`, `Low (1 week) supply.`, `Low (1 week).`, `**Dosage: Low (1 week)**`, `High (4 weeks) supply.` |

Scripts such as `analyze_gini_impurity.py` map dosage strings to categories with simple substring rules (`low`, `medium`, `high`). Variants like `Low (1 week) supply.` can fail to match cleanly and appear as **Other** → **`?`** on heatmaps even when the clinical choice was obvious.

Normalization collapses labels to a small fixed vocabulary analysis code expects.

---

## Normalization rules

### `*_answer` → `Yes` or `No`

Parsing order (using `*_answer` and, when present, `*_full`):

1. Explicit line: `Answer: Yes` / `Answer: No` (including markdown bold variants)
2. Leading token: `Yes.` / `No —` style at start of text
3. First standalone `yes` / `no` word in the blob

If nothing matches, the cell is set to an empty string.

### `*_dosage` → `Low`, `Medium`, or `High`

The script inspects `*_dosage`, `*_answer`, and `*_full` (lowercased, concatenated) and assigns:

| Detected keyword | Canonical value |
|------------------|-----------------|
| `medium` | `Medium` |
| `high` | `High` |
| `low` | `Low` |

Priority is **Medium → High → Low** (first match wins).

- **None / N/A** and other non-dose answers → blank dosage (empty string)
- Tier 1 mg-style wording (`Low (0.5 mg)`) still maps to **`Low`** / **`High`** via the `low` / `high` keywords

**Note:** Canonical dosage is intentionally **short** (`Low`, not `Low (1 week)`). Tier context (1 week vs 4 weeks vs 2 weeks) is implied by which tier CSV you are analyzing.

---

## When to use it

Run after backfill (if you used it) and **before** re-running analysis:

```bash
python analysis_scripts/backfill_experiment_dosage_from_answer.py   # optional, if dosage was empty
python analysis_scripts/normalize_gpt54_answer_dosage.py
python analysis_scripts/analyze_gini_impurity.py
python analysis_scripts/analyze_kl_divergence.py
# ... other analyses
```

Use normalize when:

- Gini/KL/escalation plots still show **`?`** despite non-empty dosage cells
- You want `*_answer` limited to **`Yes`** / **`No`** only
- You want `*_dosage` limited to **`Low`** / **`Medium`** / **`High`** only

---

## Usage

From the Q-Pain project root:

```bash
# Default: gpt54 + gpt54_retrieved, all *.csv in each folder
python analysis_scripts/normalize_gpt54_answer_dosage.py

# Preview only
python analysis_scripts/normalize_gpt54_answer_dosage.py --dry-run

# Single directory
python analysis_scripts/normalize_gpt54_answer_dosage.py \
  --dirs experiment_results/gpt54

# Specific files
python analysis_scripts/normalize_gpt54_answer_dosage.py \
  --dirs experiment_results/gpt54 \
  --glob "*tier1*ff*.csv"
```

### CLI options

| Option | Default | Description |
|--------|---------|-------------|
| `--dirs` | `experiment_results/gpt54`, `experiment_results/gpt54_retrieved` | Directories to process |
| `--glob` | `*.csv` | Filename pattern per directory |
| `--dry-run` | off | Report change counts without writing |

---

## Column detection

The script infers the model prefix from the dosage column name:

- `gpt54_dosage` → updates `gpt54_answer`, `gpt54_dosage` (and uses `gpt54_full` if present)
- `gpt4o_dosage` → updates `gpt4o_answer`, `gpt4o_dosage` (retrieved-evidence CSVs)

If multiple `*_dosage` columns exist, `gpt54_dosage` is preferred over `gpt4o_dosage`.

---

## Backups and safety

Each modified file gets a timestamped backup:

```text
results_post_op_gpt54_mini_tier1_ff.csv.bak.20260515T120000Z
```

The script rewrites **every row** in matched CSVs when answer or dosage values change (not only rows that were empty).

---

## Example output

```text
experiment_results/gpt54/results_post_op_gpt54_mini_tier1_ff.csv: rows=80, answer_changed=78, dosage_changed=80
experiment_results/gpt54/results_post_op_gpt54_mini_tier3_ff.csv: rows=1600, answer_changed=878, dosage_changed=1600

Processed files=6, rows=6560, answer_changed=5777, dosage_changed=6560
```

After normalization, typical uniques are:

- **Tier 1 / 2:** `gpt54_answer` ∈ {`Yes`} (or `Yes`/`No` if No responses exist); `gpt54_dosage` ∈ {`Low`, `High`}
- **Tier 3:** `gpt54_dosage` ∈ {`Low`, `Medium`, `High`}

---

## Interaction with other utilities

| Step | Script | Purpose |
|------|--------|---------|
| 1 (optional) | `backfill_experiment_dosage_from_answer.py` | Fill **empty** `*_dosage` from answer/full text |
| 2 | `normalize_gpt54_answer_dosage.py` | **Canonical** Yes/No and Low/Medium/High |
| 3 | Analysis scripts | Gini, KL, escalation, ROUGE-L, etc. |

See `docs/BACKFILL_EXPERIMENT_DOSAGE_README.md` for backfill details.

**ROUGE-L** (`analyze_rouge_l_explanations.py`) uses explanation text, not these columns; it is unaffected by normalize.

---

## Limitations

- **GPT‑5.4 folders only by default** — other model directories are not processed unless you pass `--dirs`.
- **Overwrites** answer/dosage strings; original wording is only preserved in `*_full` / `*_explanation` if those columns exist and are not modified by this script.
- **Blank dosage** for true “none of the above” or unparseable rows — verify Tier 3 “None” rates if that category matters for your analysis.
- Does not re-parse or fix `*_full` or `*_explanation` columns.

---

## Related files

| File | Role |
|------|------|
| `analysis_scripts/normalize_gpt54_answer_dosage.py` | This utility |
| `analysis_scripts/backfill_experiment_dosage_from_answer.py` | Fill missing dosage before normalize |
| `docs/BACKFILL_EXPERIMENT_DOSAGE_README.md` | Backfill documentation |
| `docs/GINI_IMPURITY_README.md` | Primary consumer of clean dosage categories |
| `docs/KL_DIVERGENCE_README.md` | Tier 3 KL analysis (`*_dosage` detection) |
