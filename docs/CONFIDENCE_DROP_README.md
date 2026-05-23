# Confidence Drop Analysis

## Overview

This analysis examines how the model's confidence in its dosage recommendations changes as experimental complexity increases across three tiers:

- **Tier 1**: Demographics only (race, gender) + Binary dosage (Low/High)
- **Tier 2**: Demographics + Risk factors (mental health, opioid status, pain) + Binary dosage (Low/High)
- **Tier 3**: Demographics + Risk factors + 3-level dosage (Low/Medium/High/None)

## Two Types of Analysis

We conduct **two complementary types** of confidence analysis:

### Type A: Confidence in Chosen Dosage (Overall Confidence)
Measures confidence in whatever dosage was ultimately chosen (may differ between tiers)

### Type C: Probability of Original Choice (Preference Stability)
Measures what happens to the probability of the *original* choice when complexity increases

## Research Questions

### Type A Analyses

#### Comparison A1: Tier 1 → Tier 2 (Adding Risk Factors)
**Question**: Does adding clinical complexity (risk factors) reduce the model's confidence in its dosage recommendations?

**Hypothesis**: When the model must consider additional risk factors (mental health conditions, opioid tolerance, chronic pain), it should become less certain about its recommendations, even if the final dosage choice remains the same.

#### Comparison A2: Tier 2 → Tier 3 (Adding Dosage Granularity)
**Question**: Does providing more dosage options reduce confidence in the chosen option?

**Hypothesis**: When given a "Medium" option between Low and High, the model may:
1. **Confidence Transfer**: Switch from "Low" to "Medium" with lower confidence
2. **Confidence Erosion**: Stay with "Low" but with reduced confidence due to the presence of a middle option

### Type C Analyses

#### Comparison C1: Tier 1 Choice Probability in Tier 2
**Question**: When the model chose "Low" based on demographics alone (Tier 1), what probability does it assign to "Low" when risk factors are added (Tier 2)?

**Hypothesis**: Risk factors may either:
1. **Reinforce**: Increase P(Low) if risk factors support the original choice
2. **Weaken**: Decrease P(Low) if risk factors contradict the original choice
3. **Reverse**: Switch to "High" if risk factors strongly contradict

**Key Insight**: Isolates the effect of risk factors on the original demographic-based preference, regardless of what was ultimately chosen.

#### Comparison C2: Tier 2 Choice Probability in Tier 3
**Question**: When the model chose "Low" in binary choice (Tier 2), what probability does it assign to "Low" when "Medium" is available (Tier 3)?

**Hypothesis**: Adding a middle option may:
1. **Dilute**: Decrease P(Low) as probability mass shifts to Medium
2. **Maintain**: Keep P(Low) stable if Low remains clearly appropriate
3. **Clarify**: Increase P(Low) if Medium makes High less attractive

**Key Insight**: Measures "option dilution" - how adding choices affects existing preferences.

## Methodology

### Confidence Extraction

**For Binary Dosage (Tier 1 & 2)**:
```
Confidence = prob_gpt4o_low    (if "Low" was chosen)
           = prob_gpt4o_high   (if "High" was chosen)
```

**For 3-Level Dosage (Tier 3)**:
```
Confidence = prob_gpt4o_low     (if "Low" was chosen)
           = prob_gpt4o_medium  (if "Medium" was chosen)
           = prob_gpt4o_high    (if "High" was chosen)
           = prob_gpt4o_none    (if "None" was chosen)
```

### Type A Analysis (Confidence in Chosen)

Compares the confidence in whatever dosage was ultimately chosen:
- **May involve different dosages** (e.g., comparing confidence in "Low" from Tier 1 to confidence in "High" from Tier 2)
- Captures overall confidence trends
- Mixes two effects: choice changes + confidence changes

### Type C Analysis (Probability of Original Choice)

Compares the probability of the *same* dosage across tiers:
- **Always tracks the same dosage** (e.g., P(Low) in Tier 1 vs. P(Low) in Tier 2)
- Isolates the effect on specific preferences
- Separates choice changes from probability changes

### Example Illustrating the Difference

**Scenario**: Patient with demographics suggesting "Low"

| Tier | Chosen Dosage | P(Low) | P(High) | Type A Metric | Type C Metric |
|------|---------------|--------|---------|---------------|---------------|
| Tier 1 | Low | 0.90 | 0.10 | Confidence = 0.90 | P(Low) = 0.90 |
| Tier 2 | High | 0.40 | 0.60 | Confidence = 0.60 | P(Low) = 0.40 |

**Type A Result**: Confidence dropped from 0.90 to 0.60 (-0.30)
- Interpretation: Model is less confident overall

**Type C Result**: P(Low) dropped from 0.90 to 0.40 (-0.50)
- Interpretation: Risk factors strongly weakened the original Low preference

**Key Insight**: Type C shows a larger drop because it tracks the specific effect on the Low option, while Type A shows a smaller drop because the model found a new preference (High) with moderate confidence.

### Matching Strategy

**Tier 1 → Tier 2 Comparison**:
- Match on: `vignette_idx + race + gender`
- Note: Tier 1 has no risk factors, so we compare across all risk factor combinations
- This shows the average effect of adding risk factors

**Tier 2 → Tier 3 Comparison**:
- Match on: `vignette_idx + race + gender + risk_mh + risk_op + risk_pain`
- Requires identical patient profiles (full factorial design)
- Current pilot data has randomized risk factors, so matches may be limited

### Statistical Tests

- **Paired permutation test (mean delta)**: Tests if the mean change (e.g., Tier 2 − Tier 1) is significantly different from zero using random sign-flips of paired differences (nonparametric).
- **Bootstrap 95% CI (mean / mean delta)**: Percentile bootstrap confidence intervals for the mean confidence (per tier) and mean deltas (\(\Delta_{12}\), \(\Delta_{23}\)). This emphasizes effect size + uncertainty rather than only p-values.
- **Stratified analysis**: Examines drops by:
  - Demographics (race, gender)
  - Risk factors (mental health, opioid status, preoperative pain)
  - Decision consistency (did the dosage choice change?)

### New Summary Output (recommended for slides/paper)

For each model, the script writes:

- `confidence_trajectory_summary_ff.csv`: mean confidence at Tier 1/2/3 with 95% CIs, plus mean deltas (\(\Delta_{12}\), \(\Delta_{23}\), \(\Delta_{13}\)) and paired permutation p-values for \(\Delta_{12}\), \(\Delta_{23}\).

## Key Findings (Pilot Data)

### Overall Confidence Levels

| Tier | Mean Confidence | Interpretation |
|------|----------------|----------------|
| Tier 1 | 1.0000 | Near-perfect confidence (binary, no risk factors) |
| Tier 2 | 0.9999 | Still near-perfect (binary with risk factors) |
| Tier 3 | 0.7709 | **Substantial drop** (3-level dosage) |

### Type A Results: Tier 1 → Tier 2 (Adding Risk Factors)

- **Mean confidence drop**: -0.000045 (essentially no change)
- **Statistical significance**: p = 0.037 (significant, but effect size is tiny)
- **Interpretation**: Adding risk factors causes a statistically significant but practically negligible decrease in confidence when dosage choices remain binary

**By Risk Factor**:
- Opioid-Tolerant patients show slightly larger confidence drops (-0.000148)
- Anxiety Disorder shows the largest drop among mental health conditions (-0.000161)

### Type A Results: Tier 2 → Tier 3 (Adding Dosage Granularity)

- **Status**: No matches found in pilot data (risk factors were randomized)
- **Expected in full factorial**: This comparison will show the largest confidence drops
- **Prediction**: Mean confidence drop of ~0.2-0.3 based on Tier 3's overall confidence of 0.77

### Type C Results: Tier 1 Choice Probability in Tier 2

- **Expected Pattern**: Small probability changes since binary choices remain
- **Key Questions**:
  - Do risk factors reinforce or weaken demographic-based preferences?
  - Which risk factors cause the largest probability shifts?
  - Do certain demographics show more stable preferences?

### Type C Results: Tier 2 Choice Probability in Tier 3

- **Status**: No matches found in pilot data (risk factors were randomized)
- **Expected Pattern**: Larger probability drops as Medium option attracts probability mass
- **Key Questions**:
  - How much probability shifts from Low/High to Medium?
  - Do decisions that change show larger probability drops?
  - Is the probability drop symmetric for Low vs. High?

## Key Findings (Full Factorial Experiment)

### Overall Confidence Levels

| Tier | Mean Confidence | Median | Std Dev | Interpretation |
|------|----------------|--------|---------|----------------|
| Tier 1 | 0.8455 | 0.8927 | 0.1480 | Moderate confidence with wide distribution |
| Tier 2 | 0.9433 | 0.9914 | 0.1033 | **High confidence**, distribution shifts right |
| Tier 3 | 0.9785 | 0.9958 | 0.0528 | **Very high confidence**, extremely peaked |

**Key Insight**: Contrary to initial hypothesis, confidence **increases** with each tier. The model becomes more certain about its final choice as more information and options are provided.

### Type A Results: Tier 1 → Tier 2 (Adding Risk Factors)

- **Mean confidence drop**: **+0.0978** (confidence **increased**)
- **Median confidence drop**: +0.0576
- **Statistical significance**: reported via paired permutation p-value (see `confidence_trajectory_summary_ff.csv`)
- **Interpretation**: Adding risk factors **increases** model confidence in its dosage recommendations, contrary to the hypothesis that complexity would reduce certainty.

**By Risk Factor**:
- **Opioid Status**: 
  - Opioid-Naive: +0.0519 (moderate increase)
  - Opioid-Tolerant: **+0.1438** (large increase) — risk factors provide most clarity for this group
- **Mental Health**: All conditions show similar increases (~0.09-0.10)
- **Preoperative Pain**: Similar increases for both chronic pain and no pain (~0.098)

**By Demographics**:
- **Race**: 
  - Black: +0.1284 (largest increase)
  - Asian: +0.1022
  - White: +0.0916
  - Hispanic: +0.0690 (smallest increase)
- **Gender**: 
  - Women: +0.1129 (larger increase)
  - Men: +0.0828

**Key Finding**: Risk factors provide **clarifying context** that increases confidence, especially for Opioid-Tolerant patients and Black patients.

### Type A Results: Tier 2 → Tier 3 (Adding Dosage Granularity)

- **Mean confidence drop**: **+0.0352** (confidence **increased**)
- **Median confidence drop**: +0.0027
- **Statistical significance**: reported via paired permutation p-value (see `confidence_trajectory_summary_ff.csv`)
- **Interpretation**: Adding dosage granularity slightly increases confidence, as the model can better match the clinical scenario.

**By Decision Consistency**:
- **Decision Changed** (n=1,555): Mean drop = +0.0330
- **Decision Stayed Same** (n=45): Mean drop = +0.1122 (larger increase when decision remains stable)

**By Risk Factor**:
- **Opioid Status**: 
  - Opioid-Naive: +0.0852 (increase)
  - Opioid-Tolerant: -0.0148 (slight decrease, but still near zero)
- **Mental Health**: Small increases across all conditions (~0.026-0.044)
- **Preoperative Pain**: Similar small increases (~0.035)

**By Demographics**:
- **Race**: Small increases across all groups (~0.028-0.049)
- **Gender**: Similar small increases for both (~0.033-0.037)

**Key Finding**: Adding "Medium" option allows better clinical matching, increasing confidence in the chosen dosage.

### Type C Results: Tier 1 Choice Probability in Tier 2

- **Mean probability drop**: -0.0428 (original choice probability decreased)
- **Median probability drop**: +0.0174 (median actually increased)
- **Statistical significance**: p < 1.67e-07 (highly significant)
- **Interpretation**: Overall, the probability of Tier 1's original choice decreases slightly when risk factors are added, but the model becomes more confident about its new choice.

**By Decision Consistency**:
- **Decision Changed** (n=284): Mean drop = **-0.6486** (large decrease)
  - Most common switch: Low → High (243 cases)
- **Decision Stayed Same** (n=1,316): Mean drop = **+0.0879** (increase)
  - When decision stays, the probability of that choice increases

**By Risk Factor**:
- **Opioid Status**: 
  - Opioid-Naive: -0.0420
  - Opioid-Tolerant: -0.0436
- **Mental Health**: Small negative drops across all conditions (~-0.034 to -0.054)
- **Preoperative Pain**: 
  - Chronic pain: -0.0513
  - No pain: -0.0343

**By Tier 1 Choice**:
- **High** (n=1,280): Mean drop = +0.0755, 96.8% stayed with High
- **Low** (n=320): Mean drop = **-0.5162**, only 24.1% stayed with Low
  - Most Low choices from Tier 1 switched to High in Tier 2

**Key Finding**: Risk factors often **reverse** Low preferences (from Tier 1) to High, but when the decision stays the same, confidence in that choice increases.

### Type C Results: Tier 2 Choice Probability in Tier 3

- **Mean probability drop**: **-0.9166** (dramatic decrease)
- **Median probability drop**: -0.9914
- **Mean probability mass transferred to Medium**: **0.8801** (88%)
- **Statistical significance**: p < 1e-300 (highly significant)
- **Interpretation**: When "Medium" option is added, probability mass shifts dramatically from Low/High to Medium.

**By Decision Consistency**:
- **Decision Changed** (n=1,555): 
  - Mean drop = -0.9464
  - Mean P(Medium) = 0.9056
  - **92.3% switched to Medium**
- **Decision Stayed Same** (n=45): 
  - Mean drop = +0.1122 (increase)
  - Mean P(Medium) = 0.0000 (no shift to Medium)

**By Tier 2 Choice**:
- **High** (n=1,482): 
  - Mean drop = -0.9538
  - Mean P(Medium) = 0.9061
  - Only 0.1% stayed with High
- **Low** (n=118): 
  - Mean drop = -0.4501
  - Mean P(Medium) = 0.5532
  - 37.3% stayed with Low

**By Risk Factor**:
- **Opioid Status**: 
  - Opioid-Naive: -0.8446, Mean P(Med) = 0.8786
  - Opioid-Tolerant: -0.9886, Mean P(Med) = 0.8816
- **Mental Health**: All conditions show similar patterns (~-0.91, P(Med) ~0.88)
- **Preoperative Pain**: Similar patterns for both (~-0.90 to -0.93, P(Med) ~0.88)

**Key Finding**: The "Medium" option captures **88% of probability mass** on average, with 92.3% of decisions switching to Medium. This suggests that the binary Low/High choice was forcing the model into suboptimal decisions, and Medium better matches the clinical scenarios.

### Summary of Full Factorial Findings

**Contrary to Initial Hypothesis**: Adding complexity (risk factors and dosage options) **increases** confidence rather than decreasing it. This occurs because:

1. **Risk factors provide clarifying context**: Clinical information helps the model make more confident decisions, especially for Opioid-Tolerant patients.

2. **Dosage granularity enables better matching**: The "Medium" option allows the model to select a dosage that better fits the clinical scenario, resulting in higher confidence in that choice.

3. **Decision changes reflect optimization**: When the model switches from "High" to "Medium", it's because Medium is a better match, leading to higher confidence in Medium than High had.

4. **Probability mass redistribution**: 88% of probability mass shifts to Medium when it becomes available, indicating that the binary choice was forcing suboptimal decisions.

## Visualizations Generated

### Type A Visualizations (Confidence in Chosen Dosage)

**Pilot Data Files** (suffix: `.png`):
1. **`confidence_distributions_all_tiers.png`**
   - Histograms showing confidence distributions for each tier
   - Shows dramatic shift in Tier 3

2. **`confidence_drop_heatmap_tier1_to_tier2.png`**
   - Heatmap: Race × Gender
   - Blue = confidence decreased, Red = confidence increased

3. **`confidence_drop_by_risk_factors_tier1_to_tier2.png`**
   - Bar plots showing confidence drop by:
     - Mental health condition
     - Opioid status
     - Preoperative pain status

4. **`confidence_drop_boxplot_comparison.png`**
   - Side-by-side boxplots comparing both transitions
   - Shows distribution of confidence drops

**Full Factorial Files** (suffix: `_ff.png`):
1. **`confidence_distributions_all_tiers_ff.png`**
   - Histograms showing confidence distributions for each tier
   - Shows increasing confidence across tiers (contrary to hypothesis)

2. **`confidence_drop_heatmap_tier1_to_tier2_ff.png`**
   - Heatmap: Race × Gender
   - Shows mostly positive values (confidence increased)

3. **`confidence_drop_by_risk_factors_tier1_to_tier2_ff.png`**
   - Bar plots showing confidence drop by risk factors
   - All bars show positive values (confidence increased)

4. **`confidence_drop_heatmap_tier2_to_tier3_ff.png`**
   - Heatmap: Race × Gender for Tier 2→3 transition
   - Shows small positive values (confidence slightly increased)

5. **`confidence_drop_by_risk_factors_tier2_to_tier3_ff.png`**
   - Bar plots showing confidence drop by risk factors
   - Shows small increases, with Opioid-Naive showing larger increases

6. **`confidence_drop_boxplot_comparison_ff.png`**
   - Side-by-side boxplots comparing both transitions
   - Tier 1→2 shows wider spread with positive median
   - Tier 2→3 shows narrow spread centered near zero

### Type C Visualizations (Probability of Original Choice)

**Pilot Data Files** (suffix: `.png`):
7. **`tier1_choice_probability_in_tier2.png`** (4-panel figure)
   - **Panel 1**: Scatter plot showing Tier 1 confidence vs. Tier 2 P(Tier 1 choice)
     - Color-coded by whether decision changed
     - Diagonal line shows "no change"
   - **Panel 2**: Box plot of probability drops by Tier 1 choice and decision consistency
   - **Panel 3**: Bar plot of probability drops by opioid status
   - **Panel 4**: Bar plot of probability drops by mental health status

**Full Factorial Files** (suffix: `_ff.png`):
7. **`tier1_choice_probability_in_tier2_ff.png`** (4-panel figure)
   - **Panel 1**: Scatter plot showing Tier 1 confidence vs. Tier 2 P(Tier 1 choice)
     - Most "Decision Stayed" points above diagonal (probability increased)
     - "Decision Changed" points below diagonal (probability decreased)
   - **Panel 2**: Box plot showing large negative drops when decision changed
   - **Panel 3**: Bar plot showing small negative drops by opioid status
   - **Panel 4**: Bar plot showing small negative drops by mental health

8. **`tier2_choice_probability_in_tier3_ff.png`** (4-panel figure)
   - **Panel 1**: Scatter plot showing Tier 2 confidence vs. Tier 3 P(Tier 2 choice)
     - "Decision Changed" points clustered at bottom (P ≈ 0)
     - "Decision Stayed" points in upper-right (P ≈ 0.5-1.0)
   - **Panel 2**: Box plot showing dramatic negative drops when decision changed
   - **Panel 3**: Bar plot showing large negative drops by opioid status
   - **Panel 4**: Bar plot showing large negative drops by mental health (all similar)

## Output Files

### CSV Files

**Note**: Full factorial experiment files have `_ff` suffix (e.g., `confidence_drop_tier1_to_tier2_ff.csv`)

#### Type A Files

1. **`confidence_drop_tier1_to_tier2_ff.csv`**
   - Columns: `vignette_idx`, `race`, `gender`, `risk_mh`, `risk_op`, `risk_pain`
   - `confidence_tier1`, `chosen_dosage_tier1`
   - `confidence_tier2`, `chosen_dosage_tier2`
   - `confidence_drop` (Tier 2 - Tier 1)
   - **Full factorial**: 1,600 matched records

2. **`confidence_drop_tier2_to_tier3_ff.csv`**
   - Same structure as above, plus:
   - `confidence_tier3`, `chosen_dosage_tier3`
   - `decision_changed` (boolean: did dosage choice change?)
   - **Full factorial**: 1,600 matched records

#### Type C Files

3. **`tier1_choice_probability_in_tier2_ff.csv`**
   - Columns: `vignette_idx`, `race`, `gender`, `risk_mh`, `risk_op`, `risk_pain`
   - `tier1_chosen`, `tier1_confidence`
   - `tier2_chosen`, `tier2_prob_of_tier1_choice`
   - `prob_drop` (Tier 2 P(Tier 1 choice) - Tier 1 confidence)
   - `decision_changed`
   - **Full factorial**: 1,600 matched records

4. **`tier2_choice_probability_in_tier3_ff.csv`**
   - Same structure as above, plus:
   - `tier3_chosen`, `tier3_prob_of_tier2_choice`
   - `prob_mass_to_medium` (how much probability went to Medium option)
   - All Tier 3 probabilities (`tier3_prob_none`, `tier3_prob_low`, `tier3_prob_medium`, `tier3_prob_high`)
   - **Full factorial**: 1,600 matched records

## Usage

```bash
python analyze_confidence_drop.py
```

The script will:
1. Load all three tier datasets
2. Extract confidence for each record
3. Match records between tiers
4. Calculate confidence drops
5. Generate statistical summaries
6. Create visualizations

All outputs are saved to the `results/` directory.

## Interpretation Guide

### Confidence Drop Values (Type A)

- **Negative values** (e.g., -0.05): Confidence **decreased** (model became less certain)
- **Zero**: No change in confidence
- **Positive values** (e.g., +0.05): Confidence **increased** (model became more certain)

### Probability Drop Values (Type C)

- **Negative values** (e.g., -0.30): Original choice probability **decreased** (preference weakened)
- **Zero**: Original choice probability unchanged
- **Positive values** (e.g., +0.10): Original choice probability **increased** (preference strengthened)

### What to Look For

#### Type A Insights (Overall Confidence Trends)
1. **Overall Trend**: Do confidence levels decrease with complexity?
2. **Demographic Disparities**: Do certain groups show larger confidence drops?
3. **Risk Factor Effects**: Which risk factors cause the most uncertainty?
4. **Decision Consistency**: Is confidence drop larger when the decision changes?

#### Type C Insights (Preference Stability)
1. **Preference Erosion**: Does adding complexity weaken original preferences?
2. **Preference Reinforcement**: Do risk factors sometimes strengthen original choices?
3. **Probability Redistribution**: Where does probability mass go (e.g., to Medium option)?
4. **Decision Reversal**: When does the model completely reverse its preference?

### Combining Type A and Type C

Four possible patterns when combining both analyses:

| Type A | Type C | Interpretation | Example |
|--------|--------|----------------|---------|
| Small drop | Large drop | Confident but changed | "I was sure about Low, now sure about High" |
| Large drop | Small drop | Uncertain but stable | "I still prefer Low, but much less confident" |
| Large drop | Large drop | Uncertain and changed | "I was sure about Low, now uncertain about High" |
| Small drop | Small drop | Stable preference | "Low was appropriate, still appropriate" |

### Clinical Interpretation Examples

**Example 1: Risk Factors Reinforce Original Choice**
- Tier 1: Chose "Low" (0.92 confidence) based on demographics
- Tier 2 Type A: Chose "Low" (0.98 confidence) → confidence increased
- Tier 2 Type C: P(Low) = 0.98 → probability increased
- **Interpretation**: Risk factors confirmed the original demographic-based judgment

**Example 2: Risk Factors Reveal Complexity**
- Tier 1: Chose "Low" (0.95 confidence) based on demographics
- Tier 2 Type A: Chose "High" (0.65 confidence) → confidence decreased
- Tier 2 Type C: P(Low) = 0.30 → probability dropped dramatically
- **Interpretation**: Risk factors revealed patient is high-risk; demographic-based Low was inappropriate

**Example 3: Medium Option Dilutes Confidence**
- Tier 2: Chose "Low" (0.92 confidence) in binary choice
- Tier 3 Type A: Chose "Medium" (0.55 confidence) → confidence decreased
- Tier 3 Type C: P(Low) = 0.35 → probability dropped, 0.55 went to Medium
- **Interpretation**: Medium option better captured the model's true uncertainty between Low and High

## Limitations (Pilot Data)

1. **Small sample size**: Only 80 records per tier (10 vignettes × 8 demographic groups)
2. **Randomized risk factors**: Tier 2 and Tier 3 have different risk factor assignments
3. **No Tier 2→3 matches**: Cannot analyze the effect of dosage granularity yet

## Full Factorial Experiment Status

**Completed**: Full factorial experiment has been run with:
- **Sample size**: 1,600 records per tier
  - 10 vignettes × 4 races × 2 genders × 5 mental health × 2 opioid status × 2 pain
- **Full matching**: All records match between Tier 2 and Tier 3 (identical patient profiles)
- **Robust analysis**: Sufficient power to detect demographic and risk factor interactions

**Key Achievements**:
- All Type A and Type C analyses now have complete data
- Tier 2→3 comparisons are fully available
- Statistical power sufficient for subgroup analyses
- All visualizations generated with full factorial data

**Model Used**: `gpt-4o-mini` (consistent across all tiers for comparability)

## References

- Tier mapping: See `TIER_MAPPING.md`
- Related analyses:
  - Escalation rate: `analyze_dosage_escalation.py`
  - Gini impurity: `analyze_gini_impurity.py`
  - Dosage deviation: `analyze_dosage_deviation.py`
  - KL divergence: `analyze_kl_divergence.py`

## Summary

This analysis provides a comprehensive view of how model confidence evolves across experimental tiers by examining both:

1. **Type A (Confidence in Chosen)**: Overall confidence trends
2. **Type C (Probability of Original Choice)**: Specific preference stability

### Key Findings from Full Factorial Experiment

**Contrary to Initial Hypothesis**: Adding complexity (risk factors and dosage options) **increases** confidence rather than decreasing it.

**Type A Results**:
- **Tier 1→2**: Confidence increases by +0.098 (risk factors provide clarifying context)
- **Tier 2→3**: Confidence increases by +0.035 (dosage granularity enables better matching)
- **Overall trend**: Mean confidence increases from 0.846 (Tier 1) → 0.943 (Tier 2) → 0.979 (Tier 3)

**Type C Results**:
- **Tier 1→2**: Original choice probability decreases slightly (-0.043), but when decision stays, probability increases (+0.088)
- **Tier 2→3**: Original choice probability decreases dramatically (-0.917) as 88% of probability mass shifts to Medium option
- **Decision changes**: 92.3% of Tier 2 decisions switch to Medium in Tier 3

**Interpretation**: The model becomes more certain about its final choice as it receives more information and better-fitting options. This is not a contradiction—it reflects that more information and better options lead to more confident decisions, even if the specific choice changes.

Together, these analyses reveal:
- **Information reduces uncertainty**: More clinical context increases confidence in final decisions
- **Granularity improves fit**: More dosage options allow better matching, increasing confidence
- **Decision changes are informative**: When the model changes its decision, it's usually because it found a better match
- **Probability mass redistribution**: 88% shifts to Medium when available, indicating binary choice was suboptimal

The dual approach provides richer insights than either analysis alone, allowing us to distinguish between "the model became more confident overall" (Type A) and "the model's original preference weakened but found a better match" (Type C).

