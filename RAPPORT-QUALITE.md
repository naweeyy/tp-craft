# Rapport qualité, module inventaire

Nom : Noé Marchand
Date : 2026-09-16
Empreinte du commit de départ : d686390ff29b45f4c9eccfcf954d603884c496e8

---

## 1. Tableau de bord initial

Mesures relevées avant toute modification.

### Complexité par fonction

| Fonction | Ligne | Complexité cyclomatique | Rang |
|---|---|---|---|
| rapport | 122 | 22 | D |
| par_cat | 94 | 10 | B |
| mouv | 37 | 9 | B |
| classer | 74 | 5 | A |
| val | 19 | 3 | A |
| alerte | 29 | 3 | A |
| cout | 62 | 3 | A |
| rot | 87 | 2 | A |
| maj_prix | 175 | 1 | A |
| export_json | 185 | 1 | A |

Commande utilisée :

```bash
radon cc -s inventaire/inventaire.py
```

### Synthèse du fichier

| Mesure | Valeur | Commande |
|---|---|---|
| Lignes de code réelles (SLOC) | 159 | `radon raw inventaire/inventaire.py` |
| Complexité moyenne | B (5.9) | `radon cc -s -a inventaire/inventaire.py` |
| Indice de maintenabilité | A (36.80) | `radon mi -s inventaire/inventaire.py` |
| Score pylint | 7.76 / 10 | `pylint inventaire/inventaire.py` |
| Problèmes ruff | 14 | `ruff check inventaire/inventaire.py` |
| Entrées vulture | 14 (dont 1 à 100 % de confiance : arguments inutilisés de `maj_prix`) | `vulture inventaire/inventaire.py` |
| Couverture de branches | 0 % (aucun test n'existe encore) | `pytest --cov=inventaire --cov-branch --cov-report=term-missing` |
| Barrière xenon (max B / A moyen) | Échec : `rapport` en rang D, moyenne en rang B | `xenon --max-absolute B --max-modules B --max-average A inventaire/inventaire.py` |

---

## 2. Catalogue des odeurs

| # | Ligne | Odeur ou défaut | Détecté par | Conséquence concrète |
|---|---|---|---|---|
| 1 | 32 et 63 | Comparaison stricte `a["q"] < a["seuil"]` au lieu de `<=` dans `alerte()` et `cout()`, alors que la règle M2 dit qu'un article exactement au seuil est en alerte | **Aucun outil** — code syntaxiquement valide | Un article dont le stock tombe pile au seuil (ex. `GANT-L`, q=5, seuil=5 dans `exemple_utilisation.py`) n'est jamais signalé ni chiffré pour réappro : c'est très probablement la cause des ruptures de stock passées inaperçues le mois dernier. |
| 2 | 88-91 | `rot()` avale toute exception (y compris division par zéro) et renvoie silencieusement 0, alors que la règle M7 exige de lever une erreur explicite en l'absence de vente | Le `except:` nu est vu par pylint/ruff, mais **pas** le fait que la valeur de repli (0) contredit la règle métier | Un article sans historique de vente affiche « 0 jour de stock restant » au lieu de signaler l'absence de donnée : la direction peut croire à une rupture imminente inexistante, ou l'inverse. |
| 3 | tout le fichier (ex. `a`, `q`, `t`, `l`, `d`, `n`, `c`, `j`) | Noms de variables et de paramètres à une ou deux lettres, aucun ne porte de sens métier | **Aucun outil** — identifiants valides, la configuration pylint par défaut ne les signale pas | Personne ne peut relire une ligne sans reconstruire mentalement ce que représente chaque lettre ; c'est très probablement la raison pour laquelle « personne n'ose plus toucher » ce fichier. |
| 4 | 122-172 | `rapport()` mélange calcul métier, affichage console (`print`) et écriture disque (`export`) dans une seule fonction, alors que M8 exige qu'un rapport ne dépende d'aucune ressource externe et ne modifie rien | **Aucun outil** — les effets de bord sont du Python parfaitement légal | Impossible de tester le calcul de la valeur du stock sans déclencher des `print` ou risquer une écriture de fichier ; impossible aussi de réutiliser le calcul seul ailleurs. |
| 5 | 37 | Argument par défaut mutable `j=[]` dans `mouv()` | ruff `B006` | Tous les appels qui omettent `j` partagent la même liste : le journal d'un mouvement peut se retrouver pollué par les mouvements d'un appel précédent. |
| 6 | 185 | Argument par défaut mutable `hist=[]` dans `export_json()` | ruff `B006` | Le fichier exporté peut contenir l'historique de rapports totalement sans rapport avec l'appel en cours, un bug très difficile à reproduire. |
| 7 | 90 | `except:` nu dans `rot()` | ruff `E722`, pylint `W0702` | N'importe quelle erreur (même une faute de frappe dans le code appelant) est masquée et remplacée par 0, sans trace. |
| 8 | 169 et 187 | `open()` sans gestionnaire de contexte et sans encodage explicite | pylint `R1732`/`W1514`, ruff `SIM115` | Fuite de descripteur de fichier si une exception survient entre l'ouverture et la fermeture ; comportement d'encodage différent selon la plateforme d'exécution. |
| 9 | 122 | `rapport()` a 7 paramètres et une complexité cyclomatique de 22 (rang D) | pylint `R0913`/`R0912`, radon, xenon | Ajouter un nouveau filtre ou un nouveau mode de sortie oblige à modifier une fonction déjà proche de l'incompréhensible, avec un risque élevé de casser une autre branche sans s'en apercevoir. |
| 10 | 94-119 | `par_cat()` répète le même bloc de 6 lignes pour chaque catégorie (`outil`, `consommable`, `piece`, `autre`) | pylint (complexité, branches) | Ajouter une 5ᵉ catégorie de matériel impose de copier-coller un bloc de plus, avec le même risque de typo que dans les quatre précédents. |
| 11 | 14-16, 38, 56, 58 | État global mutable `JOURNAL` / `DERNIER`, modifié via `global DERNIER` dans `mouv()` | pylint `W0603` (uniquement pour le mot-clé `global`) | Deux appels à `mouv()` faits par des parties différentes du programme (ou par deux tests) se marchent dessus via le même compteur et le même journal partagé, rendant le comportement dépendant de l'ordre d'exécution. |
| 12 | 175-182 | `maj_prix()` : corps entièrement commenté, la fonction ne fait plus rien et renvoie toujours `None` | vulture (fonction jamais utilisée) | Quiconque appelle cette fonction en pensant mettre à jour un prix ne reçoit aucune erreur : l'opération échoue silencieusement, sans aucun signal. |

---

## 3. Faut-il tout réécrire

Non. Les chiffres ne montrent pas un code irrécupérable : l'indice de maintenabilité
est en rang A (36,80) et le score pylint est de 7,76/10 — c'est d'ailleurs le piège
de cette mission, ces deux indicateurs sont flatteurs parce qu'ils mesurent des
moyennes et ne voient ni les comparaisons `<` au lieu de `<=` (odeur n°1, la cause
probable des ruptures de stock du mois dernier), ni le mélange calcul/effets de bord
de `rapport()`, ni les noms de variables illisibles. Ce que les outils voient bien,
en revanche, est sans appel : `rapport()` est en rang D avec une complexité de 22,
xenon fait échouer le fichier, et la couverture de tests est de 0 %. Le problème
n'est donc pas que le code soit fondamentalement mauvais, c'est qu'il est
insuffisamment testé et localement trop complexe à trois ou quatre endroits précis.

Le cours cite le cas Netscape (1997-2003) : une réécriture complète du moteur de
rendu, décidée parce que « plus personne ne comprenait » le code existant, a laissé
l'éditeur sans version compétitive pendant plus de deux ans, le temps qu'Internet
Explorer passe de 20 % à plus de 80 % de part de marché. Le code de 2019 tourne en
production depuis sept ans et encode une connaissance métier réelle (les règles
M1 à M8) qu'une réécriture from scratch devrait entièrement redécouvrir, probablement
en réintroduisant des bugs déjà corrigés dans la version actuelle mais jamais
documentés. Réécrire, c'est reproduire Netscape en miniature ; ajouter des tests
de caractérisation puis refactoriser par étranglement, c'est suivre l'exemple de
Twitter cité en cours.

Ordre d'intervention proposé : d'abord un filet de tests de caractérisation sur
tout le fichier (mission 3), pour pouvoir toucher au code sans peur ; ensuite corriger
en priorité l'odeur n°1 (`<` au lieu de `<=`), parce qu'elle a un impact chiffrable
immédiat sur les ruptures de stock et ne nécessite qu'un changement d'une ligne ;
puis simplifier `rapport()` et `par_cat()`, qui concentrent à eux deux la quasi-totalité
de la complexité cyclomatique du fichier ; le nettoyage des noms de variables et de
l'état global vient en dernier, car il améliore la lisibilité sans changer le
comportement observable.

---

## 4. Écarts constatés entre le code et les règles métier

Rempli pendant la mission 3, sans rien corriger.

| Règle | Ligne | Ce que le code fait | Ce que la règle dit |
|---|---|---|---|
|  |  |  |  |

---

## 5. Tableau de bord après refactoring

Mêmes mesures, mêmes commandes qu'en partie 1.

| Mesure | Avant | Après | Écart |
|---|---|---|---|
|  |  |  |  |

Ce que ce delta prouve, en trois phrases maximum :

---

## 6. Bugs prouvés puis corrigés

| Règle violée | Ligne d'origine | Commit red | Commit fix | Conséquence métier |
|---|---|---|---|---|
|  |  |  |  |  |

Pour au moins un de ces bugs, la conséquence est chiffrée en euros ou en ruptures de stock.
