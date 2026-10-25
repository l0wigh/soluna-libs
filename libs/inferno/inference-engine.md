# Écrire un moteur d'inférence : guide complet

Ce document explique comment fonctionne un petit moteur d'inférence à base de règles (chaînage arrière, à la Prolog), donne son pseudo-code indépendant du langage, puis l'implémentation Soluna complète en annexe. Il est pensé pour être réécrit à la main, brique par brique, et éventuellement porté dans un autre langage.

## Sommaire

1. Ce que fait un moteur d'inférence
2. Les idées clés
3. Les structures de données
4. Les variables
5. Substitutions : `walk` et `resolve`
6. L'unification
7. Le moteur : chaînage arrière et backtracking
8. L'API de requête
9. La couche utilisateur (DSL et état global)
10. Trace complète à la main
11. Ordre de réécriture conseillé et tests
12. Conseils pour porter dans un autre langage
13. Limites connues et pistes d'amélioration
14. Annexe : implémentation Soluna complète

---

## 1. Ce que fait un moteur d'inférence

Un moteur d'inférence reçoit :

- des **faits** : ce qu'on sait (`alice est parent de bob`) ;
- des **règles** : comment en déduire d'autres (`si X est parent de Y et Y parent de Z, alors X est grand-parent de Z`) ;
- une **question** : ce qu'on veut savoir (`qui sont les grands-parents de dave ?`).

Il répond en cherchant, parmi les faits et les règles, tout ce qui permet de **prouver** la question, et en indiquant quelles valeurs prennent les inconnues.

Il ne connaît rien du domaine. Généalogie, médecine, graphes : pour lui, ce ne sont que des motifs à rapprocher. C'est ce qui le rend réutilisable.

Exemple de ce que la lib permet :

```lisp
(facts
  (symptom p1 fievre) (symptom p1 toux)
  (symptom p2 fievre)
  (symptom p3 fievre) (symptom p3 toux))

(rule (maladie P grippe) (symptom P fievre) (symptom P toux))

(ask (symptom X toux))     ; => X = p1, X = p3
(ask (maladie X grippe))   ; => X = p1, X = p3
(ask (maladie p3 X))       ; => X = grippe
(ask (maladie p2 X))       ; => aucune preuve
```

Une seule règle répond à trois questions de sens différents. C'est la propriété centrale : **la direction de lecture est décidée par la position de l'inconnue dans la question**, pas par la règle.

---

## 2. Les idées clés

Le moteur tient sur cinq idées. Tout le reste est de la plomberie.

### 2.1 Un fait est une règle sans conditions

Une règle s'écrit `tête :- but1, but2, ...` (« pour prouver la tête, prouve tous les buts »). Un fait est une règle dont la liste de buts est vide : il est prouvé sans condition. Résultat : la base n'est qu'**une liste de règles**, et le moteur n'a qu'un seul chemin de code.

### 2.2 Une variable est un trou à remplir

`X` dans `(parent X bob)` signifie « n'importe quoi, à trouver ». Les constantes (`alice`, `42`) sont des valeurs fixes.

### 2.3 L'unification est la « comparaison intelligente »

Unifier deux termes, c'est chercher des valeurs pour leurs variables qui les rendent identiques.

```
(parent X bob)  et  (parent alice Y)   →   X = alice, Y = bob
(parent X bob)  et  (parent alice eve) →   échec (bob ≠ eve)
```

### 2.4 Une substitution mémorise ce qu'on a appris

Au fil de la preuve, on accumule des liens `variable → valeur`. Cette liste s'appelle une **substitution**. On la transmet d'un but au suivant : c'est ainsi que le `Y` trouvé dans un but contraint le même `Y` dans le but suivant.

### 2.5 Le backtracking est un effet de bord des listes de résultats

Pour prouver un but, on essaie **toutes** les règles. Chaque essai produit zéro, une ou plusieurs substitutions. On les collecte toutes. Si un essai échoue, il produit simplement une liste vide et on passe au suivant : pas de mécanisme de retour arrière explicite, la structure de boucles imbriquées suffit.

---

## 3. Les structures de données

```
Terme         = Constante                      ex: alice, 42
              | Variable                       ex: X
              | Liste de Termes                ex: [parent, alice, X]

Substitution  = liste de couples (Variable, Terme)
                ex: [(X, bob), (Y, X)]

Règle         = (tête, corps)
                tête  : un Terme
                corps : liste de Termes (les buts à prouver)

Fait          = Règle dont le corps est vide
Base          = liste de Règles
```

Un « atome logique » comme `(parent alice bob)` est simplement une liste de termes dont le premier élément est le nom du prédicat. Le moteur ne donne aucun sens particulier à ce premier élément, sauf pour deux mots réservés : `not` et `equal` (section 7).

---

## 4. Les variables

### 4.1 Reconnaître une variable

Convention de Prolog : un symbole dont la **première lettre est une majuscule** est une variable.

```
fonction est_variable(x):
    retourne x est un symbole ET sa première lettre est une majuscule
```

Dans un autre langage, une classe `Var(nom)` est plus propre. La convention de la majuscule sert seulement à écrire confortablement les règles.

### 4.2 Lister les variables d'un terme

Utile pour savoir quelles variables afficher dans la réponse à une requête.

```
fonction variables_de(terme, acc):
    si est_variable(terme):
        retourne acc + [terme]   si terme n'est pas déjà dans acc
        retourne acc             sinon
    si terme est une liste:
        pour chaque élément e:
            acc = variables_de(e, acc)
    retourne acc
```

### 4.3 Renommer : des variables fraîches à chaque usage

Voici le point le plus subtil du moteur. Considérons :

```
(rule (ancestor X Z) (parent X Y) (ancestor Y Z))
```

Cette règle est récursive : pour prouver `(ancestor a R)`, on doit prouver `(ancestor b Z)`, ce qui réutilise **la même règle**, donc les mêmes noms `X`, `Y`, `Z`. Si on ne faisait rien, le `Y` du premier niveau serait confondu avec le `Y` du second, et la preuve donnerait n'importe quoi.

Solution : **chaque fois qu'on utilise une règle, on renomme toutes ses variables** avec un suffixe unique.

```
fonction renommer(terme, suffixe):
    si est_variable(terme):  retourne Variable(nom(terme) + suffixe)
    si terme est une liste:  retourne [renommer(e, suffixe) pour chaque e]
    sinon:                   retourne terme
```

Premier usage : `X#1`, `Y#1`, `Z#1`. Deuxième usage : `X#2`, `Y#2`, `Z#2`. Plus aucun risque de collision.

Le suffixe doit être unique à chaque usage. Deux approches :

- **Compteur global** (simple) : `compteur = compteur + 1`, suffixe `"#" + compteur`.
- **Chemin de preuve** (pur, sans état) : le suffixe encode la position dans l'arbre de preuve, par exemple `#.0/1.2/0`. C'est ce que fait l'implémentation Soluna, parce qu'elle voulait éviter tout état caché dans le moteur.

---

## 5. Substitutions : `walk` et `resolve`

Une substitution est une liste de couples `(variable, valeur)`. La valeur peut elle-même être une variable (`Y` liée à `X`, `X` liée à `bob`). Il faut donc pouvoir suivre les chaînes.

### 5.1 `walk` : suivre les liens d'une variable

```
fonction walk(x, s):
    si est_variable(x):
        si x est liée dans s à une valeur v:   retourne walk(v, s)
        sinon:                                 retourne x
    retourne x
```

Exemple avec `s = [(Y, X), (X, bob)]` : `walk(Y, s)` suit `Y → X → bob` et retourne `bob`.

`walk` ne regarde que la **tête** du terme : si c'est une liste, elle est renvoyée telle quelle, même si elle contient des variables liées à l'intérieur.

### 5.2 `resolve` : tout remplacer

```
fonction resolve(x, s):
    w = walk(x, s)
    si w est une liste:  retourne [resolve(e, s) pour chaque e de w]
    retourne w
```

`resolve` descend dans les listes. On l'utilise à la toute fin pour fabriquer la réponse affichée.

Règle pratique : `walk` pendant le calcul (on a juste besoin de la tête), `resolve` pour produire un résultat lisible.

---

## 6. L'unification

C'est le cœur du moteur.

```
fonction unifier(a, b, s):          // retourne ÉCHEC, ou une substitution étendue
    x = walk(a, s)
    y = walk(b, s)
    si x == y:                      retourne s
    si est_variable(x):             retourne s + [(x, y)]
    si est_variable(y):             retourne s + [(y, x)]
    si x et y sont des listes de même longueur:
        pour chaque paire (xi, yi):
            s = unifier(xi, yi, s)
            si s == ÉCHEC:  retourne ÉCHEC
        retourne s
    retourne ÉCHEC
```

### 6.1 Lecture cas par cas

1. **Déjà égaux** : rien à faire, on rend la substitution inchangée.
2. **Une variable libre d'un côté** : on la lie à l'autre côté.
3. **Deux listes** : on unifie élément par élément, en **transmettant** la substitution de l'un à l'autre. C'est ce qui garantit la cohérence : `(f X X)` contre `(f a b)` échoue, car après `X = a`, le second `X` vaut `a` et ne peut pas s'unifier avec `b`.
4. **Sinon** : deux constantes différentes, ou une liste contre une constante. Échec.

### 6.2 Exemples à tester

| a | b | résultat |
|---|---|----------|
| `[f, X, b]` | `[f, a, Y]` | `X = a, Y = b` |
| `[f, X, X]` | `[f, a, b]` | échec |
| `[f, X, X]` | `[f, a, a]` | `X = a` |
| `[f, X]` | `[f, Y]` | `X = Y` |
| `[f, a]` | `[g, a]` | échec |
| `[f, a]` | `[f, a, b]` | échec (longueurs différentes) |
| `a` | `a` | succès, substitution vide |

### 6.3 Piège : « vide » ne veut pas dire « échec »

Une substitution **vide** est un **succès** : « ça s'unifie, sans rien avoir à lier » (cas `a` contre `a`). Dans un langage où la liste vide est considérée comme fausse (c'est le cas de Soluna), on ne peut pas utiliser `[]` à la fois pour « échec » et pour « succès sans liaison ».

Deux solutions :

- Langage avec `null` / `None` / `Option` : `ÉCHEC = null`. `[]` reste un succès.
- Langage sans cela : renvoyer une **liste de résultats**. `[s]` pour un succès, `[]` pour un échec. C'est ce que fait l'implémentation Soluna, et ça s'intègre bien avec le reste du moteur, qui manipule déjà des listes de substitutions.

### 6.4 L'occurs check (volontairement omis)

`unifier(X, [f, X])` lie `X` à une structure qui contient `X` : une structure infinie. Un moteur robuste vérifie, avant de lier une variable à un terme, que la variable n'apparaît pas dans ce terme. Prolog ne le fait pas par défaut, pour des raisons de performance. Pour apprendre, on peut l'ignorer.

---

## 7. Le moteur : chaînage arrière et backtracking

### 7.1 Principe

*Pour prouver un but, on essaie chaque règle de la base. Si la tête de la règle s'unifie avec le but, il faut alors prouver le corps de la règle, but après but, en transmettant la substitution.*

On dit **chaînage arrière** parce qu'on part de la conclusion voulue (le but) et qu'on remonte vers les faits qui la justifient. L'inverse, le chaînage avant, part des faits et applique les règles pour en dériver de nouveaux jusqu'à saturation.

### 7.2 Résoudre une conjonction de buts

```
fonction résoudre_buts(buts, s, chemin, base, n):
    // retourne la liste de TOUTES les substitutions qui satisfont tous les buts
    si buts est vide:  retourne [s]
    résultats = []
    pour chaque s2 dans résoudre_but(buts[0], s, chemin + "." + n, base):
        résultats += résoudre_buts(buts[1:], s2, chemin, base, n + 1)
    retourne résultats
```

Le premier but peut avoir **plusieurs** solutions. Pour chacune, on tente de prouver le reste des buts. C'est exactement le backtracking : si le but suivant échoue pour une solution, sa liste de résultats est vide, et on passe à la solution suivante du premier but.

Exemple : prouver `(parent X Y), (parent Y Z)`.

```
solutions de (parent X Y) :  [X=a,Y=b]  [X=b,Y=c]  [X=c,Y=d]
pour X=a,Y=b : prouver (parent b Z)  →  Z=c       ✓ une solution
pour X=b,Y=c : prouver (parent c Z)  →  Z=d       ✓ une solution
pour X=c,Y=d : prouver (parent d Z)  →  aucune    ✗ liste vide, on passe
```

### 7.3 Résoudre un seul but

```
fonction résoudre_but(but, s, chemin, base):
    si longueur(chemin) > MAX:           retourne []        // garde-fou
    si but = (not g):
        retourne [s]  si résoudre_but(g, s, chemin, base) est vide
        retourne []   sinon
    si but = (equal a b):
        retourne [unifier(a, b, s)]  si pas ÉCHEC, sinon []
    sinon:
        retourne résoudre_règles(but, s, chemin, base, base, 0)
```

Trois cas :

- **`(not g)`** : négation **par échec**. On essaie de prouver `g`. Si on n'y arrive pas, `not g` est considéré vrai. Attention : ce n'est pas une vraie négation logique, c'est « je ne peux pas le prouver ». C'est l'**hypothèse du monde clos** : ce qui n'est pas dans la base ni déductible est supposé faux.
- **`(equal a b)`** : prédicat intégré, réussit si `a` et `b` s'unifient. Utile pour `(not (equal X Y))` (« X est différent de Y »).
- **Tout le reste** : un but ordinaire, qu'on cherche à prouver avec les règles de la base.

Le garde-fou `MAX` coupe les preuves trop profondes. Sans lui, une règle récursive à gauche comme `(ancestor X Z) :- (ancestor X Y), (parent Y Z)` bouclerait à l'infini.

### 7.4 Essayer toutes les règles

```
fonction résoudre_règles(but, s, chemin, base, règles, k):
    résultats = []
    pour chaque règle r (numérotée k) de règles:
        p = chemin + "/" + k
        r = renommer(r, p)                                 // variables fraîches
        pour chaque s2 dans unifier(but, tête(r), s):      // vide si ÉCHEC
            résultats += résoudre_buts(corps(r), s2, p, base, 0)
    retourne résultats
```

Pour chaque règle :

1. on la **renomme** (voir section 4.3) ;
2. on **unifie** le but avec sa tête ;
3. si ça marche, on prouve son **corps** avec la substitution obtenue.

Pour un fait, le corps est vide : `résoudre_buts([], s2, ...)` retourne directement `[s2]`. Aucun traitement spécial.

### 7.5 Le rôle du « chemin »

Le paramètre `chemin` est une chaîne qui décrit la position dans l'arbre de preuve. On l'utilise uniquement pour fabriquer des suffixes de renommage uniques, sans compteur global. Il est allongé à deux endroits :

- `"." + n` quand on passe d'un but au suivant dans un corps ;
- `"/" + k` quand on choisit la k-ième règle.

Deux utilisations différentes d'une même règle sont à des positions différentes dans l'arbre, donc ont des chemins différents, donc des variables différentes.

---

## 8. L'API de requête

```
fonction query(base, but):
    vars = variables_de(but, [])
    pour chaque s dans résoudre_buts([but], [], "#", base, 0):
        émettre [(v, resolve(v, s)) pour chaque v de vars]

fonction holds(base, but):
    retourne résoudre_buts([but], [], "#", base, 0) n'est pas vide
```

- `query` renvoie, pour chaque solution, la valeur finale de chaque variable de la question. On filtre ainsi toutes les variables internes (renommées) qui n'intéressent pas l'utilisateur.
- `holds` répond juste par oui ou non.

Format des réponses :

```
(ask (maladie X grippe))   →  (((X p1)) ((X p3)))     // deux solutions
(ask (maladie p3 X))       →  (((X grippe)))          // une solution
(ask (maladie p2 X))       →  ()                      // aucune solution
(ask? (parent a b))        →  true / false
```

Une requête **sans variable** qui est prouvée donne une liste contenant une solution vide `(())`, d'où l'intérêt de `holds` pour les questions fermées.

---

## 9. La couche utilisateur

### 9.1 L'état global

Le moteur est **pur** : il reçoit la base en argument et ne modifie rien. La seule variable mutable est la base courante, au niveau du DSL.

```
BASE = []                                   // le seul état mutable

fonction add(règle):   BASE = BASE + [règle];  retourne règle
fonction fact(motif):  add((motif, []))
fonction rule(tête, buts...):  add((tête, buts))
fonction ask(but):     query(BASE, but)
fonction ask?(but):    holds(BASE, but)
```

Avantages de cette séparation : le moteur est facile à tester (on lui passe des bases écrites à la main), et on peut proposer des fonctions comme `suppose` sans toucher au moteur.

### 9.2 Hypothèses temporaires

```
fonction suppose(corps):
    sauvegarde = BASE
    résultat = exécuter(corps)
    BASE = sauvegarde
    retourne résultat
```

Permet les questions « et si... » : on ajoute temporairement des faits, on pose une question, puis on revient à l'état précédent.

```lisp
(suppose
  (fact (symptom p2 toux))
  (ask (maladie p2 X)))      ; (((X grippe)))
; la base est restaurée ici
```

Limite : si le corps échoue à mi-chemin, la base n'est pas restaurée. Dans un langage avec `try/finally`, c'est facile à corriger.

### 9.3 Pourquoi des macros en Soluna

Dans `(fact (parent alice bob))`, l'argument ne doit pas être évalué : `parent` n'est pas une fonction. Les macros reçoivent leurs arguments non évalués et ajoutent le `quote` à notre place. Dans un autre langage, `fact`, `rule` et `ask` sont de simples fonctions qui reçoivent des données (listes, tableaux, objets), et la question de l'évaluation ne se pose pas.

---

## 10. Trace complète à la main

**Base**

```
1. (parent a b)
2. (parent b c)
3. (ancestor X Y) :- (parent X Y)
4. (ancestor X Z) :- (parent X Y), (ancestor Y Z)
```

**Requête** : `(ancestor a R)`

### Essai de la règle 3

```
Renommage : (ancestor X#1 Y#1) :- (parent X#1 Y#1)

unifier((ancestor a R), (ancestor X#1 Y#1))
   a  ~ X#1   →  X#1 = a
   R  ~ Y#1   →  R = Y#1
   s = [(X#1,a), (R,Y#1)]

Prouver le corps : (parent X#1 Y#1), c'est-à-dire (parent a Y#1)
   règle 1 : (parent a b)  → Y#1 = b       ✓
   règle 2 : (parent b c)  → a ≠ b         ✗
   
Solution : R = Y#1 = b
```

### Essai de la règle 4

```
Renommage : (ancestor X#2 Z#2) :- (parent X#2 Y#2), (ancestor Y#2 Z#2)

unifier((ancestor a R), (ancestor X#2 Z#2))
   X#2 = a,  R = Z#2

Prouver le corps, but par but :

  but 1 : (parent a Y#2)
     règle 1 : Y#2 = b   ✓
     règle 2 : échec

  but 2 (avec Y#2 = b) : (ancestor b Z#2)
     → appel récursif, nouvelle utilisation des règles 3 et 4
       avec des variables fraîches (#3, #4...)

     règle 3 renommée : (ancestor X#3 Y#3) :- (parent X#3 Y#3)
        X#3 = b, Z#2 = Y#3
        prouver (parent b Y#3) :  règle 2 donne Y#3 = c  ✓
        solution : Z#2 = c

     règle 4 renommée : (ancestor X#4 Z#4) :- (parent X#4 Y#4), (ancestor Y#4 Z#4)
        X#4 = b, Z#2 = Z#4
        prouver (parent b Y#4) → Y#4 = c
        prouver (ancestor c Z#4)
           règle 3 : (parent c ?) → aucun fait → ✗
           règle 4 : (parent c ?) → aucun fait → ✗
        aucune solution

Solution : R = Z#2 = c
```

### Résultat

```
R = b
R = c
```

Les variables `X#1`, `Y#2`... n'apparaissent jamais dans la réponse : `query` ne retient que `R`, la seule variable de la question.

---

## 11. Ordre de réécriture conseillé et tests

Chaque étape ne dépend que des précédentes et se teste seule.

### Étape 1 : variables et termes

Écrire `est_variable`, `variables_de`, `renommer`.

```
est_variable(X)              → vrai
est_variable(alice)          → faux
variables_de([f, X, [g, Y, X]], [])  → [X, Y]
renommer([f, X, a], "#1")    → [f, X#1, a]
```

### Étape 2 : substitutions

Écrire `walk` et `resolve`.

```
walk(X, [(X, bob)])                          → bob
walk(Y, [(Y, X), (X, bob)])                  → bob
walk(Z, [(X, bob)])                          → Z
resolve([f, X, Y], [(X, a), (Y, X)])         → [f, a, a]
```

### Étape 3 : unification

C'est le morceau à bien tester, avec le tableau de la section 6.2. Ajoute les cas à plusieurs variables et les chaînes de liaisons.

```
unifier([f, X, b], [f, a, Y], [])           → X = a, Y = b
unifier([f, X, X], [f, a, b], [])           → échec
unifier([f, X, Y], [f, Y, a], [])           → X = Y, Y = a (donc X vaut a)
```

### Étape 4 : moteur avec faits seulement

Une base sans règles à corps non vide.

```
base = [(parent a b), (parent b c)]
query(base, (parent a X))     → X = b
query(base, (parent X c))     → X = b
query(base, (parent X Y))     → (X=a,Y=b), (X=b,Y=c)
```

### Étape 5 : règles et renommage

Ajoute `grandparent`, puis `ancestor` (récursive). Vérifie la terminaison.

### Étape 6 : `not` et `equal`

```
(rule (sibling X Y) (parent P X) (parent P Y) (not (equal X Y)))
```

Sans le `not (equal X Y)`, chaque personne serait sa propre sœur.

### Étape 7 : couche utilisateur

`fact`, `rule`, `ask`, `suppose`. À ce stade, le plus dur est fait.

---

## 12. Conseils pour porter dans un autre langage

- **Variables fraîches** : un compteur global (`compteur++`) est plus simple que le chemin de preuve. L'implémentation Soluna l'évite uniquement pour ne pas avoir d'état caché dans le moteur.
- **Résultats paresseux** : mes listes de solutions calculent *tout*. Avec des générateurs (`yield` en Python, itérateurs en Rust ou JS, `Seq` en F#), tu obtiens les solutions à la demande et tu peux t'arrêter à la première.
- **Garde-fou** : un compteur de profondeur explicite est plus lisible qu'une longueur de chaîne.
- **Représentation des variables** : une classe `Var(nom)` est plus propre qu'une convention de majuscule.
- **Échec d'unification** : avec `null`, `None` ou `Option`, pas besoin d'envelopper la substitution dans une liste.
- **Occurs check** : à ajouter pour la robustesse (section 6.4).
- **Substitutions** : une liste d'associations suffit pour apprendre. Une table de hachage persistante (ou un dictionnaire copié à chaque liaison) est plus rapide.
- **Indexation des règles** : pour de grosses bases, regrouper les règles par nom de prédicat évite de toutes les essayer à chaque but.

Esquisse en Python, pour fixer les idées :

```python
def unify(a, b, s):
    a, b = walk(a, s), walk(b, s)
    if a == b: return s
    if is_var(a): return {**s, a: b}
    if is_var(b): return {**s, b: a}
    if isinstance(a, list) and isinstance(b, list) and len(a) == len(b):
        for x, y in zip(a, b):
            s = unify(x, y, s)
            if s is None: return None
        return s
    return None

def solve(goals, s, base, depth=0):
    if not goals:
        yield s
        return
    if depth > MAX_DEPTH: return
    first, rest = goals[0], goals[1:]
    for s2 in solve_one(first, s, base, depth):
        yield from solve(rest, s2, base, depth)
```

Avec les générateurs, le backtracking devient presque gratuit.

---

## 13. Limites connues et pistes d'amélioration

| Sujet | État actuel | Amélioration |
|-------|-------------|--------------|
| Doublons | Deux preuves menant à la même réponse la donnent deux fois | Dédupliquer à la fin de `query`, ou dans `add` |
| Variable anonyme `_` | Absente : il faut inventer des noms (`Ignore1`) | Chaque `_` devient une variable distincte à la lecture |
| Occurs check | Absent | Vérifier avant de lier une variable |
| Récursion à gauche | Coupée par `MAX`, sans erreur claire | Tabling, ou détection de cycles |
| Négation | Par échec, hypothèse du monde clos | Documenter, ou ajouter une logique à trois valeurs |
| Explication | Non exposée | Renvoyer l'arbre de preuve avec chaque solution |
| `suppose` | Ne restaure pas la base en cas d'erreur | `try` / `finally` |
| Plusieurs bases | Une seule base globale | Repasser `kb` en paramètre des macros |
| Performance | Toutes les règles essayées à chaque but | Index par prédicat |
| Chaînage avant | Absent | Boucle à point fixe sur les règles |

### Piste : ajouter l'explication

Comme le moteur sait déjà *comment* il prouve chaque but, on peut renvoyer, avec chaque solution, l'arbre de preuve :

```
résoudre_but retourne des couples (substitution, preuve)
preuve = (but_résolu, [sous-preuves])
```

Pour `(maladie p3 grippe)` on obtiendrait :

```
(maladie p3 grippe)
├── (symptom p3 fievre)   [fait]
└── (symptom p3 toux)     [fait]
```

C'est la fonction qui distingue un moteur d'inférence d'un système expert : pouvoir répondre à « pourquoi ? ».

---

## 14. Annexe : implémentation Soluna complète

Version écrite pour Soluna 0.10.4 (symboles comparables avec `=`, `string->symbol`, `set!`). Les constantes ne doivent pas commencer par une majuscule.

```lisp
;;; ============ Utilitaires ============

(function append-map (f lst)
  (reduce (lambda (acc x) (concat acc (f x))) (list) lst))

(function pair? (x) (= (type x) "List"))

;; = protégé : types différents => false (au lieu d'une erreur)
(function same? (a b)
  (if (= (type a) (type b))
      (if (= (type a) "Nil") true (= a b))
      false))

;;; ============ Variables ============
;; Variable = symbole dont la première lettre est une majuscule

(function var? (x)
  (if (= (type x) "Symbol")
      (let ((c (get 0 (symbol->string x))))
        (if (>= c "A") (<= c "Z") false))
      false))

(function vars-of (x acc)
  (if (var? x)
      (if (null (filter (lambda (y) (= x y)) acc)) (concat acc (list x)) acc)
      (if (pair? x) (vars-of (rst x) (vars-of (fst x) acc)) acc)))

;; Variables fraîches : X devient X#/0.1 (suffixe = chemin de preuve)
(function rename (x suffix)
  (if (var? x)
      (string->symbol (concat (symbol->string x) suffix))
      (if (pair? x)
          (cons (rename (fst x) suffix) (rename (rst x) suffix))
          x)))

;;; ============ Unification ============
;; Substitution = liste de (variable valeur).
;; unify renvoie (list s) si succès, (list) si échec : une substitution
;; vide () serait sinon confondue avec "faux".

(function lookup (v s)
  (if (null s)
      (list)
      (if (= v (fst (fst s))) (list (get 1 (fst s))) (lookup v (rst s)))))

(function walk (x s)
  (if (var? x)
      (let ((b (lookup x s)))
        (if (null b) x (walk (fst b) s)))
      x))

(function resolve (x s)
  (let ((w (walk x s)))
    (if (pair? w)
        (cons (resolve (fst w) s) (resolve (rst w) s))
        w)))

(function unify (a b s)
  (let* ((x (walk a s))
         (y (walk b s)))
    (case
      ((same? x y) (list s))
      ((var? x) (list (cons (list x y) s)))
      ((var? y) (list (cons (list y x) s)))
      ((if (pair? x) (pair? y) false)
        (append-map (lambda (s2) (unify (rst x) (rst y) s2))
                    (unify (fst x) (fst y) s)))
      (default (list)))))

;;; ============ Moteur pur (chaînage arrière) ============
;; kb = liste de règles ; règle = (tête but1 but2 ...) ; fait = (tête)

(defvar max-path 400)   ; garde-fou : longueur max du chemin de preuve

(function solve-goals (goals s path kb n)
  (if (null goals)
      (list s)
      (append-map
        (lambda (s2) (solve-goals (rst goals) s2 path kb (+ n 1)))
        (solve-goal (fst goals) s (concat path "." (str n)) kb))))

(function solve-goal (goal s path kb)
  (case
    ((> (length path) max-path) (list))
    ((same? (fst goal) 'not)
      (if (null (solve-goal (get 1 goal) s path kb)) (list s) (list)))
    ((same? (fst goal) 'equal)
      (unify (get 1 goal) (get 2 goal) s))
    (default (solve-rules goal s path kb kb 0))))

(function solve-rules (goal s path kb rules k)
  (if (null rules)
      (list)
      (let* ((p    (concat path "/" (str k)))
             (rule (rename (fst rules) p)))
        (concat
          (append-map (lambda (s2) (solve-goals (rst rule) s2 p kb 0))
                      (unify goal (fst rule) s))
          (solve-rules goal s path kb (rst rules) (+ k 1))))))

(function query (kb goal)
  (let ((vars (vars-of goal (list))))
    (map (lambda (s) (map (lambda (v) (list v (resolve v s))) vars))
         (solve-goals (list goal) (list) "#" kb 0))))

(function holds? (kb goal)
  (not (null (solve-goals (list goal) (list) "#" kb 0))))

;;; ============ Base globale (seul état mutable) ============

(defvar *base* (list))

;; Ajoute une règle à la base et la renvoie (set! modifie le global)
(function add! (r)
  (do (set! *base* (concat *base* (list r)))
      r))

(function reset! ()
  (set! *base* (list)))

;;; ============ DSL ============

;; (fact (parent alice bob))
(defmacro fact (pattern)
  `(add! (list (quote ,pattern))))

;; (rule (ancestor X Z) (parent X Y) (ancestor Y Z))
(defmacro rule (head &rest body)
  `(add! (quote ,(cons head body))))

;; (facts (parent alice bob) (parent bob carol) ...)
(defmacro facts (&rest patterns)
  `(do ,@(map (lambda (p) (list 'fact p)) patterns)))

(defmacro ask (goal)
  `(query *base* (quote ,goal)))

(defmacro ask? (goal)
  `(holds? *base* (quote ,goal)))

;; Hypothèse : exécute le corps, puis restaure la base d'origine
(defmacro suppose (&rest body)
  `(let ((%saved *base*))
     (let* ((%result (do ,@body))
            (%restore (set! *base* %saved)))
       %result)))
```

### Exemple d'utilisation

```lisp
(facts
  (symptom p1 fievre) (symptom p1 toux)
  (symptom p2 fievre)
  (symptom p3 fievre) (symptom p3 toux))

(rule (maladie P grippe) (symptom P fievre) (symptom P toux))
(rule (a-consulter P) (maladie P grippe))
(rule (isoler P) (a-consulter P) (symptom P toux))

(writeln (ask (symptom X toux)))     ; (((X p1)) ((X p3)))
(writeln (ask (maladie X grippe)))   ; (((X p1)) ((X p3)))
(writeln (ask (maladie p3 X)))       ; (((X grippe)))
(writeln (ask (maladie p2 X)))       ; Nil
(writeln (ask (isoler X)))           ; (((X p1)) ((X p3)))

(writeln (suppose
  (fact (symptom p2 toux))
  (ask (maladie p2 X))))             ; (((X grippe)))

(writeln (ask? (symptom p2 toux)))   ; false : base restaurée
```

### Mémo : correspondance pseudo-code / Soluna

| Pseudo-code | Soluna |
|-------------|--------|
| `est_variable` | `var?` |
| `variables_de` | `vars-of` |
| `renommer` | `rename` |
| `walk`, `resolve` | `walk`, `resolve` |
| `unifier` | `unify` (renvoie `(list s)` ou `(list)`) |
| `résoudre_buts` | `solve-goals` |
| `résoudre_but` | `solve-goal` |
| `résoudre_règles` | `solve-rules` |
| `query`, `holds` | `query`, `holds?` |
| `BASE`, `add` | `*base*`, `add!` |
| `fact`, `rule`, `ask`, `suppose` | macros du même nom |
