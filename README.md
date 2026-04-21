# Insurance Claim Analytics Dashboard — Power BI

An end-to-end insurance analytics solution analyzing customer policies, premiums, claims, and risk profiles to identify loss-making segments and generate actionable, data-driven recommendations for portfolio optimization.

![Power BI](https://img.shields.io/badge/Power%20BI-F2C811?style=for-the-badge&logo=powerbi&logoColor=black)
![DAX](https://img.shields.io/badge/DAX-blue?style=for-the-badge)
![Data Analysis](https://img.shields.io/badge/Data%20Analysis-green?style=for-the-badge)

---

## Table of Contents

1. [Business Problem](#business-problem)
2. [Dataset Overview](#dataset-overview)
3. [Data Model](#data-model)
4. [Dashboard Pages](#dashboard-pages)
5. [Key Measures and Calculated Columns](#key-measures-and-calculated-columns)
6. [Key Findings](#key-findings)
7. [Recommendations](#recommendations)
8. [Methodology Notes](#methodology-notes)
9. [Tech Stack](#tech-stack)
10. [How to Reproduce](#how-to-reproduce)

---

## Business Problem

Insurance companies face a fundamental challenge: not every customer is profitable. Some customers file so many claims that the claims paid exceed the premiums collected, creating a **loss-making policy**. Identifying these customers — and deciding what to do about them — is central to portfolio management.

This dashboard answers three questions every insurance executive asks:

1. **Are we making money on this book of business?** (Portfolio loss ratio)
2. **Where is the loss concentrated?** (Segment, product, demographic analysis)
3. **What should we do about it?** (Data-driven recommendations per segment)

The core metric is the **Loss Ratio**:

```
Loss Ratio = Total Claims Paid / Total Premiums Earned
```

- Loss Ratio below 60%: Highly profitable
- 60–80%: Healthy range (industry benchmark)
- 80–100%: High risk, requires action
- Above 100%: Loss-making, immediate intervention required

---

## Dataset Overview

**Source:** Synthetic health insurance dataset (`data_synthetic.csv`) — 53,503 customer records with 30 columns covering demographics, policy details, claims history, and risk profiling.

**Key fields used in analysis:**

| Field | Description |
|---|---|
| CustomerID | Unique customer identifier |
| Age, Gender, MaritalStatus | Demographics |
| Occupation, Income, Education | Socioeconomic attributes |
| State | Geography |
| PolicyStartDate, PolicyRenewalDate | Policy lifecycle dates |
| PolicyType | Group, Family, Individual, Business |
| AnnualPremium | Annual premium paid |
| CoverageAmount | Sum insured per policy |
| Deductible | Out-of-pocket threshold |
| NoOfClaims | Claim count per customer (0–5) |
| PreviousClaims | Historical claim count |
| RiskProfile | Risk score (0–3) |
| CreditScore | Credit-based insurance indicator |
| Segment | Pre-built customer segmentation group |

**Scope:** Policies spanning 2018–2023 across 20 U.S. states.

---

## Data Model

The model uses a **star schema** with a fact-style customer/policy table and a conforming Date dimension:

```
┌─────────────┐         ┌──────────────────┐
│    Date     │────→────│    Insurance     │
│ (Dimension) │  1:M    │   (Fact-style)   │
└─────────────┘         └──────────────────┘
                                 │
                                 ▼
                        ┌──────────────────┐
                        │    _Measures     │
                        │ (DAX container)  │
                        └──────────────────┘
```

**Relationship:** `Date[Date] → Insurance[PolicyStartDate]` (single direction, one-to-many).

The `_Measures` table is a blank table created purely to organize all DAX measures separately from columns, improving model readability.

---

## Dashboard Pages

### Page 1: Executive Summary
**Purpose:** One-screen portfolio health view for leadership.

Key visuals:
- 8 KPI cards (Total Customers, Total Premium, Total Claims Paid, Loss Ratio, Avg Premium/Customer, Loss-Making Customers, High Risk Customer Count, Claim Frequency)
- Premium vs. Claims monthly trend line
- Loss Ratio gauge (target 70%, max 150%)
- Customer distribution by Risk Tier (donut)
- Top states by Total Premium (bar, loss ratio in tooltip)

### Page 2: Loss Ratio Deep Dive
**Purpose:** Where is the loss concentrated?

Key visuals:
- Matrix: PolicyType × AgeBand with Loss Ratio (gradient formatting)
- Treemap: Loss Ratio by Segment
- Scatter: AnnualPremium vs. ClaimAmount (customer-level, color by Risk Tier)
- Bar charts: Loss Ratio by PolicyType, RiskProfileLabel, AgeBand

### Page 3: Customer Risk Profile
**Purpose:** Drill-down to individual customer records for action.

Key visuals:
- 4 KPI cards: Healthy / Monitor / High Risk / Critical Loss customer counts
- Detail table: 50k-row customer view with Loss Ratio per Customer and Risk Tier
- Stacked column: Customer count by Risk Tier × PolicyType
- Slicers: Risk Tier, AgeBand, PolicyType, State, CreditBand, Gender

### Page 4: Claims Analysis
**Purpose:** Understand claim patterns and operational signals.

Key visuals:
- 3 KPI cards: Avg Claim Severity, Avg Deductible, Avg Credit Score
- Column chart: Distribution of customers by claim count (0–5)
- Bar chart: Loss Ratio by DeductibleTier
- Scatter: CreditScore vs. Loss Ratio per Customer, color by Risk Tier, size by NoOfClaims
- Line chart: Claims and customers over time

### Page 5: Recommendations
**Purpose:** What should leadership do? Prescriptive analytics output.

Key visuals:
- 3 text boxes: Executive findings in plain English
- Matrix: Recommendations by Segment (dynamic text based on each segment's loss ratio)
- 2 financial impact cards: Expected Premium Uplift ($), Loss Ratio Improvement (pp)
- Dynamic recommendation card that responds to slicer filters
- Segment, Risk Tier slicers for what-if exploration

---

## Key Measures and Calculated Columns

### Calculated Columns (on Insurance table)

All calculated columns are row-level attributes evaluated for each customer.

#### ClaimAmount
Derived because the source data contains claim COUNT but not claim AMOUNT.

```dax
ClaimAmount = 
Insurance[NoOfClaims] 
  * Insurance[CoverageAmount] 
  * 0.0013 
  * (1 + Insurance[RiskProfile] * 0.25)
```

**Why this formula:** Each claim pays a base rate of 0.13% of the customer's coverage amount, adjusted upward by a risk-loading factor. The `(1 + RiskProfile * 0.25)` term ensures customers with `RiskProfile = 0` still receive realistic claim amounts if they filed claims — avoiding a multiplicative zero-out bug where `Risk × Claim = 0` despite the customer having actual claims. The 0.0013 coefficient was calibrated so the book-wide loss ratio lands around 75%, matching real-world insurance norms.

#### Loss Ratio per Customer
```dax
Loss Ratio per Customer = 
DIVIDE(Insurance[ClaimAmount], Insurance[AnnualPremium], 0)
```
**Format:** Percentage, 1 decimal place.
**Why:** The customer-level loss ratio enables per-row risk classification and appears in the Page 3 detail table. Must be a calculated column (not a measure) because visuals need it as a row attribute, not an aggregate.

#### Risk Tier
```dax
Risk Tier = 
VAR LR = Insurance[Loss Ratio per Customer]
RETURN SWITCH(TRUE(),
  LR > 1,   "Critical Loss",
  LR > 0.8, "High Risk",
  LR > 0.6, "Monitor",
  "Healthy"
)
```
**Why:** Classifies every customer into an action-oriented bucket. Thresholds align with industry-standard loss ratio interpretation (60% / 80% / 100%).

#### Customer Recommendation
```dax
CustomerRecommendation = 
VAR LR = Insurance[Loss Ratio per Customer]
RETURN SWITCH(TRUE(),
  LR > 1.0, "Non-renewal review; investigate fraud; reprice +25–40%",
  LR > 0.8, "Increase premium 10–20%; require risk assessment",
  LR > 0.6, "Monitor closely; offer wellness programs",
  "Profitable — target for upsell and retention"
)
```
**Why:** Provides a customer-level action prescription. Appears in the Page 3 detail table so case managers have a ready recommendation next to each customer.

#### AgeBand
```
AgeBand =
  If Age <= 25, "18-25"
  Else if Age <= 35, "26-35"
  Else if Age <= 45, "36-45"
  Else if Age <= 55, "46-55"
  Else if Age <= 65, "56-65"
  Else "65+"
```
*(Built in Power Query as a Conditional Column)*

#### CreditBand
```
CreditBand =
  If CreditScore < 580, "Poor"
  Else if CreditScore < 670, "Fair"
  Else if CreditScore < 740, "Good"
  Else if CreditScore < 800, "Very Good"
  Else "Excellent"
```
**Why:** Credit-based insurance scoring is a well-established industry practice. Banding enables cleaner slicer options and visual aggregation.

#### DeductibleTier
```
DeductibleTier =
  If Deductible < 500, "Low ($100-500)"
  Else if Deductible < 1000, "Medium ($500-1K)"
  Else if Deductible < 1500, "High ($1K-1.5K)"
  Else "Very High ($1.5K-2K)"
```
**Why:** Tiers were calibrated to the actual dataset's deductible range ($100–$2,000), producing 4 balanced cohorts for comparison.

#### RiskProfileLabel
```
RiskProfileLabel =
  If RiskProfile = 0, "Low Risk"
  Else if RiskProfile = 1, "Medium Risk"
  Else if RiskProfile = 2, "High Risk"
  Else "Very High Risk"
```
**Why:** Translates numeric risk codes into human-readable labels for visual display.

---

### Measures (in `_Measures` table)

All measures are aggregated values that respond to filter/slicer context.

#### Core metrics
```dax
Total Customers     = DISTINCTCOUNT(Insurance[CustomerID])
Total Premium       = SUM(Insurance[AnnualPremium])
Total Claims Paid   = SUM(Insurance[ClaimAmount])
Total Claims Count  = SUM(Insurance[NoOfClaims])
```

#### Ratio metrics
```dax
Loss Ratio = DIVIDE([Total Claims Paid], [Total Premium], 0)

Claim Frequency = DIVIDE([Total Claims Count], [Total Customers], 0)

Avg Premium per Customer = DIVIDE([Total Premium], [Total Customers], 0)

Avg Claim Severity = DIVIDE([Total Claims Paid], [Total Claims Count], 0)
```

**Format:** Loss Ratio as Percentage (1 decimal); currency measures as USD (0 decimals).

#### Risk tier counts
```dax
Healthy Count        = CALCULATE([Total Customers], Insurance[Risk Tier] = "Healthy")
Monitor Count        = CALCULATE([Total Customers], Insurance[Risk Tier] = "Monitor")
High Risk Customer Count = CALCULATE([Total Customers], Insurance[Risk Tier] = "High Risk")
Critical Loss Count  = CALCULATE([Total Customers], Insurance[Risk Tier] = "Critical Loss")
Loss Making Customer Count = CALCULATE([Total Customers], Insurance[Risk Tier] = "Critical Loss")
```
**Why these are measures, not columns:** They compute dynamic counts under current filter context. Slicing to a specific PolicyType or State updates these counts automatically.

#### Operational KPIs
```dax
Avg Credit Score = AVERAGE(Insurance[CreditScore])
Avg Deductible   = AVERAGE(Insurance[Deductible])
```

#### Recommendation engine (the differentiator)
```dax
Recommendation = 
VAR LR = DIVIDE([Total Claims Paid], [Total Premium], 0)
RETURN SWITCH(TRUE(),
  LR > 1.0, "Non-renewal review; fraud investigation; reprice +25–40%",
  LR > 0.8, "Premium increase 10–20%; require risk assessment",
  LR > 0.6, "Monitor closely; offer wellness programs",
  "Profitable — target for upsell and retention"
)
```
**Why as a measure (not column):** This enables context-aware recommendations. When a stakeholder filters to a specific segment, the measure recalculates that segment's loss ratio and returns the matching recommendation. The same measure powers both the recommendations matrix (one row per segment) and the dynamic card (current filter selection).

#### Financial projections
```dax
Expected Premium Uplift = 
VAR CriticalLossPremium = CALCULATE([Total Premium], Insurance[Risk Tier] = "Critical Loss")
VAR HighRiskPremium     = CALCULATE([Total Premium], Insurance[Risk Tier] = "High Risk")
RETURN (CriticalLossPremium * 0.30) + (HighRiskPremium * 0.15)
```
**Logic:** Projects annual premium uplift if Critical Loss customers are repriced +30% and High Risk customers are repriced +15% at renewal. Assumes 100% retention — real retention sensitivity discussed in [Methodology Notes](#methodology-notes).

```dax
Loss Ratio Improvement = 
VAR CriticalLossClaims = CALCULATE([Total Claims Paid], Insurance[Risk Tier] = "Critical Loss")
VAR ProjectedReduction = CriticalLossClaims * 0.20
VAR NewPremium = [Total Premium] + [Expected Premium Uplift]
VAR NewClaims  = [Total Claims Paid] - ProjectedReduction
VAR NewLR      = DIVIDE(NewClaims, NewPremium, 0)
RETURN [Loss Ratio] - NewLR
```
**Logic:** Projects the percentage-point improvement in portfolio loss ratio if recommendations are implemented, assuming wellness programs and fraud controls reduce Critical Loss claims by 20%.

#### Gauge support
```dax
Min    = 0
Max    = 1.5
Target = 0.7
```
Used only for the Page 1 gauge visual's reference values.

#### Utility
```dax
ZLast Refresh = "Refreshed: " & FORMAT(TODAY(), "dd-mmm-yyyy")
```
Powers the refresh-date card on every page header. The "Z" prefix pushes it to the bottom of alphabetical field lists.

---

## Key Findings

### 1. Portfolio loss ratio is elevated but not critical
The overall Loss Ratio of approximately **75%** is above the industry-healthy benchmark of 60–70%, but below the 100% break-even crisis line. The book is running thin margins, not losing money outright.

### 2. Loss concentration is more customer-driven than segment-driven
An unexpected finding: loss ratios at the **Segment and PolicyType level cluster tightly between 70–78%**, suggesting those pre-built groupings don't differentiate risk well. The real variation is at the **individual customer level**, where loss ratios range from 0% to well over 200%.

**Implication:** Action should focus on customer-by-customer renewal reviews, not segment-wide repricing.

### 3. Risk Profile drives meaningful variation
Customers in the highest Risk Profile tier (Very High Risk, Risk=3) show loss ratios approximately **50% higher** than the Low Risk tier. Risk Profile is a stronger predictor than age or policy type.

### 4. Approximately 12% of customers are loss-making
The Critical Loss tier (Loss Ratio > 100%) contains roughly 12% of customers but generates a disproportionate share of total claim dollars. This is the priority cohort for action.

### 5. Deductible and Credit Score show expected patterns
Higher deductibles correlate with lower loss ratios (stakeholder can absorb more of small claims). Credit-based tiers show the classic insurance pattern: lower credit tiers, higher loss ratios.

---

## Recommendations

Based on the findings, the dashboard generates four tiered recommendations:

| Tier | Criteria | Action | Financial Lever |
|---|---|---|---|
| **Critical Loss** | Loss Ratio > 100% | Non-renewal review, fraud investigation, reprice +25–40% | 30% premium increase on renewal |
| **High Risk** | Loss Ratio 80–100% | Require risk assessment, reprice +10–20% | 15% premium increase on renewal |
| **Monitor** | Loss Ratio 60–80% | Offer wellness programs, tighten claim review | No price change; claim reduction |
| **Healthy** | Loss Ratio < 60% | Target for upsell and retention | Cross-sell opportunity |

**Projected portfolio impact** (from `Expected Premium Uplift` and `Loss Ratio Improvement` measures):
- Annual premium uplift: visible on Page 5 financial card
- Portfolio loss ratio improvement: visible on Page 5 financial card

**Implementation plan:**
1. **30 days:** Pilot the Critical Loss non-renewal/reprice strategy on the highest-risk 1% of customers
2. **60 days:** Measure retention response, recalibrate pricing sensitivity
3. **90 days:** Scale to full Critical Loss and High Risk cohorts

---

## Methodology Notes

### Claim Amount modeling
The source data contains claim counts but not claim amounts. Claim amounts were modeled as:

```
ClaimAmount = NoOfClaims × CoverageAmount × 0.0013 × (1 + RiskProfile × 0.25)
```

Two design decisions worth highlighting:

**1. Risk as an additive loading, not a multiplier.** An earlier version of the formula used `× RiskProfile` directly. This created a zero-bias bug: roughly 21% of customers have `RiskProfile = 0`, and multiplying by zero zeroed-out their claim amounts even when they had filed actual claims. The corrected formula uses `(1 + RiskProfile × 0.25)` so the risk factor acts as a 0–75% uplift on a baseline severity, never zeroing it out.

**2. Coefficient calibrated for realism.** The 0.0013 coefficient was calibrated so the book-wide loss ratio lands around 75%, matching realistic insurance portfolio norms. Any coefficient can be chosen; what matters is that the relative rankings of segments, customers, and risk tiers preserve the underlying relationships in the source data.

### Why aggregated claim counts, not row-per-visit
The original project scope envisioned one row per hospital visit. The available data uses aggregated claim counts (0–5) per customer. Rather than synthesize fake visit-level data, the analysis uses the aggregated form and is honest about this in the methodology. The insights are equivalent at the customer level.

### Recommendation engine as a measure, not a column
The `Recommendation` measure uses `SWITCH` on the current filter context's Loss Ratio. This means the same measure produces different recommendations when the stakeholder filters to different segments or risk tiers — more flexible than a per-row `CustomerRecommendation` column.

Both exist in the model for different purposes: the column shows each customer's individual recommendation (used in Page 3 detail table), and the measure shows the recommendation for whatever subset the user has filtered (used in Page 5 matrix and dynamic card).

### What this analysis does NOT cover
Honest limitations for future enhancement:
- **Reinsurance and reserves:** Not modeled. A true combined ratio would include these.
- **Expense ratio:** Not modeled. Operational costs would complete the profitability picture.
- **Retention sensitivity:** The projected uplift assumes 100% customer retention. Real renewal retention is typically 75–90%. A retention model would refine the projections.
- **Time-to-claim and reserves development:** Claims data is static; real insurance accounting requires reserve development factors.

---

## Tech Stack

- **Power BI Desktop** (report authoring)
- **Power Query / M** (data transformation)
- **DAX** (measures and calculated columns)
- **CSV** (data ingestion)
- **Row-Level Security** (role-based access, optional)

---

## How to Reproduce

1. **Clone this repository**
2. **Open** `InsuranceClaimDashboard.pbix` in Power BI Desktop
3. **Refresh data source:** Home → Transform Data → point to `data_synthetic.csv` in the `/data` folder
4. **Close & Apply** to reload the model
5. Open each page and interact with slicers to explore

**Alternate view:** A published version is available on Power BI Service — see the Service link in the repo's About section.

---
## Contact

**Author:** Adarsh Upadhyay
**Role:** Data Analyst
**Email:** *upahdhyay17ab@gmail.com*
**LinkedIn:** *https://www.linkedin.com/in/adarsh-upadhyay-10a579104/*

This project is part of a portfolio demonstrating end-to-end Power BI development, DAX modeling, and stakeholder-facing analytics storytelling.
