# Rapport — ALO pour le Flow Shop Scheduling Problem (PFSP)

## Ant Lion Optimizer (Mirjalili, 2015) adapté à l'optimisation de permutations

---

## 1. Introduction

### 1.1 Problème traité

Le **Permutation Flow Shop Scheduling Problem (PFSP)** est un problème d'optimisation combinatoire classique où :
- **n jobs** doivent être traités sur **m machines** dans le même ordre
- Chaque job visite les machines $1, 2, \dots, m$ séquentiellement
- Chaque machine ne peut traiter qu'un seul job à la fois
- Chaque job ne peut être traité que sur une seule machine à la fois
- **Objectif** : Trouver une permutation $\pi = [\pi_1, \pi_2, \dots, \pi_n]$ qui **minimise le makespan** $C_{\max}$ (temps d'achèvement du dernier job sur la dernière machine)

### 1.2 Algorithme utilisé : Ant Lion Optimizer (ALO)

L'ALO, proposé par **Mirjalili (2015)**, mime le comportement de chasse des fourmilions (antlions) :
1. **Marche aléatoire des fourmis** — exploration via somme cumulée de pas aléatoires
2. **Construction de pièges** — sélection par roulette wheel basée sur la fitness des antlions
3. **Pris au piège** — rétrécissement adaptatif des frontières autour de l'antlion sélectionné
4. **Capture de proies** — remplacement quand une fourmi surpasse un antlion
5. **Élitisme** — le meilleur antlion est toujours préservé

### 1.3 Adaptation continue → permutation

| ALO Original (Continue) | Notre ALO (PFSP) |
|---|---|
| Vecteur position $\in \mathbb{R}^n$ | Vecteur position $\in \mathbb{R}^n$ **(inchangé)** |
| $\downarrow$ | $\downarrow$ |
| Évaluer fonction objectif $f(x)$ | **Random Keys** → Permutation $\pi$ |
| | $\downarrow$ |
| | Calculer **Makespan** $C_{\max}(\pi)$ |

**Seule l'évaluation de fitness change.** Toute la structure algorithmique — marche aléatoire, roulette wheel, élite, remplacement, frontières adaptatives — reste **100% identique** à l'article de Mirjalili.

---

## 2. Définition du problème

### Instance utilisée : Taillard tai20_5

| Propriété | Valeur |
|---|---|
| **Nombre de jobs ($n$)** | 20 |
| **Nombre de machines ($m$)** | 5 |
| **Matrice $P$** | $20 \times 5$ |
| **Benchmark** | Taillard (1993) |

La matrice des temps de traitement $P_{j,k}$ représente le temps de traitement du job $j$ sur la machine $k$. Les valeurs sont des entiers allant de 2 à 99.

---

## 3. Architecture de l'implémentation

Le notebook est structuré en **29 cellules** (15 markdown + 14 code) organisées comme suit :

### 3.1 Pipeline de l'algorithme

```
                    INITIALISATION
                         │
                         ▼
              Évaluation des populations
                   (continue → permutation → makespan)
                         │
                         ▼
                    Trier Antlions
                         │
                         ▼
              Elite = meilleur Antlion
                         │
                         ▼
            ┌───── BOUCLE PRINCIPALE ─────┐
            │   Pour chaque fourmi i :     │
            │   1. Roulette Wheel          │
            │   2. Random Walk (antlion)   │
            │   3. Random Walk (elite)     │
            │   4. Moyenne des 2 walks     │
            │                              │
            │   Clip aux frontières        │
            │   Évaluer fitness            │
            │   Remplacer antlions         │
            │   Mettre à jour elite        │
            └──────────────────────────────┘
                         │
                         ▼
                     RÉSULTATS
              Meilleure permutation
              Meilleur makespan
              Courbe de convergence
              Diagramme de Gantt
```

### 3.2 Composants implémentés

| Composant | Description | Préservé de Mirjalili |
|---|---|---|
| **`calculate_makespan()`** | Calcule $C_{\max}$ pour une permutation donnée | ✅ Nouveau (PFSP) |
| **`continuous_to_permutation()`** | Random Keys : argsort d'un vecteur continu → permutation | ✅ Nouveau (PFSP) |
| **`initialize_population()`** | Initialisation uniforme dans $[lb, ub]$ | ✅ 100% |
| **`roulette_wheel()`** | Sélection proportionnelle inversée (minimisation) | ✅ 100% |
| **`random_walk()`** | Somme cumulée de $(2r > 1) - 1$, coefficient $I$ adaptatif, normalisation min-max | ✅ 100% |
| **`clip_population()`** | Contrainte des positions dans $[lb, ub]$ | ✅ 100% |
| **`evaluate_population()`** | Pipeline continu → permutation → makespan → fitness | ✅ Adapté |
| **`replace_antlions()`** | Remplacement si fourmi plus fitte que l'antlion | ✅ 100% |
| **`update_elite()`** | Mise à jour de l'élite | ✅ 100% |
| **`alo()`** | Boucle principale complète | ✅ 100% |

### 3.3 Coefficient adaptatif $I$ (exactement Mirjalili)

Le coefficient $I$ contrôle le rétrécissement des frontières :

$$I = 1 + 10 \cdot \frac{t}{\text{maxIter}}$$

Avec des paliers de rétrécissement accéléré :

| Condition | $I$ |
|---|---|
| $t > 0.1 \cdot \text{maxIter}$ | $1 + 10 \cdot (t/\text{maxIter})$ |
| $t > 0.5 \cdot \text{maxIter}$ | $1 + 10 \cdot (t/\text{maxIter})$ |
| $t > 0.75 \cdot \text{maxIter}$ | $1 + 100 \cdot (t/\text{maxIter})$ |
| $t > 0.9 \cdot \text{maxIter}$ | $1 + 1000 \cdot (t/\text{maxIter})$ |
| $t > 0.95 \cdot \text{maxIter}$ | $1 + 10000 \cdot (t/\text{maxIter})$ |

### 3.4 Marche aléatoire (exactement Mirjalili)

$$X(t) = \left[0, \text{cumsum}(2r(t_1)-1), \text{cumsum}(2r(t_2)-1), \dots\right]$$

où $r(t) = \begin{cases} 1 & \text{si rand} > 0.5 \\ 0 & \text{sinon} \end{cases}$

Normalisation min-max :

$$X_{\text{norm}} = \frac{(X - a)(d - c)}{b - a} + c$$

---

## 4. Résultats expérimentaux

### 4.1 Paramètres d'exécution

| Paramètre | Valeur |
|---|---|
| **Taille population** | 40 |
| **Nombre d'itérations** | 200 |
| **Domaine de recherche** | $[-4.0, 4.0]$ |
| **Instance** | Taillard tai20_5 (20×5) |
| **Seed aléatoire** | 42 |

### 4.2 Résultats numériques

| Métrique | Valeur |
|---|---|
| **Makespan initial** | 1412 |
| **Makespan final** | **1363** |
| **Amélioration** | **3.47%** |
| **Temps d'exécution** | **4.78 secondes** |

### 4.3 Meilleure permutation trouvée

```
Schedule (1-based) : [3, 1, 11, 18, 19, 6, 5, 2, 8, 9, 10, 12, 13, 14, 7, 4, 16, 15, 17, 20]
```

### 4.4 Courbe de convergence

La courbe de convergence montre une décroissance rapide dans les premières itérations (1412 → 1371 dès l'itération 1), suivie d'une stabilisation progressive jusqu'à 1363. Le comportement est typique d'un algorithme métaheuristique avec élitisme :

| Itération | Makespan |
|---|---|
| 0 | 1412 |
| 1 | 1371 |
| 50 | 1371 |
| 100 | 1363 |
| 150 | 1363 |
| 200 | 1363 |

---

## 5. Éléments préservés à 100% de Mirjalili (2015)

| Composant | Statut |
|---|---|
| ✅ Marche aléatoire (somme cumulée, pas binaires) | Préservé |
| ✅ Coefficient adaptatif $I$ avec paliers | Préservé |
| ✅ Rétrécissement des frontières $(\text{lb}/I, \text{ub}/I)$ | Préservé |
| ✅ Normalisation min-max | Préservé |
| ✅ Roulette Wheel (sélection proportionnelle) | Préservé |
| ✅ Élitisme (meilleur antlion toujours conservé) | Préservé |
| ✅ Remplacement antlion ← fourmi plus fitte | Préservé |
| ✅ Tri de la population d'antlions | Préservé |

## 6. Adaptations pour le PFSP

| Adaptation | Justification |
|---|---|
| **Random Keys** (`argsort`) | Convertit un vecteur continu $\mathbb{R}^n$ en permutation valide |
| **Makespan** ($C_{\max}$) | Fonction objectif spécifique au Flow Shop |
| **Minimisation** | Fitness inversée pour la roulette wheel |

---

## 7. Analyse et interprétation

### Forces
- **Implémentation fidèle** à l'algorithme original de Mirjalili
- **Architecture modulaire** facilitant les modifications et extensions
- **Visualisations complètes** (courbe de convergence + diagramme de Gantt)
- **Code documenté** avec paramètres, types et explications mathématiques

### Limitations
- L'amélioration de 3.47% est modeste, ce qui suggère que 200 itérations sont insuffisantes pour cette instance de 20 jobs
- La population de 40 individus pourrait être augmentée pour une meilleure exploration
- Pas de comparaison avec d'autres algorithmes (NEH, PSO, GA)

### Perspectives d'amélioration
1. Augmenter le nombre d'itérations (500-1000)
2. Augmenter la taille de la population (50-100)
3. Intégrer une phase de recherche locale (Hill Climbing, VNS)
4. Initialiser avec l'heuristique NEH
5. Comparer avec PSO, GA, ou d'autres métaheuristiques

---

## 8. Structure du projet

```
ALO_FlowShop/
│
├── ALO_FlowShop.ipynb    ← Notebook principal (29 cellules, ~960 lignes)
├── RAPPORT.md            ← Ce rapport
├── data/
│   └── (Taillard instances)
└── README.md
```

---

## 9. Conclusion

Ce notebook implémente l'**Ant Lion Optimizer (Mirjalili, 2015)** pour le **Permutation Flow Shop Scheduling Problem**. L'adaptation conserve **100% de la logique algorithmique** de l'ALO original, en modifiant uniquement l'évaluation de fitness via :

1. **Random Keys** pour la conversion continu → permutation
2. **Makespan** $C_{\max}$ comme fonction objectif

Les résultats sur l'instance **Taillard tai20_5** montrent une convergence de **1412 à 1363** (amélioration de **3.47%**) en **4.78 secondes**, démontrant la capacité de l'ALO à résoudre efficacement des problèmes d'ordonnancement combinatoires.
