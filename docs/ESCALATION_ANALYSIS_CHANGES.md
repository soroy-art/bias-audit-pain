# Escalation / De-escalation Analysis Script Updates

## Summary of Changes

The `analyze_dosage_escalation.py` script compares Tier~2 (binary choice: Low/High) against Tier~3 (3-choice: Low/Medium/High) on *matched* factorial patient profiles. It now produces **both**:

- **Escalation**: Tier~2 Low \(\rightarrow\) Tier~3 Medium/High  
- **De-escalation**: Tier~2 High \(\rightarrow\) Tier~3 Medium/Low  

## Key Changes

### 1. **Two-File Input System**

**Previous:** Single CSV input (3-dosage only)
```python
csv_path = "results/results_acute_cancer_gpt4o_risk_factors_3_dosages.csv"
df = load_and_prepare_data(csv_path)
```

**Updated:** Compares baseline vs. 3-dosage experiments
```python
baseline_path = "experiment_results/results_acute_cancer_gpt4o_risk_factors.csv"
three_dosage_path = "experiment_results/results_acute_cancer_gpt4o_risk_factors_3_dosages.csv"
df = load_and_prepare_data(baseline_path, three_dosage_path)
```

### 2. **Escalation Definition**

**Escalation** is now properly defined as:
- **Baseline experiment**: Low dosage was chosen
- **3-dosage experiment**: Medium or High dosage was chosen

The script verifies that baseline dosages are "Low" and warns/excludes any records that don't meet this criterion.

### 2b. **De-escalation Definition (NEW)**

**De-escalation** is defined as:
- **Baseline experiment (Tier 2)**: High dosage was chosen
- **3-dosage experiment (Tier 3)**: Medium or Low dosage was chosen

This is computed on the same matched patient profiles, but with an **eligibility filter of baseline=High** rather than baseline=Low.

### 3. **Matching Strategy**

Records are matched between experiments based on:
- `vignette_idx` (same clinical scenario)
- `race` (patient demographics)
- `gender` (patient demographics)
- `risk_op` (opioid status: naive/tolerant)
- `risk_mh` (mental health history)
- `risk_pain` (preoperative pain status)

**Important:** This requires both experiments to have the **same patient profiles**. The script will verify matches exist and provide helpful error messages if not.

**For Full Factorial Experiments:**
- Run both baseline and 3-dosage with identical combinations of all factors
- Example: 4 races × 2 genders × 5 mental_health × 2 opioid_status × 2 pain = 160 profiles per vignette
- This ensures we compare identical patients, with only the dosage choice options differing

### 4. **Removed Probability-Based Escalation**

The probability-based escalation metric (`prob_escalation = prob_medium + prob_high`) has been removed as requested. The analysis now focuses solely on the binary escalation flag.

### 5. **Automatic Directory Creation**

The script now automatically creates the `results/` directory if it doesn't exist, preventing file save errors.

## Data Flow

```
1. Load baseline CSV (binary choice: Low/High)
   ↓
2. Load 3-dosage CSV (three choices: Low/Medium/High)
   ↓
3. Create match keys: vignette_idx + race + gender + risk factors
   ↓
4. Merge datasets on match keys
   ↓
5. Verify matches exist (exit with helpful error if not)
   ↓
6a. Escalation branch: Filter baseline=Low, then flag Tier 3 Medium/High
   ↓
6b. De-escalation branch: Filter baseline=High, then flag Tier 3 Medium/Low
   ↓
7. Analyze & visualize both metrics (parallel outputs)
```

## Output Structure

All results are saved under `analysis_results/escalation_rate/<model_id>/`:

### Cross-Vignette Analysis
- **Tables:**
  - Escalation:
    - `escalation_table_overall_ff.csv`
    - `escalation_table_opioid_naive_ff.csv`
    - `escalation_table_opioid_tolerant_ff.csv`
    - `escalation_pvalues_*.csv`
    - `escalation_eligibility_summary_ff.csv` (eligible = baseline Low; includes matched total and analysis denominator)
  - De-escalation (NEW):
    - `deescalation_table_overall_ff.csv`
    - `deescalation_table_opioid_naive_ff.csv`
    - `deescalation_table_opioid_tolerant_ff.csv`
    - `deescalation_pvalues_*.csv`
    - `deescalation_eligibility_summary_ff.csv` (eligible = baseline High; includes matched total and analysis denominator)

- **Visualizations:**
  - Escalation:
    - `escalation_barplot_faceted_ff.png`
    - `escalation_heatmap_*.png`
    - `escalation_pvalues_heatmap_*.png`
  - De-escalation (NEW):
    - `deescalation_barplot_faceted_ff.png`
    - `deescalation_heatmap_*.png`
    - `deescalation_pvalues_heatmap_*.png`

## Statistical Significance & Confidence Intervals (Recommended Reporting)

This project produces several fairness-related metrics with different data types:

- **Escalation / De-escalation**: binary outcome per matched profile (0/1)
- **Gini Impurity (GI)**: continuous per vignette (often zero-inflated)
- **KL divergence**: continuous summary per subgroup + a global dosage×subgroup contingency test
- **Confidence trajectory**: continuous confidence values across tiers (paired comparisons)

To support claims like “increases/decreases,” we recommend reporting **effect sizes with 95\% confidence intervals (CIs)** and using **global hypothesis tests** sparingly.

### Escalation / De-escalation

- **Subgroup rates** should be reported with **95\% Wilson CIs** (binomial proportion interval).
- **Pairwise subgroup comparisons** (if needed) should use two-proportion tests and apply **multiple-comparisons correction** (Benjamini–Hochberg FDR).
- A **global association test** (subgroup × outcome) can be reported as a chi-squared test with **Cramer’s V** as effect size. Note: for extremely sparse tables (e.g., \(n=1\) in a stratum), chi-squared may be undefined; the script records this as NA.

Outputs:
- `*_table_*.csv` now include `CI Low (%)` and `CI High (%)` columns and `Successes`.
- `*_pvalues_raw_*.csv` contains raw two-proportion p-values; `*_pvalues_*.csv` contains FDR-adjusted p-values.
- `*_global_significance_ff.csv` records chi-squared p-values and Cramer’s V for All / by opioid status.

### Confidence trajectory (Tier 1→2→3)

- Use **paired bootstrap** CIs for mean confidence changes and **paired permutation tests** for p-values of mean deltas.
- Output: `confidence_trajectory_summary_ff.csv` per model (means, 95\% CIs, and permutation p-values for \(\Delta_{12}\), \(\Delta_{23}\)).

### Gini Impurity (GI)

- Report percent of vignettes with **GI=0** (perfect agreement) and thresholds (e.g., GI>0.25) with **binomial 95\% Wilson CIs**.
- These CIs are added to the `gini_impurity_summary_*` CSVs.

### KL divergence

- Prefer the **global chi-squared test** of dosage category × demographic subgroup, reported in `kl_summary_ff.csv` (p-value + Cramer’s V).
- Optional: `kl_summary_ff.csv` now includes a bootstrap CI for the **max KL** across subgroups (descriptive uncertainty).

### Within-Vignette Analysis
Examines whether patients with identical clinical presentations receive different dosages based on demographics.

- **Tables:**
  - `within_vignette_summary.csv` - Summary statistics
  - `within_vignette_details.csv` - Detailed variance by vignette
  - `within_vignette_consistency_tests.csv` - Chi-square test results

- **Visualizations:**
  - `within_vignette_heatmap.png` - Escalation rates by vignette × demographics
  - `within_vignette_variance_plot.png` - Disagreement scores

## Scalability for Future Experiments

The updated script is designed to scale for larger experiments:

1. **Automatic validation**: Verifies baseline dosages are "Low" before calculating escalation
2. **Flexible matching**: Works with different risk factor combinations between experiments
3. **Warning system**: Alerts if unexpected dosages are found in baseline
4. **Robust merging**: Handles missing matches gracefully

## Usage

```bash
python analyze_dosage_escalation.py
```

The script discovers model folders under `experiment_results/` and expects Tier files:
- Tier 2 (baseline): `*tier2*_ff*.csv`
- Tier 3 (three-dosage): `*tier3*_ff*.csv`

## Current Status with Pilot Data

The current pilot datasets use **randomized risk factors** independently in each experiment, so they have **0 matching records**. This is expected behavior.

When you run the script with the current pilot data, it will:
1. Detect no matches
2. Display a helpful error message explaining the issue
3. Exit gracefully

## For Full Factorial Experiments

Once you run both experiments with the same patient profiles:
- The script will successfully match records
- Calculate escalation rates for truly identical patients
- Generate all tables and visualizations showing demographic disparities

## Notes

- The script uses risk factors from the **3-dosage experiment** as the authoritative source
- Records are excluded if baseline dosage is not "Low"
- The matching ensures apples-to-apples comparison between binary and 3-level dosage choices

