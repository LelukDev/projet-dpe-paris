# DPE Paris project

Project for the course **MSA-DATI07-03 · Advanced EDA & Data Collection**.
Group: Elie Maarek, Joshai Bensoussan, Lucas Noury.

We collect energy performance diagnoses (DPE) of Paris dwellings from the ADEME API, document their quality, explore them to see what makes a dwelling labelled A or G, and compare three models that predict the label.

## Question

How are DPE labels (A to G) distributed in Paris, and how do they vary with construction period, surface and heating? Which model predicts the label best?

## Data

- Source: ADEME, dataset `dpe03existant` ([data.ademe.fr](https://data.ademe.fr/datasets/dpe03existant)), Licence Ouverte 2.0.
- Collected on 5 November 2026 through the API: 10,000 DPE drawn at random (`sort=_rand`) out of the 862,377 of Paris (`code_departement_ban:"75"`).
- Details (date, filters, columns) are in `data/raw/README.md`.
- After cleaning: 9,754 dwellings and 20 columns.

## Repository content

```
notebooks/01_collecte_qualite.ipynb   collection, schema, data quality, exploration (sections 1 to 6)
notebooks/02_models.ipynb             logistic regression by hand, decision tree, random forest
data/raw/raw.parquet                  raw collected data (and its README)
data/processed/clean.parquet          clean data (written by notebook 01)
reports/data_quality_report.md        data quality report (2 pages)
reports/decision_table.csv            cleaning decisions
reports/schema_failures.csv           schema errors on the raw data
reports/5_1.png, 5_2.png, 5_3.png     exploration figures
reports/6_1_loss.png, 6_2_confusion_matrix.png, model_comparison.csv   model results
docs/brief_projet.pdf                 project brief
requirements.txt                      packages and versions
```

## Run the project

```
python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt jupyter
jupyter nbconvert --to notebook --execute --inplace notebooks/01_collecte_qualite.ipynb
jupyter nbconvert --to notebook --execute --inplace notebooks/02_models.ipynb
```

Notebook 01 prints the number of rows and a fingerprint of the clean data at the end. It reads the committed `data/raw/raw.parquet`, so every run gives the same fingerprint (`78abd427b5da`). With `RECOLLECT = True` it calls the API again and draws other random rows, so the fingerprint changes: this is expected. Notebook 02 reads `data/processed/clean.parquet`, so run notebook 01 first.

## Main results

- 18 % of dwellings are F or G, classes A and B are only 1.3 %.
- Building age matters most: 26 % F or G before 1948, about 1 % from 1978.
- For the same period, small dwellings are more often F or G. Electric heating looks worse than gas, but mostly because electric dwellings are smaller.
- Models (test set split by building): on descriptors only, the three models reach 41 to 43 % accuracy (31 % for the most frequent class). With intermediate variables (heat loss, ubat, insulation), the random forest reaches 64.9 % accuracy and 97 % within one class. Our logistic regression coded by hand gives the same accuracy as scikit-learn with the same L2 penalty.

## Limits

- The label is computed from consumption and GHG emissions: these columns must not be used to predict it.
- A third of the DPE are generated from the building DPE: train and test must be split by building.
- The intermediate variables come from the same calculation as the consumption: the 64.9 % is not a prediction made before the diagnosis.
- For DPE generated from the building, the heat loss is the one of the whole building.
- The sample (1.2 % of Paris, DPE made for sales or rentals) does not represent all dwellings.

More details in `reports/data_quality_report.md`.
