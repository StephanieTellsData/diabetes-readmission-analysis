# DAX Measures and Calculations
## 30-Day Diabetes Readmission Analysis

This document presents the principal DAX calculated columns and measures used in the Power BI analysis, together with their purpose and relevant observed results.

**Tools:** Power BI Desktop, Power Query and DAX

---

## 1. Readmission Metrics

### Total Encounters

**Type:** Measure

**Purpose:** Counts the hospital encounters in the analytical dataset.

```dax
Total Encounters =
COUNTROWS('diabetic_encounter_raw(1)')
```

**Observed result:** 99,343 eligible encounters.

### Total 30-Day Readmission

**Type:** Measure

**Purpose:** Sums the 30-day readmission indicator across eligible encounters. The indicator is assumed to be coded 1 for readmission and 0 otherwise.

```dax
Total 30_Day Readmission =
SUM('diabetic_encounter_raw(1)'[30_day readmission])
```

**Observed result:** 11,314 readmissions.

### 30-Day Readmission Rate

**Type:** Measure

**Purpose:** Calculates the proportion of eligible encounters resulting in readmission within 30 days.

```dax
30_Day Readmission Rate =
DIVIDE(
    [Total 30_Day Readmission],
    [Total Encounters]
)
```

**Observed result:** 11.39%.

---

## 2. Risk Stratification

### Risk Score

**Type:** Calculated column

**Purpose:** Assigns each encounter a points-based score using five characteristics: previous inpatient visits, previous emergency visits, length of stay, number of diagnoses and number of medications.

```dax
Risk_Score =
IF(
    'diabetic_encounter_raw(1)'[number_inpatient] = 1, 1,
    IF('diabetic_encounter_raw(1)'[number_inpatient] >= 2, 2, 0)
)
+
IF(
    'diabetic_encounter_raw(1)'[number_emergency] = 1, 1,
    IF ('diabetic_encounter_raw(1)'[number_emergency]>= 2, 2, 0)
)
+
IF('diabetic_encounter_raw(1)'[time_in_hospital] >= 4, 1, 0)
+
IF(
    'diabetic_encounter_raw(1)'[number_diagnoses] >= 8, 2,
    IF('diabetic_encounter_raw(1)'[number_diagnoses] >= 6, 1, 0)
)
+
IF(
    'diabetic_encounter_raw(1)'[num_medications] >= 21, 2,
    IF('diabetic_encounter_raw(1)'[num_medications] >= 11, 1, 0)
)
```

The score ranges from 0 to 9.

| Variable | Scoring criteria | Points |
|---|---|---:|
| Previous inpatient visits | 0 / 1 / 2+ | 0 / 1 / 2 |
| Previous emergency visits | 0 / 1 / 2+ | 0 / 1 / 2 |
| Length of stay | 1–3 days / 4+ days | 0 / 1 |
| Number of diagnoses | 1–5 / 6–7 / 8+ | 0 / 1 / 2 |
| Number of medications | 1–10 / 11–20 / 21+ | 0 / 1 / 2 |

### Risk Tier

**Type:** Calculated column

**Purpose:** Classifies encounters into Low, Moderate and High Risk categories according to their risk scores.

```dax
Risk_Tier =
SWITCH(
    TRUE(),
    'diabetic_encounter_raw(1)'[Risk_Score] >= 5, "High Risk",
    'diabetic_encounter_raw(1)'[Risk_Score] >= 4, "Moderate Risk",
    "Low Risk"
)
```

| Risk category | Score |
|---|---:|
| Low Risk | 0–3 |
| Moderate Risk | 4 |
| High Risk | 5–9 |

---

## 3. Risk-Tier Analysis

### Tier Share

**Type:** Measure

**Purpose:** Calculates the proportion of encounters in each risk tier relative to all tiers, while retaining other applicable report filters.

```dax
Tier Share =
DIVIDE(
    [Total Encounters],
    CALCULATE(
        [Total Encounters],
        ALL('diabetic_encounter_raw(1)'[Risk_Tier])
    )
)
```

### Percentage of Encounters per Tier

**Type:** Measure

**Purpose:** Calculates the percentage of encounters represented by a tier using a separate denominator measure.

```dax
%Encounters per Tier =
DIVIDE(
    [Total Encounters],
    [Tier Share Denominator],
    0
)
```

### High Risk Readmission Rate

**Type:** Measure

**Purpose:** Calculates the 30-day readmission rate among encounters classified as High Risk.

```dax
High Risk Readmission Rate =
CALCULATE(
    [30_Day Readmission Rate],
    'diabetic_encounter_raw(1)'[Risk_Tier] = "High Risk"
)
```

**Observed result:** 16.54%.

### High Risk 30-Day Readmission

**Type:** Measure

**Purpose:** Calculates the proportion of all 30-day readmissions attributable to High Risk encounters.

```dax
High Risk 30-Day Readmission =
DIVIDE(
    [High Risk Readmissions],
    [Total 30_Day Readmission]
)
```

**Observed result:** 43.29%.

### Readmissions Outside High Risk %

**Type:** Measure

**Purpose:** Calculates the proportion of all 30-day readmissions that occurred outside the High Risk category.

```dax
Readmissions outside High Risk% =
1 - ('IDS_mapping(2) (1)'[High Risk 30-Day Readmission])
```

**Observed result:** 56.71%.

---

## 4. Interpreting the Readmission Metrics

These measures describe different aspects of readmission and use different denominators.

| Metric | Result | Interpretation |
|---|---:|---|
| Overall 30-Day Readmission Rate | 11.39% | Share of eligible encounters resulting in readmission |
| High Risk Readmission Rate | 16.54% | Share of High Risk encounters resulting in readmission |
| High Risk 30-Day Readmission | 43.29% | Share of all readmissions attributable to High Risk encounters |
| Readmissions Outside High Risk % | 56.71% | Share of all readmissions occurring outside the High Risk category |

---

## 5. DAX Functions Used

| Function | Application in this analysis |
|---|---|
| `SUM()` | Aggregated the binary 30-day readmission indicator. |
| `COUNTROWS()` | Counted hospital encounters. |
| `DIVIDE()` | Calculated ratios while handling zero denominators. |
| `IF()` | Implemented conditional scoring logic. |
| `SWITCH(TRUE())` | Assigned risk categories based on score thresholds. |
| `CALCULATE()` | Modified filter context for risk-specific calculations. |
| `ALL()` | Removed the risk-tier filter when calculating the overall tier denominator. |

**Note:** The risk score is an exploratory, points-based stratification framework. It is not a clinically validated prediction model.
