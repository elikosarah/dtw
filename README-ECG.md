# Classification de battements cardiaques par Dynamic Time Warping

**Le DTW améliore-t-il la détection des battements anormaux ?**

Analyse en R/Rcpp sur le jeu de données ECG5000. J'y compare une classification par k plus proches voisins (kNN) avec trois distances : euclidienne, DTW complet et DTW avec bande de Sakoe-Chiba. J'évalue chaque méthode classe par classe et je mesure son temps de calcul.

📄 Rapport : [`classification_ecg.Rmd`](classification_ecg.Rmd)

---

## Contexte et question

Un électrocardiogramme (ECG) contient des milliers de battements. Classer automatiquement chaque battement, normal ou anormal, aide à repérer des arythmies sans tout relire à la main.

Le **Dynamic Time Warping** (DTW) compare deux séries en tolérant des décalages et des étirements dans le temps. Il devrait donc mieux reconnaître deux battements de même forme dont le rythme diffère. Son coût est en revanche quadratique, en O(n²). La **bande de Sakoe-Chiba** limite l'alignement autour de la diagonale pour réduire ce coût.

> **La distance DTW permet-elle de mieux reconnaître les battements anormaux qu'une distance euclidienne, et à quel coût de calcul ?**

## Données

**ECG5000** (archive UCR des séries temporelles, issue de la base PhysioNet *BIDMC Congestive Heart Failure Database*) : 5 000 battements de 140 points, répartis en 5 classes. J'ai conservé le découpage officiel entre entraînement et test.

| Classe | Train | Test | Part du total |
|---|---|---|---|
| 1 (normal) | 292 | 2 627 | 58,4 % |
| 2 | 177 | 1 590 | 35,3 % |
| 3 | 10 | 86 | 1,9 % |
| 4 | 19 | 175 | 3,9 % |
| 5 | 2 | 22 | 0,5 % |

Les classes sont **fortement déséquilibrées**, alors que ce sont justement les battements anormaux, minoritaires, qu'il importe de détecter.

Le découpage officiel est **stratifié** : les 5 classes sont présentes dans les deux ensembles, dans les mêmes proportions (par exemple 2,0 % de classe 3 dans le train et 1,9 % dans le test). Le test reflète donc fidèlement l'entraînement.

En revanche, le jeu d'entraînement est **petit pour les classes rares** : il ne contient que 10 battements de classe 3, 19 de classe 4 et **2 de classe 5**. C'est la principale contrainte de cette étude, car le kNN ne peut reconnaître une classe qu'à partir des exemples qu'il a en mémoire.

## Méthode

- **Classifieur** : kNN avec vote majoritaire, calculé à partir d'une matrice de distances (4 500 battements de test × 500 battements d'entraînement, soit 2,25 millions de distances par méthode). Les méthodes sont d'abord comparées avec k = 5 et r = 5, puis k et r sont réglés par validation croisée.
- **Distances comparées** :
  - euclidienne, comme référence ;
  - DTW complet, en Rcpp ;
  - DTW avec bande de Sakoe-Chiba de rayon r = 5, en Rcpp.
- **Référence naïve** : toujours prédire la classe majoritaire.
- **Évaluation** : l'accuracy seule est trompeuse quand les classes sont déséquilibrées. Je rapporte donc plusieurs indicateurs complémentaires :
  - l'**accuracy équilibrée** (moyenne des rappels des 5 classes) et le **F1 macro**, qui donnent le même poids à chaque classe ;
  - la **sensibilité** et la **spécificité** pour la question clinique « normal ou anormal ? » (classe 1 contre classes 2 à 5) ;
  - la **précision**, le **rappel** et le **F1** de chaque classe, ainsi que les **matrices de confusion**.
- **Temps** : celui du calcul de la matrice de distances, mesuré avec `system.time`.
- **Réglage de k et r** : validation croisée **stratifiée à 5 plis sur le seul jeu d'entraînement**, avec k ∈ {1, 3, 5, 7, 9} et r ∈ {0, 1, 2, 3, 5, 8, 12, 20}, plus le DTW complet et la distance euclidienne. Le critère est l'accuracy équilibrée. Le jeu de test n'est utilisé qu'une fois, pour évaluer la combinaison retenue.

## Résultats

*Première comparaison, avec k = 5 et r = 5 fixés.*

### Indicateurs globaux

| Méthode | Accuracy | Acc. équilibrée | F1 macro | Sensibilité (anormal) | Spécificité (anormal) | Temps |
|---|---|---|---|---|---|---|
| Classe majoritaire | 0,584 | 0,200 | 0,147 | 0,000 | 1,000 | — |
| Euclidienne | 0,939 | **0,523** | **0,564** | 0,947 | **0,999** | **4 s** |
| DTW Sakoe-Chiba (r = 5) | 0,940 | 0,518 | 0,557 | 0,946 | 0,998 | 47 s |
| DTW complet | **0,941** | 0,515 | 0,554 | **0,950** | 0,997 | 17 min 27 s |

*Temps mesurés sur un MacBook Pro. La classe majoritaire est une référence naïve, qui prédit toujours « normal ».*

### Par classe

| Méthode | Rappel 1 · 2 · 3 · 4 · 5 | Précision 1 · 2 · 3 · 4 · 5 | F1 1 · 2 · 3 · 4 · 5 |
|---|---|---|---|
| Euclidienne | 0,999 · 0,951 · 0,302 · 0,360 · 0 | 0,964 · 0,917 · 0,812 · 0,656 · — | 0,981 · 0,934 · 0,441 · 0,465 · 0 |
| DTW Sakoe-Chiba (r = 5) | 0,998 · 0,955 · 0,267 · 0,371 · 0 | 0,963 · 0,926 · 0,821 · 0,602 · — | 0,980 · 0,940 · 0,404 · 0,459 · 0 |
| DTW complet | 0,997 · 0,964 · 0,267 · 0,349 · 0 | 0,965 · 0,920 · 0,719 · 0,678 · — | 0,981 · 0,941 · 0,390 · 0,460 · 0 |

La classe 5 n'est jamais prédite : sa précision n'est donc pas définie (—).

### Matrice de confusion — DTW complet

| Prédit ↓ / Réel → | 1 | 2 | 3 | 4 | 5 |
|---|---|---|---|---|---|
| **1** | **2 619** | 26 | 24 | 30 | 14 |
| **2** | 7 | **1 532** | 36 | 83 | 7 |
| **3** | 0 | 8 | **23** | 1 | 0 |
| **4** | 1 | 24 | 3 | **61** | 1 |
| **5** | 0 | 0 | 0 | 0 | **0** |

### Ce que montrent les résultats

1. **Le modèle sépare très bien les battements normaux des battements anormaux.** Avec le DTW complet, 95 % des battements anormaux sont signalés comme anormaux, et presque aucun battement normal ne déclenche de fausse alerte (spécificité de 0,997). Pour un premier tri, c'est un résultat solide.
2. **La difficulté est d'identifier le *type* d'anomalie.** Les erreurs principales confondent les anomalies entre elles : 83 des 175 battements de classe 4, et 36 des 86 de classe 3, sont prédits en classe 2. Les rappels des classes 3 et 4 restent compris entre 27 % et 37 %.
3. **La classe 5 n'est jamais reconnue**, et 14 de ses 22 battements sont classés comme normaux. Ce sont des anomalies manquées. L'explication tient à la règle du vote : avec k = 5 voisins et seulement 2 exemples de classe 5 dans l'entraînement, cette classe obtient au mieux 2 voix. Elle ne peut gagner que si les 3 autres voisins appartiennent tous à des classes différentes, ce qui est presque impossible. Et en cas d'égalité à 2 voix contre 2, l'implémentation choisit la première classe dans l'ordre, jamais la classe 5. Les classes 3 et 4, avec 10 et 19 exemples, souffrent du même problème dans une moindre mesure.
4. **L'accuracy globale masque ce problème.** Elle est élevée (environ 0,94) parce que les classes 1 et 2, bien reconnues, représentent 94 % des données. L'accuracy équilibrée (environ 0,52) et le F1 macro (environ 0,56), qui donnent le même poids à chaque classe, reflètent mieux la difficulté sur les classes rares.
5. **Le DTW n'apporte pas de gain mesurable.** Les trois distances sont à moins de 0,01 l'une de l'autre sur tous les indicateurs : le DTW complet est légèrement devant en accuracy et en sensibilité, l'euclidienne en F1 macro et en accuracy équilibrée. Le DTW complet demande pourtant **environ 250 fois plus de temps de calcul**. Une explication probable : les battements d'ECG5000 sont déjà découpés et alignés, il reste donc peu de décalage temporel à corriger.
6. **La bande de Sakoe-Chiba conserve la qualité du DTW complet tout en étant 22 fois plus rapide.** Si l'on utilise le DTW, c'est la variante à privilégier.
7. **Rcpp est indispensable.** Les versions en R pur donnent exactement les mêmes distances, mais elles sont environ 10 fois plus lentes pour le DTW complet et 15 fois pour Sakoe-Chiba. Le kNN complet en R se compterait en heures.

### Réglage de k et r par validation croisée

Accuracy équilibrée en validation croisée (extrait du tableau complet du rapport) :

| Distance | k = 1 | k = 3 | k = 5 | k = 7 | k = 9 | Temps (s) |
|---|---|---|---|---|---|---|
| Euclidienne | 0,580 | 0,564 | 0,568 | 0,527 | 0,457 | 0,3 |
| Sakoe-Chiba r = 5 | 0,588 | 0,586 | 0,587 | 0,587 | 0,526 | 2,5 |
| Sakoe-Chiba r = 12 | 0,568 | 0,567 | 0,589 | 0,590 | 0,488 | 4,0 |
| Sakoe-Chiba r = 20 | 0,766 | 0,547 | 0,589 | 0,590 | 0,489 | 5,3 |
| DTW complet | **0,770** | 0,549 | 0,570 | 0,561 | 0,470 | 56,0 |

La combinaison retenue est **DTW complet, k = 1**. Évaluation sur le jeu de test :

| | Accuracy | Acc. équilibrée | F1 macro | Sensibilité (anormal) | Spécificité (anormal) |
|---|---|---|---|---|---|
| Avant : Sakoe-Chiba r = 5, k = 5 | **0,940** | 0,518 | 0,557 | 0,946 | **0,998** |
| Après : DTW complet, k = 1 | 0,930 | **0,551** | **0,579** | **0,959** | 0,995 |

Rappels par classe après réglage : 0,995 · 0,925 · 0,360 · 0,383 · **0,091**.

- **Le réglage améliore la détection des anomalies.** La classe 5 est reconnue pour la première fois (2 battements sur 22), la classe 3 passe de 27 % à 36 % de rappel, et les battements anormaux pris pour des normaux passent d'environ 100 à 76. En contrepartie, plus de battements de classe 2 sont confondus avec la classe 4, et l'accuracy baisse légèrement. C'est le compromis attendu : avec k = 1, une classe rare n'a plus besoin de gagner un vote pour être prédite.
- **Le gain en validation croisée est surestimé.** Avec k = 1, l'accuracy équilibrée saute de 0,568 (r = 12) à 0,766 (r = 20), soit +0,20, exactement le poids d'une classe sur cinq. L'explication la plus probable : les 2 battements de classe 5 du jeu d'entraînement deviennent le plus proche voisin l'un de l'autre. Le choix repose donc sur 2 battements, et le gain réel sur le test n'est que de +0,033.
- **Sakoe-Chiba r = 20 fait presque aussi bien que le DTW complet** en validation croisée (0,766 contre 0,770), en 10 fois moins de temps. Par parcimonie, c'est une alternative défendable.

**Conclusion** : le kNN sépare bien les battements normaux des anormaux, mais distingue mal les types d'anomalies, en particulier les plus rares. Sur des battements déjà alignés, le choix de la distance compte peu. Le réglage de k et r, surtout k = 1, améliore modestement la détection des classes rares, mais reste fragile avec si peu d'exemples : **la vraie difficulté est le déséquilibre des classes**.

## Limites et pistes

- **Le choix de k et r repose sur très peu de battements rares** : la validation croisée est dominée par les 2 battements de classe 5. Une validation croisée répétée, avec plusieurs découpages aléatoires, rendrait le choix plus stable.
- **Un seul patient** : d'après l'archive UCR, les 5 000 battements proviennent d'un seul enregistrement de 20 heures. Les performances ne garantissent donc pas une généralisation à d'autres patients, ce qui demanderait des données multi-patients et un découpage par patient.
- **Le jeu d'entraînement officiel est petit pour les classes rares** (2 battements de classe 5). On pourrait réduire k (k = 1 ou 3), pondérer le vote, ou utiliser l'autre découpage (4 500 battements en entraînement). Ce dernier choix laisserait cependant trop peu de battements rares dans le test pour les évaluer de façon fiable.
- **Les écarts entre méthodes sont très faibles.** Un test de McNemar dirait s'ils sont statistiquement significatifs.
- **Pour mieux détecter les classes rares**, plusieurs pistes sont possibles : un vote pondéré par la distance ou par l'inverse de la fréquence des classes, k = 1, ou un rééchantillonnage.
- **Pour le temps de calcul**, une fonction qui renvoie uniquement la distance DTW, sans matrice de chemin, réduirait fortement le temps du DTW complet.

## Reproduire

Prérequis : R ≥ 4.1, Rcpp ≥ 1.0.13, devtools et knitr.

```r
# 1. Ouvrir le projet RStudio (ProjetM2Algo.Rproj), puis préparer les données une seule fois
source("data-raw/preparer_ecg5000.R")   # crée inst/extdata/ECG5000.rds

# 2. Générer le rapport
rmarkdown::render("classification_ecg.Rmd")
```

Le calcul du DTW complet prend environ 20 minutes. Grâce à `cache = TRUE`, il n'est refait que si le bloc de code change.

## Crédits

Les fonctions DTW utilisées (`dtw`, `dtw_rcpp`, `dtw_sakoe_chiba`, `dtw_sakoe_chiba_rcpp`) proviennent du **projet d'algorithmique de M2** réalisé avec Samira Sonfack et Fatima-Zahra Aklila ([dépôt d'origine](https://github.com/samirasonfack/ProjetM2Algo)).

L'application à la classification d'ECG présentée ici est mon travail personnel.

**Sources des données** : UCR Time Series Classification Archive ; PhysioNet, *BIDMC Congestive Heart Failure Database*.

---

Sarah Eliko · M2 Data Science, Université Paris-Saclay
