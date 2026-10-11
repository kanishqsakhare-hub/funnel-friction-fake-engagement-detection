# Funnel Friction & Fake Engagement Detection

A reproducible, notebook-first product analytics project that measures an e-commerce session funnel, investigates behavioral friction, and compares transparent rules with unsupervised anomaly detection.

## Business problem and objectives

Aggregate conversion rates do not explain where a journey breaks down or whether unusually intense activity is genuine interest. This project asks:

1. Where do sessions leave the product-view-to-purchase funnel?
2. Which observed behaviors co-occur with abandonment or long sessions?
3. Which sessions have unusual engagement patterns?
4. How well do fixed behavioral rules and Isolation Forest recover injected suspicious scenarios?
5. Which product questions should be investigated next?

The unit of analysis is a session. Funnel reach always uses unique sessions, not event-row counts.

## Dataset

The repository contains 212,476 event rows across 20,000 sessions. `events.csv` records ordered actions and behavioral telemetry; `sessions.csv` supplies session context, outcomes, simulator scenarios, and ground-truth labels.

All data is synthetic. `ground_truth_suspicious` identifies an injected scenario, not verified fraud or actual user intent. Performance on this dataset demonstrates the evaluation workflow only and does not establish real-world detection accuracy.

See [the dataset notes](data/raw/DATASET_README.md) and [data dictionary](data/raw/DATA_DICTIONARY.md).

## Project structure

```text
.
├── data/
│   ├── raw/                 # Supplied immutable CSVs and documentation
│   └── processed/           # Notebook-generated analysis tables
├── notebooks/
│   ├── 01_data_cleaning_eda.ipynb
│   ├── 02_funnel_analysis.ipynb
│   ├── 03_friction_analysis.ipynb
│   └── 04_anomaly_detection.ipynb
├── .gitignore
├── LICENSE
├── README.md
└── requirements.txt
```

Generated outputs are:

- `funnel_summary.csv`: unique-session reach and explicitly defined conversion/drop-off metrics.
- `friction_comparison.csv`: affected/unaffected cohort sizes, purchase rates, and median durations.
- `session_features.csv`: engineered session features, rule/model flags, scores, and evaluation groups.

## Installation and execution

Python 3.11 is recommended.

```bash
python -m venv .venv
# Windows
.venv\Scripts\activate
python -m pip install -r requirements.txt
jupyter lab
```

Run the notebooks in numeric order. Notebooks 01–03 read the immutable raw CSV files directly and are independently runnable. Notebook 04 consumes `data/processed/session_features.csv`, which Notebook 03 regenerates.

Paths resolve correctly when notebooks are launched from the repository root or from `notebooks/`. For non-interactive reproducibility checks:

```bash
python -m nbconvert --to notebook --execute --inplace notebooks/01_data_cleaning_eda.ipynb --ExecutePreprocessor.timeout=300
python -m nbconvert --to notebook --execute --inplace notebooks/02_funnel_analysis.ipynb --ExecutePreprocessor.timeout=300
python -m nbconvert --to notebook --execute --inplace notebooks/03_friction_analysis.ipynb --ExecutePreprocessor.timeout=300
python -m nbconvert --to notebook --execute --inplace notebooks/04_anomaly_detection.ipynb --ExecutePreprocessor.timeout=300
```

## Methodology

- `01_data_cleaning_eda.ipynb` validates identifiers, timestamps, event types, referential integrity, and session aggregates, then establishes baseline distributions.
- `02_funnel_analysis.ipynb` defines the ordered funnel as all sessions → product view → add to cart → checkout start → payment attempt → purchase.
- `03_friction_analysis.ipynb` engineers chronological session features and compares affected and unaffected cohorts. Reproducible z- and t-tests include power-based sample-size calculations and assumption checks.
- `04_anomaly_detection.ipynb` applies a fixed percentile rule and a label-blind Isolation Forest. Identifiers, scenario/ground-truth fields, and purchase outcomes are excluded from model inputs.
- Injected suspicious labels are used only after fitting for evaluation and error analysis.

## Main findings

- Observed purchase conversion was **20.88%** (4,176 of 20,000 sessions). Strict ordered-funnel completion was **18.52%** (3,703 sessions): **473** purchase sessions have no recorded payment-attempt event. These records were retained; missing events were not fabricated.
- The largest absolute and relative funnel loss was add to cart → checkout start: **9,039 sessions (55.08%)**. Ordered payment attempt → purchase conversion was **55.88%**.
- With the explicit duration-above-30-seconds definition, long-page signals affected **10,715 sessions (53.58%)**. Payment failures affected **3,496 (17.48%)**. These descriptive indicators overlap and depend on analytical thresholds.
- Comparisons used one seed-42 randomly selected session per user (11,003 sessions) to avoid repeated-user dependence. Among payment-attempt sessions, the payment-failure cohort's purchase rate was **31.12 percentage points lower** (95% CI −34.12 to −28.04). Sessions with back navigation had **1.946 times** the geometric-mean duration (95% CI 1.911–1.982). Both associations passed Holm correction; planning benchmarks were met, without any achieved-power claim.
- Isolation Forest and percentile rules used only browsing-event/page features; outcomes, IDs, scenarios and synthetic labels were excluded from inputs. Training contained **14,044 sessions**; evaluation held out **5,956 sessions from disjoint users**. Labels were loaded only after fitting and flagging.
- On held-out synthetic sessions, the fixed percentile rule flagged **139**: precision **0.950**, recall **0.297**, F1 **0.453** (7 false positives; 312 false negatives).
- With fixed 5% training contamination, Isolation Forest flagged **340 held-out sessions**: precision **0.926**, recall **0.709**, F1 **0.804** (25 false positives; 129 false negatives; false-positive rate **0.45%**).
- All four notebooks passed sequential clean-kernel execution. All three exported CSVs reproduced byte-for-byte; schema, session uniqueness, funnel accounting and feature-provenance checks passed.

Limitations: all records are synthetic; 1,385 sessions end with click events, and timestamps differ from finer metadata durations by at most 0.95 seconds. Detection is retrospective; the holdout is by user, not future time. These findings establish neither causality, confirmed fraud, nor real-world detection accuracy. Dataset notes and the data dictionary are stored with the raw data.

## Product investigations

The notebook prioritizes hypotheses rather than promised impact: instrument the add-to-cart/checkout transition; analyze payment-failure codes and retry recovery; inspect repeated-click and navigation-loop journeys for latency or usability issues; and use anomaly scores only for review prioritization until validated on time-separated, human-reviewed examples.

## Limitations and future work

- Synthetic distributions and injected labels may be easier to separate than real behavior.
- Session aggregates lose within-session sequence detail and shared-device/account context.
- Percentile rules and the 5% review capacity are analytical choices, not universal thresholds.
- No causal design, human review, temporal holdout, or production drift analysis is available.
- Future work should validate instrumentation, collect reviewed outcomes, evaluate stability over time, calibrate thresholds to review capacity/costs, and test product hypotheses experimentally.
