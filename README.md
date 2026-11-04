 Attendance and Residence Classification

This project prepares student attendance data and uses it to classify whether a
student lives **on campus** or **off campus**. The complete workflow is
implemented in [`01_transformation.ipynb`](./01_transformation.ipynb), from
loading the raw files through model evaluation and feature-importance
visualization.

## Project structure

```text
.
├── 01_transformation.ipynb
├── data/
│   ├── raw/
│   │   ├── offcampus.xlsx
│   │   ├── oncampus.xlsx
│   │   └── 01_transformation.ipynb
│   └── processed/
│       └── attendance_active_only.csv
├── LICENSE
└── README.md
```

- `data/raw/oncampus.xlsx` and `data/raw/offcampus.xlsx` are the source
  attendance files.
- `data/processed/attendance_active_only.csv` is the generated modeling
  dataset.
- `data/raw/01_transformation.ipynb` is an older/empty notebook copy and is
  not the active workflow.

## What the notebook does

 1. Load and inspect the source data

The notebook loads both Excel files with pandas and prints their dimensions:

- On-campus: 905 rows and 84 columns
- Off-campus: 3,890 rows and 83 columns

2. Remove identifying fields

The `Name` and `Admission No` columns are removed when present. This prevents
direct student identifiers from being used as model inputs.

### 3. Align attendance sessions

Column names are converted to strings and stripped of surrounding whitespace.
Only columns shared by both datasets are retained, producing 83 common session
columns before later duplicate removal.

### 4. Clean and encode attendance values

Attendance values are normalized by:

- stripping whitespace;
- converting text to uppercase;
- converting `PRESENT` to `P`;
- converting `ABSENT` to `A`;
- converting empty, `NaN`, and `None`-like values to missing values;
- removing duplicate rows and duplicate columns.

The cleaned codes are mapped to numeric values:

- `P` → `1`
- `A` → `0`

Rows from the on-campus file receive `Residence = 1`, while rows from the
off-campus file receive `Residence = 0`. The two datasets are then concatenated.

### 5. Create summary features and keep active students

Two summary columns are calculated:

- `Attendance_Rate`: mean attendance across session columns.
- `Total_Absences`: number of sessions encoded as absent.

Students with an attendance rate of zero are removed. The resulting processed
dataset contains 2,452 active students and is saved as
`data/processed/attendance_active_only.csv`.

### 6. Split the modeling data

The session attendance columns are used as model features. The summary columns
and `Residence` target are excluded from `X`. The data is divided into:

- 80% training data: 1,961 students
- 20% test data: 491 students

The split uses `random_state=42` and stratification so that the residence-group
distribution is preserved.

### 7. Train and evaluate models

Two class-balanced classifiers are evaluated:

1. **Decision Tree**
   - Gini criterion
   - maximum depth of 5
   - `random_state=42`
2. **Random Forest**
   - 100 estimators
   - maximum depth of 8
   - `random_state=42`

Both models use five-fold cross-validation with ROC-AUC scoring. They are then
fit on the training set and evaluated on the held-out test set using
classification metrics and ROC-AUC.

Recorded results from the notebook:

| Model | 5-fold CV ROC-AUC | Test ROC-AUC | Test accuracy |
|---|---:|---:|---:|
| Decision Tree | 0.690 | 0.740 | 0.76 |
| Random Forest | 0.787 | 0.805 | 0.77 |

The random forest performed better on ROC-AUC and was used to calculate the
relative importance of attendance sessions. The notebook plots the ten most
important sessions.

## How to run the project

### Requirements

Use Python 3 with the following packages:

```bash
pip install pandas numpy scikit-learn matplotlib openpyxl jupyter
```

### Run the notebook

From the project root:

```bash
jupyter notebook 01_transformation.ipynb
```

Run the cells from top to bottom. The notebook expects the raw files at:

```text
data/raw/oncampus.xlsx
data/raw/offcampus.xlsx
```

It recreates the processed output at:

```text
data/processed/attendance_active_only.csv
```

## Interpretation and limitations

- `Residence` is a label derived from which source file a row came from; it is
  not an independently observed field.
- Missing attendance values remain missing after encoding and are ignored by
  pandas when calculating row means. The behavior of downstream estimators
  should be checked if the raw data contains substantial missingness.
- The source data is imbalanced, with more off-campus than on-campus students.
  Class weighting and stratified splitting are used to reduce the effect of
  this imbalance.
- The reported metrics are the outputs recorded in the notebook and can
  change if the source files or preprocessing steps change.

## License

This project is distributed under the Apache License 2.0. See
[`LICENSE`](./LICENSE) for the full text.
