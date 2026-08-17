# Analyse & Prédiction des Prix Immobiliers en France

Projet data end-to-end : de la collecte de données ouvertes jusqu'au déploiement d'un modèle prédictif, en passant par un pipeline ETL, un entrepôt de données, un dashboard interactif et du MLOps.

## Contexte & angle narratif

Ce projet analyse le marché immobilier français (transactions DVF 2021-2025) à travers le prisme de la remontée des taux d'intérêt en 2022-2023, et son impact sur les volumes et les prix des transactions.

## Sources de données

| Source | Contenu | Lien |
|---|---|---|
| DVF géolocalisé (DGFiP/Etalab) | Transactions immobilières réelles, 2021-2025 (20 382 915 lignes brutes, confirmé) | [data.gouv.fr](https://www.data.gouv.fr/api/1/datasets/r/d7933994-2c66-4131-a4da-cf7cd18040a4) |
| INSEE Filosofi | Revenu médian par commune | à compléter |
| DPE ADEME | Performance énergétique des logements | à compléter |
| API BAN | Géocodage / normalisation des adresses | à compléter |

## Stack technique

- **ETL** : Talend Open Studio
- **Stockage** : PostgreSQL
- **Dashboard** : Power BI
- **ML** : Python (scikit-learn, XGBoost)
- **MLOps** : MLflow
- **Automatisation** : n8n

## Structure du repo

```
projet-immo-france/
├── README.md
├── PROGRESS.md            # journal de reprise entre sessions de travail
├── data/
│   ├── raw/                # fichiers bruts téléchargés
│   └── processed/          # sorties nettoyées + cache intermédiaire (gitignoré)
├── etl/
│   └── talend_jobs/        # exports des jobs Talend
├── notebooks/
│   ├── 01_exploration_dvf.ipynb       # exploration initiale (aperçu, distributions)
│   └── 02_regles_nettoyage_dvf.ipynb  # vérifications approfondies + définition des règles de nettoyage
├── sql/
│   └── schema.sql          # création des tables PostgreSQL
├── ml/
│   ├── src/
│   └── mlruns/              # tracking MLflow (gitignoré)
├── powerbi/
│   └── dashboard.pbix
├── n8n/
│   └── workflows/
└── docs/
    ├── regles_nettoyage.md  # spécification des règles de nettoyage DVF (base des jobs Talend)
    └── architecture.png
```

## Avancement du projet

- [x] Phase 0 — Setup (Talend, PostgreSQL, Power BI, Python, Git)
- [x] Phase 1 — Collecte des données (DVF géolocalisé téléchargé, 20 382 915 lignes confirmées)
- [x] Phase 2 — EDA (exploration + qualification exhaustive des données — `notebooks/01_exploration_dvf.ipynb`, `notebooks/02_regles_nettoyage_dvf.ipynb`)
- [ ] Phase 3 — ETL (Talend) — règles de nettoyage définitivement spécifiées (`docs/regles_nettoyage.md`), implémentation des jobs Talend pas encore commencée
- [ ] Phase 4 — Dashboard Power BI
- [ ] Phase 5 — Machine Learning
- [ ] Phase 6 — MLOps (MLflow)
- [ ] Phase 7 — Automatisation (n8n)
- [ ] Phase 8 — Documentation finale

*(Note : `PROGRESS.md` utilise sa propre numérotation de phase, orientée journal de travail — sa "Phase 1"
regroupe la Collecte, l'EDA et la définition des règles de nettoyage ci-dessus, sans se caler sur les
phases du README. Se référer au README pour l'avancement global du projet, à `PROGRESS.md` pour le détail
du travail en cours.)*

## Setup local

### Prérequis
- PostgreSQL 16+
- Python 3.10+
- Talend Open Studio for Data Integration
- Power BI Desktop (Windows)

### Base de données
```sql
CREATE DATABASE immo_france;
```
