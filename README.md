# Projet DPE Paris

Projet du cours **MSA-DATI07-03 · Advanced EDA & Data Collection**.
Groupe : Elie Maarek, Joshai Bensoussan, Lucas Noury.

On collecte des diagnostics de performance énergétique (DPE) de logements parisiens via l'API de l'ADEME, on documente leur qualité, puis on les explore pour comprendre ce qui fait qu'un logement est classé A ou G.

## Question

Comment les étiquettes DPE (A à G) se répartissent à Paris, et comment varient-elles selon la période de construction, la surface et le chauffage ?

## Données

- Source : ADEME, jeu `dpe03existant` ([data.ademe.fr](https://data.ademe.fr/datasets/dpe03existant)), Licence Ouverte 2.0.
- Collecte du 5 novembre 2026 par l'API, 10 000 DPE tirés au hasard (`sort=_rand`) parmi les 862 377 de Paris (`code_departement_ban:"75"`).
- Le détail (date, filtres, colonnes) est dans `data/raw/README.md`.
- Après nettoyage : 9 754 logements et 20 colonnes.

## Contenu du dépôt

```
notebooks/01_collecte_qualite.ipynb   notebook principal (sections 1 à 6)
data/raw/raw.parquet                  données brutes collectées (et leur README)
data/processed/                       données nettoyées (générées par le notebook)
reports/rapport_qualite.md            rapport qualité (2 pages)
reports/decision_table.csv            tableau des décisions de nettoyage
reports/schema_failures.csv           erreurs du schéma sur les données brutes
reports/5_1.png, 5_2.png, 5_3.png     les trois figures
docs/brief_projet.pdf                 énoncé du projet
requirements.txt                      packages et versions
```

## Lancer le projet

```
python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
jupyter nbconvert --to notebook --execute --inplace notebooks/01_collecte_qualite.ipynb
```

Le notebook affiche à la fin le nombre de lignes et une empreinte (fingerprint) des données propres. Deux exécutions donnent la même empreinte (`78abd427b5da` avec nos données), grâce au cache de l'API. Sans le cache, l'API renvoie d'autres lignes et l'empreinte change : c'est normal.

## Résultats principaux

- 18 % des logements sont classés F ou G, les classes A et B font à peine 1 %.
- L'âge du bâtiment pèse le plus : 26 % de F ou G avant 1948, environ 1 % à partir de 1978.
- À période égale, les petits logements sont plus souvent F ou G.

## Limites

- L'étiquette est calculée à partir de la consommation et des émissions de GES : ces colonnes ne doivent pas servir à la prédire.
- Un tiers des DPE sont générés à partir du DPE de l'immeuble : il faut séparer train et test par immeuble.
- L'échantillon (1,2 % de Paris, DPE faits à la vente ou à la location) n'est pas représentatif de tout le parc.

Le détail est dans `reports/rapport_qualite.md`.
