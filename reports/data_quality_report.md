# Data-Quality Report — DPE of existing dwellings in Paris (ADEME `dpe03existant`)

**Group:** Elie Maarek, Joshai Bensoussan, Lucas Noury · **Date:** 2026-11-05 · **Notebook:** `notebooks/01_collecte_qualite.ipynb` · **Commit:** `8f682d5`
**Clean-data fingerprint:** `78abd427b5da`


## 1. Source & provenance
| | |
|---|---|
| Business question | How are DPE labels (A to G) distributed in Paris, and how do they vary with construction period, surface and heating? Which variables predict the label? |
| Source(s) & endpoint / URL | ADEME open data, data-fair API, dataset `dpe03existant` · https://data.ademe.fr/datasets/dpe03existant |
| Collection method & date | REST API with the cached session, filter `code_departement_ban:"75"`, `sort=_rand`: 10,000 random DPE out of 862,377 (1.2 %), collected on 2026-11-05 |
| Licence, ToS constraints | Licence Ouverte 2.0 (Etalab), cite ADEME. Open API, no key |
| Unit of observation | One row = one DPE. After cleaning, one row = one dwelling (whole-building DPE removed) |
| Final size (rows × columns) | Raw 10,000 × 223 → clean **9,754 × 20** |

## 2. Schema (the contract)
Strict pandera schema on 20 columns (notebook 3.4), `lazy=True`. Key checks:
- `numero_dpe` unique, not null — duplicated DPE; `date_etablissement_dpe` from 2021-07-01 — old calculation method
- `type_batiment` appartement, maison, immeuble (clean: dwellings only) — out-of-scope types
- `etiquette_dpe`, `etiquette_ges` in A to G; consumption > 0, GHG ≥ 0 — typos, impossible values
- `surface_habitable_logement` 5 to 1,000 m² raw, required and 9 to 500 m² clean; `annee_construction` 1500 to 2026; heat loss and ubat ≥ 0 — absurd or negative values
- `strict=True`, `coerce=True` — extra columns, wrong types

Validation on raw data: **3 violations**, all on one check: surface outside 5 to 1,000 m² (3 m², 2,464 m², 3,450 m²), see `reports/schema_failures.csv`. Validation on clean data: **passes**.

## 3. Missingness
| Column | % missing | Mechanism | Evidence | Decision |
|---|---|---|---|---|
| `annee_construction` | 50.2 % | MAR | 46-point gap between `methode_application_dpe` groups (4.3) | column dropped, `periode_construction` replaces it |
| `type_installation_chauffage` | 0.5 % (53) | MAR | same test, missing in one DPE type (4.3) | kept, impute in the model pipeline (train stats) |
| `deperditions_enveloppe` | 0.2 % (23) | MAR | 13-point gap between DPE types (4.3) | kept, impute in the model pipeline (train stats) |
| `numero_dpe_remplace` | 88.6 % | structural | empty when the DPE replaces nothing | kept as is |

MAR test: gap of empty-value share between `methode_application_dpe` groups of at least 50 rows, above 10 points = MAR.

## 4. Duplicates & outliers — decision table
Condensed from `reports/decision_table.csv` (notebook 4.4).

| Column | Issue | n rows | Decision | Why |
|---|---|---|---|---|
| (all) | 223 columns, many almost empty | 10,000 | keep 20 columns | only useful ones |
| `type_batiment` | whole-building DPE | 167 | removed | not a dwelling, surface empty |
| `methode_application_dpe` | apartment DPE generated from the building DPE | 3,219 | kept + `dpe_immeuble_genere` | neighbours look alike: split train/test by building |
| (all), `numero_dpe_remplace` | identical rows, old DPE next to its replacement | 0 | nothing to do | checked, none found |
| `identifiant_ban` | near-identical dwellings in one building | 429 | kept | often really different dwellings |
| `surface_habitable_logement` | empty or outside 9 to 500 m² | 79 | removed | impossible for a dwelling |
| `etiquette_dpe` | label differs from the recomputed one | 1,398 | kept | 30 % mismatch under 40 m² vs 2 % above: small-surface rule |
| `annee_construction` | 50 % missing, MAR | 4,899 | column dropped | more than 30 % empty |
| `type_installation_chauffage`, `deperditions_enveloppe` | 0.5 % and 0.2 % missing, MAR | 76 | kept, impute later | tiny, train stats |
| `zone_climatique` | 99.7 % identical (H1a) | 48 | column dropped | almost constant |
| `conso_5_usages_par_m2_ep` | extreme values (IQR, log scale) | 158 | kept + `conso_outlier` | 80 very high (all G, median 17 m²), 78 very low (A to C): real values |

## 5. Key findings (max 3)
1. **18 % of dwellings are F or G, A and B only 1.3 %** — Fig. 5.1 — so what: very unbalanced, group A-B and F-G and use macro F1.
2. **Building age is the clearest variable**: 26 % F or G before 1948, 12 % for 1948-1974, about 1 % from 1978; fuel oil 65 % (89 dwellings), electricity 23 %, gas 17 %, district heating 6 % — Fig. 5.2 — so what: period and heating are the main descriptors.
3. **For the same period, small dwellings are more often F or G**: before 1948, 32 % under 30 m² vs 15 % above 60 m² — Fig. 5.3 — so what: keep surface; part of it is the small-surface label rule.

## 6. What cannot be trusted ⚠️
- **Target leakage.** `conso_5_usages_par_m2_ep`, `emission_ges_5_usages_par_m2`, `etiquette_ges` (and `conso_outlier`) compute the label (worse of the energy and GHG classes): never use them as features. Under 40 m², 30 % of labels do not match the calculation.
- **DPE generated from the building DPE** (33 % of rows). Neighbours look alike and they are rarely F or G (6 % vs 24 %). A random split gives scores that are too good: split by `identifiant_ban`. The gap may come from the building type, not the calculation.
- **Small groups.** Fuel oil (89), LPG (15), wood (2), coal (1), period "après 2021" (24), class A (12): too few rows.
- **Intermediate variables** (heat loss, ubat, insulation) come from the same calculation as the consumption. In notebook 02 they lift the random forest from 43 % to 64.5 % accuracy, but this is not a prediction made before the diagnosis.
- **Unrepresentative sample.** 1.2 % of Paris, one date, and a DPE only exists for a sale, a rental or a new building.
- **Skewed values**: heat loss up to 92,000, consumption up to 1,334 kWh/m²/year: log scale or trees.

## 7. Hypotheses for modelling (≥ 5)
- **H1** — `periode_construction`: before 1948 gives more F or G than from 1978. Population: all 9,754 dwellings. Test: chi-square.
- **H2** — `surface_habitable_logement`: for the same period, the smaller the dwelling, the more F or G. Population: all dwellings. Test: logistic regression (period, surface).
- **H3** — `type_energie_principale_chauffage`: fuel oil and electricity give more F or G than gas and district heating. Population: groups of at least 50 dwellings. Test: chi-square, Kruskal-Wallis on consumption.
- **H4** — `qualite_isolation_enveloppe`: "insuffisante" gives more F or G than "bonne" (22 % vs under 4 %). Population: all dwellings. Test: chi-square (partly by construction, see section 6).
- **H5** — `dpe_immeuble_genere`: generated DPE are less often F or G, and the gap comes from the building type. Population: all dwellings at equal period and heating. Test: stratified comparison.

## 8. How to reproduce
```bash
python3 -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt
jupyter nbconvert --to notebook --execute --inplace notebooks/01_collecte_qualite.ipynb
```
Expected fingerprint: `78abd427b5da` (needs the API cache).
