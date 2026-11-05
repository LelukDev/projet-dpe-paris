# Data quality report: DPE of existing dwellings in Paris

**Group:** Elie Maarek, Joshai Bensoussan, Lucas Noury · MSA-DATI07-03 · Data collected on 5 November 2026

## 1. Question and data

How are DPE labels (A to G) distributed in Paris, and how do they vary with construction period, surface and heating? The data comes from the ADEME API (dataset `dpe03existant`, Licence Ouverte 2.0). We collected 10,000 DPE drawn at random (`sort=_rand`) out of 862,377 for Paris (`code_departement_ban:"75"`), about 1.2 %. The API returns 223 columns and we keep 20. One row is one DPE. After cleaning, **9,754 dwellings** remain.

## 2. Schema contract (pandera)

The schema is strict: the 20 columns have a type and about a dozen business checks (labels between A and G, unique `numero_dpe`, known building type, date after July 2021, surface between 5 and 1,000 m², positive consumption and GHG, construction year between 1500 and 2026, positive heat loss and ubat). `nullable=True` is used only for columns that can really be empty. On the raw data (`lazy=True`) only one check fails: **3 surfaces outside 5 to 1,000 m²** (3 m², 2,464 m² and 3,450 m²), summarised in `reports/schema_failures.csv`. After cleaning, the surface is required and between 9 and 500 m², and the final contract passes.

## 3. Cleaning decisions

The full table is in `reports/decision_table.csv`. Main decisions:

| Column | Problem | Rows | Decision | Why |
|---|---|---|---|---|
| (all) | 223 columns, many almost empty | 10,000 | keep 20 columns | target, descriptors and intermediate variables useful for the question |
| `type_batiment` | whole-building DPE | 167 | removed | not a dwelling, dwelling surface is empty |
| `methode_application_dpe` | apartment DPE generated from the building DPE | 3,219 | kept + `dpe_immeuble_genere` | neighbours look alike: split train/test by building |
| `numero_dpe_remplace` | old DPE next to its replacement | 0 | nothing to do | we keep the most recent, no old DPE remained |
| `identifiant_ban` | near-identical dwellings in one building | 429 | kept | often really different dwellings |
| `surface_habitable_logement` | empty or outside 9 to 500 m² | 79 | removed | impossible for a dwelling |
| `etiquette_dpe` | label differs from the one recomputed with consumption and GHG | 1,398 | kept | 30 % mismatch under 40 m² against 2 % above: probably the small-surface rule |
| `annee_construction` | 50 % empty, MAR | 4,899 | column dropped | above 30 % empty, `periode_construction` replaces it |
| `type_installation_chauffage` | 0.5 % empty, MAR | 53 | kept, impute later | imputed in the model pipeline with train statistics |
| `zone_climatique` | 99.7 % of rows identical (H1a) | 48 | column dropped | almost constant, learns nothing |
| `conso_5_usages_par_m2_ep` | extreme values (IQR on log scale) | 158 | kept + `conso_outlier` | 80 very high (all G), 78 very low (almost all A to C): consistent with the label |

For missing values we tested whether the missingness depends on the DPE type (`methode_application_dpe`) by comparing the share of empty values between types. The gap is 46 points for `annee_construction`: this is MAR, not random.

## 4. What the data says

Figures are in `reports/5_1.png`, `5_2.png` and `5_3.png`.
1. **18 % of dwellings are F or G**, classes A and B are 1.3 % (5.1).
2. **Building age is the clearest variable**: 26 % F or G before 1948, 12 % between 1948 and 1974, about 1 % from 1978. Fuel oil (65 %, 89 dwellings) and electricity (23 %) are the most affected heating types (5.2).
3. **For the same period, small dwellings are more often F or G**: before 1948, 32 % under 30 m² against 15 % above 60 m² (5.3).

## 5. What cannot be trusted

- **Target leakage.** `conso_5_usages_par_m2_ep`, `emission_ges_5_usages_par_m2` and `etiquette_ges` compute the label (the worse of the energy and GHG classes). A model using them is useless. Under 40 m² the label does not always match the calculation, so it cannot be recomputed by hand.
- **DPE generated from the building** (a third of dwellings). Neighbours look alike and these are rarely F or G (6 % against 24 %). A random train/test split would give scores that are too good: we split by `identifiant_ban`. The gap may also come from the type of building, not from the calculation.
- **Small groups.** Fuel oil (89 dwellings), LPG (15), wood (2), coal (1), period "après 2021" (24) and class A (12 DPE out of 10,000): too few rows to conclude.
- **Unrepresentative sample.** 1.2 % of Paris dwellings, at one date, and a DPE only exists for a sale, a rental or a new building. We cannot generalise to all dwellings.
- **Intermediate variables.** Heat loss, ubat and insulation quality come from the same building calculation as the consumption. In notebook 02 they raise the random forest accuracy from 43 % to 64.5 %, but this is not a prediction made before the diagnosis.
- **Very skewed values.** `deperditions_enveloppe` goes above 92,000 and consumption up to 1,334 kWh/m²/year: use a log scale or tree models.
- **Remaining empty values** (`type_installation_chauffage`, `deperditions_enveloppe`) are MAR: filled in the model pipeline with train statistics only.

## 6. Hypotheses to test

- **H1:** before 1948 a dwelling is more often F or G than from 1978 (chi-square).
- **H2:** for the same period, the smaller the surface, the higher the share of F or G (logistic regression).
- **H3:** fuel oil and electricity give more F or G than gas and district heating (chi-square, Kruskal-Wallis).
- **H4:** "insufficient" insulation increases the share of F or G, 22 % against less than 4 % for good insulation (chi-square). Insulation is computed from the building, so part of the link is by construction.
- **H5:** generated DPE are less often F or G, and the gap comes from the building type (comparison at equal period and heating).
