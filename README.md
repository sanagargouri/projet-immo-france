# Analyse & Prédiction des Prix Immobiliers en France

Projet data end-to-end : de la collecte de données ouvertes jusqu'au déploiement d'un modèle prédictif, en passant par un pipeline ETL, un entrepôt de données, un dashboard interactif et du MLOps.

## Contexte & angle narratif

Ce projet analyse le marché immobilier français (transactions DVF 2021-2025) à travers le prisme de la remontée des taux d'intérêt en 2022-2023, et son impact sur les volumes et les prix des transactions.

## Sources de données

| Source | Contenu | Lien |
|---|---|---|
| DVF géolocalisé (DGFiP/Etalab) | Transactions immobilières réelles, 2021-2025 | [data.gouv.fr](https://www.data.gouv.fr/api/1/datasets/r/d7933994-2c66-4131-a4da-cf7cd18040a4) |
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
├── data/
│   ├── raw/              # fichiers bruts téléchargés
│   └── processed/        # sorties nettoyées
├── etl/
│   └── talend_jobs/      # exports des jobs Talend
├── notebooks/            # EDA, exploration
├── sql/
│   └── schema.sql        # création des tables PostgreSQL
├── ml/
│   ├── src/
│   └── mlruns/           # tracking MLflow (gitignoré)
├── powerbi/
│   └── dashboard.pbix
├── n8n/
│   └── workflows/
└── docs/
    └── architecture.png
```

## Avancement du projet

- [x] Phase 0 — Setup (Talend, PostgreSQL, Power BI, Python, Git)
- [ ] Phase 1 — Collecte des données
- [ ] Phase 2 — ETL (Talend)
- [ ] Phase 3 — EDA
- [ ] Phase 4 — Dashboard Power BI
- [ ] Phase 5 — Machine Learning
- [ ] Phase 6 — MLOps (MLflow)
- [ ] Phase 7 — Automatisation (n8n)
- [ ] Phase 8 — Documentation finale

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

## Auteur

À compléter.
