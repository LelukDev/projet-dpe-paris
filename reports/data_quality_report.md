# Data-Quality Report — DPE of existing dwellings in Paris (ADEME `dpe03existant`)

**Group:** Elie Maarek, Joshai Bensoussan, Lucas Noury · **Date:** 2026-11-05 · **Notebook:** `notebooks/01_collection_quality.ipynb` · **Commit:** `e2d7470`
**Clean-data fingerprint:** `78abd427b5da`


## 1. Source & provenance
| | |
|---|---|
| Business question | How are DPE labels (A to G) distributed in Paris, and how do they vary with construction period, surface and heating? |
| Source(s) & endpoint / URL | ADEME open data, data-fair API, dataset `dpe03existant` · https://data.ademe.fr/datasets/dpe03existant |
| Collection method & date | REST API with the cached session, filter `code_departement_ban:"75"`, `sort=_rand`: 10,000 random DPE out of 862,377 (1.2 %), collected on 2026-11-05. Reruns read `data/raw/raw.parquet` (`RECOLLECT = False`) |
| Licence, ToS constraints | Licence Ouverte 2.0 (Etalab), cite ADEME. Open API, no key |
| Unit of observation | One row = one DPE. After cleaning, one row = one DPE of one dwelling (whole-building DPE removed) |
| Final size (rows × columns) | Raw 10,000 × 223 → clean **9,754 × 20** |

## 2. Schema (the contract)
Strict pandera schema on 20 columns (notebook 3.4), `lazy=True`. Key checks:
- `numero_dpe` unique — duplicated DPE; `date_etablissement_dpe` from 2021-07-01 — old method; `type_batiment` in appartement, maison, immeuble — out-of-scope types
- `etiquette_dpe`, `etiquette_ges` in A to G; consumption > 0, GHG ≥ 0 — typos, impossible values
- `surface_habitable_logement` 5 to 1,000 m² raw, required and 9 to 500 m² clean; `annee_construction` 1500 to 2026; heat loss and ubat ≥ 0 — absurd or negative values
- `strict=True`, `coerce=True` — extra columns, wrong types; `nullable` only on the 5 columns that have empty values in the raw data

Validation on raw data: **3 violations**, all on one check: surface outside 5 to 1,000 m² (3 m², 2,464 m², 3,450 m²), see `reports/schema_failures.csv`. Validation on clean data: **passes**.

## 3. Missingness
| Column | % missing | Mechanism | Evidence | Decision |
|---|---|---|---|---|
| `annee_construction` | 50.2 % | MAR | 46-point gap between `methode_application_dpe` groups (4.3) | column dropped, `periode_construction` replaces it |
| `type_installation_chauffage` | 0.5 % (53) | structural | empty for 100 % of houses, 0 % elsewhere (4.3) | kept, category "unknown" in the model |
| `deperditions_enveloppe` | 0.2 % (23) | MAR | 13-point gap between DPE types (4.3) | kept, impute in the model pipeline (train stats) |
| `numero_dpe_remplace` | 88.6 % | structural | empty when the DPE replaces nothing | kept as is |

Test (4.3): share of empty values per `methode_application_dpe` group (≥ 50 rows); gap > 10 points = MAR, always empty for one type = structural.

## 4. Duplicates & outliers — decision table
Condensed from `reports/decision_table.csv` (notebook 4.4).

| Column | Issue | n rows | Decision | Why |
|---|---|---|---|---|
| `type_batiment` | whole-building DPE | 167 | removed | not a dwelling, surface empty |
| `methode_application_dpe` | DPE generated from the building DPE | 3,219 | kept + `dpe_immeuble_genere` | neighbours look alike: split train/test by building |
| (all), `numero_dpe_remplace` | identical rows, old DPE next to its replacement | 0 | nothing to do | checked, none found |
| `identifiant_ban` | near-identical dwellings in one building | 429 | kept | often really different dwellings |
| `surface_habitable_logement` | empty or outside 9 to 500 m² | 79 | removed | impossible for a dwelling |
| `etiquette_dpe` | label differs from the recomputed one | 1,398 | kept | mismatch falls from 76 % under 15 m² to 2 % above 40 m², label better in 91 %: small-dwelling thresholds |
| `deperditions_enveloppe` | building-scale value for generated DPE | 3,216 | kept, not comparable | median 155 vs 2.8 W/K per m² |
| `annee_construction` | 50 % missing, MAR | 4,899 | column dropped | more than 30 % empty |
| `type_installation_chauffage` | 0.5 % missing, structural (houses) | 53 | kept, "unknown" | value does not exist for houses |
| `deperditions_enveloppe` | 0.2 % missing, MAR | 23 | kept, impute later | tiny, train stats |
| `zone_climatique` | 99.7 % identical (H1a) | 48 | column dropped | almost constant |
| `conso_5_usages_par_m2_ep` | extreme values (IQR, log scale) | 158 | kept + `conso_outlier` | 80 very high (all G, median 17 m²), 78 very low (A to C): real values |

## 5. Key findings (max 3)
1. **18 % of dwellings are F or G, A and B only 1.3 %** — Fig. 5.1 — so what: very unbalanced, group A-B and F-G and use macro F1.
2. **Building age is the clearest variable**: 26 % F or G before 1948, 12 % for 1948-1974, about 1 % from 1978; fuel oil 65 % (89 dwellings), electricity 23 %, gas 17 %, district heating 6 % — Fig. 5.2 — so what: period first; heating only next to surface (electric dwellings are smaller, at equal surface electricity ≈ gas, 5.4).
3. **For the same period, small dwellings are more often F or G**: before 1948, 32 % under 30 m² vs 15 % above 60 m² — Fig. 5.3 — so what: keep surface; part of it is the small-surface label rule.

## 6. What cannot be trusted ⚠️
- **Target leakage.** `conso_5_usages_par_m2_ep`, `emission_ges_5_usages_par_m2`, `etiquette_ges` (and `conso_outlier`) compute the label (worse of the energy and GHG classes): never use them as features. Under 40 m², 30 % of labels do not match the calculation (76 % under 15 m²): small dwellings have adjusted thresholds we do not have.
- **DPE generated from the building DPE** (33 % of rows). Neighbours look alike and they are rarely F or G (6 % vs 24 %). Split train/test by `identifiant_ban`.
- **Small groups**: fuel oil (89), LPG (15), wood (2), coal (1), "après 2021" (24), class A (9).
- **Intermediate variables** (heat loss, ubat, insulation) come from the same calculation as the consumption. In notebook 02 they lift the random forest from 43 % to 64.9 % accuracy, but this is not a prediction made before the diagnosis.
- **Sample**: 1.2 % of Paris, one date, DPE only exist for a sale, a rental or a new building.
- **Heat loss of generated DPE.** For the 3,216 generated DPE, `deperditions_enveloppe` is the heat loss of the whole building (median 155 vs 2.8 W/K per m², 4.2). It is the forest's first variable: prefer `ubat_w_par_m2_k` (per m²).

## 7. Hypotheses for modelling (≥ 5)
- **H1** — `periode_construction`: before 1948 gives more F or G than from 1978. Population: all 9,754 dwellings. Test: chi-square.
- **H2** — `surface_habitable_logement`: for the same period, the smaller the dwelling, the more F or G. Population: all dwellings. Test: logistic regression (period, surface).
- **H3** — `type_energie_principale_chauffage`: at equal surface class, fuel oil gives more F or G than gas, and electricity is close to gas. Population: heating types with at least 50 dwellings. Test: chi-square inside each surface class.
- **H4** — `qualite_isolation_enveloppe`: "insuffisante" gives more F or G than "bonne" (22 % vs under 4 %). Population: all dwellings. Test: chi-square (partly by construction, see section 6).
- **H5** — `dpe_immeuble_genere`: generated DPE are less often F or G, and the gap comes from the building type. Population: all dwellings at equal period and heating. Test: stratified comparison.

## 8. How to reproduce
```bash
python3 -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt jupyter
jupyter nbconvert --to notebook --execute --inplace notebooks/01_collection_quality.ipynb
```
Expected fingerprint: `78abd427b5da` (the notebook reads `data/raw/raw.parquet`; `RECOLLECT = True` draws a new sample).
