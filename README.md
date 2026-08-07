# Détection de fraude bancaire — IEEE-CIS

Détecter des transactions frauduleuses sur données réelles (Vesta), avec un
déséquilibre fort : 3,5 % de fraudes. Le modèle produit une probabilité de
fraude ; le seuil de blocage est un choix métier (coût d'une fraude ratée vs
coût d'une fausse alerte). 

## Données
- `train_transaction.csv` : 590 540 lignes × 394 colonnes, cible `isFraud` (3,5 % de fraudes)
- `train_identity.csv` : 40 colonnes supplémentaires (device, navigateur), disponibles
  pour 24 % des transactions — leur simple présence est un signal (fraude : 7,8 % avec, 2,1 % sans)

## Démarche
1. `01_eda.ipynb` — volumétrie, déséquilibre des classes, types, NaN massifs
   **conservés** (XGBoost les gère nativement ; l'absence d'une donnée peut être un signal)
2. `02_baseline.ipynb` — montants (log-normale), temps (6 mois ordonnés →
   **validation temporelle**, jamais aléatoire), jointure identity, baseline XGBoost

## Résultat courant
Baseline XGBoost (réglages par défaut, split temporel 80/20) : **PR-AUC = 0,497**
(un modèle aléatoire : 0,035).

Concrètement : réglé pour attraper la moitié des fraudes, le modèle a raison une
alerte sur deux ; réglé pour en attraper 70 %, une alerte sur quatre reste juste.

## Prochaines étapes
- Seuil optimal en coût (fraude ratée ≈ 100 €, fausse alerte ≈ 5 €)
- Feature engineering : décimales des montants, structure temporelle
- Dégraissage des blocs V redondants (taux de NaN identiques par bloc)
- Explicabilité SHAP