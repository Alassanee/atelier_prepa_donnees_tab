# Atelier Préparation de Données Tabulaires — Smart Building

Préparation complète d'un jeu de données de capteurs de bâtiments intelligents, en vue de prédire la présence d'une **alerte** (`alerte` : non / oui). Le travail va de l'exploration du fichier brut jusqu'à l'export d'un dataset propre, entièrement numérique.

## Structure du projet

```
atelier_prepa_donnees_tab/
├── data/
│   └── smart_building_raw.csv        # données brutes (507 lignes, 14 colonnes)
├── notebooks/
│   └── atelier_prepa_donnees_tab.ipynb
├── exports/
│   └── smart_building_cleaned.csv    # dataset nettoyé (500 lignes, 35 colonnes)
└── README.md
```

## Description des données

| Colonne | Type | Description |
|---|---|---|
| `id_mesure` | identifiant | identifiant de la mesure |
| `date` | datetime | date et heure de la mesure |
| `batiment`, `type_batiment`, `zone` | catégorielle nominale | bâtiment (B1–B8), type (6 modalités) et zone (A–D) |
| `temperature`, `humidite`, `co2`, `occupation`, `consommation_kwh` | numérique | mesures des capteurs |
| `mode_climatisation` | catégorielle ordinale | eco, normal, boost |
| `etat_systeme` | catégorielle ordinale | normal, alerte, panne |
| `jour_semaine` | catégorielle | jour de la mesure |
| `alerte` | **cible** | non / oui |

## Étapes du travail

### Partie 1 — Exploration
Chargement, dimensions, typage des variables (numériques, catégorielles, dates, identifiants), statistiques descriptives.

**Problèmes détectés :**
- valeurs manquantes (environ 1 à 2,5 % par colonne) ;
- 7 doublons (507 lignes pour 500 `id_mesure` uniques) ;
- valeurs aberrantes ou impossibles : température jusqu'à 96 °C, humidité jusqu'à 160 %, CO₂ jusqu'à 6000 ppm, occupation jusqu'à 116 ;
- catégories mal écrites : `type_batiment` avec 16 modalités au lieu de 6 (`BUREAU`, `Bureu`, ` Bureau `, `ecole`…), `mode_climatisation` avec 7 au lieu de 3 (`normale`, `BOOST`…) ;
- `date` lue comme du texte.

### Nettoyage
- **Incohérences** : valeurs impossibles (humidité < 0 ou > 100, températures extrêmes, consommation négative) transformées en valeurs manquantes.
- **Catégories** : suppression des espaces, passage en minuscules, correction des fautes (`bureu` → `bureau`, `normale` → `normal`, `ecole` → `école`, `entrepot` → `entrepôt`).
- **Valeurs manquantes** : médiane pour les numériques (distributions asymétriques, présence d'outliers), mode pour les catégorielles.
- **Doublons** : suppression.
- **Valeurs aberrantes** : détection visuelle (boxplots) et par la règle de l'IQR.
- **Déséquilibre de la cible** : environ 71 % de « non » pour 29 % de « oui ».
- **Corrélations** : matrice de corrélation (seaborn) sur les variables numériques.

### Partie 2 — Séparation X / y
`y = alerte` ; `X = temperature, occupation, consommation_kwh, co2`.

### Partie 3 — Découpage train / test
80 % / 20 %, `random_state=42` pour la reproductibilité, `stratify=y` pour conserver les proportions de classes.

### Partie 4 — Encodage des variables catégorielles

| Variable | Encodeur | Justification |
|---|---|---|
| `alerte` (cible) | `LabelEncoder` | cible binaire, simple 0/1 |
| `mode_climatisation` | `OrdinalEncoder` (eco < normal < boost) | ordre naturel d'intensité, ordre imposé avec `categories` |
| `etat_systeme` | `OrdinalEncoder` (normal < alerte < panne) | gravité croissante |
| `jour_semaine` | `OneHotEncoder` | ordonnée mais cyclique (dimanche voisin de lundi), effet non linéaire |
| `type_batiment` | `OneHotEncoder` | nominale, aucun ordre |
| `zone` | `OneHotEncoder` | simples étiquettes sans ordre réel |
| `batiment` | `OneHotEncoder` | identifiants nominaux |

### Partie 5 — Mise à l'échelle
- **Pertinence** : `co2` (jusqu'à plusieurs centaines ou milliers de ppm) et `temperature` (quelques dizaines de °C) n'ont pas la même échelle ; sans mise à l'échelle, les variables de grande amplitude dominent les modèles sensibles aux distances ou aux gradients (régression logistique, KNN, SVM).
- **Technique retenue** : `StandardScaler` (moyenne 0, écart-type 1), adapté à des mesures de capteurs à distribution à peu près symétrique. `RobustScaler` serait préférable si des outliers persistaient.
- **Pourquoi ne pas calculer la moyenne et l'écart-type sur tout le dataset ?** Cela crée une **fuite de données** : le jeu de test influencerait la transformation, ce qui rend l'évaluation trop optimiste. On fait donc `fit` sur le train uniquement, puis `transform` sur le train et le test.

### Partie 6 — Pipeline de preprocessing (scikit-learn)
`ColumnTransformer` avec deux branches :
- **numériques** : `SimpleImputer(median)` puis `StandardScaler` ;
- **catégorielles** : `SimpleImputer(most_frequent)` puis `OneHotEncoder(handle_unknown='ignore')`.

`X` est redéfini avec 4 variables numériques et 5 catégorielles (`type_batiment`, `zone`, `mode_climatisation`, `etat_systeme`, `jour_semaine`). `batiment` est exclu, car redondant avec `type_batiment`. Le pipeline est ajusté sur le train seul.

### Partie 7 — Export
Le dataset final `exports/smart_building_cleaned.csv` est **entièrement numérique** (aucune colonne texte) :
- `alerte`, `mode_climatisation`, `etat_systeme` gardent leur nom avec des valeurs entières ;
- `batiment`, `type_batiment`, `zone`, `jour_semaine` sont remplacées par leurs colonnes One-Hot ;
- 500 lignes, 35 colonnes, 0 valeur manquante, 0 doublon.

## Bonnes pratiques appliquées
- **Aucune fuite de données** : toute transformation qui apprend des statistiques (médiane, mode, moyenne, écart-type, catégories) est ajustée sur le train seul.
- **Découpage stratifié** pour préserver la proportion de la classe minoritaire.
- **Encodage adapté à la nature de la variable** : ordinal quand il existe un ordre, One-Hot sinon.
- **Pipeline reproductible**, applicable tel quel à de nouvelles données.

## Erreurs rencontrées et corrigées
- `fillna(df.mode)` sans parenthèses remplaçait les NaN par la *méthode* `mode` et non par sa valeur, ce qui faisait échouer l'`OrdinalEncoder` (`Found unknown categories`). Correction : `fillna(df[c].mode()[0])`. Le même défaut existait pour `median` sur la date : `median()`.

## Exécution
```bash
pip install pandas numpy scikit-learn seaborn matplotlib jupyter
jupyter notebook notebooks/atelier_prepa_donnees_tab.ipynb
```
Exécuter les cellules dans l'ordre, depuis le chargement des données.
