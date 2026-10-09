# Energy Communities and Electricity Consumption in the Porto Metropolitan Area

Did the creation of energy communities change electricity consumption in the Porto Metropolitan Area (AMP)? A data science project built on open data from E-REDES, following the CRISP-DM methodology.

---

## Business Context & Objective

With the rapid growth of **Collective Self-Consumption (ACC)** and **Renewable Energy Communities (CER)** across Portugal, assessing their real-world impact on grid demand is critical.

* **Target Region:** Porto Metropolitan Area (AMP)
* **Granularity:** Parish-level (*Freguesia*) electricity consumption data
* **Objective:** Evaluate whether parishes adopting energy communities (CER/ACC) exhibited significant deviations in baseline grid electricity consumption compared to non-participating parishes.

## Approach

For each parish, a model trained only on the period **before** any community existed predicts the consumption of the following 12 months (the *counterfactual*). The gap between the prediction and the observed consumption is the estimated impact, corrected for the model's own error measured on control parishes (parishes with no community).

## Data

| Dataset | Content |
|---|---|
| Monthly consumption by parish | Billed active energy (kWh) by parish and voltage level, 2020 to 2025. 19,165 records for the 17 AMP municipalities from Nov 2020 |
| Energy communities | Registry of ACC and CER by parish and date, 2022 to 2025. In the AMP: 1,546 ACC and 25 CER, from 2023 onwards |
| Weather | ERA5 reanalysis (Copernicus): hourly 2 m temperature and precipitation, 2021 to 2025 |

Source: [E-REDES Open Data Portal](https://e-redes.opendatasoft.com/).

## Data understanding

- Consumption is heavily right-skewed (skewness 8.39; mean about 2.0 GWh against a median of about 0.8 GWh per parish-month). A log transformation brings skewness to about -0.18.
- Clear seasonality: highest in January, lowest in August (Kruskal-Wallis, p < 0.001 across months and seasons).
- Energy communities are almost entirely ACC (1,546 against 25 CER). Santa Maria da Feira and Vila Nova de Gaia lead in ACC, and CER only appear at the end of the period.
- Adoption accelerates from mid-2024 with a spike in late 2025, so the impact analysis focuses on pioneer communities from 2023 and 2024.
- Data quality: about 99.9% panel completeness and no missing values. Negative consumption values come from records with no municipality or parish assigned ("OUTROS"), which were excluded.

## Data preparation

- Filtering to the AMP, standardising parish names, and joining consumption and communities on the parish code and month.
- Flags for treatment: `has_acc_cer`, `months_since_acc_cer`, `n_acc_cer`, plus a control indicator.
- Weather aggregated to parish and month through a spatial join to the nearest ERA5 grid point.
- Three parish scenarios: **established** communities (with at least 12 months after), **future** communities (used as controls), and **never** (control parishes).
- Splitting each parish's timeline at the community creation date (`months_since_acc_cer`) into a **before** dataset (7,851 parish-months, used for modelling) and an **after** dataset (2,420 parish-months, used for the impact analysis).
- Features: lags, rolling means (3, 6, 12, 24 months), momentum, year-over-year ratio, volatility, holiday rate and cyclical month encoding.

## Modelling

- **Data used:** only pre-community months. The post-community months are kept for the impact analysis.
- **Task:** one prediction per parish, the mean consumption over the last 12 months before the community, using only earlier months. 173 parishes, 21 features.
- **Validation:** forward chronological cross-validation. Parishes are sorted by the date of their target window; the oldest 86 are used only for training, and the other 87 are split into 5 groups, each predicted by a model trained only on earlier parishes. This avoids using "future" information. Hyperparameters are tuned inside each training set (nested CV).
- **Models:** four naive baselines, ten cross-sectional models (linear, regularised, tree-based, kernel, ETS) and panel sensitivity checks, plus stacking and an optimised weighted ensemble.

| Model | MAE (kWh) |
|---|---|
| Optimised ensemble (62.7% linear regression, 37.3% parish mean) | 67,589 |
| Linear regression | 81,950 |
| Parish mean (naive) | 90,259 |
| Seasonal naive (benchmark) | 103,469 |
| XGBoost | 224,296 |

The ensemble reaches a MAPE of 3.54% and R² of 0.998, about 35% lower error than the seasonal naive benchmark on the same 87 parishes. History-based signals (lags and rolling means) dominate the predictions, and weather is secondary. Complex models did not beat simple ones on this small cross-section.

## Impact analysis

- Municipality-level impact ranged from about a **0.8% reduction (Trofa)** to a **0.4% increase (Oliveira de Azeméis)**, with no clear pattern by density or location. Within municipalities, parishes show opposing effects.
- Statistical tests found **no significant impact**: Mann-Whitney U (p = 0.303) and volume-weighted least squares (p = 0.386 with all data, 0.153 without outliers).
- Likely reasons: only 12 months after creation, which is short for effects that build up over time; consumption is noisy and driven by seasonal and economic factors; and the effect may be small compared with that variability.

## Limitations and further work

- Only about half of the parishes could be scored out-of-sample under strict forward validation.
- Many parishes share the same cutoff month (Oct 2023), so the forward ordering is strict only across different cutoff dates. Among parishes with the same date, the order falls back to parish code.
- Short post-community window, and a treatment definition that does not account for community size.
- Further work: 5 and 10 year horizons, rural versus urban segmentation, socioeconomic indicators, and parish-level power quality data.

## Repository structure

The project folders follow the CRISP-DM phases described above:

```text
.
├── README.md
├── Report.pdf
│
├── input/
│   ├── 3-consumos-faturados-por-municipio-ultimos-10-anos.csv
│   ├── comunidades-de-energia.csv
│   ├── parish_code_mapping.csv
│   ├── data_before_1acc.csv
│   └── data_after_1acc.csv
│
├── 1_DU/
│   ├── data_understanding.ipynb
│   └── DU_output/
│       ├── 01_ea_distribution.png
│       ├── 02_voltage_distribution.png
│       ├── 03_decomposition.png
│       ├── 04a_monthly_seasonality.png
│       ├── 05_mun_consumption.png
│       ├── 06_treatment_control.png
│       ├── 07_community_growth.png
│       └── 08_acc_cer_by_municipality.png
│
├── 2_DP/
│   ├── 1_EnergyComunities&Consumption.ipynb
│   ├── 1_EnergyComunities&Consumption_output/
│   │   └── dataset_final_parish_amp_FINAL.csv
│   ├── 2_EnvironmentalData.ipynb
│   ├── 2_EnvironmentalData_output/
│   │   └── month_env_region_AMP.csv
│   ├── 3_AllDataAgregation.ipynb
│   ├── 3_AllDataAgregation_output/
│   │   ├── month_con_full_AMP.csv
│   │   └── parish_scenarios_plot.png
│   ├── 4_SeparateData.ipynb
│   └── 4_SeparateData_output/
│       ├── data_after_1acc.csv
│       └── data_before_1acc.csv
│
├── 3_M/
│   ├── modeling.ipynb
│   └── 1_Models_Comparison_output/
│       ├── ensemble_metric_sensitivity.png
│       ├── ensemble_sanity_checks.png
│       ├── feature_importance.png
│       ├── leakage_visual_dashboard.png
│       ├── leakage_visual_panel_d.png
│       ├── model_comparison.png
│       ├── residual_diagnostics.png
│       ├── vector_forecast_per_step.png
│       └── whitebox_coefficients.png
│
└── 4_Impact/
    └── impact_analysis.ipynb
```

### `input/`

Contains all the datasets used and generated throughout the project.

* **Raw data:** `3-consumos-faturados-por-municipio-ultimos-10-anos.csv`, `comunidades-de-energia.csv`, and `parish_code_mapping.csv` are the initial base files.
* **Processed data:** `data_before_1acc.csv` and `data_after_1acc.csv` are intermediate/final datasets automatically generated by the notebooks in the `2_DP` folder.

### `1_DU/` (Data Understanding)

* **`data_understanding.ipynb`**: Explores the raw datasets, checks for missing values and distributions, and builds an understanding of the core variables before processing.

### `2_DP/` (Data Preparation)

The sequential pipeline for cleaning and preparing the data.

* **`environment_input/`**: Raw data files required for processing the environmental features.
* **`1_EnergyComunities&Consumption.ipynb`**: Treats the energy communities and consumption data.
* **`2_EnvironmentalData.ipynb`**: Cleans and prepares the environmental data.
* **`3_AllDataAgregation.ipynb`**: Merges the cleaned datasets into a single dataframe.
* **`4_SeparateData.ipynb`**: Handles the final data splits, generating `data_before_1acc.csv` and `data_after_1acc.csv`, which are saved back to the `input/` folder.

### `3_M/` (Modeling)

* **`modeling.ipynb`**: The machine learning workflow, including model training, evaluation, and validation using the prepared datasets.

### `4_Impact/` (Impact Analysis)

* **`impact_analysis.ipynb`**: Evaluates the impact of the model's findings, extracts insights, and calculates key metrics based on the modeling phase.

## Tools

Python: pandas, NumPy, scikit-learn, statsmodels, SciPy, matplotlib, Jupyter notebooks. Git and GitHub for version control.

## Team

* [Carolina Dias](https://github.com/CarolDias18) 
* [Leonor Couto](https://github.com/Leonor2004)
* [Mariana Pereira](https://github.com/mfaria-p) 
* [Simão Bernardo](https://github.com/simaozuzarte) 
* [Sofia Fernandes](https://github.com/sofiagf04) 


The earlier phases were developed collaboratively by the whole team. For the final version, **I worked with Sofia Fernandes on the data understanding notebook**: introducing the datasets, exploring the variables and analysing their distributions. The repository contains this notebook (`1_DU/data_understanding.ipynb`); the raw data can be downloaded from the E-REDES portal.


