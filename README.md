# 30-Day Hospital Readmission in Diabetic Patients

Predicting whether a diabetic patient will be readmitted within 30 days of discharge.
101,766 hospital stays from 130 US hospitals, 1999-2008.

Final model: XGBoost, 0.6673 AUC.

## Why this project

I wanted to get into healthcare analytics, specifically Stars and HEDIS work. Readmission
seemed a direct way in. This is hospital encounter data rather than plan claims, so it is a
hospital-side readmission problem, not a plan-level quality measure.

## Files

```
Medicare/
├── data/
│   ├── diabetic_data.csv      101,766 rows, 50 columns
│   └── IDS_mapping.csv        decodes the id columns
└── notebooks/
    ├── 01_eda.ipynb           exploration, 12 charts
    └── 02_modelling.ipynb     cleaning, models, fairness
```

The two CSVs are not in the repo. Download the Diabetes 130-US Hospitals dataset from the
UCI Machine Learning Repository and put `diabetic_data.csv` and `IDS_mapping.csv` in `data/`.

```bash
pip install -r requirements.txt
jupyter lab notebooks/01_eda.ipynb
```

Both notebooks read from `../data/` and save nothing.

## Results

| Metric | Value |
|---|---|
| XGBoost AUC | 0.6673 |
| Logistic regression AUC | 0.6506 |
| `number_inpatient` alone | 0.6022 |
| Top-decile lift | 2.41x |
| Brier after calibration | 0.0953 (from 0.2122) |
| Positive rate | 11.2% |

112 features add 0.065 AUC over sorting patients by last year's admission count.

## Model choice

| Model | AUC |
|---|---|
| XGBoost | 0.6673 |
| Hist gradient boosting | 0.6659 |
| Random forest | 0.6625 |
| Extra trees | 0.6532 |
| Decision tree (depth 5) | 0.6507 |
| Logistic regression | 0.6506 |
| Gaussian naive Bayes | 0.6099 |

All tree ensembles land within 0.005 of each other. The full spread is 0.017, about the same
as the leakage correction. Model choice is not where the gains are on this dataset, so two
are enough: logistic regression as a baseline, XGBoost as the best of the rest.

KNN and SVM were skipped (112 mostly binary features, 80k rows). SHAP's TreeExplainer is
exact for trees, which the per-patient explanations rely on. These were run near-default, not
tuned.

## Four decisions

**Removed patients who cannot be readmitted.** 2,423 stays have a discharge code of expired
or hospice. Code 11 has a readmission rate of 0.000 across 1,642 patients. Removing them
drops AUC by 0.006.

**Split by patient, not by row.** 101,766 stays but 71,518 patients. A random split puts
6,798 patients on both sides and inflates AUC by 0.0066.

**Did not report accuracy as the headline.** Predicting "nobody returns" scores 88.8%. Both
models score below that and are more useful.

**Calibrated the probabilities.** Isotonic regression halves the Brier score without changing
AUC. It also showed the 0.5 threshold was meaningless here: afterwards only 0.2% of patients
sit above it.

## From the EDA

- Prior-year admissions is the strongest single field. 8.4% readmission at zero prior
  admissions, 36.4% at five or more.
- 83% of stays never got an A1C test. That group returns more often (11.42% vs 9.85%).
- A glucose test points the other way (12.36% vs 11.09%) because it is ordered reactively.
- Insulin dose changes mark unstable patients: 13.0-13.9% against 10.0% for patients on none.
- The long tails are real patients. `number_emergency` reaches 76 visits a year. Not clipped.

## What it does

Ranks patients by risk. Contacting the top 10% reaches 24% of the people who were readmitted,
against 10% at random. Contacting half reaches 72%.

It does not decide anything about an individual: at a 0.5 threshold, 82% of flagged patients
never come back. It is not causal. The data shows who returned, not what would have happened
if someone had called them.

## Limitations

- The 80-90 age band is flagged at 48% against an 11.9% actual rate. AUC drops to 0.54 for
  the over-90s. Not fixed.
- Readmission depends on housing, medication cost and access to follow-up care. None of that
  is in the data, which is why 0.67 is the ceiling.
- The data is from 1999-2008.

