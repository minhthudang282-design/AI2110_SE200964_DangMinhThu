# Cross-Battery State-of-Health Estimation for UAV-Grade Lithium-Ion Cells

## Project Overview

This project investigates battery State of Health (SOH) degradation and cross-battery SOH estimation using the NASA PCoE Randomized Battery Usage dataset.

The study has three main goals:

1. Construct a reproducible SOH target and feature table from the NASA randomized battery experiments.
2. Evaluate whether SOH can generalize to battery cells that were not used for model training.
3. Quantify uncertainty, End-of-Life (EOL) behavior, feature importance, and deployment trade-offs.

The analysis covers battery cells `RW1` to `RW28`.

---

## Final Project Structure

```text
PJ_ADY/
│
├── .venv/                         # Local Python environment on the working machine
│
├── data/
│   ├── raw/                       # Original NASA Randomized Battery Usage data
│   └── processed/
│       ├── rw_battery.parquet
│       ├── feat.parquet
│       ├── manifest.json
│       └── ady.duckdb
│
├── report/
│   ├── audit_evidence/
│   ├── profile/
│   ├── fig_rq1_*.png
│   ├── fig_rq2_*.png
│   ├── fig_rq3_*.png
│   └── table_*.csv
│
├── sql/
│   └── queries.sql
│
├── Notebook_A_Nhom7_RW_Battery_NASA.ipynb
├── Notebook_B_Nhom7_RW_Battery_NASA.ipynb
├── requirements.txt
├── .gitignore
└── README.md
```

Notes:

- The two final notebooks are intentionally kept at the project root because their relative paths assume execution from `PJ_ADY/`.
- The local `.venv` is useful on the original machine, but it should not be copied as a portable environment. Another user should create a new virtual environment from `requirements.txt`.
- Legacy notebook copies, archive folders, empty source folders, and development-only cleanup cells have been removed from the final project.

---

## Verified Environment

The final notebooks were verified with:

```text
Python          3.10.11
NumPy           2.2.6
pandas          2.3.3
SciPy           1.15.3
Matplotlib      3.10.0
DuckDB          1.5.5
scikit-learn    1.7.2
PyArrow         25.0.1
ydata-profiling 4.18.4
SHAP            0.49.1
```

The exact dependencies are recorded in:

```text
requirements.txt
```

### Setup on another machine

From the project root:

```powershell
python -m venv .venv
.\.venv\Scripts\Activate.ps1
python -m pip install --upgrade pip
pip install -r requirements.txt
```

In VS Code, select:

```text
PJ_ADY\.venv\Scripts\python.exe
```

Recommended Python version:

```text
Python 3.10.x
```

---

## Raw Data

The NASA Randomized Battery Usage dataset should be placed under:

```text
data/raw/
```

Notebook A searches recursively for:

```text
RW*.mat
```

A correct load should recover all 28 battery cells:

```text
RW1 ... RW28
```

If a submission platform has a strict file-size limit, the raw dataset may be omitted from the submitted archive and obtained separately before rerunning Notebook A.

---

# Notebook A — Data Engineering and RQ1

Main notebook:

```text
Notebook_A_Nhom7_RW_Battery_NASA.ipynb
```

Notebook A performs:

1. Raw NASA MATLAB-file ingestion.
2. Battery-step extraction and normalization.
3. True reference-discharge identification.
4. SOH target construction.
5. Data-quality auditing.
6. Temperature-sentinel handling.
7. DuckDB SQL analysis.
8. Cumulative throughput construction.
9. 50-operational-step rolling feature engineering.
10. RQ1 statistical analysis.
11. Seven RQ1 figures.
12. Final validation and handoff to Notebook B.

---

## True Reference Discharge Definition

A true reference discharge is identified using both:

```text
type == "D"
```

and an exact cleaned comment:

```text
comment == "reference discharge"
```

The exact match is required because a broad substring search also captures rows such as:

```text
rest post reference discharge
```

which are not capacity-reference measurements.

Final audit:

```text
947 true reference-discharge rows
28 battery cells
```

---

## SOH Target Definition

For each operational charge/discharge step, the target uses the next future true reference-discharge capacity from the same battery cell.

The per-cell SOH reference capacity is implemented as:

```text
capacity_initial_ah = max(true reference-discharge capacity within that cell)
```

SOH is then:

```text
SOH = next reference-discharge capacity / capacity_initial_ah
```

Therefore, the final implementation uses the maximum observed valid reference-discharge capacity for each cell as the baseline rather than assuming that the first recorded reference capacity is always the correct initial capacity.

Reference-procedure throughput is not added to operational cumulative throughput.

---

## Temperature Data Quality

The raw data contain physically implausible negative temperature values, including values near:

```text
-4099 °C
```

These are treated as invalid sensor/sentinel readings.

Notebook A applies:

```text
negative t_max -> NaN
```

while retaining the rest of the operational row.

Temperature is not imputed in Notebook A. Any downstream imputation is fitted using training cells only in Notebook B to avoid cross-cell leakage.

---

## Final Feature Handoff

Notebook A produces:

```text
data/processed/feat.parquet
data/processed/manifest.json
```

Verified feature-table dimensions:

```text
Rows    : 548,010
Columns : 17
Cells   : 28
```

Final columns:

```text
cell
step
type
dur_s
v_start
v_end
i_mean
t_max
ah
cum_ah
soh
dv
i_ma50
t_max50
ah50
temp_valid_n50
history_n
```

Every retained feature row has a complete 50-operational-step history.

Missing-temperature counts:

```text
t_max    : 13,239
t_max50  : 5,663
```

These missing values are retained intentionally and handled without cross-cell leakage in Notebook B.

---

## SQL Analysis

The reproducible SQL queries are stored in:

```text
sql/queries.sql
```

The SQL workflow covers:

- Q1: SOH summary by battery cell and cumulative-throughput bin.
- Q2: Health-state grouping.
- Q3: 50-step rolling features.
- Q4: Final quality and integrity checks.

The rolling window uses the current operational row and the previous 49 operational rows only.

---

## RQ1 Findings

Across the 28 cells, cumulative throughput has a very strong negative monotonic relationship with SOH.

Median cell-wise Spearman correlation:

```text
rho = -0.996138
```

Mean cell-wise Spearman correlation:

```text
rho = -0.994475
```

Wilcoxon signed-rank test:

```text
p < .001
```

Median raw correlations with SOH:

```text
ah50      +0.431202
i_ma50    +0.416218
t_max50   +0.159089
```

Median partial correlations after controlling for cumulative throughput:

```text
ah50      -0.065597
i_ma50    +0.121690
t_max50   +0.029429
```

Interpretation:

> Accumulated operational throughput dominates the monotonic degradation signal in this dataset. The raw associations of recent load-history features with SOH weaken substantially after controlling for cumulative throughput.

These results are descriptive and should not be interpreted as causal evidence.

---

## RQ1 Figures

Notebook A generates seven final figures:

```text
fig_rq1_distribution.png
fig_rq1_timeseries.png
fig_rq1_all_cells.png
fig_rq1_corr.png
fig_rq1_degradation_rate.png
fig_rq1_load_raw.png
fig_rq1_partial_corr.png
```

---

# Notebook B — Cross-Battery Modeling and RQ2/RQ3

Main notebook:

```text
Notebook_B_Nhom7_RW_Battery_NASA.ipynb
```

Notebook B starts from:

```text
data/processed/feat.parquet
data/processed/manifest.json
```

The final notebook was verified with a clean:

```text
Restart Kernel
Run All
```

without execution errors.

---

## Cross-Battery Evaluation Protocol

For each seed:

```text
18 training cells
5 calibration cells
5 test cells
```

All sets are cell-disjoint.

Five independent seeds:

```text
42
123
2026
27182
31415
```

Primary performance metric:

```text
macro-cell MAE
```

Macro-cell averaging prevents cells with more operational rows from dominating evaluation.

Preprocessing is fitted on training cells only, and cell identity is not used as a predictive feature.

---

## Models Evaluated

Five model families are compared:

```text
Ridge
kNN
Random Forest
LightGBM
small GRU
```

---

## RQ2 — Cross-Battery Model Comparison

Five-seed mean macro-cell MAE:

```text
Random Forest   0.024707 ± 0.004836
GRU             0.025469 ± 0.005154
LightGBM        0.027255 ± 0.003190
kNN             0.034890 ± 0.008757
Ridge           0.053463 ± 0.014459
```

Random Forest:

```text
Average rank : 1.20
Seed wins    : 4 / 5
```

Random Forest is selected as the primary RQ3 model because it provides the strongest average held-out-cell generalization.

---

# RQ3 — Reliability, Interpretation, and Practicality

## Conformal Prediction

Split conformal prediction uses held-out calibration cells that are separate from both training and final test cells.

Primary method:

```text
pooled absolute calibration residuals
nominal coverage = 90%
```

Five-seed pooled-residual results:

```text
Mean macro-cell coverage : 98.518%
Mean EOL coverage        : 97.610%
Mean interval width      : 0.227927
Worst-seed EOL coverage  : 93.472%
```

Cell-balanced sensitivity analysis:

```text
Mean macro-cell coverage : 96.135%
Mean EOL coverage        : 92.953%
Mean interval width      : 0.167266
Mean width reduction     : 24.92%
```

The cell-balanced approach is narrower, but one split shows weaker EOL coverage. The pooled-residual method is therefore retained as the primary conformal analysis.

---

## End-of-Life Evaluation

EOL is defined as:

```text
SOH <= 0.80
```

Five-seed point-estimate summary:

```text
Mean macro EOL MAE       : 0.041228
Mean macro EOL RMSE      : 0.048642
Mean macro EOL bias      : +0.031998
EOL overestimate rate    : 78.219%
Missed-EOL rate          : 24.788%
Pre-EOL false-alarm rate : 2.577%
```

All five seed-level EOL biases are positive.

Interpretation:

> Random Forest tends to overestimate SOH in the late-life region, so strong average SOH accuracy does not automatically imply equally strong EOL decision quality.

This motivates reporting conformal uncertainty intervals alongside point predictions.

---

## Feature Ablation

Five-seed mean macro-cell MAE:

```text
Full feature set                  0.024707
Without rolling load              0.024178
Without instantaneous electrical  0.029215
Without temperature               0.032235
Cum-Ah only                       0.040537
Without cumulative exposure       0.051146
```

Cross-seed interpretation:

```text
Cumulative exposure removed       worse in 5 / 5 seeds
Instantaneous electrical removed  worse in 5 / 5 seeds
Temperature removed               worse in 4 / 5 seeds
Rolling load removed              worse in 4 / 5 seeds
Cum-Ah only                       worse than full in 5 / 5 seeds
```

Removing cumulative exposure increases mean macro-cell MAE by approximately 109% relative to the full feature set.

The slightly lower mean MAE of the `without rolling load` configuration is driven by one favorable split and should not be interpreted as evidence that rolling-load information is generally harmful.

---

## SHAP Interpretation

Five-seed SHAP analysis is performed on held-out test cells using cell-balanced trajectory sampling.

Approximate TreeSHAP is used after validation against exact TreeSHAP.

Balanced validation audit:

```text
Spearman rank correlation : 0.9824
Top-5 overlap             : 5 / 5
Approximation speedup     : ~9218.6x
```

Final SHAP analysis:

```text
5 fitted Random Forest models
25 seed-cell evaluations
7,500 explained rows
```

Five-seed mean absolute SHAP importance:

```text
1. cum_ah          0.076551
2. ah50            0.028946
3. t_max50         0.025712
4. t_max           0.017890
5. i_ma50          0.007838
6. dur_s           0.007398
7. dv              0.004660
8. ah              0.003739
9. v_end           0.002941
10. i_mean         0.002849
```

`cum_ah` ranks first in all five seeds.

SHAP values represent predictive importance, not causal effects.

---

## Computational Trade-Off

Random Forest provides the best predictive accuracy, while LightGBM is a substantially cheaper alternative.

Using the final clean Run-All timing:

```text
RF / LightGBM fit-time ratio       : 23.98x
RF / LightGBM inference-time ratio : 1.86x
LightGBM MAE penalty vs RF         : 10.31%
GRU / RF inference-time ratio      : 8.05x
```

Timing is hardware- and implementation-dependent and should be interpreted descriptively rather than as a universal benchmark.

---

## Final RQ2/RQ3 Figures

Notebook B generates six final figures:

```text
fig_rq2_five_seed_macro_mae.png
fig_rq3_conformal_coverage_width.png
fig_rq3_feature_ablation_macro_mae.png
fig_rq3_shap_five_seed_importance.png
fig_rq3_eol_bias_missed_rate.png
fig_rq3_accuracy_inference_cost.png
```

---

# Main Conclusions

1. Cross-battery SOH estimation is feasible on the NASA Randomized Battery Usage dataset under strict held-out-cell evaluation.
2. Random Forest provides the strongest average generalization among the five evaluated model families.
3. Accumulated operational throughput is the dominant predictive signal across RQ1, ablation, and SHAP analyses.
4. Additional electrical, temperature, and recent-load features improve prediction beyond cumulative throughput alone, although some contributions are split-dependent.
5. Late-life SOH prediction is more difficult than average SOH estimation and exhibits systematic positive bias near EOL.
6. Conformal prediction intervals are operationally useful because they expose uncertainty that point estimates alone can hide.
7. LightGBM is a useful efficiency-oriented alternative when computational cost matters more than the final increment in predictive accuracy.

The final modeling result is best interpreted as a combined framework consisting of:

```text
cross-cell SOH estimation
+ conformal uncertainty intervals
+ EOL diagnostics
+ feature ablation
+ SHAP interpretation
+ computational trade-off analysis
```

---

# Reproducibility

## Full Rebuild

To rebuild the entire project from raw NASA data:

```text
1. Place all RW*.mat files under data/raw/
2. Activate the project environment
3. Open Notebook_A_Nhom7_RW_Battery_NASA.ipynb
4. Restart Kernel
5. Run All
6. Confirm data/processed/feat.parquet and manifest.json are produced
7. Open Notebook_B_Nhom7_RW_Battery_NASA.ipynb
8. Restart Kernel
9. Run All
```

Notebook B can also be run directly if these validated files already exist:

```text
data/processed/feat.parquet
data/processed/manifest.json
```

---

## Important Reproducibility Notes

- Do not use battery-cell identity as a predictive feature.
- Preprocessing must be fitted on training cells only.
- Calibration cells and final test cells must remain disjoint.
- The final test set must never be used to choose the conformal residual quantile.
- Many operational rows share the same next-reference SOH target, so hundreds of thousands of rows should not be interpreted as hundreds of thousands of independent battery-capacity measurements.
- Feature importance, SHAP values, and degradation associations must not be interpreted as causal evidence.
- Computational timing depends on hardware and software implementation.
- The current findings are specific to the NASA Randomized Battery Usage dataset and should not automatically be generalized to other battery chemistries, protocols, or UAV fleets.

---

## Verified Final Status

```text
Notebook A
-----------
Raw ingestion                  DONE
SOH target                     DONE
Data cleaning                  DONE
Temperature handling           DONE
SQL Q1-Q4                      DONE
50-step features               DONE
RQ1 statistics                 DONE
RQ1 figures                    DONE
Validation tests               DONE
Manual feature verification    DONE
Manifest / handoff             DONE
Clean Run All                  DONE

Notebook B
-----------
Cell-disjoint split protocol   DONE
Ridge                          DONE
kNN                            DONE
Random Forest                  DONE
LightGBM                       DONE
GRU                            DONE
Five-seed RQ2 benchmark        DONE
Conformal prediction           DONE
EOL evaluation                 DONE
Feature ablation               DONE
SHAP interpretation            DONE
Computational cost analysis    DONE
Six final RQ2/RQ3 figures      DONE
Final synthesis                DONE
Clean Restart Kernel + Run All DONE
```

---

## Group

ADY201m Research-Based Learning Project  
Group 7
