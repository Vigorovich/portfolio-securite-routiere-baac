# Sécurité routière — accidents corporels en France (2021-2023)

Analyse SQL des accidents corporels de la circulation routière, à partir des bases annuelles ONISR (fichier BAAC) publiées sur data.gouv.fr.

## Question traitée

« Quand, où et dans quelles conditions les accidents graves surviennent-ils ? »

Un **accident grave** est défini ici comme un accident corporel comptant au moins un usager tué ou blessé hospitalisé (`grav` = 2 ou 3 dans la table `Usagers`).

**Angle technique** : jointures SQL entre 4 tables liées par clé étrangère (contrairement aux CSV déjà dénormalisés des autres projets du portfolio), nettoyage de données réelles, analyse temporelle et géographique.

## Le jeu de données

[Bases de données annuelles des accidents corporels de la circulation routière](https://www.data.gouv.fr/datasets/bases-de-donnees-annuelles-des-accidents-corporels-de-la-circulation-routiere-annees-de-2005-a-2024) — ONISR / Ministère de l'Intérieur, licence ouverte.

**Périmètre retenu : 2021 à 2023** (3 années, sur les 2005-2024 disponibles), 12 fichiers CSV (4 tables × 3 ans) :

| Table | Grain | Lignes (3 ans) | Contenu |
|---|---|---|---|
| `Caractéristiques` | 1 accident | 166 642 | Date, heure, commune, luminosité, météo, type de collision |
| `Lieux` | 1 accident (voir note) | 182 680 | Catégorie de route, régime de circulation, état de la chaussée, vitesse autorisée |
| `Véhicules` | 1 véhicule impliqué | 285 393 | Catégorie, manœuvre, point de choc |
| `Usagers` | 1 personne impliquée | 381 699 | Gravité, place, trajet, équipement de sécurité |

Clé commune : `Num_Acc`.

**Non retenu** : les années antérieures à 2021 (codifications moins stables, cf. avertissement ONISR sur la qualification "blessé hospitalisé" depuis 2018) et les données d'immatriculation des véhicules (fichier séparé, hors périmètre de la question).

## Pipeline

1. **Chargement** : les 12 CSV sont parsés et chargés dans une base **SQLite** (tables `caracteristiques`, `lieux`, `vehicules`, `usagers`), avec deux corrections de nettoyage :
   - `Caractéristiques` 2022 nomme sa clé `Accident_Id` au lieu de `Num_Acc` (même colonne, même rôle sur les 2 autres années et les autres tables) — harmonisée au chargement.
   - Coordonnées (`lat`, `long`) au format `48,86638600` (virgule décimale) → converties en `REAL`.
2. **Jointures** : `Num_Acc` relie les 4 tables. `Lieux` n'est pas strictement à la granularité 1 ligne = 1 accident : ~29 % des accidents ont 2 lignes ou plus (généralement un accident à une intersection de deux voies, une ligne par voie — ex. l'accident `202300000001` a une ligne "RUE DE RIVOLI" et une ligne "RUE SAINT FLORENTIN"). Une vue `lieux_dedup` (première ligne par `Num_Acc`) est utilisée pour les jointures 1-pour-1 avec `Caractéristiques` ; la table `Lieux` brute reste disponible pour une analyse par voie.
3. **Analyse** : requêtes SQL agrégées (heure, mois, catégorie de route, département, luminosité, météo, état de la chaussée), croisées avec un indicateur de gravité.

Le détail complet (chaque requête, son résultat, son interprétation) est dans le notebook : **[`securite_routiere_2021_2023.ipynb`](./securite_routiere_2021_2023.ipynb)**.

## Résultats clés

- **166 642 accidents** sur 2021-2023, dont **35,6 % classés graves**.
- **Nuit sans éclairage public** : 3ᵉ condition la plus rare en volume, mais le taux de gravité le plus élevé (53,7 % contre 34,4 % en plein jour).
- **Routes départementales** : volume élevé (62 406 accidents, 2ᵉ catégorie) *et* sévérité élevée (50,6 % de graves) — le signal le plus net du jeu de données.
- **Paris** : 14 903 accidents sur la période mais seulement 7,5 % graves (vitesse urbaine limitée), à l'inverse de départements plus ruraux comme le Pas-de-Calais (62, 66,7 % graves sur un volume bien plus faible).
- Conditions météo dégradées (brouillard, vent fort) associées à des taux de gravité plus élevés, mais sur des effectifs trop restreints pour être aussi robustes que les constats ci-dessus.

## Limites connues

- **Pas de dénominateur d'exposition** : le dataset compte des accidents, pas de trafic (véhicules·km parcourus). Un département avec plus d'accidents n'est pas forcément plus dangereux par trajet.
- **Qualification "blessé hospitalisé"** non comparable d'une année sur l'autre depuis 2018, non labellisée par l'autorité de la statistique publique depuis 2019 (avertissement ONISR) — à garder en tête pour affiner ce travail.
- **Corrélation, pas causalité** : les écarts observés (nuit, route départementale, météo) sont des associations statistiques dans les données déclarées par les forces de l'ordre, pas une analyse causale.
- **Effectifs faibles sur certaines conditions météo rares** (ex. vent fort : 426 accidents sur 3 ans) — pourcentages associés à interpréter avec prudence.

## Outils

Python (sqlite3, pandas, matplotlib), SQL, Jupyter Notebook.
