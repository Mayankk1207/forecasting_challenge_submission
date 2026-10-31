# Forecasting Aggregate VM CPU Utilisation and Active VM Count

## 1. Executive Summary

This project forecasts two aggregate metrics from virtual-machine CPU
telemetry: the number of active virtual machines and aggregate CPU
utilisation. The source data contains 10,000 VMs, each with 134
observations recorded at five-minute intervals.

The data was aggregated into a chronological time series containing 134
timestamps. The first 110 observations were used for model development
and validation, while the final 24 observations were held out for final
evaluation. The forecast horizon was therefore two hours.

The active VM count remained constant at 10,000 throughout the observed
period. A constant forecast achieved zero MAE and RMSE on the final
holdout.

For aggregate CPU utilisation, several statistical approaches were
compared using expanding-window, rolling-origin validation. The
historical-mean baseline achieved the best average MAE and RMSE among
the tested models. On the final 24-observation holdout, it achieved an
MAE of 0.174590 percentage points, an RMSE of 0.210462 percentage points
and an sMAPE of 1.8773%.

The results support a simple forecasting approach for this dataset.
However, the constant CPU forecast cannot respond to short-term
fluctuations or workload-level changes. The final holdout also showed a
tendency to overpredict CPU utilisation.

## 2. Problem Definition

The objective is to produce a 24-step-ahead forecast for two metrics:

1.  `active_vm_count`: the number of distinct VMs with a valid CPU
    observation at each timestamp.
2.  `aggregate_cpu_utilization`: the mean of the VM-level `avg_cpu`
    readings at each timestamp.

Each observation is separated by 300 seconds, or five minutes. A 24-step
forecast therefore covers 120 minutes, or two hours.

The task is treated as a time-series forecasting problem. Chronological
ordering is preserved throughout data preparation, validation and final
evaluation.

## 3. Dataset and Data Preparation

### 3.1 Source files

Two CSV files were used:

-   `vm_cpu_readings.csv`: timestamped CPU readings containing
    `timestamp`, `vm_id`, `min_cpu`, `max_cpu` and `avg_cpu`.
-   `vm_metadata.csv`: VM-level metadata, including identifiers,
    lifecycle timestamps, CPU summary fields, VM category, virtual-core
    bucket and memory bucket.

The CPU readings file contains 1,340,000 records, representing 10,000
VMs across 134 timestamps. The metadata file contains one record per VM.

### 3.2 Data-quality checks

The inspection found:

-   10,000 unique VM identifiers.
-   134 unique timestamps.
-   Exactly 134 observations for each VM.
-   No duplicate `(vm_id, timestamp)` CPU records.
-   No duplicate VM identifiers in the metadata.
-   No missing values in the inspected CPU or metadata columns.
-   A consistent 300-second interval between consecutive timestamps.

These checks indicated that the supplied observations were complete and
regularly spaced for the selected dataset. No missing-value imputation
was required.

### 3.3 Aggregation

The VM-level readings were aggregated by timestamp.

For each timestamp:

-   `active_vm_count` was calculated as the number of distinct VM
    identifiers.
-   `aggregate_cpu_utilization` was calculated as the arithmetic mean of
    `avg_cpu` across the VMs observed at that timestamp.

This produced a 134-row time series with three columns: timestamp and
the two forecast targets.

The aggregation was performed before the chronological train/holdout
split. This ensured that the model-development data and final evaluation
data represented the same aggregate forecasting problem.

## 4. Exploratory Data Analysis

### 4.1 Active VM count

The active VM count was 10,000 at every one of the 134 timestamps. Its
mean, minimum and maximum were all 10,000, and its standard deviation
was zero.

This is a constant series in the supplied data. A persistence forecast,
which repeats the last observed count, is therefore a natural and
sufficient baseline for this target.

### 4.2 Aggregate CPU utilisation

Across all 134 observations, aggregate CPU utilisation had:

-   Mean: approximately 9.3881%.
-   Standard deviation: approximately 0.1950 percentage points.
-   Minimum: approximately 8.9836%.
-   Median: approximately 9.3673%.
-   Maximum: approximately 9.9122%.

The series fluctuated within a relatively narrow range. The
training-period plot showed changes in local level, including a
higher-utilisation period and a later decline and recovery. This
suggested that the series was not simply a perfectly stable level, even
though its overall range was limited.

A 12-observation rolling window was used to inspect the local mean and
standard deviation. Since each observation represents five minutes, this
window corresponds to one hour.

### 4.3 Autocorrelation and seasonality

The original CPU series showed strong short-lag autocorrelation. The
lag-1 autocorrelation was approximately 0.6621, with positive
correlations also visible at several early lags. The correlation
weakened at longer lags.

The lag-12 correlation was positive, but the lag-24 correlation was
close to zero. The available training history was also short relative to
the number of hourly cycles needed to establish a repeating hourly
pattern. There was therefore insufficient evidence to prioritise
seasonal models.

## 5. Stationarity Investigation

The Augmented Dickey-Fuller (ADF) and KPSS tests were applied to the
training CPU series and its first difference.

For the original series:

-   ADF statistic: -1.7348.
-   ADF p-value: 0.4133.
-   KPSS statistic: 0.3075.
-   KPSS reported p-value: 0.1.

The ADF test did not reject the unit-root null hypothesis. The KPSS test
did not reject its stationarity null hypothesis at the reported
significance bound. These results are not conclusive and should not be
interpreted as proof of either stationarity or non-stationarity.

For the first-differenced series:

-   ADF statistic: -2.6808.
-   ADF p-value: 0.0774.
-   KPSS statistic: 0.1406.
-   KPSS reported p-value: 0.1.

Differencing improved the ADF evidence, although its p-value remained
above the conventional 5% threshold. The test results were treated as
diagnostic evidence rather than an automatic rule for model selection.
ARIMA models with first differencing were included as candidates and
evaluated empirically.

## 6. Validation Design

The first 110 aggregate observations were reserved for model
development. The final 24 observations were held out and were not used
for model selection.

An expanding-window rolling-origin validation design was applied within
the first 110 observations. Three origins were used:

    Origin      History size Validation positions     Horizon
  -------- ----------------- ---------------------- ---------
        62   62 observations 62--85                        24
        74   74 observations 74--97                        24
        86   86 observations 86--109                       24

At each origin, the model was fitted using only the observations
available before that origin and forecast the next 24 observations.

The validation windows overlap. This makes the fold errors correlated
rather than independent, and the small number of folds limits the
strength of statistical conclusions. Nevertheless, the design provides a
consistent comparison under the required 24-step forecast horizon while
keeping the final holdout untouched.

The following metrics were calculated:

-   MAE: mean absolute error.
-   RMSE: root mean squared error.
-   sMAPE: symmetric mean absolute percentage error.

For CPU utilisation, MAE and RMSE are expressed in CPU percentage
points.

## 7. Model Experiments and Selection

### 7.1 Baselines

Two simple CPU baselines were evaluated:

-   Naive: repeat the last observed training value for all 24 forecast
    steps.
-   Historical mean: repeat the mean of the available training
    observations for all 24 forecast steps.

The historical-mean forecast fitted on all 110 training observations was
approximately 9.414089%.

### 7.2 Exponential smoothing

Simple Exponential Smoothing (SES) was fitted independently at each
validation origin. It estimates a smoothing parameter that controls the
weight assigned to recent observations.

Holt's method was also evaluated to test whether an explicit trend
component improved forecasts. It did not improve performance. The fitted
trend smoothing parameter was zero in two folds and very small in the
third, which is consistent with limited evidence for a stable
extrapolatable trend.

### 7.3 ARIMA

Three compact ARIMA configurations were evaluated:

-   ARIMA(1,1,0)
-   ARIMA(0,1,1)
-   ARIMA(1,1,1)

These models were selected as small candidates given the short training
history and the stationarity diagnostics. ARIMA(1,1,0) was the strongest
of the tested ARIMA configurations, but it did not outperform the
historical-mean baseline on average validation MAE or RMSE.

### 7.4 Validation results

The average results across the three rolling-origin folds were:

  Model               Average MAE   Average RMSE   Average sMAPE
  ----------------- ------------- -------------- ---------------
  Historical mean          0.1846         0.2120         1.9638%
  Naive                    0.1848         0.2174         1.9537%
  ARIMA(1,1,0)             0.1885         0.2180         1.9956%
  ARIMA(1,1,1)             0.2110         0.2406         2.2313%
  ARIMA(0,1,1)             0.2125         0.2421         2.2472%
  SES                      0.2132         0.2428         2.2540%
  Holt                     0.2451         0.2795         2.5772%

The historical mean achieved the lowest average MAE and RMSE. The naive
baseline had a marginally lower average sMAPE. The differences between
the two simplest baselines were small, and no single model dominated
every metric and fold.

The historical mean was selected for the final CPU forecast because it
achieved the best average MAE and RMSE across the predefined validation
folds. The choice favours a simple and reproducible model rather than
additional complexity that did not demonstrate a consistent improvement.

### 7.5 Active VM count model

The active VM count was constant at 10,000 throughout the training data.
A persistence forecast was evaluated across the same three validation
folds and achieved zero MAE and zero RMSE in every fold.

The persistence forecast was selected for active VM count.

## 8. Final Holdout Evaluation

After model selection, the selected models were fitted using all 110
training observations. The final 24 timestamps were then forecast.

The CPU forecast was the training-period mean, repeated across all 24
steps. The active VM count forecast repeated the final training count of
10,000.

### 8.1 Holdout metrics

  ---------------------------------------------------------------------------
  Target        Selected                  MAE            RMSE           sMAPE
                method                                        
  ------------- ------------- --------------- --------------- ---------------
  Active VM     Persistence            0.0000          0.0000    Not reported
  count                                                       

  Aggregate CPU Historical           0.174590        0.210462         1.8773%
  utilisation   mean                                          
  ---------------------------------------------------------------------------

The active VM count forecast matched all 24 holdout observations.

For CPU utilisation, the holdout MAE and RMSE were slightly lower than
the corresponding average validation results. This is encouraging for
the selected evaluation window, but a single holdout period does not
establish performance across all possible future workloads.

### 8.2 Holdout level comparison

The mean actual CPU utilisation in the holdout was 9.268988%, while the
constant forecast was 9.414089%.

The actual holdout minimum was 8.983613% and the maximum was 9.635990%.
The difference between the forecast level and actual holdout mean was
approximately 0.1451 percentage points.

## 9. Error Analysis

The signed error was defined as:

`error = actual - predicted`

The holdout error analysis produced:

  Error statistic                    Result
  ----------------------------- -----------
  Mean signed error               -0.145101
  Maximum underprediction          0.221901
  Maximum overprediction          -0.430476
  Underpredicted observations       6 of 24
  Overpredicted observations       18 of 24

A negative mean signed error indicates that the model overpredicted CPU
utilisation on average during the holdout. It overpredicted 18 of the 24
observations and underpredicted the remaining six.

The largest underprediction was 0.221901 percentage points, while the
largest overprediction was 0.430476 percentage points. The plot of
actual and predicted CPU values showed that the constant forecast
captured the broad utilisation level but did not follow local
fluctuations.

This behaviour is expected from a historical-mean forecast. It is stable
and simple, but it cannot adapt to a change in workload level or react
to short-lived increases.

The observed overprediction bias applies to this holdout window. It
should not be assumed to be a permanent property of the model.

## 10. Operational Considerations

The selected models provide a useful aggregate baseline, but they should
not be used as the sole signal for production capacity decisions.

Recommended considerations include:

-   Continue monitoring VM-level CPU utilisation rather than relying
    only on an aggregate mean.
-   Inspect maximum and high-percentile CPU utilisation when assessing
    risk from localised demand spikes.
-   Use a safety margin informed by historical forecast errors and
    service-level requirements.
-   Re-evaluate the forecast when the workload mix, active VM population
    or utilisation level changes.
-   Retrain or recalibrate models using a longer and more recent history
    when it becomes available.
-   Track forecast errors over time to identify systematic bias or
    changes in workload behaviour.

The constant active VM count forecast is appropriate for the supplied
trace, but it would not anticipate VM creation, deletion or scaling
events in a changing production environment.

## 11. Limitations

Several limitations should be considered when interpreting the results:

1.  The aggregate time series contains only 134 observations, with 110
    used for training and validation and 24 reserved for final
    evaluation.
2.  The forecast horizon is relatively long compared with the available
    training history: 24 steps versus 110 training observations.
3.  The three validation windows overlap, so their errors are not
    independent.
4.  The final holdout covers only one two-hour period.
5.  The selected CPU model forecasts a constant mean and cannot respond
    dynamically to workload changes.
6.  The active VM count is constant in this dataset, so its perfect
    forecast does not demonstrate the ability to predict scaling events.
7.  The model comparison focused on compact statistical approaches and
    simple baselines. A lag-feature machine-learning experiment has not
    yet been evaluated and is not included in the reported model
    selection.

## 12. Conclusion

The project developed and evaluated two aggregate forecasts from
VM-level CPU telemetry. Data-quality checks confirmed a complete,
regular set of observations for 10,000 VMs across 134 five-minute
timestamps.

The active VM count remained constant, and a persistence forecast
achieved zero error on the final holdout. For aggregate CPU utilisation,
the historical-mean baseline achieved the best average validation MAE
and RMSE among the tested candidates. It achieved a final holdout MAE of
0.174590 percentage points, RMSE of 0.210462 percentage points and sMAPE
of 1.8773%.

The final holdout also revealed a tendency to overpredict: 18 of 24 CPU
observations were overpredicted, and the mean signed error was -0.145101
percentage points. The model therefore provides a good estimate of the
broad utilisation level for this trace, but it does not capture
short-term changes.

The main outcome is that a simple, transparent model was more effective
than the more complex statistical alternatives tested under the
available data constraints. For operational use, the aggregate forecast
should be supplemented with VM-level monitoring, high-percentile
utilisation analysis and a capacity safety margin.

## Appendix A. Reproducibility

The notebook contains the data loading, integrity checks, aggregation,
exploratory analysis, stationarity tests, model experiments,
rolling-origin validation, prediction generation and holdout evaluation.

The raw input files are expected at:

-   `dataset/vm_cpu_readings.csv`
-   `dataset/vm_metadata.csv`

The final prediction file is written to:

-   `results/predictions/predictions.csv`

The submission CSV contains 24 chronological rows and the required
columns in this order:

`timestamp, active_vm_count, aggregate_cpu_utilization`

The notebook currently includes a machine-specific absolute path in its
data-loading cell. For portability, replace it with project-relative
paths before submitting, for example:

``` python
from pathlib import Path

PROJECT_ROOT = Path.cwd()
cpu = pd.read_csv(PROJECT_ROOT / "dataset" / "vm_cpu_readings.csv")
metadata = pd.read_csv(PROJECT_ROOT / "dataset" / "vm_metadata.csv")
```

If the notebook is launched from a subdirectory, set `PROJECT_ROOT` to
the repository root explicitly.

The reported metrics and conclusions in this report reflect the
experiments completed so far. The proposed Ridge Regression experiment
is not included because it has not yet been run and validated.
