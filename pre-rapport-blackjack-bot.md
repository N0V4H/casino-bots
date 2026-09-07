# Pré-rapport — Blackjack Bot (Module 1 du projet Casino Bots)

## 1. Contexte et objectif

Projet de portfolio visant à démontrer une maîtrise de l'algorithmique, des probabilités et de la simulation Monte Carlo, à travers des bots capables de calculer (et exploiter, quand c'est mathématiquement possible) un avantage sur des jeux de casino.

Le blackjack est le point de départ car c'est le seul jeu de casino classique où un avantage joueur réel est atteignable, via la stratégie de base combinée au comptage de cartes. Le poker sera un module ultérieur et séparé (problème joueur-vs-joueur, approche par théorie des jeux / GTO, hors périmètre de ce document).

Les jeux à espérance négative fixe et sans état exploitable (roulette, machines à sous, baccarat) sont explicitement exclus du projet : aucun calcul de probabilité ne peut y créer d'avantage.

## 2. Choix techniques

| Décision | Choix retenu |
|---|---|
| Langage | Python (pur) |
| Interface | Shell / CLI |
| Priorité de conception | Équilibre lisibilité pédagogique / performance |
| Visualisation des résultats | Texte console uniquement (V1) |
| Rigueur des tests | Tests unitaires basiques sur les fonctions clés |
| Configuration de table (V1) | Un joueur solo face au croupier |
| Documentation | README + docstrings détaillées |
| Dépendances externes | Quelques librairies légères (ex. `rich` pour le rendu CLI) |
| Reproductibilité | Seed aléatoire fixable pour reproduire une simulation exacte |

### Justifications / implications
- **Python pur + équilibre perf/lisibilité** : privilégier un code idiomatique et bien typé (type hints, dataclasses) plutôt que des micro-optimisations prématurées ; optimiser seulement si un profiling montre un vrai goulot d'étranglement (probable sur les simulations Monte Carlo à grande échelle, ex. `numpy` pour vectoriser le tirage de mains si besoin plus tard — à réévaluer, pas décidé pour la V1).
- **CLI + texte console** : pas de dépendance à un framework GUI/web pour la V1 ; `rich` permettra tout de même un affichage clair (tableaux, couleurs, barres de progression) sans complexité additionnelle.
- **Seed fixable** : nécessite un point d'entrée unique pour la génération aléatoire (ex. `random.Random(seed)` injecté explicitement, jamais l'état global `random`), afin que toute simulation soit reproductible pour le debug et les tests.
- **Tests basiques** : cibler en priorité les fonctions pures et critiques (calcul de valeur de main, détection de blackjack/bust, calcul du true count, décision de stratégie de base) plutôt qu'une couverture exhaustive.

## 3. Règles de la variante modélisée

| Paramètre | Valeur |
|---|---|
| Nombre de jeux de cartes | 6 |
| Comportement croupier sur soft 17 | Reste (S17) |
| Double autorisé | Sur n'importe quelles 2 cartes |
| Double après split (DAS) | Autorisé |
| Nombre de resplits | Illimité |
| Split des As | Autorisé plusieurs fois (pas limité à une carte par main) |
| Surrender | Non autorisé |
| Paiement blackjack | 3:2 |
| Pénétration du sabot | Aléatoire entre 65% et 75% (tirée à chaque nouveau sabot) |
| Assurance | Disponible |
| Règle Charlie | Absente |

Avantage maison théorique estimé avec cette configuration (avant comptage) : ~0.3–0.4%.

## 4. Inventaire des données nécessaires

### 4.1 Règles du jeu
Cf. section 3 — à modéliser comme un objet de configuration unique et immuable (`GameRules`), source de vérité injectée dans tous les autres modules.

### 4.2 Cartes et sabot
- Rang de carte (2–As) et valeur associée (les figures = 10, As = 1 ou 11 selon contexte)
- Couleur/enseigne : non pertinente, à ignorer dans le modèle
- Composition du sabot : nombre de cartes restantes par rang (nécessaire au comptage et au calcul exact de probabilités)

### 4.3 État de la partie (par main / par décision)
- Main du joueur (liste de cartes)
- Carte visible du croupier
- Mise en cours
- Cartes déjà sorties depuis le dernier rebattage

*(Note : pas de gestion multi-mains/splits simultanés en V1 puisque la config retenue est "un joueur solo vs croupier" — mais le modèle de données doit rester extensible pour couvrir les splits sans refonte majeure, les règles autorisant le resplit illimité.)*

### 4.4 Tables de stratégie de base
Trois tables généré par calcul d'EV (et non codées en dur) :
- Hard totals (total joueur 4–21) × carte croupier (2–As)
- Soft totals × carte croupier
- Pairs (décision de split) × carte croupier

Chaque table dépend directement de la configuration de règles (section 3).

### 4.5 Comptage de cartes
- Système de comptage (à choisir — ex. Hi-Lo pour commencer, extensible à KO/Omega II plus tard)
- Table de valeurs par rang selon le système choisi
- Running count (cumulatif)
- True count = running count / decks restants estimés
- Indices de déviation (ex. Illustrious 18) pour affiner stratégie et décision d'assurance selon le true count — l'assurance étant l'un des indices de déviation classiques, à traiter comme une décision à part entière influencée par le count

### 4.6 Bankroll / gestion de mise
- Unité de mise de base
- Table de spread de mise selon le true count
- Bankroll de départ, règles éventuelles de stop-loss/stop-win

### 4.7 Suivi statistique / simulation
- Nombre de mains à simuler (Monte Carlo, ordre de grandeur 10⁵–10⁷ pour convergence)
- Résultat par main (gain/perte)
- Métriques agrégées : EV%, écart-type, edge par tranche de true count, risque de ruine
- Seed de génération aléatoire utilisée (pour traçabilité/reproductibilité)

## 5. Architecture modulaire proposée

```
blackjack_bot/
├── rules.py          # GameRules (dataclass immuable, section 3)
├── card.py           # Card, valeur, comparaisons
├── shoe.py           # Shoe : composition, tirage, pénétration, reshuffle
├── hand.py           # Hand : calcul de valeur (soft/hard), bust, blackjack
├── strategy.py       # BasicStrategy : génération des tables par calcul d'EV
├── counter.py        # Counter : running/true count, indices de déviation
├── bankroll.py       # Gestion de mise et bankroll (spread selon count)
├── simulator.py       # Boucle de simulation Monte Carlo, orchestration
├── cli.py            # Interface shell (rich)
└── tests/
    └── test_*.py     # Tests unitaires ciblés (fonctions clés)
```

- `GameRules` est injecté dans `shoe.py`, `strategy.py` (génère les tables selon les règles) et `simulator.py`.
- `strategy.py` ne contient aucune table codée en dur : les décisions sont dérivées d'un calcul d'espérance (récursif, avec mémoïsation) sur l'état exact de la main.
- `counter.py` est indépendant de `strategy.py` mais vient moduler ses décisions (déviations) et piloter `bankroll.py` (spread de mise).

## 6. Prochaines étapes

1. Finaliser ce pré-rapport (validation des choix ci-dessus).
2. Implémenter `rules.py`, `card.py`, `shoe.py`, `hand.py` (fondations, testables immédiatement).
3. Implémenter le calcul d'EV récursif pour générer les tables de `strategy.py`.
4. Implémenter `counter.py` (Hi-Lo pour commencer) et son intégration aux décisions.
5. Implémenter `simulator.py` + `cli.py` pour lancer des simulations Monte Carlo en conditions réelles de règles (section 3).
6. Tests unitaires ciblés sur les fonctions critiques au fil de l'implémentation.
