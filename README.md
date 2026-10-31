# VM CPU Utilisation Forecasting

## Overview

This project was developed as part of the Atomity AI/ML Engineering Intern take-home challenge. The objective is to analyse virtual machine (VM) CPU utilisation data and forecast aggregate infrastructure metrics over a 2-hour horizon.

The solution focuses on time-series analysis, statistical forecasting, chronological validation and prediction reproducibility.

## Problem Statement

The task involves forecasting two aggregate metrics:

- **Active VM Count:** Number of distinct VMs with valid observations at each timestamp.
- **Aggregate CPU Utilisation:** Mean CPU utilisation across all VMs at each timestamp.

The dataset contains 10,000 VMs with 134 observations per VM, recorded at 5-minute intervals.

The first 110 timestamps are used for training and validation, while the final 24 timestamps are held out for evaluation.

## Methodology

### 1. Data Processing
- Loaded CPU readings and VM metadata.
- Checked for missing values and duplicate records.
- Aggregated VM-level observations into timestamp-level metrics.
- Preserved chronological ordering to prevent data leakage.

### 2. Exploratory Data Analysis
- Analysed aggregate CPU utilisation trends and variability.
- Examined rolling statistics and autocorrelation.
- Conducted ADF and KPSS stationarity tests.
- Investigated temporal patterns and potential seasonality.

### 3. Forecasting and Validation

The forecasting experiments included:

- Mean baseline
- Persistence (naive) baseline
- Simple Exponential Smoothing
- ARIMA models
- Holt's linear trend method

Models were evaluated using expanding-window rolling-origin validation with a 24-step forecast horizon.

The historical mean was selected for aggregate CPU utilisation based on its average MAE and RMSE across the validation folds. Persistence was used for active VM count.

## Final Holdout Results

The selected approach was evaluated on the final 24 unseen timestamps.

| Metric | Result |
|---|---:|
| CPU Utilisation MAE | 0.174590 |
| CPU Utilisation RMSE | 0.210462 |
| CPU Utilisation sMAPE | 1.8773% |
| Active VM Count MAE | 0.0 |
| Active VM Count RMSE | 0.0 |

**Final forecast values:**
- Predicted active VM count: **10,000**
- Predicted aggregate CPU utilisation: **9.414089%**

The aggregate CPU forecast is constant across the 24-step horizon because the historical mean performed competitively during validation. The active VM count remained constant throughout the observed dataset.

## Repository Structure

```text
forecasting_challenge_submission/
│
├── notebook.ipynb
├── final_report.md
├── predictions.csv
└── README.md
```

- `notebook.ipynb`: Data analysis, exploratory analysis, forecasting experiments and evaluation.
- `final_report.md`: Detailed explanation of the methodology, validation strategy, results and limitations.
- `predictions.csv`: Final 24-step forecast in the required submission format.
- `README.md`: Project overview and reproduction information.

## Prediction Format

The submitted `predictions.csv` contains 24 rows with the following columns:

```text
timestamp
active_vm_count
aggregate_cpu_utilization
```

Predictions are ordered chronologically at 5-minute intervals.

## Reproducibility

### Requirements

- Python 3.11 or later
- pandas
- NumPy
- matplotlib
- statsmodels
- scikit-learn
- Jupyter Notebook

### Setup

Clone the repository:

```bash
git clone https://github.com/Mayankk1207/forecasting_challenge_submission.git
cd forecasting_challenge_submission
```

Create and activate a virtual environment:

```bash
python3 -m venv venv
source venv/bin/activate
```

Install the required packages:

```bash
pip install numpy pandas matplotlib statsmodels scikit-learn jupyter
```

Launch the notebook:

```bash
jupyter notebook
```

Open `notebook.ipynb` and execute the cells sequentially.

The original dataset is not included in this submission. Place the provided dataset files in the expected local dataset directory and update the notebook paths if necessary.

## Limitations

- The aggregate time series contains only 134 observations.
- The validation folds are limited and partially overlapping.
- The selected mean forecast cannot capture short-term fluctuations or sudden changes in CPU utilisation.
- The constant active VM count forecast relies on the stable VM count observed in the supplied dataset.
- Further evaluation on longer time periods and changing infrastructure workloads would be required before production deployment.

## Author

**Mayank Kumar**

AI/ML Engineering Candidate

GitHub: [Mayankk1207](https://github.com/Mayankk1207)
