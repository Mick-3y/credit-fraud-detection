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
   **validation temporelle**, jamais aléatoire), jointure identity, baseline XGBoost,
   seuil de blocage optimal en coût
3. `03_features.ipynb` — feature engineering guidé par la mesure : 3e décimale des
   montants (signature de conversion de devise), heure du jour (cycle nocturne) ;
   jour de semaine mesuré et écarté (taux plat, un résultat négatif documenté)
4. `04_pruning.ipynb` — dégraissage des 339 colonnes V testé et **écarté** : les blocs
   à taux de NaN identiques ne sont pas redondants (4 % des paires à |corr| > 0,9),
   et couper les 181 colonnes corrélées dégrade le coût de 5 705 € — des colonnes
   corrélées à 0,9 ne sont pas des copies, le modèle exploitait leurs différences
5. `05_shap.ipynb` — explicabilité SHAP : vue globale (beeswarm) et explications
   locales (waterfalls) — dont le pire faux positif, décomposé feature par feature

## Résultat courant
Baseline XGBoost (réglages par défaut, split temporel 80/20) : **PR-AUC = 0,497**
(un modèle aléatoire : 0,035).

Concrètement : réglé pour attraper la moitié des fraudes, le modèle a raison une
alerte sur deux ; réglé pour en attraper 70 %, une alerte sur quatre reste juste.

Avec des coûts métier (fraude ratée ≈ 100 €, fausse alerte ≈ 5 €), le seuil
optimal sort à 0,026 : le modèle coûte **168 k€** sur les 5 semaines de
validation, contre 406 k€ sans modèle — **~238 k€ économisés**. Chiffre vérifié
par deux routes indépendantes (formules precision/recall vs recomptage direct).

Deux features construites depuis les colonnes brutes : `has_3dec` (10,5 % des
transactions portent une 3e décimale — hypothèse : montants convertis d'une devise
étrangère — et concentrent **35 % des fraudes**, taux 11,7 % contre 2,5 %) et
`hour` (la nuit, le taux de fraude triple : les clients légitimes désertent, la
fraude se voit). À configuration identique, la PR-AUC passe à **0,512** et le coût
optimal descend à **166 230 €** (−1 830 € sur la fenêtre).

Le modèle est auditable : SHAP décompose chaque score en contributions par colonne
(additives exactement, en log-odds). Globalement, l'importance est très répartie ;
le modèle a appris seul que cartes récentes (D1 faible) et navigateurs obsolètes
(`id_31`) sont des profils à risque. `hour` se classe 33e/434 ; `has_3dec` 302e —
redondante *en marginal* (son info est déjà portée en pièces détachées par
`TransactionAmt` et ses voisines) mais utile *en ablation* : signal univarié fort,
contribution marginale et delta de retrain sont trois mesures différentes.

## Limites
- Les colonnes Vesta (V, C, D...) sont anonymisées : SHAP dit **quelle** colonne
  décide (V258, C1 dominent les deux waterfalls), pas ce qu'elle signifie métier.
- Un seul split temporel : les chiffres (PR-AUC, coûts) n'ont pas de barre d'erreur.
- Coûts métier fixés à dire d'expert (100 € / 5 €) — le seuil optimal en dépend.
- Le signal "présence d'identity" (2,1 % de fraude sans, 7,8 % avec) suppose que ces
  données sont collectées au moment de la transaction, pas après coup lors d'une
  enquête fraude — hypothèse plausible (pratique connue du vrai dataset Vesta) mais
  jamais vérifiée dans ces notebooks. Si elle était fausse, ce serait une fuite.