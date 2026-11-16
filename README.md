# Dynamic Time Warping en R et Rcpp : complexité et classification d'ECG

Ce dépôt réunit deux travaux autour du **Dynamic Time Warping** (DTW), une distance qui compare deux séries temporelles en tolérant des décalages et des étirements dans le temps :

1. **Un projet d'algorithmique de M2, réalisé en équipe** : implémentation du DTW et de ses variantes en R et en C++ (Rcpp), et étude de leur complexité ;
2. **Une application personnelle** : la classification de battements cardiaques (jeu ECG5000) par k plus proches voisins, avec le DTW comme distance.

📄 Rapports : [`projet_algo.rmd`](projet_algo.rmd) (partie 1) · [`classification_ecg.Rmd`](classification_ecg.Rmd) (partie 2) · détails de la partie 2 : [`README_ECG.md`](README_ECG.md)

---

## Partie 1 — Implémentation et complexité du DTW (projet d'équipe)

### Ce qui est implémenté

Le projet est organisé comme un package R (`ProjetM2Algo`).

| Algorithme | R | Rcpp | Complexité théorique |
|---|---|---|---|
| DTW complet, avec reconstruction du chemin optimal | `dtw()` | `dtw_rcpp()` | O(n · m) |
| DTW avec bande de Sakoe-Chiba de rayon r | `dtw_sakoe_chiba()` | `dtw_sakoe_chiba_rcpp()` | O(n · r) |
| Segmental DTW (calcul par segments) | `segmental_dtw_two_series()` | `segmental_dtw_cpp()` | O(n · m) |

### Vérification expérimentale de la complexité

Le temps de calcul est mesuré pour des séries de longueur croissante. La pente d'un graphique log-log, log(temps) en fonction de log(n), donne l'exposant de la complexité : 2 pour un algorithme quadratique, 1 pour un algorithme linéaire.

- **DTW complet** : sur de grandes séries (n de 1 000 à 5 000), la pente est proche de 2 en R comme en Rcpp, ce qui confirme la complexité quadratique. **La simulation prend environ 32 minutes en R contre environ 2 minutes en C++.**
- **DTW avec bande de Sakoe-Chiba** : la pente est d'environ 1 en Rcpp (0,95 à 1,00), conformément à la complexité linéaire en n. Elle est d'environ 1,4 en R : l'interpréteur R ajoute un surcoût qui croît avec n.

### Application à des données financières

La bande de Sakoe-Chiba sert à aligner le cours de clôture d'une action et celui d'une matière première (le pétrole), deux séries de fréquences différentes, avant de les comparer.

### Équipe

Samira Sonfack, Fatima-Zahra Aklila et Sarah Eliko ([dépôt d'origine](https://github.com/samirasonfack/ProjetM2Algo)).

Ma contribution à cette partie : l'implémentation du DTW, les fonctions de simulation de la complexité (R contre Rcpp) et l'analyse log-log.

---

## Partie 2 — Classification de battements cardiaques (travail personnel)

**Question** : la distance DTW permet-elle de mieux reconnaître les battements cardiaques anormaux qu'une distance euclidienne, et à quel coût de calcul ?

**Données** : ECG5000 (archive UCR, issue de PhysioNet), soit 5 000 battements de 140 points en 5 classes très déséquilibrées : 58 % de battements normaux, et 0,5 % pour la classe la plus rare.

**Méthode** :
- kNN comparé avec trois distances : euclidienne, DTW complet et DTW Sakoe-Chiba, en utilisant les fonctions Rcpp de la partie 1 ;
- k et r réglés par validation croisée stratifiée sur le jeu d'entraînement ;
- évaluation adaptée au déséquilibre des classes : accuracy équilibrée, F1 macro, sensibilité et spécificité « normal / anormal », rappel par classe et matrices de confusion.

**Principaux résultats** :

| | Accuracy | Acc. équilibrée | Sensibilité (anormal) | Spécificité (anormal) | Temps |
|---|---|---|---|---|---|
| Référence naïve (classe majoritaire) | 0,584 | 0,200 | 0,000 | 1,000 | — |
| Euclidienne, k = 5 | 0,939 | 0,523 | 0,947 | 0,999 | 4 s |
| DTW Sakoe-Chiba r = 5, k = 5 | 0,940 | 0,518 | 0,946 | 0,998 | 47 s |
| DTW complet, k = 5 | 0,941 | 0,515 | 0,950 | 0,997 | 17 min 27 s |
| **DTW complet, k = 1 (réglé par validation croisée)** | 0,930 | **0,551** | **0,959** | 0,995 | — |

- **Le modèle distingue très bien un battement normal d'un battement anormal** (sensibilité d'environ 0,95, spécificité supérieure à 0,99). En revanche, il identifie mal le *type* d'anomalie pour les classes rares.
- **Le DTW n'apporte pas de gain mesurable** par rapport à la distance euclidienne sur ces battements déjà alignés, pour un temps de calcul environ 250 fois plus long. La bande de Sakoe-Chiba garde la même qualité en 22 fois moins de temps.
- **Le réglage de k et r** (k = 1) permet enfin de détecter des battements de la classe la plus rare et améliore la sensibilité. Ce gain reste toutefois fragile : il repose sur très peu d'exemples rares.
- **Limites** : un seul patient, et très peu de battements rares dans l'entraînement.

👉 Analyse complète, tableaux par classe et discussion : [`README_ECG.md`](README_ECG.md).

---

## Reproduire

Prérequis : R ≥ 4.1, Rcpp ≥ 1.0.13, devtools, knitr, ggplot2, dplyr et tidyr.

```r
# Ouvrir ProjetM2Algo.Rproj dans RStudio
devtools::load_all(".")                      # compile et charge les fonctions DTW
rmarkdown::render("projet_algo.rmd")         # partie 1
rmarkdown::render("classification_ecg.Rmd")  # partie 2 (le DTW complet prend ~20 min au premier calcul)
```

---

Sarah Eliko · M2 Data Science, Université Paris-Saclay
