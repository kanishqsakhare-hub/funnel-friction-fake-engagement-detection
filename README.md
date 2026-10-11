# Funnel Friction & Fake Engagement Detection

A notebook-based product analytics project for examining where e-commerce sessions leave the conversion funnel, which behaviors coincide with friction, and which browsing patterns look unusual enough to review.

The analysis works at the session level across 212,476 events and 20,000 synthetic sessions. It combines ordered-funnel measurement, transparent behavioral indicators, statistical comparisons, feature engineering, and label-blind Isolation Forest modeling. Synthetic scenario labels are kept out of model fitting and used only after scoring to evaluate the workflow.

## Key results

| Area | Result |
|---|---|
| Observed purchases | 4,176 sessions; 20.88% of all sessions |
| Strict ordered-funnel completion | 3,703 sessions; 18.52% |
| Largest funnel loss | Add to cart to checkout start: 9,039 sessions, or 55.08% of the preceding stage |
| Most common friction indicator | A page-duration signal above 30 seconds: 10,715 sessions, or 53.58% |
| Payment-failure association | 31.12 percentage-point lower purchase rate among eligible, independent-user sessions; 95% CI from -34.12 to -28.04 |
| Back-navigation association | 1.946 times the geometric-mean session duration; 95% CI from 1.911 to 1.982 |
| Isolation Forest, held-out users | Precision 0.926, recall 0.709, F1 0.804 |
| Transparent percentile rule, held-out users | Precision 0.950, recall 0.297, F1 0.453 |

These are results from a synthetic simulator. A flagged session is a review candidate, not confirmed fake engagement or fraud, and the reported model metrics are not estimates of production performance.

## Problem statement

An overall conversion rate does not show where customers leave, whether checkout problems coincide with abandonment, or whether bursts of activity reflect genuine exploration. This project addresses five practical questions:

1. How many unique sessions reach each funnel stage in the required order?
2. Which transition has the largest absolute and relative loss?
3. How common are repeated clicks, payment failures, back navigation, long page durations, technical errors, and repeated checkout attempts?
4. Are selected friction signals associated with purchase outcomes or session duration after accounting for repeated users?
5. Can a model trained without scenario labels identify the synthetic suspicious patterns on users it did not see during fitting?

The session is the unit of analysis throughout. Funnel counts use unique sessions rather than event rows.

## Dataset

Both input tables are synthetic and generated with seed 42. The simulator includes ordinary sessions, friction scenarios, and unusual activity patterns with deliberate overlap between groups.

| File | Grain | Rows | Columns | Purpose |
|---|---:|---:|---:|---|
| [`events.csv`](data/raw/events.csv) | One recorded event | 212,476 | 14 | Ordered actions, timestamps, pages, click counts, durations, errors, payment signals, device, and acquisition channel |
| [`sessions.csv`](data/raw/sessions.csv) | One session | 20,000 | 12 | Session context, journey outcomes, simulator scenario, and synthetic ground-truth fields |

The data contains 11,003 unique users, so some users have more than one session. Optional event fields are populated only when relevant: for example, `error_type` is normally blank outside error events and `payment_status` is blank outside payment-related events. The notebooks preserve those structural nulls instead of treating them as bad rows.

The session table contains 1,385 injected suspicious-pattern sessions, a prevalence of 6.93%. `scenario`, `ground_truth_friction`, and `ground_truth_suspicious` are evaluation metadata; they are never model inputs.

See the [dataset notes](data/raw/DATASET_README.md) and [data dictionary](data/raw/DATA_DICTIONARY.md) for the source-field definitions.

## Repository structure

```text
.
|-- data/
|   |-- raw/
|   |   |-- events.csv
|   |   |-- sessions.csv
|   |   |-- DATASET_README.md
|   |   `-- DATA_DICTIONARY.md
|   `-- processed/
|       |-- funnel_summary.csv
|       |-- friction_comparison.csv
|       `-- session_features.csv
|-- notebooks/
|   |-- 01_data_cleaning_eda.ipynb
|   |-- 02_funnel_analysis.ipynb
|   |-- 03_friction_analysis.ipynb
|   `-- 04_anomaly_detection.ipynb
|-- .gitignore
|-- LICENSE
|-- README.md
`-- requirements.txt
```

## Analysis workflow

### 1. Data quality and exploratory analysis

[`01_data_cleaning_eda.ipynb`](notebooks/01_data_cleaning_eda.ipynb) loads both raw tables and verifies the assumptions required by the downstream analysis:

- required identifiers and timestamps are present;
- `event_id` and `session_id` contain no duplicates;
- timestamps parse correctly and are nondecreasing within each session;
- durations and click counts are nonnegative;
- event names belong to the expected set;
- event-level user, device, and acquisition values agree with session metadata;
- event counts and recorded milestones reconcile with the session table;
- every session begins with one `session_start` event.

The audit also retains and reports imperfect sequences. There are 10,783 adjacent timestamp ties, 1,385 sessions whose final recorded event is a click, and 473 purchase sessions without a `payment_attempt` event. Timestamp-derived durations differ from the finer session metadata by no more than 0.95 seconds. None of these records is silently removed or repaired.

The notebook plots event counts per session and elapsed session duration, using a logarithmic x-axis for the skewed duration distribution.

### 2. Ordered funnel analysis

[`02_funnel_analysis.ipynb`](notebooks/02_funnel_analysis.ipynb) measures the following sequence:

```text
all sessions -> product view -> add to cart -> checkout start -> payment attempt -> purchase
```

A session reaches a stage only when that event occurs after the preceding required stage. Stable row order resolves equal timestamps, and every session is counted at most once per stage.

| Stage | Sessions | Stage conversion | Stage drop-off | Overall reach |
|---|---:|---:|---:|---:|
| All sessions | 20,000 | - | - | 100.00% |
| Product view | 18,876 | 94.38% | 5.62% | 94.38% |
| Add to cart | 16,412 | 86.95% | 13.05% | 82.06% |
| Checkout start | 7,373 | 44.92% | 55.08% | 36.86% |
| Payment attempt | 6,627 | 89.88% | 10.12% | 33.14% |
| Purchase | 3,703 | 55.88% | 44.12% | 18.52% |

The strict funnel produces fewer purchases than the 4,176 observed purchase sessions because 473 purchases have no recorded payment-attempt stage. The notebook reports both values so that an instrumentation gap is not mistaken for customer behavior.

### 3. Friction indicators and statistical comparisons

[`03_friction_analysis.ipynb`](notebooks/03_friction_analysis.ipynb) aggregates chronological events into one feature row per session. The transparent indicator definitions are:

| Indicator | Definition | Affected sessions |
|---|---|---:|
| Long page | At least one `duration_seconds` value above 30 seconds | 10,715 (53.58%) |
| Payment failure | At least one `payment_failure` event | 3,496 (17.48%) |
| Back navigation | At least one `back_navigation` event | 2,124 (10.62%) |
| Repeated checkout | At least two `checkout_start` events | 1,409 (7.05%) |
| Repeated clicks | At least one click event with `click_count >= 3` | 1,379 (6.90%) |
| Technical error | At least one `page_error` event | 414 (2.07%) |

These indicators can overlap. They identify observable behavior, not its cause: a long duration could reflect difficulty, interest, or an idle tab.

Two predeclared comparisons use one randomly selected session per user, chosen with seed 42 before eligibility filtering. This leaves 11,003 independent-user sessions for inference while preserving all 20,000 sessions in descriptive outputs.

**Payment failures and purchase rate.** Among sessions with at least one payment attempt, a two-sided proportions z-test compares sessions with and without a payment failure. Both groups exceed the planning benchmark of 1,562 observations per group. The observed risk difference is -31.12 percentage points (95% Newcombe CI: -34.12 to -28.04; z = -18.981; p = 2.43e-80).

**Back navigation and duration.** Session duration is log-transformed before comparison. Group size, skew, tail behavior, positive finite values, and variance differences are checked in code. Because Levene's test indicates unequal variances, the notebook uses Welch's t-test. Sessions with back navigation have a geometric-mean duration ratio of 1.946 (95% CI: 1.911 to 1.982; Welch t = 72.108; Cohen's d = 1.202).

Both comparisons meet their predeclared sample-size benchmarks and remain significant after Holm correction across the two questions. The calculations show association only; the observational design cannot establish causality, and achieved power is not used as evidence of validity.

### 4. Label-blind anomaly detection

[`04_anomaly_detection.ipynb`](notebooks/04_anomaly_detection.ipynb) reads the session features created by Notebook 03 and uses an explicit allowlist of 13 browsing features:

- browsing event, click, and unique-page counts;
- browsing duration, median interval, event rate, and maximum page duration;
- product-view, add-to-cart, back-navigation, page-error, checkout-start, and repeated-click counts.

Only whitelisted browsing events on homepage, search, product, cart, and checkout pages contribute to these features. IDs, scenarios, ground-truth fields, purchase outcomes, payment outcomes, payment and confirmation pages, and full-session aggregates are excluded. Missing browsing intervals and rates are imputed with medians learned from the training split only.

`GroupShuffleSplit` assigns entire users to either training or evaluation, producing 14,044 training sessions and 5,956 held-out sessions with no shared users. Labels are loaded only after both methods have been fitted or applied:

- **Transparent rule:** flag a session when its browsing rate is strictly above the training 99th percentile (16.434 events per minute) or its product-view count is above the training 99th percentile (14 views).
- **Isolation Forest:** 200 trees, `max_samples=256`, fixed 5% training contamination, `random_state=42`, and single-threaded fitting for deterministic execution.

The held-out set contains 444 injected suspicious sessions, or 7.45%.

| Method | Flagged | Precision | Recall | F1 | False positives | False negatives | False-positive rate |
|---|---:|---:|---:|---:|---:|---:|---:|
| Percentile rule | 139 | 0.950 | 0.297 | 0.453 | 7 | 312 | 0.13% |
| Isolation Forest | 340 | 0.926 | 0.709 | 0.804 | 25 | 129 | 0.45% |

The rule is more selective, while Isolation Forest recovers substantially more injected suspicious sessions at the cost of additional false positives. Anomaly scores are ranking values, not probabilities.

## Visualizations

The executed notebooks contain the project figures and their supporting tables:

- [Session activity and duration distributions](notebooks/01_data_cleaning_eda.ipynb)
- [Ordered funnel reach and stage drop-off](notebooks/02_funnel_analysis.ipynb)
- [Friction prevalence and duration by back navigation](notebooks/03_friction_analysis.ipynb)
- [Browsing-feature distributions, confusion matrix, and anomaly-score distributions](notebooks/04_anomaly_detection.ipynb)

The notebooks are committed with their outputs so the charts can be reviewed directly on GitHub without rerunning the project.

## Generated outputs

| File | Contents |
|---|---|
| [`funnel_summary.csv`](data/processed/funnel_summary.csv) | Ordered unique-session reach, stage conversion, and absolute and relative drop-off |
| [`friction_comparison.csv`](data/processed/friction_comparison.csv) | Indicator prevalence plus purchase rates and median durations for affected and unaffected sessions |
| [`session_features.csv`](data/processed/session_features.csv) | 20,000 session rows with descriptive features, browsing-only model features, anomaly scores, flags, and train/held-out membership |

`session_features.csv` deliberately excludes scenario and ground-truth columns. Notebook 04 reloads those labels separately for evaluation and overwrites the feature export with its scores and flags.

## Technology

| Tool | Role |
|---|---|
| Python 3.11 | Analysis runtime |
| pandas and NumPy | Data validation, aggregation, feature engineering, and numerical work |
| Matplotlib and seaborn | Statistical charts and diagnostic plots |
| SciPy | Distribution diagnostics, Levene's test, Welch's t-test, and fallback exact testing |
| statsmodels | Proportions testing, Newcombe confidence interval, and Holm correction |
| scikit-learn | Grouped train/holdout split, median imputation, Isolation Forest, and evaluation metrics |
| Jupyter and nbconvert | Executable analysis and clean-kernel reproducibility |

Dependency ranges are defined in [`requirements.txt`](requirements.txt).

## Reproducing the analysis

Python 3.11 is recommended.

Clone or download the repository, open a terminal in its root directory, and create a virtual environment:

```bash
python -m venv .venv
```

Activate the environment and install the dependencies.

```bash
# Windows PowerShell
.venv\Scripts\Activate.ps1
python -m pip install -r requirements.txt
```

```bash
# macOS or Linux
source .venv/bin/activate
python -m pip install -r requirements.txt
```

For interactive use:

```bash
jupyter lab
```

Run the notebooks in numeric order:

1. `01_data_cleaning_eda.ipynb`
2. `02_funnel_analysis.ipynb`
3. `03_friction_analysis.ipynb`
4. `04_anomaly_detection.ipynb`

Notebooks 01 through 03 read the raw CSV files independently. Notebook 03 creates `session_features.csv`; Notebook 04 requires that file and adds the final model results. Paths work when Jupyter is launched from the repository root or from the `notebooks` directory.

To execute the full workflow from clean kernels at the command line:

```bash
python -m nbconvert --to notebook --execute --inplace notebooks/01_data_cleaning_eda.ipynb --ExecutePreprocessor.timeout=300
python -m nbconvert --to notebook --execute --inplace notebooks/02_funnel_analysis.ipynb --ExecutePreprocessor.timeout=300
python -m nbconvert --to notebook --execute --inplace notebooks/03_friction_analysis.ipynb --ExecutePreprocessor.timeout=300
python -m nbconvert --to notebook --execute --inplace notebooks/04_anomaly_detection.ipynb --ExecutePreprocessor.timeout=300
```

The committed notebooks have sequential execution counts and no error outputs. The analysis uses seed 42 for random session selection, user-group splitting, and Isolation Forest fitting.

## Practical interpretation

The cart-to-checkout transition is the clearest place to investigate first because it loses the most sessions both absolutely and proportionally. Useful follow-up work would instrument shipping-cost exposure, checkout entry points, cart edits, and exit reasons around that transition.

Payment failures are associated with lower purchase completion among sessions that attempted payment, so failure codes and retry recovery deserve closer analysis. Repeated clicks and navigation loops are candidates for targeted journey review, particularly where latency or unclear interaction feedback could be measured directly.

The anomaly model is suitable as a prioritization mechanism for review. It should not trigger enforcement or customer-facing action without reviewed outcomes, cost-sensitive threshold selection, and monitoring on real traffic.

## Limitations

- The data and labels are synthetic; their separability may not resemble real customer behavior.
- The random holdout separates users but does not test performance on a future time period.
- Session aggregates lose detailed sequence patterns and cross-session account or device context.
- Friction indicators overlap and use explicit analytical thresholds rather than validated product definitions.
- The 5% Isolation Forest contamination setting represents a review-capacity choice, not an estimated anomaly rate.
- Structural gaps remain in the event stream, including 473 purchases without a recorded payment attempt and 1,385 sessions ending with click events.
- The statistical comparisons are observational and may be affected by confounding.
- There is no human review, causal experiment, production calibration, or drift monitoring.

## Future improvements

- Validate event instrumentation and distinguish missing events from legitimate alternate journeys.
- Collect human-reviewed outcomes and evaluate the detector on a time-based holdout.
- Calibrate thresholds against review capacity and the relative cost of false positives and false negatives.
- Add sequence-aware features such as repeated page loops, transition counts, and time spent between specific funnel stages.
- Segment friction by device and acquisition channel while controlling the number of statistical comparisons.
- Test product hypotheses through controlled experiments rather than inferring impact from observational associations.
- Add automated data-contract and notebook-execution checks to continuous integration.

## License

This project is available under the [MIT License](LICENSE).
