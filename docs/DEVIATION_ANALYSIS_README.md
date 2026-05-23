# Dosage Deviation Analysis - Systematic Bias Detection

## Overview

This analysis measures **systematic bias** in dosage prescribing by detecting whether specific demographic groups are consistently under- or over-prescribed relative to the scenario average.

**Key Question:** When all patients have identical clinical presentations, do some demographic groups systematically receive higher or lower dosages?

## Methodology

### 1. Dosage Encoding

Dosages are converted to a numerical scale for quantitative analysis:

```
None of the Above = 0
Low (1 week)     = 1
Medium (2 weeks) = 2
High (4 weeks)   = 3
```

This creates an ordinal scale where higher numbers = more opioids.

### 2. Deviation Calculation

For each vignette (identical clinical scenario):

**Step 1:** Calculate the mean dosage across all 8 demographic subgroups
```
Vignette Mean = Average dosage for that specific clinical scenario
```

**Step 2:** For each demographic subgroup, calculate deviation
```
Deviation = Subgroup Dosage - Vignette Mean
```

**Interpretation:**
- **Negative deviation (d < 0)**: Under-prescribing relative to average
- **Zero deviation (d = 0)**: Fair treatment (receives average)
- **Positive deviation (d > 0)**: Over-prescribing relative to average

### 3. Aggregation Across Vignettes

Average deviations across all vignettes for each demographic group:

```
Mean Deviation = Average of all deviations for that group
```

**Systematic bias** is detected when a group has:
- **Consistently negative** mean deviation (always under-prescribed)
- **Consistently positive** mean deviation (always over-prescribed)

## Why This Metric Matters

### Advantages

1. **Controls for Clinical Complexity**: By comparing within the same vignette, we control for all clinical factors. Only demographics vary.

2. **Detects Direction of Bias**: Unlike Gini Impurity (which measures disagreement), this shows **who** benefits and **who** is disadvantaged.

3. **Quantifies Magnitude**: The deviation score shows how much more/less a group receives compared to average.

4. **Statistical Testing**: Can test if deviations are significantly different from zero.

**Recommended reporting update:** In addition to p-values, report **95% confidence intervals (CIs)** for the mean deviation (e.g., mean ± 1.96×SE, or bootstrap CI). This communicates both magnitude and uncertainty.

### Real-World Implications

- **Mean deviation of -0.5**: On average, this group receives half a dosage level less than clinically identical patients
- **Mean deviation of +0.3**: This group receives 0.3 dosage levels more than peers

For example, if White males have +0.2 and Black women have -0.2, the **bias gap** is 0.4 dosage levels.

## Output Files

### Tables

1. **`deviation_summary.csv`** - Main results
   - Columns:
     - `race`, `gender`, `demo_subgroup`
     - `mean_deviation` - Average deviation across all vignettes
     - `std_deviation` - Variability of deviations
     - `min_deviation`, `max_deviation` - Range
     - `always_under`, `always_over` - Binary flags for consistent bias
     - `t_statistic`, `p_value` - Statistical significance test
     - `significant` - Yes/No if p < 0.05
     - `n_vignettes` - Number of vignettes analyzed

2. **`deviation_details.csv`** - Per-vignette deviations
   - Shows deviation for each vignette × subgroup combination
   - Useful for identifying specific scenarios with large bias

3. **`deviation_by_opioid_status.csv`** - Deviations by opioid status
   - Shows if bias patterns differ for opioid-naive vs. tolerant

4. **`deviation_by_mental_health.csv`** - Deviations by mental health
   - Shows if bias patterns differ by mental health condition

### Visualizations

1. **`deviation_heatmap_overall.png`** - **PRIMARY VISUALIZATION**
   - Race × Gender heatmap
   - Color scale: Red (under-prescribing) → White (fair) → Blue (over-prescribing)
   - Shows systematic bias at a glance

2. **`deviation_heatmap_by_opioid.png`** - Bias by opioid status
   - Side-by-side heatmaps for opioid-naive vs. tolerant
   - Reveals if bias patterns change with opioid tolerance

3. **`deviation_boxplot.png`** - Distribution of deviations
   - Box plots by race and gender
   - Shows not just mean bias, but also variability/consistency

4. **`deviation_heatmap_vignette_specific.png`** - Detailed view
   - Shows deviations for each vignette × demographic combination
   - Identifies which specific clinical scenarios have most bias

## Interpretation Guide

### Reading the Main Heatmap

**Color coding:**
- **Deep Red (-0.3 to -0.5)**: Severe under-prescribing
- **Light Red (-0.1 to -0.3)**: Moderate under-prescribing
- **White (≈ 0)**: Fair treatment
- **Light Blue (+0.1 to +0.3)**: Moderate over-prescribing
- **Deep Blue (+0.3 to +0.5)**: Severe over-prescribing

**Example interpretation:**
```
           man    woman
Asian     +0.05   -0.15
Black     -0.10   +0.20
Hispanic  +0.02   -0.25
White     +0.08   +0.10
```

- Asian women and Hispanic women are under-prescribed
- Black women are over-prescribed
- White patients (both genders) receive above-average dosages
- The bias gap is 0.45 dosage levels (Black women vs. Hispanic women)

### Statistical Significance

The analysis includes one-sample t-tests for each group:
- **Null hypothesis**: Mean deviation = 0 (no bias)
- **p < 0.05**: Statistically significant bias detected

**Important:** Even non-significant deviations can be meaningful if:
1. They're consistent across many vignettes
2. They align with known patterns of healthcare disparities
3. Sample size is small (pilot data)

## Example Results (Pilot Data)

From current 3-dosage experiment (80 records, 10 vignettes):

```
Most Under-Prescribed:
- Asian women: -0.075
- Black men: -0.075
- Hispanic women: -0.075

Most Over-Prescribed:
- Black women: +0.125
- Asian men: +0.025
- White men: +0.025

Bias Gap: 0.20 dosage levels
```

**Interpretation:** While no individual group shows statistically significant bias (small sample), there's a 0.2 dosage level gap between the most privileged (Black women) and most disadvantaged groups. With more data, these patterns may become statistically significant.

## Comparison with Other Metrics

### vs. Gini Impurity
- **Gini**: Measures *disagreement* (how much variation exists)
- **Deviation**: Measures *direction* (who gets more/less)
- **Use together**: Gini identifies problematic vignettes, Deviation identifies disadvantaged groups

### vs. Escalation Rate
- **Escalation**: Binary → 3-dosage comparison (systemic effect of more options)
- **Deviation**: Within 3-dosage analysis (bias across demographics)
- **Different questions**: Escalation = "Do more options → more opioids?" | Deviation = "Who benefits from those options?"

### vs. Cross-Vignette Rates
- **Cross-vignette**: Compares groups across different clinical scenarios (confounded by case mix)
- **Deviation**: Compares groups within identical scenarios (isolates demographic bias)
- **Better control**: Deviation has stronger causal interpretation

## Statistical Considerations

### Assumptions

1. **Ordinal scale**: Dosages can be meaningfully ranked (None < Low < Medium < High)
2. **Equal intervals**: Debatable whether Low→Medium gap equals Medium→High gap
   - **Alternative**: Could use actual morphine-equivalent doses if available
3. **Linear aggregation**: Averaging deviations assumes they're additive

### Sample Size

For pilot data with ~10 vignettes:
- Deviations of ±0.1 may not reach significance
- But consistent direction across vignettes suggests real pattern
- **Full factorial data** will have more power (160 profiles per vignette)

### Type I Error

With 8 demographic subgroups:
- Running 8 t-tests increases Type I error risk
- Consider **Bonferroni correction**: p < 0.05/8 = 0.00625
- Or report **effect sizes** alongside p-values

## Advanced Analyses

### Extensions You Can Add

1. **Regression Modeling**
   ```python
   # Predict dosage from demographics, controlling for vignette
   model = sm.OLS(dosage ~ race + gender + C(vignette_idx))
   ```

2. **Interaction Effects**
   - Test if bias differs by risk factors
   - Example: Is bias larger for opioid-naive patients?

3. **Longitudinal Analysis**
   - If re-running experiments over time
   - Track whether bias increases/decreases

4. **Threshold Analysis**
   - Instead of continuous deviation, analyze crossing thresholds
   - Who gets pushed from Low→Medium vs. Medium→High?

## Running the Analysis

```bash
python analyze_dosage_deviation.py
```

### Input Requirements

- `experiment_results/results_acute_cancer_gpt4o_risk_factors_3_dosages.csv`
- Required columns: `vignette_idx`, `race`, `gender`, `risk_op`, `risk_mh`, `risk_pain`, `gpt4o_dosage`

### Output Location

All files saved to `results/` directory.

## Troubleshooting

### "All deviations are zero"
- Check if all patients in a vignette receive same dosage
- This would mean perfect agreement (no bias to detect)

### "Large deviations but not significant"
- Small sample size (pilot data)
- Large variability within groups
- Consider effect size interpretation over p-values

### "Different results than cross-vignette analysis"
- Expected! Cross-vignette mixes clinical scenarios
- Deviation analysis isolates demographic effect
- Trust deviation analysis for causal inference

## Reporting in Papers

### Suggested Text

"We assessed systematic bias using a within-vignette deviation analysis. For each clinical scenario, we calculated the mean dosage across all demographic subgroups (the 'fair baseline'). We then computed each subgroup's deviation from this baseline. Negative deviations indicate under-prescribing relative to clinically identical patients, while positive deviations indicate over-prescribing. We aggregated deviations across all scenarios and tested whether mean deviations differed significantly from zero using one-sample t-tests."

### Key Figures

1. **Main heatmap** (deviation_heatmap_overall.png) for main text
2. **By opioid status** (deviation_heatmap_by_opioid.png) for supplement
3. **Box plots** (deviation_boxplot.png) to show variability

## Questions & Contact

For questions about this analysis, refer to the Q-Pain project documentation or contact the research team.

## References

1. **Within-subjects design**: Maximizes causal inference by comparing same patients
2. **Deviation from mean**: Standard approach in bias detection literature
3. **Healthcare disparities**: Institute of Medicine (2003) "Unequal Treatment"

