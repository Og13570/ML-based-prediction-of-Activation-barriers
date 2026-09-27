# ML-based Prediction of Activation Barriers

Machine-learning models that predict **activation barriers (eV)** of surface reactions on
transition-metal catalysts (C1/CO chemistry: C–H, C–O, C–C bond making and breaking) from
DFT-derived descriptors. The trained models are then used to predict C2 reaction barriers on
**Ni(111)** and **NiB** surfaces and compared against DFT values.

## Repository contents

| File | Description |
|------|-------------|
| `1st_90_10_Model_1_Hyperparameter.ipynb` | Main notebook: data processing, hyperparameter optimization of 10 regressors, prediction on the validation set, plots |
| `Train_Test_Data_with_Descriptor.xlsx` | 1548 DFT reactions (Ru, Co, Pt, Rh, Ni, NiB, Cu, Pd, Ag, Ir) with descriptors — split 90/10 into train/test |
| `Validation_Data_with_Descriptor.xlsx` | 20 C2 reactions on Ni and NiB used as an external validation set |
| `Sample dataset.xlsx` | First rows of the dataset, showing the column format |

## Dataset columns

* **Target:** `Activation Barrier_eV`
* **Reaction energy:** `Delta E_eV`
* **Catalyst:** `Period`, `Group`, `Coordination Number` (`Metal`, `Surface` are kept for bookkeeping only)
* **DFT settings (one-hot encoded):** `Functional`, `vdW Correction` (empty = none), `Energy Term`
* **Reaction descriptors:** `delta_*` (change from reactants to products) and `max_*` descriptors for
  C, CO, O, H, C–C, C–CO, CO–CO, CO–O (`a_*` and `n_*` variants)
* **Not used as features:** `Metal`, `Surface`, `Reagent A/B`, `Product C/D`, `Code`

Columns that are zero for every row (`delta_n_O`, `delta_n_H`) are dropped automatically.

## Workflow (notebook)

1. **Data processing** — one function, `prepare_features`, is used for both the training and the
   validation data: it drops the identifier columns, fills a missing vdW correction with `None`,
   drops the all-zero columns and one-hot encodes the categorical columns using the encoder fitted
   on the training data. The data is then split 90/10 into train/test (`random_state=42`), either
   randomly or by reaction group (see `SPLIT_BY_GROUP`).
2. **Hyperparameter optimization** — 5-fold `GridSearchCV` (scoring: MAE) for
   LR, RFR, GBR, XGBR, DTR, ETR, SVR, KRR, KNN and GPR. Each model runs inside a
   `StandardScaler → model` pipeline. GBR picks its number of trees by early stopping, and
   hyperparameters that do not change a model (e.g. `degree` for non-polynomial kernels) are not
   searched.
3. **Evaluation** — MAE, RMSE and R² of every best model on the train, held-out test and Ni/NiB
   validation data; parity plots of the test set; permutation feature importance of the best
   model.
4. **Validation plots** — predict the Ni / NiB C2 reactions with every model and plot ML vs. DFT
   barriers.

## Getting started

```bash
python -m venv .venv
source .venv/bin/activate        # Windows: .venv\Scripts\activate
pip install -r requirements.txt
jupyter notebook 1st_90_10_Model_1_Hyperparameter.ipynb
```

Settings in section **0.1 Run Settings** of the notebook:

| Setting | Default | Meaning |
|---------|---------|---------|
| `QUICK_RUN` | `False` | `True` uses only the first value of each hyperparameter, so the whole notebook runs in about a minute — use it to check your setup before the full search |
| `SCALE_FEATURES` | `True` | Standardize features inside the pipeline (needed for SVR, KRR, KNN, GPR; no effect on tree models) |
| `SPLIT_BY_GROUP` | `False` | `True` keeps all rows of the same reaction on the same metal surface (computed with different functionals / vdW corrections) together in train or test and in the CV folds. This gives a more honest error for new reaction–surface combinations. `False` is a random split |
| `N_JOBS` | `-1` | CPU cores used by `GridSearchCV` |
| `FIGURE_DPI` | `300` | Resolution of the saved figures |
| `RANDOM_STATE` | `42` | Seed for the split and the models |

To run the full search unattended:

```bash
pip install papermill
papermill 1st_90_10_Model_1_Hyperparameter.ipynb output.ipynb
```

## Outputs

| File | Content |
|------|---------|
| `Data_With_Reaction.xlsx` | Train/test rows with the reaction written out |
| `Best_Models.xlsx` / `.csv` | Best CV score and hyperparameters per algorithm |
| `All_GridSearch_Results.xlsx` / `.csv` | CV train/test score of every hyperparameter combination |
| `Best_Estimators.joblib` | Fitted best model per algorithm (`joblib.load("Best_Estimators.joblib")`) |
| `Model_Performance.xlsx` | MAE / RMSE / R² of every model on train, test and validation data |
| `Parity_Test_Set.jpg` | DFT vs. ML parity plot of the test set for every model |
| `Feature_Importance.xlsx` / `.jpg` | Permutation feature importance of the best model |
| `Prediction_Validation_Data_with_Descriptor.xlsx` | DFT vs. predicted barriers for the Ni / NiB validation reactions |
| `Ni NiB Comparison <model>.jpg`, `DFT ML Comparison <model>.jpg` | Comparison plots |

Results are saved after every algorithm finishes, so a crash late in the grid search does not lose
the earlier results.
