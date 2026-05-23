# Cross-Vignette KL Divergence Analysis

## Overview

This analysis uses **Kullback-Leibler (KL) divergence** to measure whether demographic subgroups exhibit systematically different dosage prescribing patterns across all clinical scenarios. Unlike within-vignette deviation analysis (which compares subgroups within identical scenarios), this cross-vignette approach aggregates across all vignettes to detect **systematic bias patterns**.

## What is KL Divergence?

### Mathematical Definition

KL divergence \(D_{KL}(P \parallel Q)\) measures how one probability distribution \(P\) differs from a reference distribution \(Q\):

\[
D_{KL}(P \parallel Q) = \sum_{i} P(i) \log\frac{P(i)}{Q(i)}
\]

Where:
- **P**: The "actual" distribution (e.g., dosage distribution for Black women across all vignettes)
- **Q**: The "reference" distribution (e.g., dosage distribution for all patients across all vignettes)
- **Result**: A non-negative value where:
  - \(D_{KL} = 0\): P and Q are identical (no bias)
  - \(D_{KL} > 0\): P differs from Q (potential bias)
  - Higher values indicate greater divergence

### Intuitive Interpretation

Think of KL divergence as measuring "surprise":
- If we expect dosages to follow distribution Q (the population average)
- But observe distribution P (a specific demographic subgroup)
- KL divergence quantifies how "surprised" we should be
- Higher surprise = greater systematic difference from expected pattern

## Why KL Divergence for Cross-Vignette Analysis?

### Advantages Over Mean Deviation

1. **Distribution-Based**: Captures the full pattern of prescribing, not just the average
2. **No Interval Assumptions**: Treats dosage categories (None, Low, Medium, High) as categorical outcomes without assuming equal spacing
3. **Statistically Principled**: Well-established information-theoretic measure with clear interpretation
4. **Sensitive to Patterns**: Can detect systematic bias even when means are similar

### Example Scenario

Consider two demographic groups with the same average dosage:

**Group A**: Always gets "Medium" → Distribution: [0, 0, 1.0, 0]
**Group B**: 50% "Low", 50% "High" → Distribution: [0, 0.5, 0, 0.5]

- **Mean deviation**: Both groups have the same average (2.0)
- **KL divergence**: Detects that Group B has higher variance/uncertainty

## Methodology

### Step 1: Calculate Reference Distribution (Q)

For all patients across all vignettes, calculate:

```
Q = [P(None), P(Low), P(Medium), P(High)]
```

Example:
```
Q = [0.125, 0.375, 0.375, 0.125]
```

This represents the "fair" or expected dosage distribution if no bias exists.

### Step 2: Calculate Subgroup Distributions (P)

For each demographic subgroup (e.g., Asian men, Black women), calculate their dosage distribution across all vignettes:

```
P_subgroup = [P(None), P(Low), P(Medium), P(High)]
```

Example for Black women:
```
P_Black_women = [0.10, 0.50, 0.30, 0.10]
```

### Step 3: Compute KL Divergence

For each subgroup:

```python
D_KL(P || Q) = sum(P[i] * log(P[i] / Q[i]) for i in [None, Low, Med, High])
```

### Step 4: Stratify by Opioid Status

Since opioid status is a major clinical factor affecting dosage, we also compute KL divergence separately for:
- Opioid-Naive patients
- Opioid-Tolerant patients

This isolates demographic bias from clinical differences.

## Interpretation Guidelines

### KL Divergence Thresholds

| Range | Interpretation | Action |
|-------|----------------|--------|
| < 0.01 | Negligible difference | No systematic bias detected |
| 0.01 - 0.05 | Notable difference | Mild systematic bias - investigate patterns |
| > 0.05 | High difference | Significant systematic bias - requires attention |

**Note**: These thresholds are context-dependent. For medical decision-making, even small divergences may be clinically significant.

### What KL Divergence Tells You

- **Higher KL = Greater Systematic Bias**: The subgroup's dosage pattern consistently differs from the population average
- **Lower KL = More Typical Pattern**: The subgroup's dosage pattern matches the population
- **Direction of Bias**: Examine the P distribution to see if bias is toward over-prescribing (more High/Medium) or under-prescribing (more Low/None)

## Comparison with Within-Vignette Deviation Analysis

| Aspect | KL Divergence (Cross-Vignette) | Mean Deviation (Within-Vignette) |
|--------|-------------------------------|----------------------------------|
| **Question** | Does this group have a different prescribing *pattern* across all scenarios? | Does this group get different dosages within *identical* scenarios? |
| **Unit of Analysis** | All vignettes aggregated | Individual vignettes |
| **Metric** | Distributional divergence | Mean difference |
| **Assumption** | Categorical outcomes | Ordinal/interval scale |
| **Detects** | Systematic bias patterns | Scenario-specific disparities |
| **Sample Size Need** | Multiple vignettes per group | Multiple subgroups per vignette |

### When to Use Each

**Use KL Divergence (Cross-Vignette) when**:
- You want to detect overall systematic bias across the entire experiment
- You have sufficient vignettes to create meaningful distributions (typically 10+)
- You want to avoid assumptions about equal spacing between dosage levels
- You care about the *pattern* of prescribing, not just averages

**Use Deviation Analysis (Within-Vignette) when**:
- You want to detect disparities in identical clinical scenarios
- You want interpretable magnitude ("0.5 dosage levels lower")
- You want to identify specific vignettes with high disparity
- You have multiple subgroups per vignette but few observations per subgroup

### Complementary Insights

Both approaches provide complementary insights:
- **Within-vignette**: "Are subgroups treated differently when presenting identically?"
- **Cross-vignette**: "Do subgroups have different overall prescribing patterns?"

A subgroup might have:
- High KL divergence + low within-vignette deviation: Consistently receives different dosages, but fairly within each scenario
- Low KL divergence + high within-vignette deviation: Overall pattern is normal, but specific scenarios show bias

## Output Files

### Tables

1. **`kl_summary.csv`**: Summary statistics
   - Mean, median, std dev, min, max KL divergence
   - Count of groups with notable/high divergence
   - Most/least divergent groups
   - **Chi-squared test of independence (dosage × subgroup)**: p-value and **Cramer’s V** effect size (global test on underlying counts)
   - **Max KL 95% CI (bootstrap)**: optional descriptive uncertainty on the maximum subgroup KL (computed by resampling rows and recomputing KL)

2. **`kl_divergence_details.csv`**: Detailed results per subgroup
   - KL divergence and JS divergence scores
   - Full P and Q distributions
   - Sample sizes

3. **`kl_divergence_by_opioid_status.csv`**: Stratified analysis
   - KL divergence for Opioid-Naive patients
   - KL divergence for Opioid-Tolerant patients

### Visualizations

1. **`kl_divergence_barplot.png`**: Bar chart showing KL divergence for each demographic subgroup
   - Sorted by divergence (highest first)
   - Color-coded by gender
   - Threshold lines for notable (0.01) and high (0.05) divergence

2. **`kl_divergence_heatmap.png`**: Heatmap showing KL divergence by race × gender
   - Easy comparison across demographics
   - Color intensity indicates divergence magnitude

3. **`distribution_comparison.png`**: Grouped bar charts comparing P vs Q for top 8 most divergent groups
   - Shows *why* each group has high KL divergence
   - Visualizes which dosage categories drive the difference

4. **`kl_by_opioid_status.png`**: Faceted bar plots showing KL divergence separately for Opioid-Naive and Opioid-Tolerant
   - Controls for major clinical factor
   - Isolates demographic bias from clinical differences

## Technical Details

### Handling Edge Cases

1. **Zero Probabilities**: Add small epsilon (1e-10) to all probabilities to avoid log(0)
2. **Small Sample Sizes**: KL divergence is more reliable with larger samples; interpret cautiously with <10 vignettes
3. **Asymmetry**: KL divergence is asymmetric: \(D_{KL}(P \parallel Q) \neq D_{KL}(Q \parallel P)\). We use \(D_{KL}(P \parallel Q)\) where P is the subgroup and Q is the reference.

### Jensen-Shannon Divergence (JS)

We also compute **Jensen-Shannon divergence**, a symmetric variant:

\[
JS(P, Q) = \frac{1}{2} D_{KL}(P \parallel M) + \frac{1}{2} D_{KL}(Q \parallel M)
\]

where \(M = \frac{1}{2}(P + Q)\)

JS divergence is:
- Symmetric: \(JS(P, Q) = JS(Q, P)\)
- Bounded: \(0 \leq JS(P, Q) \leq 1\) (when using log base 2)
- More stable for small samples

## Usage

### Running the Analysis

```bash
cd "/path/to/Q-Pain/analysis_scripts"
python analyze_kl_divergence.py
```

### Requirements

- Input: `experiment_results/results_post_op_gpt4o_risk_factors_3_dosages.csv`
- Python packages: pandas, numpy, scipy, matplotlib, seaborn

### Expected Runtime

- ~5-10 seconds for pilot data (80 records)
- ~30-60 seconds for full factorial data (1600+ records)

## Theoretical Background

### Information Theory Context

KL divergence originates from information theory and has several interpretations:

1. **Relative Entropy**: The expected excess "surprise" when using Q to model data that actually follows P
2. **Information Loss**: The information lost when Q is used to approximate P
3. **Statistical Distance**: A measure of how distinguishable two distributions are (though not a true metric since it's asymmetric)

### Connection to Maximum Likelihood

KL divergence is directly related to log-likelihood:
- Minimizing \(D_{KL}(P \parallel Q)\) is equivalent to maximizing the likelihood of Q given data from P
- This makes KL divergence a natural measure for comparing distributions in statistical modeling

### Applications in Bias Detection

KL divergence has been used in:
- Fairness in machine learning (detecting disparate impact)
- Clinical trial analysis (comparing treatment effects across subgroups)
- Healthcare disparities research (identifying systematic differences in care)

## Limitations and Considerations

1. **Sample Size Sensitivity**: Requires sufficient samples to estimate distributions reliably
2. **Categorical Assumption**: Treats dosage as categorical, ignoring ordinal structure
3. **Aggregation**: May mask scenario-specific biases that cancel out in aggregate
4. **Interpretation Complexity**: Less intuitive than simple mean differences
5. **Asymmetry**: Direction matters; \(D_{KL}(P \parallel Q) \neq D_{KL}(Q \parallel P)\)

## References

1. Kullback, S., & Leibler, R. A. (1951). On information and sufficiency. *The Annals of Mathematical Statistics*, 22(1), 79-86.

2. Cover, T. M., & Thomas, J. A. (2006). *Elements of information theory*. John Wiley & Sons.

3. Lin, J. (1991). Divergence measures based on the Shannon entropy. *IEEE Transactions on Information Theory*, 37(1), 145-151. (Jensen-Shannon divergence)

4. Mehrabi, N., et al. (2021). A survey on bias and fairness in machine learning. *ACM Computing Surveys*, 54(6), 1-35.

## Contact

For questions about this analysis, please refer to the main project documentation or contact the research team.

---

**Last Updated**: February 2026  
**Script**: `analysis_scripts/analyze_kl_divergence.py`  
**Related**: See also `DEVIATION_ANALYSIS_README.md` for within-vignette analysis

