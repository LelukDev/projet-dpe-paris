# Rapport qualité des données : DPE des logements existants à Paris

**Groupe :** Elie Maarek, Joshai Bensoussan, Lucas Noury · MSA-DATI07-03 · Données collectées le 5 novembre 2026

## 1. Question et données

On cherche à comprendre comment les étiquettes DPE (A à G) se répartissent à Paris, selon la période de construction, la surface et le chauffage. Les données viennent de l'API de l'ADEME (jeu `dpe03existant`, Licence Ouverte 2.0). On a collecté 10 000 DPE tirés au hasard (`sort=_rand`) sur 862 377 pour Paris (`code_departement_ban:"75"`), soit environ 1,2 %. L'API renvoie 223 colonnes, on en garde 20. Une ligne correspond à un DPE. Après nettoyage, il reste **9 754 logements**.

## 2. Contrat de schéma (pandera)

Le schéma est strict : les 20 colonnes ont un type, avec une douzaine de contrôles métier (étiquettes entre A et G, `numero_dpe` unique, type de bâtiment connu, date après juillet 2021, surface entre 5 et 1000 m², consommation et GES positifs, année de construction entre 1500 et 2026, déperditions et ubat positifs). `nullable=True` n'est utilisé que pour les colonnes qui peuvent vraiment être vides. Sur les données brutes (validation `lazy=True`), un seul contrôle échoue : **3 surfaces hors de 5 à 1000 m²** (3 m², 2 464 m² et 3 450 m²). Le résumé est dans `reports/schema_failures.csv`. Après nettoyage, la surface est obligatoire et comprise entre 9 et 500 m², et le contrat final est validé.

## 3. Décisions de nettoyage

Le tableau complet est dans `reports/decision_table.csv`. Les principales décisions :

| Colonne | Problème | Lignes | Décision | Pourquoi |
|---|---|---|---|---|
| (toutes) | 223 colonnes, beaucoup presque vides | 10 000 | on garde 20 colonnes | cible, descripteurs et variables intermédiaires utiles à la question |
| `type_batiment` | DPE d'immeuble entier | 167 | supprimées | ce n'est pas un logement, la surface du logement est vide |
| `methode_application_dpe` | appartement dont le DPE est généré à partir de celui de l'immeuble | 3 219 | gardés + colonne `dpe_immeuble_genere` | des logements voisins se ressemblent, il faudra séparer train et test par immeuble |
| `numero_dpe_remplace` | ancien DPE présent à côté de son remplaçant | 0 | rien à faire | on garde le plus récent, aucun ancien DPE n'est resté |
| `identifiant_ban` | logements quasi identiques dans le même immeuble | 429 | gardés | souvent de vrais logements différents |
| `surface_habitable_logement` | surface vide ou hors de 9 à 500 m² | 79 | supprimées | valeurs impossibles pour un logement |
| `etiquette_dpe` | étiquette différente de celle recalculée avec conso et GES | 1 398 | gardées | écart de 30 % sous 40 m² contre 2 % au-dessus, probablement la règle des petites surfaces |
| `annee_construction` | 50 % de valeurs vides, MAR (dépend du type de DPE) | 4 899 | colonne supprimée | plus de 30 % vide, `periode_construction` la remplace |
| `type_installation_chauffage` | 0,5 % vide, MAR | 53 | gardée, à remplir plus tard | on remplira dans le pipeline du modèle, avec les stats du train |
| `zone_climatique` | 99,7 % des lignes identiques (H1a) | 48 | colonne supprimée | quasi constante, n'apprend rien |
| `conso_5_usages_par_m2_ep` | valeurs extrêmes (IQR sur échelle log) | 158 | gardées + colonne `conso_outlier` | 80 très hautes (G) et 78 très basses (A à C) : cohérent avec l'étiquette |

Pour les valeurs manquantes, on a testé si le manque dépend du type de DPE (`methode_application_dpe`), en comparant le taux de valeurs vides entre les types. L'écart est de 46 points pour `annee_construction` : c'est du MAR, pas du hasard.

## 4. Ce que disent les données

Les figures sont dans `reports/5_1.png`, `5_2.png` et `5_3.png`.
1. **18 % des logements sont F ou G**, les classes A et B font 1,3 % (5.1).
2. **L'âge du bâtiment est la variable la plus marquée** : 26 % de F ou G avant 1948, 12 % entre 1948 et 1974, environ 1 % à partir de 1978. Le fioul (65 %) et l'électricité (23 %) sont les chauffages les plus touchés (5.2).
3. **À période égale, les petits logements sont plus souvent F ou G** : avant 1948, 32 % sous 30 m² contre 15 % au-dessus de 60 m² (5.3).

## 5. Ce qui n'est pas fiable

- **Fuite de la cible.** `conso_5_usages_par_m2_ep`, `emission_ges_5_usages_par_m2` et `etiquette_ges` servent à calculer l'étiquette (c'est la pire des classes énergie et GES). Un modèle qui les utilise est inutile. Pour les logements de moins de 40 m², l'étiquette ne correspond pas toujours au calcul : on ne peut pas la recalculer à la main.
- **DPE générés à partir de l'immeuble** (un tiers des logements). Ils se ressemblent entre voisins et sont rarement F ou G (6 % contre 24 % pour les autres). Un découpage train/test aléatoire donnerait des scores trop beaux : il faut séparer par `identifiant_ban`. Cet écart peut aussi venir du type d'immeuble, pas du calcul.
- **Petits groupes.** Fioul (89 logements), GPL (15), bois (2), charbon (1), période « après 2021 » (24) et classe A (12 DPE sur 10 000) : trop peu de lignes pour conclure.
- **Échantillon non représentatif.** 1,2 % du parc parisien, photographié à une date, et un DPE n'existe que pour une vente, une location ou un bâtiment neuf. On ne peut pas généraliser à tous les logements de Paris.
- **Valeurs très étalées.** `deperditions_enveloppe` va jusqu'à plus de 92 000 et la consommation jusqu'à 1 334 kWh/m²/an : passer en log ou utiliser des modèles à arbres.
- **Valeurs vides à remplir.** Les valeurs vides restantes (`type_installation_chauffage`, `deperditions_enveloppe`) sont MAR : on les remplit dans le pipeline du modèle, avec les statistiques du train uniquement.

## 6. Hypothèses à tester

- **H1 :** avant 1948, un logement est plus souvent F ou G qu'à partir de 1978 (khi-deux).
- **H2 :** à période égale, plus la surface est petite, plus la part de F ou G augmente (régression logistique).
- **H3 :** le fioul et l'électricité donnent plus de F ou G que le gaz et le chauffage urbain (khi-deux, Kruskal-Wallis).
- **H4 :** une isolation « insuffisante » augmente la part de F ou G, 22 % contre moins de 4 % pour une bonne isolation (khi-deux).
- **H5 :** les DPE générés depuis l'immeuble sont moins souvent F ou G, et l'écart vient du type d'immeuble (comparaison à période et chauffage égaux).
