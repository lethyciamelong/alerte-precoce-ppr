


#  PPR Early Warning — Système d'alerte précoce pour la Peste des Petits Ruminants (PPR) au Cameroun

Prédiction du niveau de risque de foyers de PPR par arrondissement, à partir de données climatiques, spatiales, temporelles et épidémiologiques, avec une anticipation d'au moins 4 semaines.

## Contexte

La Peste des Petits Ruminants (PPR) est une maladie animale dévastatrice pour l'élevage des petits ruminants. Une détection précoce des zones à risque permet une intervention rapide et ciblée, limitant les pertes économiques et la propagation de la maladie.

Ce projet a été réalisé dans le cadre du cours **INF4248** (Université de Yaoundé I). L'objectif : construire le meilleur modèle capable de classifier chaque observation (arrondissement × semaine épidémiologique) en trois niveaux de risque — **Confirmé**, **Suspicion**, **Sans suspicion** — en amont de la semaine épidémiologique critique.

## Données

Le jeu de données couvre 5 années d'observations (2020–2025) croisant :
- **Variables spatiales** : Région, Arrondissement, Zone Agro-Écologique (ZAE)
- **Variables temporelles** : Année, Mois, Semaine épidémiologique
- **Variables climatiques** (NASA POWER) : précipitations, humidité relative, température (moy./min/max), vitesse et direction du vent
- **Variables épidémiologiques** : animaux malades, cas susceptibles, animaux morts
- **Cible** : `Diagnosis_basis` (statut du foyer)

Le dictionnaire de données complet est disponible dans [`Dictionnaire_PPR.docx`](./Dictionnaire_PPR.docx).

**Défi principal : un déséquilibre de classes extrême** — `sans suspicion` (~96%), `Suspicion` (~3,5%), `Confirmé` (~0,5%, soit une vingtaine de cas sur l'ensemble des données).

## Méthodologie

| Étape | Décisions clés |
|---|---|
| **1. Exploration (EDA)** | Détection du déséquilibre extrême, de valeurs de précipitation négatives (artefact satellite), d'une colonne redondante, et d'un signal géographique/temporel fort. Identification d'un risque potentiel de fuite de données sur les variables épidémiologiques. |
| **2. Prétraitement** | Nettoyage (doublons, redondances, valeurs aberrantes), feature engineering temporel (encodage cyclique semaine/mois), encodage ordinal des variables géographiques, ratios épidémiologiques dérivés. **Split chronologique strict** : entraînement 2020–2024, test 2025, pour respecter la contrainte d'anticipation. |
| **3. Modélisation** | Validation croisée `StratifiedGroupKFold` (groupe = arrondissement) pour éviter toute fuite spatiale, gestion du déséquilibre à l'intérieur de chaque pli (pondération de classe / sur-échantillonnage). **8 modèles comparés** (Régression Logistique, KNN, SVM, MLP, Random Forest, Gradient Boosting, XGBoost, LightGBM), classés par F1-macro. Les deux meilleurs sont combinés en un **ensemble par vote souple**. |
| **4. Évaluation** | Test final sur l'année 2025 (indépendante), matrices de confusion, importance des variables, **test de sensibilité au data leakage**, analyse des foyers manqués. |

## Résultats clés

- **Modèle final recommandé : Gradient Boosting**, avec un **F1-macro de 0,73** sur le jeu de test 2025.
- L'ensemble (XGBoost + Gradient Boosting) obtient le meilleur score moyen en validation croisée ; en test, Gradient Boosting seul le devance légèrement — écart attendu sur une seule année de test.
- **Rappel de la classe `Confirmé` : ~29%** (le point le plus fragile du système, cohérent avec seulement 13 exemples en entraînement).
- **Rappel de la classe `Suspicion` : ~88%**, mieux représentée.
- **Test de sensibilité** : retirer les variables épidémiologiques (`Sick_animals`, `Susceptible_cases`, `Dead_animals`) fait chuter le F1-macro de 0,73 à 0,34 — un point de vigilance majeur à valider avec l'équipe vétérinaire terrain avant tout déploiement (disponibilité réelle de ces comptages *avant* diagnostic).

## Limites

- Très faible volume pour la classe `Confirmé` (13 exemples en train, 7 en test) → métriques statistiquement fragiles sur cette classe.
- Incertitude non tranchée sur la disponibilité temporelle réelle des variables épidémiologiques.
- Une seule année de test ; une validation *rolling origin* sur plusieurs années serait plus robuste.
- Encodage ordinal des zones géographiques : ne capture pas explicitement la proximité spatiale réelle entre arrondissements.

## Structure du repository

```
├── PPR_Prediction.ipynb         # Notebook complet (EDA → prétraitement → modélisation → évaluation)
├── Data_PPR.xlsx                # Jeu de données brut
├── Dictionnaire_PPR.docx        # Dictionnaire des données
├── modeling_results.joblib      # Modèles sauvegardés 
├── preprocessed_data.joblib     
└── README.md
```

## Utilisation

```bash
git clone https://github.com/<votre-utilisateur>/ppr-early-warning.git
cd ppr-early-warning
pip install -r requirements.txt
jupyter notebook PPR_Prediction.ipynb
```

Le notebook est structuré pour être exécuté séquentiellement (`Kernel → Restart & Run All`). Toutes les graines aléatoires sont fixées (`random_state = 42`) pour garantir la reproductibilité.

## Pistes d'amélioration

- Valider avec l'équipe terrain la disponibilité temporelle réelle des variables épidémiologiques.
- Enrichir les données géographiques (voisinage, mouvements de transhumance).
- Collecter davantage de données pour renforcer la classe `Confirmé`.
- Explorer un seuillage de décision ajusté selon le coût réel des faux négatifs.


