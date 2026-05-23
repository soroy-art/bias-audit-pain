# ROUGE-L Explanation Similarity (Tier 1)

This analysis compares **Tier 1 model explanations** to the **ground-truth explanation** text using **ROUGE-L (F1)**.

## What this measures

- **Ground truth**: `data/data_post_op.csv`, column `Explanation`
- **Model output**: each model’s Tier 1 CSV under `experiment_results/<model_id>/`, using an `*_explanation` column (auto-detected)
- **Metric**: **ROUGE-L F1**, computed from the **token-level longest common subsequence (LCS)** between:
  - reference = ground-truth explanation
  - candidate = model explanation

ROUGE-L is in \([0, 1]\). Higher means the model explanation’s wording overlaps more with the reference (not necessarily “more correct”, just more similar).

## Matching logic (“same case”)

- Tier 1 experiment rows must include `vignette_idx`, `race`, and `gender`.
- Ground truth does not include demographics; it provides one explanation per vignette template.
- We match each experiment row to ground truth by `vignette_idx`.
  - If the ground-truth file does not have `vignette_idx`, the script assigns `vignette_idx = 0..N-1` by row order.

## How to run

From the `Q-Pain/` directory:

```bash
python analysis_scripts/analyze_rouge_l_explanations.py
```

The script automatically discovers model folders under `experiment_results/` and scores any model that has a Tier 1 CSV.

## Outputs

All outputs are written to:

`analysis_results/rouge_l_explanations/`

### Per-model

`analysis_results/rouge_l_explanations/<model_id>/rouge_l_scores_tier1_ff.csv`

Columns:
- `model_id`
- `vignette_idx`
- `race`
- `gender`
- `rouge_l_f1`
- `tier1_path`

### Combined + summary

- `analysis_results/rouge_l_explanations/rouge_l_scores_all_models_tier1_ff.csv`
- `analysis_results/rouge_l_explanations/rouge_l_summary_tier1_ff.csv`
  - `mean_rouge_l_f1`, `median_rouge_l_f1`
  - `ci_low`, `ci_high`: bootstrap 95% CI for the mean (5000 resamples)
- `analysis_results/rouge_l_explanations/rouge_l_mean_barplot_tier1_ff.png`

## Notes / caveats

- ROUGE-L rewards **lexical overlap**. It does not capture semantic equivalence well (synonyms, paraphrase).
- Explanations in your ground-truth CSV are templates without demographic personalization; ROUGE-L may penalize models that add clinically reasonable details not present in the reference.

