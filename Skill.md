---
name: outlier-page-map
description: Use this skill whenever someone asks about the Process Outlier Correction page, outlier correction, demand-history spikes or dips, manual correction, corrected history, or the SHC000/SHC100/SHC101 logic in the 5.2. Demand Analysis model. Build a read-only, source-grounded page map that explains the Global (GL), Edit, and Effective (EF) settings layers, what the relevant line items represent, and how a weekly outlier flag is produced.
compatibility: Requires Anaplan catalog tools and access to the 5.2. Demand Analysis model. This skill is read-only: do not write cells, run imports/processes, or otherwise alter model data.
---

# Outlier Page Map

## Purpose and guardrails

Explain the **Process Outlier Correction** page in business terms using the model structures below. Treat the cataloged module and line-item metadata as the source of truth; distinguish a formula-backed fact from a user-facing interpretation. Keep the work read-only. Do not infer current setting values from formulas or metadata alone; use reads only if the user separately requests actual values and the needed dimensions are known.

Use the planner's vocabulary explicitly: **outlier**, **spike**, **dip**, **manual correction**, and **corrected history**. Explain technical formula names in plain language, but preserve exact Anaplan line-item names in backticks so planners or model builders can locate them.

## Model and evidence

Target model: workspace **Agentic Enterprise - Supply Chain**, model **5.2. Demand Analysis** (production model). The relevant modules were discovered and read using `catalog_line_items(detail="full")`:

- `SHC000 - Statistics and History Correction Parameters` (SHC000): 68 line items; product/customer parameter settings and effective flags.
- `SHC100  - Statistics and History Correction` (SHC100): 110 line items; weekly history, statistical limits, flags, manual and system corrections, and corrected history.
- `SHC101 - History Correction Summary - Annual` (SHC101): 10 line items; annual rollups and correction summaries.

SHC100's native time grain is **Week**; SHC101's is **Year**. Both use `HLC080:CUST_IsForecast` and `HLP080:PROD_IsForecast`; SHC100 also has native Time. Do not present SHC101 annual summary outputs as weekly flags.

## The three settings layers: GL, Edit, EF

Explain these as a settings-resolution stack, not three independent algorithms:

1. **GL — Global layer.** In SHC000, `GL ...` settings are global defaults sourced from `'PRM120 - Global Parameters: Effective'`. The important defaults are `GL Correct From`, `GL Outlier Magnitude`, `GL Correct What`, `GL History Correction Adjustment` (the global Correct To), `GL Period by Period`, and the promotion-outlier settings `GL Correct Promotion Outliers?` / `GL Correction of promotion outliers`.
2. **Edit layer.** The `Edit Parameters` section is documented in SHC000 as product-level parameter settings, dimensional by customer and product. Editable inputs include `Correct From`, `Outlier Magnitude`, `Correct What`, `Correct To`, `Period by Period`, and the promotion-outlier choices. These are local overrides, subject to the valid customer/product-combination access driver. A literal zero in `Outlier Magnitude` is treated as “use the global default,” not as a zero-width threshold (the line item notes call zero erroneous).
3. **EF — Effective layer.** `EF ...` line items resolve the usable setting for a valid combination: generally use an entered Edit setting when present, otherwise fall back to GL. `EF Outlier Magnitude` uses GL magnitude when the Edit magnitude equals zero and is gated by `ValidCombinationCounter`. `EF Correct From` and `EF Correct To` also apply correction eligibility/substitution rules (for example, insufficient history or a non-effective current period can force the correction basis Off; Trend Seasonal substitutions can apply when decomposition history is insufficient). `EF Correct What`, `EF Period by Period`, and promotion options resolve the corresponding edit/global choice. The effective layer is what the weekly calculation consumes.

Name the editable and effective variants precisely: SHC000 uses `Correct From` / `EF Correct From`, `Correct What` / `EF Correct What`, `Correct To` / `EF Correct To`, `Outlier Magnitude` / `EF Outlier Magnitude`, `Period by Period` / `EF Period by Period`, plus `Correct Promotion Outliers?` / `EF Correct Promotion Outliers?` and `Correction of promotion outliers` / `EF Correction of promotion outliers`.

## How a week gets flagged

Describe the logic as this sequence, with qualifications:

1. **Eligible series and history:** `SHC100.ISValidforCorrection` requires a valid customer/product combination, an enabled correction basis (`SHC000.ISCorrectionBasisOn`), enough effective history (`SHC000.ISEnoughHistoryforCorrection`), and a history-effective product/week (`EFF200 ... ISProdEffective (HIST)`). Thus a numerical extreme is not automatically an actionable outlier.
2. **Build the reference and limits:** SHC100 derives a mean (`Calc Mean (U)` / `Mean (U)`), a regression trend (`Trend (U)`), and, where applicable, a Trend Seasonal series (`Trend Seasonal (U)`). `Upper Limit` and `Lower Limit` are based on the selected correction basis and the effective outlier magnitude: mean ± standard deviation × magnitude, trend ± trend RMSE × magnitude, or trend-seasonal ± residual RMSE × magnitude. The magnitude is the tolerance multiplier, not a forecast amount.
3. **Compare the actual history:** `ISUpperLimitBreached` is true when eligible `Chained History (U)` is above `Upper Limit`; `ISLowerLimitBreached` is true when eligible history is below `Lower Limit`.
4. **Apply the planner's detection choice:** `ISOutsideEffectiveLimits` uses `EF Correct What` to test both sides (`Both`), only low-side breaches (`Dips`), or only high-side breaches (`Spikes`). In planner language, an unusually high week can be a **spike**, an unusually low week a **dip**, and either can be an **outlier** if it is outside the effective limits and passes the eligibility/selection tests.
5. **Apply correction mode:** `ISCorrectionNeeded` combines `CALC1ISCorrectionNeeded` (period-by-period mode selected, that week's `Period by Period` toggle on, and limits breached) and `CALC2ISCorrectionNeeded` (period-by-period mode not selected, and limits breached), with correction basis enabled. Therefore a flagged threshold breach and a week selected for correction are related but not identical concepts; describe the relevant boolean explicitly.
6. **Show the result and manual choice:** `System Corrected History (U)` applies the configured system correction when needed (e.g., mean, trend, a breached limit, or Trend Seasonal); otherwise it retains `Chained History (U)`. `Manual Correction (U)` is a planner-entered weekly override. `Corrected History (U)` uses the manual value whenever it is nonzero; otherwise it uses `System Corrected History (U)`. This is the core distinction between **manual correction**, system correction, and final **corrected history**.

Do not claim that every limit breach is corrected: mention product/week eligibility, the chosen spike/dip scope, correction-basis settings, and period-by-period mode. Do not call annual SHC101 measures weekly flags.

## Line-item map: SHC000

SHC000 is a parameter-resolution and eligibility module at customer/product combination grain. Heading/separator rows (`Global Parameters`, `Edit Parameters`, `Effective Parameters`, `Formatting`, `Booleans`, `Indicators`, `Error Alerts`, `Outlier Identification`, `Outlier Correction`, `EF Outlier Identification`, `EF Outlier Correction`, `Demand Sensing`) are section labels, not measures.

### Global, edit, and effective settings

| Line item | What it holds |
|---|---|
| `GL Correct From` | Global default history-correction start/basis selection, sourced from PRM120. |
| `GL Outlier Magnitude` | Global default numerical tolerance multiplier. |
| `GL Correct What` | Global default detection scope: both, dips, or spikes. |
| `GL History Correction Adjustment` | Global default correction destination/adjustment choice (the global `Correct To`). |
| `GL Period by Period` | Global default yes/no for period-by-period correction. |
| `GL Correct Promotion Outliers?` | Global default switch for promotion outlier treatment. |
| `GL Correction of promotion outliers` | Global default promotion correction basis. |
| `Correct From` | Editable customer/product correction start/basis. |
| `Outlier Magnitude` | Editable customer/product threshold multiplier; zero is treated as fallback to the global value. |
| `Correct What` | Editable customer/product scope (`Both`, `Dips`, `Spikes`). |
| `Correct To` | Editable customer/product correction destination/adjustment. |
| `Period by Period` | Editable customer/product yes/no mode setting. |
| `Correct Promotion Outliers?` | Editable customer/product promotion-outlier switch. |
| `Correction of promotion outliers` | Editable customer/product promotion correction basis. |
| `CALCEF Correct From` | For a valid combination, uses Edit `Correct From` when supplied, else GL `Correct From`. |
| `EF Correct From` | Effective correction basis, including Off-for-ineligible conditions and Trend Seasonal substitution rules. |
| `EF Outlier Magnitude` | Effective threshold multiplier; falls back to GL when the Edit input is zero and is gated by valid combination. |
| `EF Correct What` | Effective both/dips/spikes choice, preferring Edit, else GL. |
| `CALCEF Correct To` | For a valid combination, uses Edit `Correct To` when supplied, else the global adjustment. |
| `EF Correct To` | Effective destination/adjustment, including possible Trend Seasonal substitution. |
| `EF Period by Period` | Effective period-by-period choice, preferring Edit, else GL. |
| `EF Correct Promotion Outliers?` | Effective promotion-outlier on/off choice. |
| `EF Correction of promotion outliers` | Effective promotion correction basis, preferring Edit, else GL. |

### SHC000 support flags, indicators, and messages

| Line item | What it holds |
|---|---|
| `Period by Period Global Yes` | Numeric 1/0 representation of the GL period-by-period setting, gated by valid-combination counter. |
| `Period by Period User Yes` | Numeric 1/0 representation of the Edit period-by-period setting, gated by valid-combination counter. |
| `Period by Period Effective Yes` | Numeric 1/0 representation of the effective period-by-period setting, gated by valid-combination counter. |
| `NumberofProductEffectivePeriods` | Count of product-effective history periods used for the history-sufficiency checks. |
| `ISEffectiveCurrentPeriod` | Whether the product is effective in the current period. |
| `ISNotEffectiveCurrentPeriod` | Inverse current-period effectiveness check. |
| `ISEnoughHistoryforCorrection` | Effective history count meets the global minimum history requirement. |
| `ISNotEnoughHistoryforCorrection` | Effective history count is below the minimum required for correction. |
| `ISEnoughHistoryForDemandSensing` | Inverse of the demand-sensing history-shortfall condition. |
| `ISNotSufficientHistoryforDecompAnalysis` | Pulls the decomposition-analysis insufficient-history flag. |
| `ISNOTEnoughHistoryForDemandSensing` | Demand-sensing analysis horizon exceeds the available effective history. |
| `ISCorrectFromTrendSeasonalButNoDecomp` | The selected effective start basis is Trend Seasonal despite insufficient decomposition history. |
| `ISCorrectToTrendSeasonalButNoDecomp` | The selected effective destination is Trend Seasonal despite insufficient decomposition history. |
| `ISCorrectionBasisOff` | Effective `Correct From` equals Off. |
| `ISCorrectionBasisOn` | Valid combination and correction basis is not Off. |
| `ISCorrectionBasisFromMean` | Effective start basis is From Mean. |
| `ISCorrectionBasisFromTrend` | Effective start basis is From Trend. |
| `ISCorrectionBasisFromTrendSeasonal` | Effective start basis is From Trend Seasonal. |
| `ISCorrectionAdjToLimit` | Effective correction destination is To Limit. |
| `ISCorrectionAdjToMean` | Effective correction destination is To Mean. |
| `ISCorrectionAdjToTrend` | Effective correction destination is To Trend. |
| `ISCorrectionAdjToTrendSeasonal` | Effective correction destination is To Trend Seasonal. |
| `ISPeriodByPeriodYes` | Effective period-by-period setting is Yes. |
| `ISPeriodByPeriodNo` | Effective period-by-period setting is No. |
| `ISCorrectionTypeBoth` | Effective scope is Both. |
| `ISCorrectionTypeDips` | Effective scope is Dips. |
| `ISCorrectionTypeSpikes` | Effective scope is Spikes. |
| `INDCorrectToSubstituted` | Numeric indicator that effective Correct To differs from the calculated pre-substitution choice. |
| `INDCorrectFromSubstituted` | Numeric indicator that effective Correct From differs from the calculated pre-substitution choice. |
| `IND History Correction Alert Text` | Numeric indicator for history-correction alert conditions. |
| `IND Demand Sensing Alert Text` | Numeric indicator for demand-sensing alert conditions. |
| `History Correction Alert Text` | Human-readable status: product not currently effective, insufficient history, Trend Seasonal history issue, or history correction possible. |
| `Demand Sensing Alert Text` | Human-readable demand-sensing eligibility/history status. |

## Line-item map: SHC100

SHC100 is the weekly calculation and correction module. Its line items are listed below by function; headings are included to orient the Blueprint but have no values of their own.

### History and baseline/statistics calculations

| Line item(s) | What they hold |
|---|---|
| `Statistics`, `Stat Inputs`, `Standard Deviation Calcs`, `Linear Regression Calcs`, `Standard Dev from Trend`, `Standard Dev from Residual`, `History Correction`, `History Correction Inputs`, `Hist Correction Calculations`, `Corrected Standard Dev and Error Calcs`, `Corrected History Linear Regression Calcs`, `Corrected History Stats`, `Demand Sensing`, `Demand Analyst KPIs`, `Graph Only Outputs`, `Booleans` | Section headings / display group labels, not data measures. |
| `Chained History (U)` | Base weekly history quantity from the demand-chaining summary. |
| `One` | Constant 1 by week for sample-count calculations. |
| `Calc Mean (U)` | Average chained history over the product's effective historical window. |
| `Mean (U)` | Weekly mean reference, distributed across time. |
| `x - x bar`, `x-x bar2`, `Calc TimeSum x-x bar2`, `Calc TimeSum n-1`, `Standard Deviation`, `C-Var (Mean)` | Mean-deviation, squared-deviation, time-summed sample variance, standard deviation, and coefficient of variation calculations. |
| `xy`, `x2`, `sum of x`, `sum of y`, `sum of xy`, `sum of x2`, `CALC1m`, `CALC2m`, `m`, `CALC1b`, `CALC2b`, `b` | Regression intermediate products/sums and slope (`m`) / intercept (`b`) calculations over the effective history window. |
| `Trend (U)` | Weekly regression trend reference. |
| `y = mx + b` | Text rendering of the regression equation. |
| `x-xbar2  (Trend)`, `Calc RMSE (Trend)`, `RMSE (Trend)`, `C-Var (Trend)` | Trend residual squared error, RMSE, and coefficient of variation calculations. |
| `Trend Seasonal (U)` | Weekly trend-seasonal reference pulled from decomposition analysis. |
| `RMSE (Residual)` | Residual RMSE used for trend-seasonal limits. |
| `Selected History (U)` | Selected weekly history from the selected-history module. |

### Detection and correction logic

| Line item | What it holds |
|---|---|
| `Period by Period` | Editable weekly on/off switch for a week when the effective mode is period-by-period; note says it controls correction on/off when that criterion is selected. |
| `ISValidforCorrection` | Week-level eligibility: valid combination, correction basis on, enough history, and history-effective period. |
| `Upper Limit` | Effective upper statistical boundary, selected-basis reference plus applicable dispersion × effective outlier magnitude. |
| `Lower Limit` | Effective lower statistical boundary, selected-basis reference minus applicable dispersion × effective outlier magnitude. |
| `ISUpperLimitBreached` | True when an eligible week's chained history is above the upper limit. |
| `ISLowerLimitBreached` | True when an eligible week's chained history is below the lower limit. |
| `ISOutsideEffectiveLimits` | Breach flag after applying the effective Both/Dips/Spikes selection. |
| `ISOutsideEffectiveandPeriodbyPeriod` | Outside-limits flag combined with effective period-by-period mode. |
| `CALC1ISCorrectionNeeded` | Correction-needed path when effective period-by-period is Yes and the week's `Period by Period` switch is on, with basis enabled and limits exceeded. |
| `CALC2ISCorrectionNeeded` | Correction-needed path when effective period-by-period is No and limits are exceeded, with basis enabled. |
| `ISCorrectionNeeded` | OR of the two correction-needed paths. |
| `System Corrected History (U)` | System-selected corrected history: applies correction only when needed, selecting configured correction target/limit; otherwise keeps chained history. Promotion-period handling may take precedence. |
| `Manual Correction (U)` | Planner-entered weekly manual correction value; zero means no manual override in the final selection formula. |
| `Corrected History (U)` | Final weekly corrected history: Manual Correction when nonzero, otherwise System Corrected History. |
| `Corrected History (R)` | Corrected units converted to revenue using average selling price. |
| `System Corrected Promotion History (U)` | Promotion-specific system-corrected history choice. |
| `ISPromotionPeriod` | Historical promotion-period flag, gated by effective promotion-outlier setting and positive promotional impact. |
| `Promotion` | Promotion adjustment amount in a promotion period, floored at zero. |
| `Outlier Range` | Upper limit minus lower limit. |
| `History Correction Comments` | Section label for comments. |
| `Comments` | Editable free-text comments at customer/product/week grain. |

### Corrected-history statistical diagnostics

| Line item(s) | What they hold |
|---|---|
| `x-xbar 2 (Corrected)`, `CALC1Standard Deviation (Corrected)`, `CALC2Standard Deviation (Corrected)`, `Mean (Corrected)`, `Standard Deviation (Corrected)`, `C-Var (Corrected Mean)` | Squared deviations and derived mean, standard deviation, and coefficient-of-variation diagnostics on corrected history. |
| `xy (Corrected)`, `sum of y (Corrected)`, `sum of xy (Corrected)`, `CALC1m (Corrected)`, `CALC2m (Corrected)`, `m (Corrected)`, `CALC1b (Corrected)`, `CALC2b (Corrected)`, `b (Corrected)`, `Corrected Trend (U)` | Regression inputs, corrected-history slope/intercept, and corrected trend series. |
| `x-xbar 2 (Corrected Trend)`, `CALC1Standard Deviation (Corrected Trend)`, `CALC2Standard Deviation (Corrected Trend)`, `Mean (Corrected Trend)`, `Standard Deviation (Corrected Trend)`, `C-Var (Corrected Trend)` | Corrected-trend deviation, mean, standard deviation, and variability diagnostics. |
| `Mean (U).`, `Trend (U).`, the visually blank/hidden-named line item with formula `Lower Limit` | Display/helper line items that expose the mean, trend, or lower-limit series for presentation/access; the hidden-named item is a formula alias for `Lower Limit`, not a separate business concept. |
| `History Sum for Effective Window`, `Regression Sum for Effective Window`, `Effective Window` | Effective-window history/trend summaries used for effective-window reporting/graph support. |

### Demand-sensing and count support

| Line item(s) | What they hold |
|---|---|
| `ISDemandSensingTimeRange` | Periods in the configured demand-sensing analysis window that also meet effective-history criteria. |
| `ISProductEffective (HIST)` | Weekly history-period product-effectiveness flag. |
| `ISProcess` | Valid customer/product combination indicator for processing. |
| `DemandSensingUpperBreach`, `DemandSensingLowerBreach` | Numeric upper/lower breach markers within the demand-sensing time range. |
| `DemandSensingUpperCount`, `DemandSensingLowerCount` | Breach counts summarized across time. |
| `CountOutsideEffectiveLimits` | Numeric 1/0 count-support version of `ISOutsideEffectiveLimits`. |
| `SUMCommercialPlanningProcessLevel`, `SUMExportToStatProcessLevel` | Valid-combination counters used by export/process-level checks. |

## Line-item map: SHC101 annual summary

SHC101 contains annual summary measures sourced from SHC100. It is not the weekly page's detection engine.

| Line item | What it holds |
|---|---|
| `Inputs`, `Outputs` | Section headings. |
| `ISHistoryCurrent` | Whether the year is the current-history year according to the base-calendar time module. |
| `Chained History (U)` | Annual rollup of the original chained history. |
| `System Corrected History (U)` | Annual rollup of the system-corrected history. |
| `Final Corrected History (U)` | Annual rollup of final corrected history (manual correction takes precedence in SHC100). |
| `Net System Correction` | Annual system-corrected history less annual chained history. |
| `% System Correction Magnitude` | Net system correction divided by chained history, formatted as a percentage. |
| `Net Total Correction` | Annual final corrected history less annual chained history. |
| `% Final Correction Magnitude` | Net total correction divided by chained history, formatted as a percentage. |

## Response format when applying this skill

Use this structure unless the user asks for another format:

1. **Page purpose** — one short paragraph in planner language.
2. **Settings layers (GL → Edit → EF)** — summarize defaults, local overrides, and resolved settings; give examples for magnitude, correction type, and period-by-period mode.
3. **How a week gets flagged** — numbered chain from eligibility through limits, spike/dip selection, and correction-needed logic.
4. **What planners see/change** — distinguish base history, system correction, `Manual Correction (U)`, and final `Corrected History (U)`; mention comments.
5. **Line-item map by module** — identify SHC000 as parameters, SHC100 as weekly calculation, SHC101 as annual summary. Use concise grouped descriptions from this reference and include exact line-item names for traceability.
6. **Caveat** — distinguish “outside limits / outlier flag” from “correction applied,” and state that manual correction overrides the system result when nonzero.

Always include the words **outlier**, **spike**, **dip**, **manual correction**, and **corrected history** in the explanatory prose.