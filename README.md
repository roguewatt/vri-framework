# VRI Framework

**Valid / Reference / Issue — A Governance and Auditing Convention for Production Forecasting**

---

## Purpose

VRI is a governance and auditing convention for production forecasting systems. It provides a strict operational vocabulary for managing temporal leakage, data eligibility, and forecast reproducibility.

It does not introduce a new forecasting methodology or temporal theory.

---

## Core Concepts

### Valid Time
The business-defined time point or interval that a record or prediction describes. It is a property of the modelled reality, independent of when the record was created, collected, or published.
- For the half-hourly load of 08:00–08:30 on a given day, the Valid Time is 08:00, the start of the settlement period.
- For a weather forecast for tomorrow, the Valid Time is the future time that the forecast describes.

### Issue Time
The time at which a record or prediction becomes available to its intended consumer.

| Type | Valid Time | Issue Time | Description |
| :--- | :--- | :--- | :--- |
| Outturn | 08:00 on 2026-06-18 | 2026-06-19 02:00 | Published at 02:00 on the following day |
| Outturn (revised) | 08:00 on 2026-06-18 | 2026-06-19 10:00 | Revised value published later the same day |
| Forecast | 08:00 on 2026-06-20 | 2026-06-19 08:15 | Forecast released at 08:15 the day before |

Each revision or forecast release is a separate record version with its own Issue Time.

### Reference Time

Reference Time is the temporal anchor of a sample or inference. It is the timestamp that defines the information state from which the sample's forecast outputs are produced.

Each sample or inference has one Reference Time.

Any input used by the sample must satisfy:

`Issue Time ≤ Reference Time`

For example, consider a half-hourly solar-generation model using `Solar Radiation Forecast` and `Cloud Cover Forecast` as input features. The model jointly produces `HH0`, `HH1`, and `HH2` as separate outputs. This forms a MIMO structure in which the multiple outputs represent different horizons of the same target series.

The final training row constructed by the I/O builder may be:

| Reference Time | Solar Radiation Forecast | Cloud Cover Forecast | Solar Generation HH0 | Solar Generation HH1 | Solar Generation HH2 |
| :--- | ---: | ---: | ---: | ---: | ---: |
| 08:00 | 320 W/m² | 65% | 110 MW | 145 MW | 180 MW |

The first testing input row contains the same input features, but no realised target outputs:

| Reference Time | Solar Radiation Forecast | Cloud Cover Forecast |
| :--- | ---: | ---: |
| 08:30 | 410 W/m² | 48% |

After inference, the model produces three outputs for that testing row:

| Reference Time | Predicted Solar Generation HH0 | Predicted Solar Generation HH1 | Predicted Solar Generation HH2 |
| :--- | ---: | ---: | ---: |
| 08:30 | 150 MW | 190 MW | 225 MW |

The Forecast Schedule then maps those outputs to their Target Valid Times:

| Reference Time | Horizon | Target Valid Time | Solar Generation Forecast |
| :--- | :--- | :--- | ---: |
| 08:30 | HH0 | 08:30 | 150 MW |
| 08:30 | HH1 | 09:00 | 190 MW |
| 08:30 | HH2 | 09:30 | 225 MW |

Reference Time therefore defines the information state from which every forecast output for that sample or inference is produced. The Forecast Schedule resolves each output to its Target Valid Time using Reference Time as the schedule anchor.

### Cutoff
Cutoff is the global data boundary applied to one dataset construction, training run, backtest, or inference job.

A run normally has one Cutoff, while the samples or inferences contained within that run may have many Reference Times.

Cutoff determines the latest data state available to the run and therefore constrains:
- which record versions can be retrieved;
- which samples can be constructed;
- whether required labels are available;
- where a training or evaluation dataset must end.

Cutoff does not replace Reference Time.
- **Reference Time** governs the information state of an individual sample or inference.
- **Cutoff** governs the overall data boundary of the run.

---

## Training

Training samples are constructed using the same input eligibility rule as inference.

For each training sample:

- A historical point on the Target Valid Time axis is assigned as the sample's Reference Time.
- Every input record used by the sample satisfies `Issue Time ≤ Reference Time`.
- The label is the realised value corresponding to the sample's Target Valid Time.
- The label may be issued after the Reference Time and is attached later as an outcome for model fitting.
- The model architecture, such as XGBoost, MLP, TCN, or LSTM, is independent of VRI.

### Label Completeness and Past-Covariate Availability

A training sample may be included only when every required label is available by the dataset Cutoff. This is a label-completeness constraint, not an input eligibility constraint. Labels may have Issue Times later than the sample's Reference Time.

A realised observation is not automatically eligible merely because its Valid Time is before the Reference Time. It may still be unavailable if its Issue Time is later than the Reference Time. Different input series may therefore have different latest eligible Valid Times for the same sample.

For each input series, the forecasting strategy must define which eligible record version is selected, any required safety lag, whether an eligible forecast or nowcast replaces an unavailable outturn, how missing values are handled, and whether the series should be excluded when its historical availability cannot be reproduced reliably.

Historical training samples must reproduce the source-specific availability state that existed at each sample's Reference Time. Later publications, revisions, corrected values, or backfilled observations must not be used unless they were already eligible at that Reference Time.

For a regular half-hourly schedule with labels available immediately at their Target Valid Times, the following may be useful:

`Latest Training Reference Time = Cutoff − Maximum Horizon × 30 minutes`

However, this formula is not general. If labels are published later, revised asynchronously, or governed by an irregular Forecast Schedule, the actual inclusion condition is:

`Required Label Issue Time ≤ Cutoff`

The core VRI eligibility rule remains unchanged for every input record:

`Issue Time ≤ Sample Reference Time`

---

## Testing (Inference)

Testing follows the same input eligibility rule as training. The model is evaluated by simulating historical inferences or running live predictions.

- Each test sample or inference has its own Reference Time.
- Only input records satisfying `Issue Time ≤ Reference Time` are eligible.
- Future covariates are allowed when their record versions were issued by the Reference Time.
- Realised target values are never used as model inputs.
- Realised targets are used only for scoring after predictions have been generated.
- For out-of-sample backtesting, the completed model is frozen before the first simulated test inference.
- No test-period samples or labels enter model fitting.

### Single Inference vs. Multiple Inferences

- **Single inference**: one Reference Time, one logical model input, and one set of outputs. The output may be single-horizon or multi-horizon.
- **Multiple inferences**: multiple logical model inputs, each governed by its own Reference Time. They may be processed individually or together in a batch.

The resulting predictions are aligned with their Target Valid Times and scored against realised values.

Multiple-inference evaluation is the standard structure for out-of-sample backtesting.

It is orthogonal to multi-horizon:

- one inference may produce one or multiple horizons;
- an evaluation may contain multiple inferences;
- each inference may use a single-horizon or multi-horizon output structure.

---

## Multi-horizon and I/O Schema

### Multi-horizon
Multi-horizon means that one output series contains predictions for multiple Target Valid Times, such as:

`y(t+1), y(t+2), ..., y(t+H)`

The horizon represents an ordered position in the applicable Forecast Schedule. It does not necessarily represent a fixed elapsed duration.

Multi-horizon is independent of the number of semantic input and output series.

### I/O Schema

SISO, SIMO, MISO, and MIMO describe the logical input and output structure presented to a model.

- **SISO**: single input, single output
- **SIMO**: single input, multiple outputs
- **MISO**: multiple inputs, single output
- **MIMO**: multiple inputs, multiple outputs

Inputs and outputs may represent different semantic series or separately constructed model dimensions.

A jointly generated multi-horizon target may therefore form a special MIMO structure when its horizons are represented as separate output dimensions. In this case, the multiple outputs belong to the same semantic target series but resolve to different Target Valid Times.

Multi-horizon and I/O schema describe different aspects of the model:

- **Multi-horizon** describes the temporal coverage of the outputs.
- **I/O schema** describes the logical input and output structure presented to the model.

---

## Examples

### Example 1: Intraday Single Horizon (MISO)
GB Zone A. A prediction is issued at 08:15 to forecast load for the next settlement period.
- **I/O Schema**: MISO.
- **Reference Time**: 2026-06-19 08:00.
- **Input records**: all records used satisfy `Input Record Issue Time ≤ 08:00`.
- **Target Valid Time**: 2026-06-19 08:30.
- **Issue Time**: 2026-06-19 08:15.
- **Check**: input-compliant.

### Example 2: Intraday Multi-Horizon (MISO)
GB Zone A. A prediction is issued at 08:15 to forecast load for the next three settlement periods.
- **I/O Schema**: MISO.
- **Multi-horizon**: 3.
- **Reference Time**: 2026-06-19 08:00.
- **Input records**: all records used satisfy `Input Record Issue Time ≤ 08:00`.
- **Target Valid Times**: 08:30, 09:00, and 09:30.
- **Issue Time**: 2026-06-19 08:15.
- **Check**: input-compliant.

### Example 3: Two Zones, Two Days Each (MIMO)
Zone A and Zone B are forecast for each of the next two days.
- **I/O Schema**: MIMO.
- **Multi-horizon**: 2.
- **Reference Time**: 2026-06-19 03:00.
- **Input records**: all records used satisfy `Input Record Issue Time ≤ 2026-06-19 03:00`.
- **Target Valid Times**: 2026-06-20 and 2026-06-21 for both output series.
- **Check**: compliant.

### Example 4: Single Zone, Two Metrics (MIMO)
GB Zone A. Predict load and price for the next settlement period.
- **I/O Schema**: MIMO.
- **Reference Time**: 2026-06-19 08:00.
- **Input records**: all records used satisfy `Input Record Issue Time ≤ 08:00`.
- **Target Valid Time**: 2026-06-19 08:30.
- **Check**: compliant.

### Example 5: Revision Handling
A record describes Valid Time 08:00 on 2026-06-18.
- **Version 1 Issue Time**: 2026-06-19 02:00.
- **Version 2 Issue Time**: 2026-06-19 10:00.
- **Inference A**, with Reference Time 2026-06-19 06:00: only Version 1 is eligible.
- **Inference B**, with Reference Time 2026-06-19 12:00: both versions are eligible.

A predefined deterministic rule, such as selecting the latest eligible version, determines which record is used.

### Example 6: Out-of-Sample Backtesting
A model is trained using labels whose Valid Times end on 2026-05-31. The completed model is frozen before the first simulated test inference.
- Each simulated inference uses its own historical Reference Time.
- Only input records satisfying `Input Record Issue Time ≤ Reference Time` are eligible.
- Realised target values are never used as model inputs.

### Example 7: GB Electricity Intraday Prediction with Horizon 0
A job runs at 08:15 on 2026-06-19. The current settlement period started at 08:00.
- **Reference Time**: 2026-06-19 08:00.
- **Input records**: all records used satisfy `Input Record Issue Time ≤ 08:00`.
- **Forecast Issue Time**: 2026-06-19 08:15.
- **Future Target Valid Times**: 08:30, 09:00, 09:30, and later settlement periods.

If horizon 0 is included, its Target Valid Time is 08:00.

Because the forecast is issued at 08:15, the horizon-0 output is published after its target settlement period has started. Its inputs may still satisfy the VRI input eligibility rule, but the output is not an ahead-of-time forecast.

### Example 8: Cutoff and Label Completeness
A training dataset is constructed with:

`Cutoff = 2026-06-19 08:30`

The dataset contains multiple historical samples, each with its own Reference Time.

For every input record used by a sample:

`Input Record Issue Time ≤ Sample Reference Time`

The Cutoff does not replace the sample Reference Time. It represents the latest data state available when the training dataset is constructed.

For a multi-horizon sample, every required label must satisfy:

`Required Label Issue Time ≤ Cutoff`

Samples whose required labels are not yet available by the Cutoff are excluded.

For a regular time grid where labels are available immediately at their Target Valid Times, this may reduce to:

`Latest Training Reference Time = Cutoff − Maximum Horizon × Time Step`

This fixed-step formula is only a special case. When labels are published later or the Forecast Schedule is irregular, label availability must be evaluated using the actual Issue Times of the required labels.

### Example 9: Decision Time Is Outside VRI
A trader acts at 10:00 after receiving a forecast.

VRI governs the temporal semantics of the prediction and its inputs. It does not govern the later business decision.

---

## Auditing

A VRI-compliant system should be able to reconstruct, for each logical inference:
- Reference Time
- Target Valid Time
- Forecast horizon
- Applicable Forecast Schedule
- Input record identifiers with their Valid Times and Issue Times
- Selected record versions and the deterministic rule used to select them
- Output record identifiers with their Valid Times and Issue Times
- Model version
- Run Cutoff

For training datasets, the system should additionally record:
- Sample Reference Time
- Target labels with their Valid Times and Issue Times
- Dataset Cutoff
- Training and test periods
- Target-buffer or label-completeness rule

---

## What VRI Is Not

VRI is not:
- a forecasting algorithm;
- a temporal database theory;
- a leakage detection theory;
- a replacement for feature stores;
- a replacement for forecasting methodology.

It is a governance and auditing convention for production forecasting systems.

---

## References

[1] **Databricks.** (n.d.). "Point-in-time feature joins." *Databricks Feature Store Documentation.*  
Accessed 2026-06-19.  
https://docs.databricks.com/aws/en/machine-learning/feature-store/time-series

[2] **Tecton.** (n.d.). "Construct Training Data." *Tecton Documentation.*  
Accessed 2026-06-19.  
https://docs.tecton.ai/docs/reading-feature-data/reading-feature-data-for-training/constructing-training-data

[3] **Simha, N.** (2023). "Chronon — A Declarative Feature Engineering Framework." *Airbnb Engineering Blog.*  
Accessed 2026-06-19.  
https://medium.com/airbnb-engineering/chronon-a-declarative-feature-engineering-framework-b7b8ce796e04

[4] **Hyndman, R. J., & Athanasopoulos, G.** (2021). *Forecasting: Principles and Practice.* 3rd ed. OTexts.  
https://otexts.com/fpp3/

[5] **Ben Taieb, S., Bontempi, G., Atiya, A. F., & Sorjamaa, A.** (2012). "A review and comparison of strategies for multi-step ahead time series forecasting based on the NN5 forecasting competition." *Expert Systems with Applications*, 39(8), 7067–7083.  
https://doi.org/10.1016/j.eswa.2012.01.039

[6] **Salinas, D., Flunkert, V., Gasthaus, J., & Januschowski, T.** (2020). "DeepAR: Probabilistic forecasting with autoregressive recurrent networks." *International Journal of Forecasting*, 36(3), 1181–1191.  
https://doi.org/10.1016/j.ijforecast.2019.07.001

[7] **Lim, B., Arık, S. Ö., Loeff, N., & Pfister, T.** (2021). "Temporal Fusion Transformers for interpretable multi-horizon time series forecasting." *International Journal of Forecasting*, 37(4), 1748–1764.  
https://doi.org/10.1016/j.ijforecast.2021.03.012

[8] **Bontempi, G., Ben Taieb, S., & Le Borgne, Y.-A.** (2013). "Machine Learning Strategies for Time Series Forecasting." In *Business Intelligence* (pp. 62–77). Springer.  
https://doi.org/10.1007/978-3-642-36318-4_3

[9] **Kaufman, S., Rosset, S., Perlich, C., & Stitelman, O.** (2012). "Leakage in data mining: Formulation, detection, and avoidance." *ACM Transactions on Knowledge Discovery from Data*, 6(4), Article 15, 21 pages.  
https://doi.org/10.1145/2382577.2382579

[10] **Snodgrass, R. T., & Ahn, I.** (1985). "A taxonomy of time in databases." *Proceedings of the 1985 ACM SIGMOD International Conference on Management of Data*, 236–246.  
https://doi.org/10.1145/22733.22745

[11] **Jensen, C. S., et al.** (1998). "The consensus glossary of temporal database concepts — February 1998 version." In *Temporal Databases: Research and Practice* (pp. 367–405). Springer.  
https://doi.org/10.1007/BFb0053710

---

## License

Licensed under the [Apache License 2.0](LICENSE).

---

## Citation

```bibtex
@misc{vri-framework,
  author       = {Fan Zhang},
  title        = {VRI: A Governance and Auditing Convention for Production Forecasting},
  year         = {2026},
  howpublished = {\url{https://github.com/roguewatt/vri-framework}}}
