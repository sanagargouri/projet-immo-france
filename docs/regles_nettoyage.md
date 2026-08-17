# Règles de nettoyage DVF — Document de synthèse (Phase 2, Talend)

**Statut : FINALISÉ.** Toutes les règles sont tranchées, le pipeline complet a été exécuté de bout en bout,
et le volume final du périmètre résidentiel nettoyé est confirmé : **4 748 082 transactions**
(2 595 784 Maison, 2 152 298 Appartement). Ce document est la spécification de référence pour
l'implémentation des jobs Talend (Phase 2).

Source : `notebooks/01_exploration_dvf.ipynb` (exploration initiale) et
`notebooks/02_regles_nettoyage_dvf.ipynb` (vérifications et décisions détaillées ci-dessous).

Dataset source : `data/raw/dvf.csv` — DVF géolocalisé, 20 382 915 lignes brutes.

---

## Ordre d'application des règles

Les règles doivent s'appliquer dans cet ordre (chaque étape suppose les précédentes déjà faites) :

1. Dédoublonnage strict
2. Filtre nature de mutation (Vente + VEFA)
3. Filtre périmètre résidentiel (type_local)
4. Suppression des dépendances rattachées
5. Règle vente en bloc (agrégation / exclusion)
6. Exclusion des lignes sans coordonnées valides / hors bornes
7. Exclusion des anomalies surface / pièces
8. Seuils outliers `valeur_fonciere` *(à recalculer en dernier, une fois 1-7 appliqués)*

---

## 1. Dédoublonnage strict

**Constat** : 2 334 425 lignes (11.45% du fichier brut) sont des doublons stricts — lignes identiques sur
toutes les colonnes, confirmé par 2 méthodes indépendantes (hashing `Counter` + `numpy.unique`).

**Règle** : dédupliquer sur la ligne complète, garder la première occurrence.

```sql
-- Équivalent logique (Talend : tFilterRow ou tUniqRow sur le hash de toutes les colonnes)
SELECT * FROM (
    SELECT *, ROW_NUMBER() OVER (PARTITION BY <hash_ligne_complète> ORDER BY <ordre_fichier>) AS rn
    FROM dvf_brut
) WHERE rn = 1
```

**Volume après règle** : 18 859 504 lignes (92.53% du brut).

---

## 2. Filtre nature de mutation

**Constat** : `nature_mutation` compte 93.1% Vente, 5.1% VEFA (Vente en l'état futur d'achèvement), le
reste (Echange, Adjudication, Expropriation, Vente terrain à bâtir) hors périmètre.

**⚠️ Piège évité** : un filtre naïf `nature_mutation == 'Vente'` exclut silencieusement le VEFA (modalité
distincte : `'Vente en l'état futur d'achèvement'`), alors que la décision de principe est de le garder
comme variable d'analyse séparée, pas de l'exclure.

**Règle** :
```sql
WHERE nature_mutation IN ('Vente', 'Vente en l''état futur d''achèvement')
```
Ajouter une colonne dérivée :
```sql
est_vefa = (nature_mutation = 'Vente en l''état futur d''achèvement')
```

**Volume après règle** (cumulé avec §1) : 18 524 590 lignes (90.88% du brut).

---

## 3. Filtre périmètre résidentiel

**Constat** : `type_local` = NaN 40.2% (terrains), Dépendance 26.0%, Maison 16.5%, Appartement 14.1%,
Local commercial 3.2%. Le croisement avec `nature_culture` (Vérification 1) confirme que `nature_culture`
caractérise la parcelle cadastrale, pas le type de bien — non exploitable comme variable de recoupement,
mais sans impact sur la règle de filtrage ci-dessous.

**Règle** :
```sql
WHERE type_local IN ('Maison', 'Appartement')
```

**Volume après règle** (cumulé avec §1-2) : 5 917 612 lignes (29.03% du brut) — dont 5 803 868 Vente
classique (98.08%) et 113 744 VEFA (1.92%, `est_vefa = TRUE`).

---

## 4. Dépendances rattachées

**Constat** : 2 661 868 transactions mixtes Maison/Appartement + Dépendance sur le même `id_mutation`,
99.925% avec `valeur_fonciere` strictement identique entre les lignes (le prix total est porté sur une
seule ligne, pas réparti).

**Règle** :
```sql
-- Étape A : exclure les cas incohérents (valeur différente entre Dépendance et Maison/Appartement du même id_mutation)
DELETE FROM dvf WHERE id_mutation IN (
    SELECT id_mutation FROM dvf
    GROUP BY id_mutation
    HAVING COUNT(DISTINCT CASE WHEN type_local IN ('Maison','Dépendance') THEN valeur_fonciere END) > 1
)  -- 2 002 transactions (0.075%) exclues à ce titre

-- Étape B : supprimer la ligne Dépendance quand une ligne Maison/Appartement existe pour le même id_mutation
DELETE FROM dvf WHERE type_local = 'Dépendance' AND id_mutation IN (
    SELECT id_mutation FROM dvf WHERE type_local IN ('Maison', 'Appartement')
)
```

*(Cette règle est déjà satisfaite mécaniquement dès lors que le filtre §3 ne garde que Maison/Appartement —
les lignes Dépendance sont exclues par construction. La règle formelle ci-dessus reste nécessaire si un
export intermédiaire à la maille `id_mutation` est produit ailleurs dans le pipeline, pour éviter de
compter deux fois la valeur d'une transaction.)*

---

## 5. Vente en bloc

### 5.1 Cas à 2 lignes (553 885 transactions) — **TRANCHÉ**

**Constat** : sur les 695 857 candidats "vente en bloc" résiduels (multi-lignes, `valeur_fonciere`
identique, après dédoublonnage), 553 885 (79.6%) sont des transactions à exactement 2 lignes. Composition :
81.0% (Maison, Maison), 17.6% (Appartement, Appartement), 1.4% mixte. Surface cumulée médiane 192 m² (~2.5x
la médiane d'un bien individuel seul, 75 m²) — cohérent avec une propriété à deux structures (bâtiment
principal + annexe codée par erreur en Maison/Appartement plutôt qu'en Dépendance), pas avec un doublon ni
une vraie vente en bloc d'immeuble.

**Règle** :
```sql
-- Agrégation : sommer les surfaces, garder 1 ligne par id_mutation, valeur_fonciere inchangée (déjà le prix total)
SELECT id_mutation, MAX(valeur_fonciere) AS valeur_fonciere,
       SUM(surface_reelle_bati) AS surface_reelle_bati, ...
FROM dvf
WHERE id_mutation IN (<liste des 553 885 id_mutation à 2 lignes, valeur identique>)
GROUP BY id_mutation
```

### 5.2 Cas à 3 lignes ou plus (142 578 transactions) — **TRANCHÉ**

**Résultat de l'investigation composition/surface par tranche** :

| Tranche | Homogène type_local | Surface médiane | Surface/ligne implicite |
|---|---|---|---|
| 3-4 | 93.7% | 241 m² | ~69 m² |
| 5-9 | 88.4% | 328 m² | ~47 m² |
| 10-19 | 86.1% | 720 m² | ~50 m² |
| 20-49 | 81.9% | 1 648 m² | ~48 m² |
| 50-99 | 75.6% | 4 089 m² | ~54 m² |
| 100-499 | 47.5% | 10 335 m² | ~34 m² |
| 500+ | 9.1% | 86 529 m² | ~86 m² |

**Constat** : la surface cumulée ne fournit pas de rupture nette (elle croît proportionnellement au nombre
de lignes, avec une surface/ligne implicite qui reste plausible — 35 à 70 m² — sur toutes les tranches).
Le signal discriminant est l'homogénéité `type_local`, qui reste élevée (>85%) jusqu'à 10-19 lignes puis
décroît progressivement avant de chuter nettement au-delà de 100 lignes (47.5%, puis 9.1%). **Il n'y a pas
de seuil unique évident dans les données — la transition est progressive.**

**Règle retenue (arbitrage assumé, pas une certitude mathématique)** :
```sql
-- Agrégation (comme §5.1) si nb_lignes <= 9 après dédoublonnage
-- Exclusion comme vraie vente en bloc si nb_lignes > 9
```
Seuil choisi à 9 lignes : l'homogénéité y reste >85% et le volume concerné reste minoritaire dans la traîne
(136 845 sur 142 578, 96%). **Limite assumée** : contamination croisée résiduelle possible dans les deux
sens (petits immeubles agrégés à tort, grandes propriétés légitimes exclues à tort) — impact jugé
négligeable au vu du volume total (5,9M transactions).

---

## 6. Géographie

**Constat** : 19 954 941 lignes avec longitude/latitude renseignées. 0 coordonnée `(0,0)` suspecte.
194 273 lignes hors bornes métropole sont des DOM-TOM légitimes (`code_departement` 971-976, à garder).
Seulement 334 lignes (0.002%) sont réellement hors bornes et hors DOM-TOM.

**Règle** (déjà actée depuis le début du projet + affinée) :
```sql
WHERE longitude IS NOT NULL AND latitude IS NOT NULL
  AND (
        code_departement IN ('971','972','973','974','975','976')  -- DOM-TOM, pas de contrainte de bornes
        OR (longitude BETWEEN -5.5 AND 10.0 AND latitude BETWEEN 41.0 AND 51.5)  -- métropole
      )
```

---

## 7. Cohérence surface / pièces

**Constat** : `surface_reelle_bati` est quasi complète et cohérente sur le périmètre résidentiel (0.01%
manquant, aucune valeur à 0) — colonne fiable, aucun nettoyage nécessaire. `nombre_pieces_principales`
manquant à 0.01%, à 0 pour 0.19% des lignes (plausible pour certains ateliers/lofts).

**Anomalie repérée** : 394 lignes avec `nombre_pieces_principales > 20` (max observé : 198). Ratio
surface/pièces médian de ce sous-groupe : 3.38 m²/pièce — 3x sous le seuil légal français de pièce
habitable (9 m² minimum, critère de décence). Très majoritairement une erreur de saisie plutôt qu'un vrai
grand bien.

**Règle** :
```sql
WHERE NOT (
    nombre_pieces_principales > 20
    AND surface_reelle_bati / NULLIF(nombre_pieces_principales, 0) < 9
)
```

---

## 8. Cas "valeur différente" — **AUCUNE RÈGLE SPÉCIFIQUE, POINT DÉFINITIVEMENT CLOS**

**Constat (bug identifié et confirmé à 100%)** : les 1 469 transactions initialement classées
"multi-lignes, valeur différente" se sont révélées être **intégralement** un artefact de comparaison
`NaN == NaN` (qui vaut `False` en SQL/Python), pas un vrai phénomène de prix réparti par lot — vérification
directe : sur les 1 469, 1 469 ont `valeur_fonciere` manquant sur toutes leurs lignes, 0 cas réel.

**Règle** : déjà couvertes par la règle générique "lignes sans `valeur_fonciere` → exclure" (actée dès le
début du projet, section Valeurs manquantes). Aucune règle Talend additionnelle à écrire pour ce cas.

**Point méthodologique à retenir pour Talend** : toute comparaison d'égalité entre deux colonnes pouvant
contenir des valeurs manquantes doit être doublée d'un test de complétude explicite — une transformation
qui compare `NULL` à `NULL` ne doit jamais être interprétée comme "valeurs différentes".

---

## 9. Seuils outliers `valeur_fonciere` — **FINALISÉS**

**Résultat obtenu après application complète des règles §1-§7** (4 758 845 transactions, Maison 2 601 594 /
Appartement 2 157 251) :

| | Maison | Appartement |
|---|---|---|
| Médiane | 200 000€ | 170 500€ |
| 1er percentile | 15 000€ | 23 000€ |
| 99e percentile | 1 270 250€ | 1 380 000€ |
| 99.9e percentile | 4 300 000€ | 4 315 000€ |
| Min | 0.15€ | 0.15€ |
| Max | 722 590 000€ | 340 400 000€ |

**⚠️ Point important** : le maximum (722 590 000€, Maison) est identique à la valeur brute repérée avant
toute règle de nettoyage — c'est une transaction à **ligne unique** (`nb_lignes = 1`), donc la règle vente
en bloc (§5, basée sur le nombre de lignes) ne peut structurellement pas l'exclure. Seul un seuil direct
sur `valeur_fonciere` peut l'attraper — d'où la nécessité de cette dernière étape, indépendante de §5.

Le minimum (0.15€ dans les deux cas) confirme la nécessité du seuil bas : caractéristique de cessions
symboliques (donations déguisées, erreurs de saisie), pas de transactions de marché.

**Règle définitive** :
```sql
WHERE valeur_fonciere BETWEEN 1000
  AND (CASE WHEN type_local = 'Maison' THEN 4300000 ELSE 4315000 END)
```
Seuil haut basé sur le 99.9e percentile de chaque sous-population — exclut la queue extrême (erreurs de
saisie manifestes, ventes symboliques) tout en conservant les biens de luxe plausibles.

---

## Tableau entonnoir — volumétrie cumulée

| Étape | Lignes/Transactions | % du brut |
|---|---|---|
| 1. Total brut | 20 382 915 | 100.00% |
| 2. Après dédoublonnage strict (§1) | 18 859 504 | 92.53% |
| 3. Vente + VEFA (§2) | 18 524 590 | 90.88% |
| 4. Périmètre résidentiel Maison/Appartement (§3) | 5 917 612 | 29.03% |
| 5. Après géo + surface/pièces, lignes (§6-§7) | 5 804 584 | 28.48% |
| 6. Après agrégation/exclusion vente en bloc, transactions (§5) | 4 758 845 | 23.35% |
| 7. **Après seuil outlier valeur_fonciere, final (§9)** | **4 748 082** | **23.29%** |

Le filtre le plus sélectif est de loin le périmètre résidentiel (§3, -61.85 points) — terrains, locaux
commerciaux et dépendances représentent la majorité du fichier brut. Toutes les étapes suivantes sont des
raffinements marginaux en comparaison (moins de 6 points cumulés) : la règle vente en bloc ne retire
réellement que 5 874 transactions (le reste de la baisse à cette étape vient de l'agrégation des
transactions à 2-9 lignes, pas d'une exclusion), et le seuil outlier final retire 10 763 transactions
supplémentaires (0.23%).

**Volume final du périmètre résidentiel nettoyé : 4 748 082 transactions**
(2 595 784 Maison, 2 152 298 Appartement).

---

## Phase 1 — terminée

Le volume final du périmètre résidentiel nettoyé est confirmé : **4 748 082 transactions**
(2 595 784 Maison, 2 152 298 Appartement), soit 23.29% du fichier brut d'origine (20 382 915 lignes).

Prochaine étape (hors Phase 1) : traduire les 9 règles ci-dessus en jobs Talend, charger le résultat dans
PostgreSQL (`immo_france`).
