# Methodology & Technical Notes

Supplementary technical documentation for `InsuranceClaimDashboard.pbix`.

---

## 1. Data Source Selection

### Two candidate files were evaluated

The project initially considered combining two sources:
- `data_synthetic.csv` — 53,503 rows, 30 columns including CustomerID, Premium, Coverage, Claims, Risk Profile, Segment
- `insurance_dataset.csv` — 13,000 rows, 7 columns: Age, Gender, Income, Marital Status, Education, Occupation, Claim_Amount

**Decision:** Use `data_synthetic.csv` exclusively.

**Reasoning:** The second file had no `CustomerID` or any other joinable key to link records across files. Its attributes (demographics + Claim_Amount) were entirely a subset of what the primary file already contained. Combining files would have added complexity without analytical gain.

This is a deliberate data source decision — in real analytical work, choosing the right source from multiple candidates is part of the analyst's judgment.

---

## 2. Schema Transformation in Power Query

The source CSV had 30 columns, many of which were cryptic placeholder fields. The transformation pipeline:

### Columns renamed for clarity
Original column names like `Claim History` were renamed to `NoOfClaims`; `Geographic Information` to `State`; etc.

### Columns removed (non-analytical)
- `Behavioral Data` — cryptic placeholder
- `Purchase History` — duplicate information
- `Location` (numeric) — unmappable to geography
- `Interactions with Customer Service` — not relevant to loss analysis
- `Customer Preferences`, `Preferred Communication Channel`, `Preferred Contact Time`, `Preferred Language` — marketing metadata, not risk-relevant
- `Driving Record` — auto-insurance field not used here
- `Life Events` — too sparse to analyze

### Data type corrections
- `PolicyStartDate`, `PolicyRenewalDate` → Date type (with locale-aware parsing for DD-MM-YYYY format)
- `CustomerID` → Text (to prevent accidental aggregation)
- `AnnualPremium`, `CoverageAmount`, `Deductible`, `Income` → Decimal Number
- `Age`, `NoOfClaims`, `RiskProfile`, `PreviousClaims`, `CreditScore` → Whole Number

### Calculated columns added in Power Query
- `AgeBand` (conditional column based on Age)
- `CreditBand` (conditional column based on CreditScore)
- `DeductibleTier` (conditional column based on Deductible)
- `RiskProfileLabel` (conditional column translating 0–3 to "Low Risk" through "Very High Risk")

### Sort helper columns
Because Power BI sorts categorical axes alphabetically by default, numeric sort helper columns were added:
- `AgeBand_Sort` (1–6)
- `CreditBand_Sort` (1–5)
- `DeductibleTier_Sort` (1–4)

Each tier column is then configured via **Column tools → Sort by column** to use its helper column, producing logical ordering (Low → High) in all visuals.

---

## 3. Date Dimension Table

A standalone Date table was created using M code rather than derived from `PolicyStartDate`. This enables clean time intelligence (YoY, MoM, YTD) and avoids the common pitfall of time calculations failing when the underlying date column has gaps.

```m
let
    StartDate = #date(2018, 1, 1),
    EndDate = #date(2026, 12, 31),
    Dates = List.Dates(StartDate, Duration.Days(EndDate - StartDate) + 1, #duration(1,0,0,0)),
    T = Table.FromList(Dates, Splitter.SplitByNothing(), {"Date"}),
    Typed = Table.TransformColumnTypes(T, {{"Date", type date}}),
    Year = Table.AddColumn(Typed, "Year", each Date.Year([Date]), Int64.Type),
    Quarter = Table.AddColumn(Year, "Quarter", each "Q" & Text.From(Date.QuarterOfYear([Date]))),
    Month = Table.AddColumn(Quarter, "Month", each Date.Month([Date]), Int64.Type),
    MonthName = Table.AddColumn(Month, "MonthName", each Date.MonthName([Date])),
    MonthYear = Table.AddColumn(MonthName, "MonthYear", each Date.ToText([Date], "MMM-yyyy"))
in
    MonthYear
```

The table was **marked as a Date Table** via right-click → Mark as Date Table → Date column.

---

## 4. ClaimAmount Modeling — Full Derivation

This is the most analytically consequential modeling decision in the project.

### Problem statement
The source data contains `NoOfClaims` (count of claims: 0, 1, 2, 3, 4, or 5) but no `ClaimAmount` (dollar value of claims). Loss Ratio analysis requires claim amounts.

### Initial naive formula (rejected)
```
ClaimAmount_v1 = NoOfClaims × CoverageAmount × 0.015 × RiskProfile
```

### Bug discovered: zero-bias
`RiskProfile` in the data ranges from 0 to 3. Roughly 21% of customers have `RiskProfile = 0`. Under the naive formula, any customer with `RiskProfile = 0` received `ClaimAmount = 0` regardless of how many claims they actually filed. Over 9,000 customers with claims but `RiskProfile = 0` would have been assigned $0 in claims — a systematic error.

### Coefficient miscalibration
A second issue: with coefficient `0.015`, the book-wide loss ratio computed to approximately 870%, which is nonsensical. Real insurance books rarely exceed 150% loss ratio.

### Corrected formula
```
ClaimAmount = NoOfClaims × CoverageAmount × 0.0013 × (1 + RiskProfile × 0.25)
```

**Change 1: Additive risk loading.** Replacing `× RiskProfile` with `(1 + RiskProfile × 0.25)` means:
- RiskProfile = 0 → multiplier 1.00 (baseline severity)
- RiskProfile = 1 → multiplier 1.25
- RiskProfile = 2 → multiplier 1.50
- RiskProfile = 3 → multiplier 1.75

The risk factor now acts as a 0–75% uplift on a baseline, never zeroing it out.

**Change 2: Coefficient recalibration.** The 0.0013 coefficient was solved algebraically by targeting a book-wide loss ratio of approximately 75%:

```
Target Total Claims = Total Premium × 0.75
0.75 × Σ(AnnualPremium) = Σ(NoOfClaims × CoverageAmount × k × (1 + RiskProfile × 0.25))
Solve for k → k ≈ 0.0013
```

### Validation
After applying the corrected formula:
- Book-wide Loss Ratio: approximately 75% (realistic)
- Zero-bug check: 0 customers with claims > 0 and ClaimAmount = 0
- Risk Profile still drives variation (Risk 3 loss ratio > Risk 0 loss ratio)
- 52% of customers in Healthy tier, 12% in Critical Loss tier

---

## 5. Measure vs. Calculated Column Design

Power BI offers two places to create derived values. Choosing correctly is critical.

### Rule of thumb applied in this model
- **Many rows → one number** (for a chart, card, or total): **Measure**
- **Each row gets its own value** (appears in a table, filter, or slicer): **Calculated Column**

### Examples from this model

| Field | Type | Reason |
|---|---|---|
| ClaimAmount | Calculated Column | Each customer has their own value |
| Loss Ratio per Customer | Calculated Column | Appears per row in Page 3 detail table |
| Risk Tier | Calculated Column | Used as slicer and legend |
| CustomerRecommendation | Calculated Column | Per-customer action label |
| Loss Ratio | Measure | Aggregates across filter context |
| Total Premium | Measure | Aggregates across filter context |
| Recommendation | Measure | Generates segment/portfolio-level recommendation dynamically |

### Why Recommendation is a measure (advanced)

The `Recommendation` measure is a deliberate design choice. A measure recomputes under each filter context, so when the user filters to a specific segment or policy type, the measure computes that subset's loss ratio and returns the matching recommendation. A calculated column could not do this.

This also explains why both `CustomerRecommendation` (column) and `Recommendation` (measure) coexist:
- `CustomerRecommendation` shows a specific customer's recommendation in the Page 3 detail table
- `Recommendation` shows the current filter selection's recommendation in Page 5's dynamic card

---

## 6. Why Loss Ratio Thresholds Match Industry Benchmarks

The SWITCH thresholds for Risk Tier and Recommendation are:

| Threshold | Tier | Industry Meaning |
|---|---|---|
| Loss Ratio > 100% | Critical Loss | Claims exceed premium; book is losing money |
| Loss Ratio 80–100% | High Risk | Approaching break-even; profitability at risk |
| Loss Ratio 60–80% | Monitor | Above industry-healthy benchmark |
| Loss Ratio < 60% | Healthy | Profitable book |

These thresholds are standard in the insurance industry. Major insurers publicly target combined ratios below 100% (loss ratio + expense ratio). Loss ratio alone below 70% is typically considered a healthy book.

---

## 7. Visual Design Decisions

### Color palette
A 6-color palette was chosen to support the risk/health narrative:

| Purpose | Hex | Use |
|---|---|---|
| Primary Navy | `#1F4E78` | Headers, titles, neutral bars |
| Accent Blue | `#2E75B6` | Secondary bars, KPI backgrounds |
| Success Green | `#548235` | Healthy tier, profitable segments |
| Warning Orange | `#ED7D31` | Monitor tier, caution flags |
| Alert Red | `#C00000` | High Risk, Critical Loss |
| Dark Red | `#833C0C` | Critical Loss (deepest severity) |

Color is used consistently across all 5 pages: red always means "problem," green always means "healthy."

### Takeaway-style titles
Chart titles were written to describe the insight, not the chart:
- Instead of *"Loss Ratio by PolicyType"* → *"Group policies drive highest loss ratios in the book"*
- Instead of *"Customer Distribution by Risk Tier"* → *"12% of customers are loss-making"*

This elevates the dashboard from student project quality to stakeholder-ready output.

### Sync slicers
`Year` and `PolicyType` slicers are synced across all 5 pages (View → Sync slicers). Page-specific slicers (Risk Tier, AgeBand, CreditBand, Gender, State) appear only on Page 3 where their analytical relevance is highest.

---

## 8. Known Limitations

Documented honestly for portfolio review:

### Matrix uniformity finding
The PolicyType × AgeBand matrix on Page 2 shows loss ratios clustering tightly in the 69–78% range across all combinations. This surprised the analysis. It means **PolicyType and AgeBand are not the drivers of risk variation in this dataset**. The real driver is Risk Profile and, at a finer level, the individual customer. This finding redirected the recommendation focus from segment-wide pricing action to customer-by-customer renewal review.

### Segment uniformity
Similarly, pre-built Segments (Segment1–Segment5) show tight loss ratio variation. The Segment-level recommendations in Page 5 reflect this — most segments fall into the "Monitor" tier under absolute-threshold logic. An alternative approach using relative deviation from portfolio average was explored.

### Retention assumption
The `Expected Premium Uplift` measure assumes 100% customer retention after repricing. Real-world insurance retention after a 30% premium increase is typically 50–75%. A retention-adjusted uplift would be a natural Phase 2 enhancement.

### Claim amount derivation
Because ClaimAmount is modeled rather than observed, the absolute dollar values should be treated as directional rather than precise. The relative rankings (which segments are worse, which customers are loss-making) are robust; the specific dollar impacts are approximations.

---

## 9. Interview Talking Points

For portfolio interviews, this project provides answers to common questions:

**Q: Tell me about a time you caught a bug in your own analysis.**
*"In my claim amount modeling, I had a multiplicative formula where risk profile zeroed out actual claim amounts for customers with Risk Profile = 0 — about 21% of the dataset. I redesigned the formula as a baseline-plus-loading structure, which also required recalibrating the coefficient to produce a realistic portfolio loss ratio."*

**Q: How do you decide between a measure and a calculated column?**
*"If the formula needs to aggregate across rows or respond to filters, it's a measure. If it's a per-row attribute used in a table, slicer, or visual legend, it's a calculated column. In this project, Loss Ratio per Customer is a column because it appears per row in Page 3, but Loss Ratio is a measure because it aggregates across whatever filter is applied."*

**Q: How did you handle the situation where your data didn't show the pattern you expected?**
*"The PolicyType × AgeBand matrix came out nearly uniform — 69–78% across all combinations. Rather than force a finding, I flagged this as an insight in itself: our pre-built segmentation doesn't differentiate risk. The real variation was at the individual customer level, which directed the recommendation strategy toward per-customer renewal reviews instead of segment-wide repricing."*

**Q: What would you do differently with more time?**
*"Three things: (1) retention-adjusted uplift projections, (2) a fraud detection layer on the Critical Loss tier using claim frequency and credit score anomalies, (3) reserve development modeling for a proper combined ratio."*

---

## 10. Files in This Repository

```
/
├── README.md                         (main project documentation)
├── METHODOLOGY.md                    (this file — technical deep dive)
├── InsuranceClaimDashboard.pbix      (the Power BI report)
├── /data/
│   └── data_synthetic.csv            (source dataset)
└── /screenshots/
    ├── page1_executive_summary.png
    ├── page2_loss_ratio_deep_dive.png
    ├── page3_customer_risk.png
    ├── page4_claims_analysis.png
    └── page5_recommendations.png
```
