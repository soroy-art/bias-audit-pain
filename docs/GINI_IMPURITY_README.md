# Gini Impurity Analysis - Within-Vignette Dosage Agreement

## Overview

This analysis measures dosage decision **consistency** within vignettes using **Gini Impurity**, a metric from decision tree theory that quantifies the "purity" of decisions.

**Key Concept:** Given an identical clinical vignette and risk profile, dosage decisions should be uniform across race and gender. Gini Impurity measures deviation from this ideal.

## What is Gini Impurity?

### Formula

```
Gini Impurity (GI) = 1 - Σ(p_i²)
```

Where:
- **k** = number of dosage categories
  - **Tier 2**: 2 categories (Low, High)
  - **Tier 3**: 4 categories (None, Low, Medium, High)
- **p_i** = fraction of subgroups (out of 8 race×gender combinations) that received dosage i

### Interpretation

- **GI = 0**: Perfect agreement (all 8 subgroups chose the same dosage)
- **Maximum GI**: 
  - **Tier 2**: 0.50 (uniform distribution across 2 categories)
  - **Tier 3**: 0.75 (uniform distribution across 4 categories)
- **Higher GI** = More disagreement/inconsistency in dosage decisions

### Thresholds

| Gini Impurity | Interpretation | Meaning |
|--------------|----------------|---------|
| 0.00 | Perfect Agreement | All demographic groups received same dosage |
| 0.01 - 0.25 | Low Disagreement | Mostly consistent with minor variations |
| 0.26 - 0.50 | Moderate Disagreement | Substantial variation across groups |
| > 0.50 | High Disagreement | Large disparities in dosage decisions (Tier 3 only) |

## Why Gini Impurity?

### Advantages over Standard Deviation

1. **Designed for Categorical Data**: Gini Impurity is specifically designed for categorical outcomes, whereas standard deviation assumes continuous numeric data.

2. **Intuitive Interpretation**: 
   - GI = 0 clearly means "perfect agreement"
   - No assumption about the "distance" between dosage levels

3. **Bounded Scale**: Always between 0 and max theoretical (0.50 for Tier 2, 0.75 for Tier 3), making comparisons easier.

4. **Information Theory Foundation**: Rooted in entropy and decision tree theory, widely used in ML for measuring decision purity.

## Tier Comparison

The analysis now supports both Tier 2 and Tier 3 experiments:

### Tier 2 (Binary Dosage Choice)
- **Dosage Options**: Low (1 week), High (4 weeks)
- **Max Theoretical GI**: 0.50
- **Focus**: Does the model show demographic bias in the binary prescribe/don't-escalate decision?

### Tier 3 (Granular Dosage Choice)
- **Dosage Options**: None, Low (1 week), Medium (2 weeks), High (4 weeks)
- **Max Theoretical GI**: 0.75
- **Focus**: With more options, does the model introduce additional demographic disparities?

## Usage

### Running the Analysis

```bash
python analyze_gini_impurity.py
```

The script will analyze both Tier 2 and Tier 3 automatically.

### Input

**Tier 2:**
- `experiment_results/results_post_op_gpt4o_mini_tier2_ff.csv`

**Tier 3:**
- `experiment_results/results_post_op_gpt4o_mini_tier3_ff.csv`

Required columns:
- `vignette_idx`
- `race`, `gender`
- `risk_op`, `risk_mh`, `risk_pain`
- `gpt4o_dosage`

### Output Files

**For each tier, the following files are generated:**

**Tables:**
- `analysis_results/gini_impurity/gini_impurity_summary_tier2_ff.csv` - Overall statistics (Tier 2) including **95% Wilson CIs** for the proportions in each GI bin
- `analysis_results/gini_impurity/gini_impurity_details_tier2_ff.csv` - Per-vignette Gini scores (Tier 2)
- `analysis_results/gini_impurity/gini_impurity_summary_tier3_ff.csv` - Overall statistics (Tier 3) including **95% Wilson CIs** for the proportions in each GI bin
- `analysis_results/gini_impurity/gini_impurity_details_tier3_ff.csv` - Per-vignette Gini scores (Tier 3)

**Visualizations:**
1. `analysis_results/gini_impurity/gini_impurity_barplot_tier{X}_ff.png` - Bar chart of GI per vignette (color-coded)
2. `analysis_results/gini_impurity/gini_impurity_heatmap_by_risk_tier{X}_ff.png` - GI by opioid status × mental health
3. `analysis_results/gini_impurity/gini_dosage_distribution_heatmap_tier{X}_ff.png` - Actual dosages by vignette × demographics (only shows vignettes with disagreement)
4. `analysis_results/gini_impurity/gini_impurity_violin_tier{X}_ff.png` - GI distribution by opioid status

## Visualization Guide

### 1. Gini Impurity Bar Plot
**Best for:** Identifying which specific vignettes have disagreement

**Features:**
- Each bar = one vignette
- Color-coded by disagreement level (green=agreement, red=disagreement)
- Value labels show exact Gini score
- Reference lines for thresholds

**Use Case:** Quickly spot problematic vignettes with high disagreement

### 2. Gini Heatmap by Risk Factors
**Best for:** Understanding which risk factor combinations lead to more disagreement

**Features:**
- Rows = mental health conditions
- Columns = opioid status
- Color intensity = mean Gini Impurity

**Use Case:** Identify if certain risk profiles (e.g., opioid-naive + depression) consistently show more disagreement

### 3. Dosage Distribution Heatmap
**Best for:** Seeing the actual dosage patterns that create disagreement

**Features:**
- Rows = vignettes (with risk factors)
- Columns = 8 demographic groups (4 races × 2 genders)
- Colors = dosage level (green=Low, yellow=Medium, red=High)

**Use Case:** Visualize exactly which demographic groups received different dosages within the same vignette

### 4. Violin Plot
**Best for:** Comparing overall disagreement patterns between opioid-naive vs. tolerant

**Features:**
- Distribution shape shows how GI scores are spread
- Individual points show each vignette
- Wider sections = more vignettes at that GI level

**Use Case:** Compare whether opioid-naive or tolerant patients have more consistent dosing across demographics

## Example Results (Pilot Data)

From the current 3-dosage experiment (80 records, 10 vignettes):

```
Mean Gini Impurity: 0.1625
- Perfect Agreement: 6/10 vignettes (60%)
- Moderate Disagreement: 3/10 vignettes (30%)
- High Disagreement: 0/10 vignettes (0%)

By Opioid Status:
- Opioid-Naive: Mean GI = 0.385 (more disagreement)
- Opioid-Tolerant: Mean GI = 0.067 (more agreement)
```

## Key Findings (Full Factorial Experiment)

### Tier 2 Results (Low/High Binary Choice)
- **Mean Gini Impurity**: 0.0366
- **Perfect Agreement**: 175/200 vignettes (87.5%)
- **Disagreement Cases**: 25/200 vignettes (12.5%)
  - 17 with low disagreement (GI ≤ 0.25)
  - 8 with moderate disagreement (0.25 < GI ≤ 0.5)
  - 0 with high disagreement (GI > 0.5)
- **Highest GI**: 0.50 (V6-ON-Bip-NP — opioid-naive, bipolar disorder, no preop pain)
- **By Opioid Status**:
  - **Opioid-Naive**: Mean GI = 0.0709 (76/100 perfect agreement)
  - **Opioid-Tolerant**: Mean GI = 0.0022 (99/100 perfect agreement)

**Interpretation**: When the model has only 2 dosage options, it shows relatively high agreement across demographics. However, opioid-naive patients have **32× more disagreement** than opioid-tolerant patients, suggesting demographic bias is more pronounced when the model lacks prior opioid history information.

### Tier 3 Results (None/Low/Medium/High Granular Choice)
- **Mean Gini Impurity**: 0.0056
- **Perfect Agreement**: 196/200 vignettes (98.0%)
- **Disagreement Cases**: 4/200 vignettes (2.0%)
  - 3 with low disagreement (GI ≤ 0.25)
  - 1 with moderate disagreement (0.25 < GI ≤ 0.5)
  - 0 with high disagreement (GI > 0.5)
- **Highest GI**: 0.469 (V6-ON-None-NP — opioid-naive, no mental health history, no preop pain)
- **By Opioid Status**:
  - **Opioid-Naive**: Mean GI = 0.0091 (97/100 perfect agreement)
  - **Opioid-Tolerant**: Mean GI = 0.0022 (99/100 perfect agreement)

**Interpretation**: With 4 dosage options, overall disagreement **decreases dramatically** (0.0056 vs. 0.0366 in Tier 2). This suggests that having more granular dosage choices allows the model to converge on "Medium" for most cases, reducing demographic disparities.

### Cross-Tier Comparison
- **Tier 2 → Tier 3**: Mean GI drops by **84%** (0.0366 → 0.0056)
- **Disagreeing vignettes**: Drop from 25 → 4 (84% reduction)
- **Opioid-Naive disagreement**: Drops by 87% (0.0709 → 0.0091)
- **Opioid-Tolerant disagreement**: No change (0.0022 stays constant)

**Key Insight**: Adding more dosage options (Tier 3) paradoxically **reduces** demographic bias. This is likely because "Medium" becomes a "safe middle ground" that the model defaults to across all demographics, whereas in Tier 2, the binary Low/High choice forces the model to make more demographic-dependent decisions.

**Interpretation:** Dosage decisions are relatively consistent overall, but opioid-naive patients show more variation across demographic groups compared to opioid-tolerant patients.

## Statistical Design

### Grouping Structure

For each unique combination of:
- Vignette index
- Risk factors (opioid status, mental health, preoperative pain)

We have **8 demographic subgroups**:
- 4 races (Asian, Black, Hispanic, White)
- 2 genders (man, woman)

Gini Impurity measures how "pure" the dosage decision is across these 8 subgroups.

## Statistical Reporting Notes (CIs)

GI is computed **per vignette**. For summary statements like “% of vignettes with GI=0” or “% with moderate disagreement,” the analysis now reports **95% Wilson confidence intervals** in `gini_impurity_summary_tier*_ff.csv`. These CIs quantify uncertainty on the **proportion of vignettes** falling into each disagreement bin.

### Example Calculation

**Vignette 2: All 8 subgroups chose "Medium"**
```
Dosage counts: {Medium: 8}
p_medium = 8/8 = 1.0

GI = 1 - (1.0²) = 1 - 1 = 0.0 ✓ Perfect agreement
```

**Vignette 3: 5 chose "Medium", 3 chose "Low"**
```
Dosage counts: {Medium: 5, Low: 3}
p_medium = 5/8 = 0.625
p_low = 3/8 = 0.375

GI = 1 - (0.625² + 0.375²)
   = 1 - (0.391 + 0.141)
   = 1 - 0.532
   = 0.468 ✓ Moderate disagreement
```

## Connection to Other Metrics

### Relationship to Entropy

Gini Impurity is closely related to Shannon Entropy:
- Both measure "impurity" or "disorder"
- Gini is computationally simpler (no logarithms)
- Both used in decision tree algorithms

### Comparison to Standard Deviation

For the same vignette with 5 Medium and 3 Low:
- **Standard Deviation:** Treats as binary (0/1), gives ~0.48 or 48%
- **Gini Impurity:** Measures categorical distribution, gives 0.468

Gini is more appropriate because:
1. Dosages are **categorical**, not ordinal (we don't assume High=3, Medium=2, Low=1)
2. Clearer interpretation (what does 48% std dev mean for a binary?)
3. Generalizes naturally to >2 categories

## Extending the Analysis

### For Full Factorial Experiments

With 160 profiles per vignette:
- Still compute GI the same way
- More subgroups = more nuanced disagreement detection
- Can subset by specific demographics (e.g., only compare across races for male patients)

### Additional Analyses

You can extend this script to:
1. **Compare Gini by context** (acute vs. chronic pain)
2. **Test if GI differs significantly** between opioid-naive vs. tolerant (t-test on GI scores)
3. **Regression**: Predict GI from risk factors to identify which combinations cause most disagreement
4. **Temporal analysis**: Track GI changes if you re-run experiments over time

## Technical Notes

### Dosage Encoding

The script automatically standardizes dosage strings:
- "Low (1 week)" → "Low"
- "Medium (2 weeks)" → "Medium"
- "High (4 weeks)" → "High"
- "None of the above" → "None"

### Missing Data

If a vignette has fewer than 8 subgroups (e.g., missing demographics), GI is calculated on available subgroups. The denominator adjusts automatically.

### Maximum Theoretical GI

For 4 categories with uniform distribution:
```
GI_max = 1 - 4×(0.25²) = 1 - 0.25 = 0.75
```

This represents maximum possible disagreement (each dosage chosen by 25% of subgroups).

## Questions & Troubleshooting

### "Why are all Gini scores 0?"
This could mean:
1. Perfect agreement across all vignettes (good!)
2. Only one dosage is ever prescribed (check dosage distribution)
3. Data loading issue (verify CSV columns)

### "Should I use Gini or Standard Deviation?"
Use **Gini Impurity** because:
- Your outcome (dosage) is **categorical**
- You want to measure **agreement/disagreement** across groups
- You care about **which categories** were chosen, not just variance

Use Standard Deviation if:
- You have a truly **continuous** outcome (e.g., actual opioid dosage in mg)
- You want to compare to existing literature using SD

## References

1. **Gini Impurity in Decision Trees**: Breiman et al., "Classification and Regression Trees" (1984)
2. **Information Theory**: Shannon, "A Mathematical Theory of Communication" (1948)
3. **Measuring Bias**: Obermeyer et al., "Dissecting racial bias in an algorithm" (Science, 2019)

## Contact

For questions about this analysis, refer to the Q-Pain project documentation or contact the research team.

