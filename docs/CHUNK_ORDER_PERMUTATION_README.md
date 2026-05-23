# Retrieved Chunk Order Permutation (Tier 1)

## Overview

Tests whether **reordering the same 10 retrieved guideline chunks** changes model responses. Baseline (original retrieval order) stays in the existing Tier 1 retrieved CSV; this pipeline runs **two new orders** per row:

| `permutation_id` | Definition |
|------------------|------------|
| `reverse` | Reverse the 10 chunks for that vignette |
| `random` | Seeded shuffle per `vignette_idx` (same order for all 8 race×gender rows in that vignette) |

**API calls (per retrieved model):** 80 rows × 2 permutations = **160** (baseline not re-run).

## Scripts

| Script | Role |
|--------|------|
| `analysis_scripts/retrieved_chunk_order.py` | Parse / permute / reassemble `open_prompts` |
| `analysis_scripts/run_retrieved_chunk_order_tier1.py` | Run API calls for all `*_retrieved` models |
| `analysis_scripts/analyze_chunk_order_rouge_l.py` | ROUGE-L vs baseline explanations |

## Run experiment

From Q-Pain root (requires `OPENAI_API_KEY` in `.env`):

```bash
# Preview parsing (no API)
python analysis_scripts/run_retrieved_chunk_order_tier1.py --dry-run

# One model
python analysis_scripts/run_retrieved_chunk_order_tier1.py --models gpt54_retrieved

# All retrieved folders (gpt54_retrieved, gpt4o_mini_retrieved, gpt41_mini_retrieved)
python analysis_scripts/run_retrieved_chunk_order_tier1.py

# Pilot: first 2 rows only
python analysis_scripts/run_retrieved_chunk_order_tier1.py --limit 2
```

### Outputs

Per model:

```text
experiment_results/<model>_retrieved/chunk_order_tier1/results_tier1_chunk_order_permutations_ff.csv
```

Key columns: `permutation_id`, `baseline_chunk_order`, `chunk_order`, `open_prompts`, plus the same `*_answer` / `*_dosage` / `*_explanation` / `*_full` pattern as the source CSV.

Resume is on by default (skips rows already present). Use `--no-resume` to start fresh.

### Model IDs

| Folder | Default OpenAI model |
|--------|----------------------|
| `gpt54_retrieved` | `OPENAI_MODEL` env or `gpt-5.4` |
| `gpt4o_mini_retrieved` | `gpt-4o-mini` |
| `gpt41_mini_retrieved` | `gpt-4.1-mini` |

## ROUGE-L analysis

After permutation runs finish:

```bash
python analysis_scripts/analyze_chunk_order_rouge_l.py
```

Writes:

```text
analysis_results/chunk_order_rouge_l/<model>_retrieved/
  chunk_order_rouge_l_scores_ff.csv
  chunk_order_rouge_l_summary_ff.csv
```

Compares each permuted explanation to the **baseline Tier 1** explanation for the same `(vignette_idx, race, gender)`, with name/pronoun standardization from `analyze_rouge_l_explanations.py`.

## Visualizations

After ROUGE scores exist under `analysis_results/chunk_order_rouge_l/`:

```bash
python analysis_scripts/visualize_chunk_order_results.py
```

Writes **plots** to `analysis_results/chunk_order_rouge_l/plots/`:

| Figure | What it shows |
|--------|----------------|
| `chunk_order_mean_rouge_bar_ff.png` | Mean ROUGE-L vs original order (reverse / random) by model |
| `chunk_order_rouge_violin_ff.png` | Full distribution of row-level ROUGE-L (80 prompts per model×perm) |
| `chunk_order_material_rouge_drop_ff.png` | % prompts with ROUGE-L below 0.5 / 0.7 / 0.9 |
| `chunk_order_vignette_rouge_heatmap_ff.png` | Per clinical vignette (10), mean ROUGE over 8 demographics |
| `chunk_order_reverse_vs_random_scatter_ff.png` | Vignette-level reverse vs random (both vs baseline) |

**Tables:** `chunk_order_stability_summary_ff.csv`, `chunk_order_vignette_rouge_ff.csv`

If `experiment_results/<model>/chunk_order_tier1/results_tier1_chunk_order_permutations_ff.csv` is present, also produces dosage/Yes agreement barplots (`chunk_order_decision_agreement_bar_ff.png`, `chunk_order_vignette_dosage_match_ff.png`).

**Interpretation:** ROUGE-L compares permuted explanations to the **original-order** Tier 1 run (not permuted-vs-permuted). Low mean ROUGE (~0.43–0.59) and ~26–56% of rows below 0.5 suggest chunk **order alone** often changes wording materially; if dosage agreement plots are added and show flips, that supports **positional** rather than purely evidentiary reasoning.

## Design notes

- Only **retrieved** Tier 1 CSVs (non-retrieved has no RAG block).
- Chunk order is **per vignette**; all 8 demographic variants for that vignette share the same permutation.
- `random` uses `random_seed + vignette_idx` (default seed **42**).
- Original-order prompts round-trip exactly through parse → reassemble (validated on 80 rows).
