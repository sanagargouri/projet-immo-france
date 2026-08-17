# PROGRESS — Projet Analyse & Prédiction Prix Immobiliers France

Dernière mise à jour : **Phase 1 (exploration + nettoyage DVF) TERMINÉE.**

---

## Contexte général

- Environnement : Windows, terminal **cmd** (pas PowerShell — préférence confirmée)
- Repo GitHub : https://github.com/sanagargouri/projet-immo-france
- Projet local : `C:\Users\eree\Documents\projet-immo-france`
- Venv Python actif : `venv\Scripts\activate` (cmd), pandas 3.0.3, `pyarrow` requis (cache parquet)
- Éditeur : VS Code, notebooks exécutés directement dans VS Code (extension Jupyter), kernel = venv
- PostgreSQL installé, base `immo_france` créée mais **vide** — le chargement réel se fait en Phase 2

## Objectif final rappelé

Pipeline data end-to-end : DVF (+ INSEE Filosofi, DPE ADEME, API BAN) → Talend (ETL) →
PostgreSQL → Power BI (dashboard) → Python/ML (XGBoost) → MLflow (MLOps) → n8n (automatisation).
Angle storytelling : impact de la remontée des taux d'intérêt 2022-2023 sur le marché immobilier
(confirmé sur le périmètre nettoyé, cf. `01_exploration_dvf.ipynb` section 7).

Contexte perso : projet destiné à renforcer un CV/portfolio pour recherche de stage PFE
en France (février 2027).

---

## ✅ Phase 0 — Setup (terminée)

PostgreSQL, pgAdmin 4, Talend Open Studio, Power BI Desktop installés. Repo GitHub structuré
(`notebooks/`, `docs/`, `data/{raw,processed}/`, `etl/`, `sql/`, `ml/`, `powerbi/`, `n8n/`).

## ✅ Phase 1 — Exploration & règles de nettoyage DVF (TERMINÉE)

### Fichiers produits

- `notebooks/01_exploration_dvf.ipynb` — exploration initiale pure (aperçu, comptage, valeurs manquantes,
  répartitions nature_mutation/type_local, distribution valeur_fonciere, répartition temporelle brute).
  Le notebook a été **découpé en deux** en cours de route (il était devenu trop long, 96 cellules) — voir
  ci-dessous.
- `notebooks/02_regles_nettoyage_dvf.ipynb` — toutes les vérifications approfondies et décisions de règles
  (dédoublonnage, vente en bloc, VEFA, géographie, surface/pièces, entonnoir, pipeline complet). Notebook
  **autonome** (redéfinit ses propres imports/chemins, ne dépend d'aucune variable de `01`).
- `docs/regles_nettoyage.md` — **document de synthèse final**, spécification de référence pour la Phase 2
  (Talend). Contient les 9 règles dans l'ordre d'application, chacune en pseudo-SQL directement
  transposable.

### Données

- DVF géolocalisé : `data/raw/dvf.csv`, ~3,55 Go, **20 382 915 lignes brutes**

### Résultat final

**4 748 082 transactions** dans le périmètre résidentiel nettoyé (2 595 784 Maison, 2 152 298 Appartement),
soit 23.29% du fichier brut. Détail complet des 9 règles et du tableau entonnoir : voir
`docs/regles_nettoyage.md`.

### Points méthodologiques notables (utiles pour l'entretien)

- **Un bug NaN a été identifié et corrigé** : une comparaison `min_val == max_val` classait à tort les
  transactions à `valeur_fonciere` manquante comme "prix différent par ligne" (`NaN == NaN` vaut `False`).
  Vérifié exhaustivement : 100% des 1 469 cas concernés étaient cet artefact, 0 cas réel.
- **La règle "vente en bloc"** (transactions multi-lignes à prix identique) a nécessité plusieurs itérations :
  hypothèse initiale sur petit échantillon (5 cas) invalidée à l'échelle du dataset complet, dédoublonnage
  strict qui n'expliquait que 2% du phénomène, puis analyse de composition (`type_local` + surface) pour
  trancher un seuil (`nb_lignes <= 9` → agréger, au-delà → exclure) — assumé comme un arbitrage raisonné, la
  donnée ne montrant pas de rupture nette.
- **Un outlier extrême (722 590 000€, une Maison) a survécu à toutes les règles** jusqu'au bout — c'était une
  transaction à ligne unique, donc invisible pour la règle vente en bloc (basée sur le nombre de lignes).
  Seul le seuil outlier final (percentile 99.9) l'a exclu — bon rappel que différentes règles attrapent
  différents types d'anomalies, aucune n'est suffisante seule.
- Cache disque ajouté (`data/processed/keep_mask.npy`, `final_resume_dedup.parquet`) pour éviter de
  recalculer le dédoublonnage (passe complète sur 20M lignes) à chaque reprise de session.

---

## ⏳ Phase 2 — Implémentation Talend (à démarrer)

Prochaine étape du projet : traduire les 9 règles de `docs/regles_nettoyage.md` en jobs Talend, avec
PostgreSQL (`immo_france`) comme cible. Rien n'a encore été commencé sur cette phase.

---

## Format de travail convenu avec l'utilisateur

- Terminal : **cmd** (pas PowerShell)
- Notebook exécuté directement dans VS Code (pas de terminal Jupyter séparé)
- Après chaque résultat de vérification, TOUJOURS fournir : interprétation + conclusion + règle Talend
  correspondante, en bloc Markdown prêt à coller dans le notebook
- L'utilisateur commit/push régulièrement mais pas après chaque petite étape
- L'utilisateur souhaite comprendre chaque bout de code (pas de code non expliqué)
- L'utilisateur pousse activement à vérifier sur l'ensemble du dataset plutôt que sur des échantillons
- Objectif CV/stage en tête : garder une trace propre et justifiable de chaque décision méthodologique

---

## Comment reprendre dans une nouvelle conversation

Donner ce fichier + préciser : "Phase 1 terminée (exploration + nettoyage DVF), tout est dans
`docs/regles_nettoyage.md`. On attaque la Phase 2 : traduire ces règles en jobs Talend et charger le
résultat dans PostgreSQL (`immo_france`)."
