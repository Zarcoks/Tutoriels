# Guide Python → Go
## Apprendre Go quand on connaît Python

---

## Avant de commencer : Pourquoi Go existe-t-il ?

Python est un langage fantastique pour prototyper, écrire des scripts, faire de la data science ou du web. Mais il a des limites connues :

- **Lent à l'exécution** : le GIL (Global Interpreter Lock) empêche le vrai parallélisme
- **Typage dynamique** : les bugs de type ne se découvrent qu'à l'exécution
- **Pas fait pour les serveurs haute performance** : gérer 100 000 connexions simultanées en Python demande des contorsions (asyncio, gevent...)
- **Déploiement lourd** : dépendances, environnements virtuels, interpréteur à installer

Go a été conçu par Google en 2009 précisément pour répondre à ces problèmes sur des systèmes à grande échelle (serveurs, outils CLI, infrastructure). Ses caractéristiques clés :

- **Compilé en binaire natif** : une seule exécutable, aucune dépendance à installer
- **Typage statique** : les erreurs de type sont détectées avant même de lancer le programme
- **Concurrence native et légère** : les goroutines permettent de gérer des millions de tâches simultanées facilement
- **Syntaxe minimaliste** : le langage tient en peu de mots-clés, on apprend vite

**Exemples de logiciels écrits en Go** : Docker, Kubernetes, Terraform, Prometheus, Caddy.

---
---

## MODULE 1 — Pas d'héritage : Structs et Interfaces

### 1.1 — Théorie

#### En Python : classes et héritage

En Python, on organise les données et comportements dans des **classes**. L'héritage permet à une classe enfant de réutiliser le code d'une classe parente :

```python
class Animal:
    def __init__(self, nom):
        self.nom = nom

    def parler(self):
        raise NotImplementedError

class Chien(Animal):  # Chien hérite de Animal
    def parler(self):
        return f"{self.nom} dit Ouaf"

class Chat(Animal):
    def parler(self):
        return f"{self.nom} dit Miaou"

animaux = [Chien("Rex"), Chat("Whiskers")]
for a in animaux:
    print(a.parler())
```

Duck typing Python : si un objet a la méthode `parler()`, on peut l'utiliser comme un `Animal`, même sans héritage formel.

#### En Go : séparation des données et des comportements

Go **n'a pas de classes, pas d'héritage**. À la place, deux outils distincts :

- **`struct`** : pour regrouper des données (équivalent d'un `dataclass` Python)
- **`interface`** : pour définir un comportement attendu (équivalent du duck typing Python, mais formalisé)

Une struct implémente une interface **implicitement** : si elle possède toutes les méthodes requises, c'est tout. Pas de mot-clé `implements`.

```go
// Une struct : que des données
type Chien struct {
    Nom string
}

// Une méthode rattachée à la struct (le "receiver")
// (c Chien) joue le rôle du "self" Python
func (c Chien) Parler() string {
    return c.Nom + " dit Ouaf"
}

type Chat struct {
    Nom string
}

func (c Chat) Parler() string {
    return c.Nom + " dit Miaou"
}

// L'interface : définit ce qu'on attend
type Animal interface {
    Parler() string
}

// Chien et Chat implémentent Animal automatiquement
// car ils ont tous les deux la méthode Parler() string
func main() {
    animaux := []Animal{Chien{"Rex"}, Chat{"Whiskers"}}
    for _, a := range animaux {
        fmt.Println(a.Parler())
    }
}
```

#### Points clés à retenir

| Python | Go |
|--------|-----|
| `class Animal:` | `type Animal interface { ... }` |
| `class Chien(Animal):` | `type Chien struct { ... }` + méthode avec receiver |
| Héritage explicite | Implémentation implicite d'interface |
| `self` | receiver `(c Chien)` |
| `__init__` | pas de constructeur magique, souvent une fonction `NewChien(...)` |

#### La composition à la place de l'héritage

Quand on veut "réutiliser" du code dans une struct, on l'**embarque** (embedding) :

```go
type Adresse struct {
    Rue  string
    Ville string
}

type Personne struct {
    Nom     string
    Adresse          // Adresse est embarquée : ses champs sont accessibles directement
}

p := Personne{
    Nom:     "Alice",
    Adresse: Adresse{Rue: "10 rue des Lilas", Ville: "Paris"},
}

fmt.Println(p.Ville) // Accès direct, comme si Ville était un champ de Personne
```

---

### 1.2 — Exercices

#### Exercice 1A — Modéliser des formes géométriques

On veut calculer l'aire de différentes formes géométriques.

**À faire :**

1. Crée une struct `Rectangle` avec les champs `Largeur` et `Hauteur` (type `float64`)
2. Crée une struct `Cercle` avec le champ `Rayon` (type `float64`)
3. Définis une interface `Forme` avec une seule méthode : `Aire() float64`
4. Implémente `Aire()` pour `Rectangle` et `Cercle`
5. Dans `main()`, crée une slice `[]Forme` contenant un rectangle et un cercle, et affiche l'aire de chacun

**Indices :**
- Aire d'un rectangle : `Largeur * Hauteur`
- Aire d'un cercle : `math.Pi * Rayon * Rayon` (pense à importer `"math"`)
- Pour afficher avec deux décimales : `fmt.Printf("Aire : %.2f\n", f.Aire())`

---

#### Exercice 1B — Composition avec embedding

On modélise un système d'employés.

**À faire :**

1. Crée une struct `Personne` avec les champs `Nom` et `Age` (int)
2. Crée une struct `Employe` qui **embarque** `Personne`, et ajoute un champ `Poste` (string)
3. Crée une struct `Manager` qui **embarque** `Employe`, et ajoute un champ `Equipe` ([]string — une liste de noms)
4. Définis une interface `Identifiable` avec une méthode `Identite() string`
5. Implémente `Identite()` sur `Personne` qui retourne `"Nom (Age ans)"`
6. Vérifie que `Employe` et `Manager` satisfont aussi `Identifiable` sans rien rajouter (grâce à l'embedding)
7. Dans `main()`, crée un `Manager` et affiche son `Identite()` et son `Equipe`

---

### 1.3 — Corrections

#### Correction 1A

```go
package main

import (
    "fmt"
    "math"
)

// --- Structs ---

type Rectangle struct {
    Largeur float64
    Hauteur float64
}

type Cercle struct {
    Rayon float64
}

// --- Interface ---

type Forme interface {
    Aire() float64
}

// --- Implémentations ---

// (r Rectangle) est le "receiver" : l'équivalent du self Python
func (r Rectangle) Aire() float64 {
    return r.Largeur * r.Hauteur
}

func (c Cercle) Aire() float64 {
    return math.Pi * c.Rayon * c.Rayon
}

// --- Main ---

func main() {
    formes := []Forme{
        Rectangle{Largeur: 5, Hauteur: 3},
        Cercle{Rayon: 4},
    }

    for _, f := range formes {
        fmt.Printf("Aire : %.2f\n", f.Aire())
    }
}
// Sortie :
// Aire : 15.00
// Aire : 50.27
```

**Points importants :**
- `Rectangle{Largeur: 5, Hauteur: 3}` : c'est l'initialisation d'une struct par champs nommés (recommandé)
- Le receiver `(r Rectangle)` ne modifie pas r, il lit juste ses champs. On utilise un receiver valeur (pas de `*`) car on ne modifie rien
- `[]Forme{...}` : une slice dont chaque élément satisfait l'interface `Forme` — Go accepte `Rectangle` et `Cercle` car ils ont tous les deux `Aire() float64`

---

#### Correction 1B

```go
package main

import (
    "fmt"
    "strings"
)

// --- Structs avec embedding ---

type Personne struct {
    Nom string
    Age int
}

type Employe struct {
    Personne        // Embedding : Employe "hérite" des champs et méthodes de Personne
    Poste   string
}

type Manager struct {
    Employe         // Embedding : Manager "hérite" de tout Employe
    Equipe  []string
}

// --- Interface ---

type Identifiable interface {
    Identite() string
}

// --- Implémentation sur Personne seulement ---
// Employe et Manager l'obtiennent automatiquement par embedding

func (p Personne) Identite() string {
    return fmt.Sprintf("%s (%d ans)", p.Nom, p.Age)
}

// --- Main ---

func main() {
    m := Manager{
        Employe: Employe{
            Personne: Personne{Nom: "Alice", Age: 35},
            Poste:    "Lead Engineer",
        },
        Equipe: []string{"Bob", "Claire", "David"},
    }

    // Accès direct aux champs embarqués (pas besoin de m.Employe.Personne.Nom)
    fmt.Println("Nom   :", m.Nom)
    fmt.Println("Poste :", m.Poste)
    fmt.Println("ID    :", m.Identite()) // Fonctionne car Personne implémente Identifiable

    // Manager satisfait Identifiable sans qu'on ait rien écrit
    var id Identifiable = m
    fmt.Println("Via interface :", id.Identite())

    fmt.Println("Équipe :", strings.Join(m.Equipe, ", "))
}
// Sortie :
// Nom   : Alice
// Poste : Lead Engineer
// ID    : Alice (35 ans)
// Via interface : Alice (35 ans)
// Équipe : Bob, Claire, David
```

**Points importants :**
- L'embedding n'est **pas** de l'héritage : c'est de la **composition**. `Manager` contient un `Employe`, qui contient une `Personne`. Go remonte la chaîne pour trouver les méthodes.
- On peut toujours accéder explicitement : `m.Employe.Personne.Nom` — l'accès court `m.Nom` est un raccourci
- Si `Manager` avait lui-même une méthode `Identite()`, elle aurait priorité sur celle de `Personne`

---
---

## MODULE 2 — Gestion d'erreurs explicite

### 2.1 — Théorie

#### En Python : les exceptions

Python utilise le mécanisme `try/except`. Une fonction peut lever une exception n'importe où, et si personne ne la capture, le programme plante :

```python
def diviser(a, b):
    if b == 0:
        raise ValueError("Division par zéro !")
    return a / b

try:
    result = diviser(10, 0)
except ValueError as e:
    print(f"Erreur : {e}")
```

L'inconvénient : rien dans la signature de `diviser` ne t'indique qu'elle peut échouer. Tu dois lire la doc ou le code source.

#### En Go : les erreurs sont des valeurs

Go **n'a pas d'exceptions**. Une fonction qui peut échouer retourne **deux valeurs** : le résultat et une erreur.

```go
func diviser(a, b float64) (float64, error) {
    if b == 0 {
        return 0, errors.New("division par zéro")
    }
    return a / b, nil  // nil = pas d'erreur (équivalent de None en Python)
}

result, err := diviser(10, 0)
if err != nil {
    fmt.Println("Erreur :", err)
    return
}
fmt.Println("Résultat :", result)
```

Le type `error` est une interface standard de Go :
```go
type error interface {
    Error() string
}
```

Toute valeur qui implémente `Error() string` est une erreur.

#### Le pattern `if err != nil`

C'est le pattern le plus courant en Go. Il revient à **chaque appel de fonction susceptible d'échouer** :

```go
// En Python, tu écris une fois try/except pour tout
// En Go, tu traites l'erreur à chaque étape
data, err := os.ReadFile("config.json")
if err != nil {
    return fmt.Errorf("impossible de lire le fichier : %w", err)
}

config, err := parseJSON(data)
if err != nil {
    return fmt.Errorf("JSON invalide : %w", err)
}
```

`fmt.Errorf("... : %w", err)` permet d'**envelopper** une erreur avec du contexte (le `%w` "wrappe" l'erreur originale, elle reste inspecatable).

#### Créer des erreurs personnalisées

```go
// Méthode simple (pour des erreurs statiques)
var ErrNotFound = errors.New("élément introuvable")

// Méthode avancée : erreur avec données (struct qui implémente error)
type ErrValidation struct {
    Champ   string
    Message string
}

func (e ErrValidation) Error() string {
    return fmt.Sprintf("validation du champ '%s' : %s", e.Champ, e.Message)
}

func validerAge(age int) error {
    if age < 0 {
        return ErrValidation{Champ: "age", Message: "doit être positif"}
    }
    return nil
}
```

#### `panic` et `recover` — l'exception de dernier recours

Go a bien un mécanisme proche des exceptions : `panic`. Mais son usage est réservé aux **erreurs irrécupérables** (bug de programmation, état incohérent), pas aux erreurs métier normales.

```go
func diviserStrict(a, b int) int {
    if b == 0 {
        panic("division par zéro : cas impossible selon la spec")
    }
    return a / b
}
```

`recover()` permet de "rattraper" un panic (dans un `defer`), mais c'est rare et déconseillé pour la logique métier. À éviter au début.

---

### 2.2 — Exercices

#### Exercice 2A — Convertir un code Python avec try/except

Voici une fonction Python qui lit un fichier de configuration (clé=valeur) et retourne un dictionnaire :

```python
def lire_config(chemin: str) -> dict:
    config = {}
    try:
        with open(chemin, "r") as f:
            for ligne in f:
                ligne = ligne.strip()
                if "=" not in ligne:
                    raise ValueError(f"Ligne invalide : '{ligne}'")
                cle, valeur = ligne.split("=", 1)
                config[cle.strip()] = valeur.strip()
    except FileNotFoundError:
        raise RuntimeError(f"Fichier '{chemin}' introuvable")
    return config
```

**À faire :**

1. Écris la version Go de cette fonction : `func lireConfig(chemin string) (map[string]string, error)`
2. Pour lire un fichier : utilise `os.ReadFile(chemin)` qui retourne `([]byte, error)`
3. Pour découper le contenu en lignes : utilise `strings.Split(string(contenu), "\n")`
4. Pour chercher `"="` : utilise `strings.Contains(ligne, "=")` et `strings.SplitN(ligne, "=", 2)`
5. Dans `main()`, appelle ta fonction avec un chemin inexistant et affiche l'erreur, puis avec un chemin valide (crée un petit fichier texte).

**Imports utiles :** `"os"`, `"strings"`, `"fmt"`, `"errors"`

---

#### Exercice 2B — Erreur personnalisée + chaîne d'erreurs

On simule un système de validation d'un formulaire d'inscription.

**À faire :**

1. Définis une struct `ErrChamp` avec les champs `Nom` (string) et `Raison` (string), qui implémente l'interface `error`
2. Écris une fonction `validerNom(nom string) error` :
   - Retourne une `ErrChamp` si le nom est vide
   - Retourne une `ErrChamp` si le nom fait moins de 2 caractères
   - Retourne `nil` sinon
3. Écris une fonction `validerEmail(email string) error` :
   - Retourne une `ErrChamp` si l'email ne contient pas `"@"`
   - Retourne `nil` sinon
4. Écris une fonction `inscrire(nom, email string) error` qui appelle les deux validations et, en cas d'erreur, l'enveloppe avec `fmt.Errorf("inscription impossible : %w", err)`
5. Dans `main()`, teste avec des données invalides et affiche les erreurs

---

### 2.3 — Corrections

#### Correction 2A

```go
package main

import (
    "errors"
    "fmt"
    "os"
    "strings"
)

func lireConfig(chemin string) (map[string]string, error) {
    // os.ReadFile retourne ([]byte, error) — on traite l'erreur immédiatement
    contenu, err := os.ReadFile(chemin)
    if err != nil {
        // On enveloppe l'erreur avec du contexte
        return nil, fmt.Errorf("fichier '%s' introuvable : %w", chemin, err)
    }

    config := make(map[string]string)

    lignes := strings.Split(string(contenu), "\n")
    for _, ligne := range lignes {
        ligne = strings.TrimSpace(ligne)
        if ligne == "" {
            continue // On ignore les lignes vides
        }
        if !strings.Contains(ligne, "=") {
            return nil, fmt.Errorf("ligne invalide : '%s'", ligne)
        }
        parties := strings.SplitN(ligne, "=", 2)
        cle := strings.TrimSpace(parties[0])
        valeur := strings.TrimSpace(parties[1])
        config[cle] = valeur
    }

    return config, nil
}

func main() {
    // Test avec fichier inexistant
    _, err := lireConfig("inexistant.txt")
    if err != nil {
        fmt.Println("Erreur :", err)
    }

    // Créer un fichier de test
    contenuTest := "host = localhost\nport = 5432\ndbname = mabase\n"
    os.WriteFile("config.txt", []byte(contenuTest), 0644)

    // Test avec fichier valide
    config, err := lireConfig("config.txt")
    if err != nil {
        fmt.Println("Erreur :", err)
        return
    }
    fmt.Println("Config chargée :")
    for cle, val := range config {
        fmt.Printf("  %s = %s\n", cle, val)
    }

    // Nettoyage
    os.Remove("config.txt")
}
// Sortie :
// Erreur : fichier 'inexistant.txt' introuvable : open inexistant.txt: no such file or directory
// Config chargée :
//   host = localhost
//   port = 5432
//   dbname = mabase
```

**Points importants :**
- `return nil, fmt.Errorf(...)` : on retourne la valeur zéro (`nil` pour une map) + l'erreur. Convention Go : si erreur ≠ nil, les autres valeurs de retour sont ignorées
- `%w` dans `fmt.Errorf` enveloppe l'erreur originale. On peut ensuite utiliser `errors.Is()` ou `errors.As()` pour l'inspecter
- `make(map[string]string)` : initialise une map vide (une map non-initialisée est `nil`, écrire dedans causerait un panic)

---

#### Correction 2B

```go
package main

import (
    "errors"
    "fmt"
)

// --- Erreur personnalisée ---

type ErrChamp struct {
    Nom    string
    Raison string
}

// ErrChamp implémente l'interface error
func (e ErrChamp) Error() string {
    return fmt.Sprintf("champ '%s' invalide : %s", e.Nom, e.Raison)
}

// --- Fonctions de validation ---

func validerNom(nom string) error {
    if nom == "" {
        return ErrChamp{Nom: "nom", Raison: "ne peut pas être vide"}
    }
    if len(nom) < 2 {
        return ErrChamp{Nom: "nom", Raison: "doit contenir au moins 2 caractères"}
    }
    return nil
}

func validerEmail(email string) error {
    if !strings.Contains(email, "@") {
        return ErrChamp{Nom: "email", Raison: "doit contenir un '@'"}
    }
    return nil
}

// --- Fonction principale ---

func inscrire(nom, email string) error {
    if err := validerNom(nom); err != nil {
        return fmt.Errorf("inscription impossible : %w", err)
    }
    if err := validerEmail(email); err != nil {
        return fmt.Errorf("inscription impossible : %w", err)
    }
    fmt.Printf("Inscription réussie pour %s (%s)\n", nom, email)
    return nil
}

// --- Main ---

func main() {
    // Cas 1 : nom vide
    err := inscrire("", "alice@example.com")
    if err != nil {
        fmt.Println(err)

        // On peut inspecter l'erreur sous-jacente avec errors.As
        var errChamp ErrChamp
        if errors.As(err, &errChamp) {
            fmt.Printf("  → Problème sur le champ : %s\n", errChamp.Nom)
        }
    }

    // Cas 2 : email invalide
    err = inscrire("Alice", "pas-un-email")
    if err != nil {
        fmt.Println(err)
    }

    // Cas 3 : tout valide
    err = inscrire("Alice", "alice@example.com")
    if err != nil {
        fmt.Println(err)
    }
}
// Sortie :
// inscription impossible : champ 'nom' invalide : ne peut pas être vide
//   → Problème sur le champ : nom
// inscription impossible : champ 'email' invalide : doit contenir un '@'
// Inscription réussie pour Alice (alice@example.com)
```

**Points importants :**
- `errors.As(err, &errChamp)` : "déballe" la chaîne d'erreurs pour trouver une `ErrChamp` dedans. C'est possible grâce au `%w` utilisé dans `fmt.Errorf`. C'est l'équivalent Go du `except ErrChamp as e` Python
- `if err := validerNom(nom); err != nil { ... }` : le `:=` dans la condition `if` est un idiome Go fréquent. Il déclare et assigne `err` dans la portée du `if` seulement

---
---

## MODULE 3 — Goroutines et channels

### 3.1 — Théorie

#### Le problème de la concurrence en Python

En Python, le GIL (Global Interpreter Lock) empêche plusieurs threads de s'exécuter en parallèle réel. On contourne ça avec `asyncio` (concurrence coopérative) ou `multiprocessing` (processus séparés, lourd).

#### En Go : goroutines légères

Une **goroutine** est une fonction qui s'exécute de manière concurrente. On la lance avec le mot-clé `go` :

```go
go maFonction() // Lance maFonction() dans une nouvelle goroutine
```

Les goroutines sont très légères (~2 Ko de mémoire au démarrage, contre ~1 Mo pour un thread OS). Un programme Go peut en lancer des millions.

```python
# Python asyncio
import asyncio

async def tache(nom):
    await asyncio.sleep(1)
    print(f"Tâche {nom} terminée")

async def main():
    await asyncio.gather(tache("A"), tache("B"), tache("C"))

asyncio.run(main())
```

```go
// Go : beaucoup plus direct
func tache(nom string) {
    time.Sleep(time.Second)
    fmt.Println("Tâche", nom, "terminée")
}

func main() {
    go tache("A")
    go tache("B")
    go tache("C")
    time.Sleep(2 * time.Second) // Attendre (on verra mieux avec WaitGroup)
}
```

#### Les channels : communiquer entre goroutines

Un **channel** (`chan`) est un tuyau typé pour envoyer/recevoir des valeurs entre goroutines. La philosophie Go : *"Ne communique pas en partageant de la mémoire, partage de la mémoire en communiquant."*

```go
ch := make(chan int)      // Crée un channel non-bufferisé
ch <- 42                  // Envoie la valeur 42 dans le channel (bloquant)
valeur := <-ch            // Reçoit depuis le channel (bloquant)
```

Un channel non-bufferisé **bloque** l'envoyeur jusqu'à ce qu'un receveur soit prêt, et vice-versa. C'est une synchronisation naturelle.

```go
func calculer(a, b int, resultat chan int) {
    resultat <- a + b  // Envoie le résultat dans le channel
}

func main() {
    ch := make(chan int)
    go calculer(3, 4, ch)
    res := <-ch  // Attend que la goroutine envoie le résultat
    fmt.Println("Résultat :", res) // 7
}
```

#### `sync.WaitGroup` : attendre plusieurs goroutines

L'équivalent Go de `asyncio.gather` :

```go
import "sync"

func main() {
    var wg sync.WaitGroup

    for i := 0; i < 3; i++ {
        wg.Add(1) // On annonce qu'on va lancer une goroutine
        go func(n int) {
            defer wg.Done() // Décrémente le compteur quand la goroutine se termine
            fmt.Println("Tâche", n)
        }(i)
    }

    wg.Wait() // Bloque jusqu'à ce que toutes les goroutines aient appelé Done()
}
```

`defer` : exécute une instruction juste avant que la fonction retourne (peu importe comment). Très utilisé pour les nettoyages (fermer un fichier, libérer un lock...).

---

### 3.2 — Exercices

#### Exercice 3A — Pipeline de traitement

On veut traiter une liste de nombres en parallèle : chaque nombre est mis au carré, puis les résultats sont collectés.

**À faire :**

1. Crée une fonction `carrer(n int, ch chan int)` qui envoie `n*n` dans le channel
2. Dans `main()`, crée un channel `chan int`
3. Pour les nombres 1 à 5, lance une goroutine `carrer` par nombre
4. Reçois les 5 résultats depuis le channel (dans une boucle) et affiche-les
5. Affiche la somme des carrés

**Attention :** tu n'as pas besoin de `WaitGroup` ici car tu sais exactement combien de résultats attendre.

---

#### Exercice 3B — Producteur / Consommateur

Voici un pattern classique : un **producteur** génère des données, un **consommateur** les traite.

**À faire :**

1. Crée une fonction `producteur(ch chan string)` qui envoie 5 messages `"message-1"`, `"message-2"`, ..., puis **ferme** le channel (`close(ch)`)
2. Crée une fonction `consommateur(ch chan string, id int, wg *sync.WaitGroup)` qui :
   - Reçoit des messages du channel jusqu'à ce qu'il soit fermé (`for msg := range ch`)
   - Affiche `"Consommateur N a reçu : message"`
   - Appelle `wg.Done()` quand terminé
3. Dans `main()` :
   - Crée un channel bufferisé de taille 5 : `make(chan string, 5)`
   - Lance `producteur` dans une goroutine
   - Lance **2** consommateurs en goroutines
   - Attends la fin avec un `WaitGroup`

**À noter :** `for msg := range ch` reçoit les valeurs d'un channel jusqu'à sa fermeture. `close(ch)` signale qu'il n'y aura plus de données.

---

### 3.3 — Corrections

#### Correction 3A

```go
package main

import "fmt"

func carrer(n int, ch chan int) {
    ch <- n * n  // Envoie le carré dans le channel
}

func main() {
    ch := make(chan int) // Channel non-bufferisé

    // Lance 5 goroutines
    for i := 1; i <= 5; i++ {
        go carrer(i, ch)
    }

    // Reçoit exactement 5 résultats
    somme := 0
    for i := 0; i < 5; i++ {
        valeur := <-ch
        fmt.Println("Reçu :", valeur)
        somme += valeur
    }

    fmt.Println("Somme des carrés :", somme)
}
// Sortie (ordre non garanti) :
// Reçu : 1
// Reçu : 9
// Reçu : 25
// Reçu : 4
// Reçu : 16
// Somme des carrés : 55
```

**Points importants :**
- L'ordre des résultats est **non-déterministe** : les goroutines s'exécutent en parallèle et arrivent quand elles peuvent. C'est voulu.
- `<-ch` dans une boucle bloque jusqu'à recevoir chaque valeur. Comme on sait qu'il y en a exactement 5, on boucle 5 fois.
- Si on boucle trop peu de fois, des goroutines restent bloquées à envoyer (goroutine leak). Si on boucle trop, on bloque indéfiniment.

---

#### Correction 3B

```go
package main

import (
    "fmt"
    "sync"
)

func producteur(ch chan string) {
    for i := 1; i <= 5; i++ {
        msg := fmt.Sprintf("message-%d", i)
        ch <- msg
        fmt.Println("Produit :", msg)
    }
    close(ch) // Signale qu'il n'y aura plus de messages — les consommateurs sortiront de leur boucle range
}

func consommateur(ch chan string, id int, wg *sync.WaitGroup) {
    defer wg.Done() // Sera appelé quand le consommateur termine (quand le channel est fermé et vidé)

    for msg := range ch { // Boucle jusqu'à la fermeture du channel
        fmt.Printf("Consommateur %d a reçu : %s\n", id, msg)
    }
}

func main() {
    ch := make(chan string, 5) // Channel bufferisé : le producteur peut envoyer 5 messages sans bloquer

    var wg sync.WaitGroup

    go producteur(ch)

    // Lance 2 consommateurs
    wg.Add(2)
    go consommateur(ch, 1, &wg) // &wg : on passe un pointeur (le WaitGroup doit être partagé)
    go consommateur(ch, 2, &wg)

    wg.Wait() // Attend que les 2 consommateurs aient terminé
    fmt.Println("Tout terminé.")
}
// Sortie possible (ordre variable) :
// Produit : message-1
// Produit : message-2
// Consommateur 1 a reçu : message-1
// Consommateur 2 a reçu : message-2
// ...
// Tout terminé.
```

**Points importants :**
- `make(chan string, 5)` : channel **bufferisé**. Le producteur peut y déposer jusqu'à 5 messages sans qu'un consommateur soit prêt. Sans buffer, le producteur bloquerait à chaque envoi.
- `close(ch)` : indispensable pour que `for msg := range ch` se termine. Sans `close`, les consommateurs attendraient des messages à l'infini.
- `&wg` : on passe le WaitGroup **par pointeur**. Si on le passait par valeur, chaque goroutine aurait sa propre copie et `wg.Done()` n'affecterait pas le WaitGroup du `main`.

---
---

## MODULE 4 — Valeurs vs pointeurs 

### 4.1 — Théorie (rapide)

#### En Python : tout (ou presque) passe par référence

En Python, assigner un objet à une variable crée une **référence** vers cet objet. Passer un objet à une fonction ne copie pas l'objet :

```python
def modifier(liste):
    liste.append(99)  # Modifie la liste originale

ma_liste = [1, 2, 3]
modifier(ma_liste)
print(ma_liste)  # [1, 2, 3, 99] — la liste a été modifiée
```

#### En Go : tu choisis explicitement

En Go, **tout est passé par valeur par défaut** (une copie est faite). Pour modifier l'original, on passe un **pointeur**.

```go
// Passage par valeur — la struct est copiée, l'originale n'est pas modifiée
func doubler(r Rectangle) {
    r.Largeur *= 2 // Modifie la copie, pas l'original
}

// Passage par pointeur — on modifie l'original
func doublerPointeur(r *Rectangle) {
    r.Largeur *= 2 // Modifie la struct originale
}

rect := Rectangle{Largeur: 5, Hauteur: 3}
doublerPointeur(&rect) // & = "donne-moi l'adresse de rect"
fmt.Println(rect.Largeur) // 10
```

#### Règle pratique : quand utiliser un pointeur ?

- **Receiver `(r *Rectangle)` vs `(r Rectangle)`** : utilise un pointeur receiver si la méthode **modifie** la struct, ou si la struct est grande (éviter les copies coûteuses)
- **Argument `*Type` vs `Type`** : utilise un pointeur si tu veux modifier l'original dans la fonction
- Les slices, maps, et channels sont déjà des références implicitement — pas besoin de pointeurs pour eux

---

### 4.2 — Exercice

#### Exercice 4 — Compteur

1. Crée une struct `Compteur` avec un champ `Valeur int`
2. Écris une méthode `Incrementer()` avec un **receiver pointeur** (`*Compteur`) qui incrémente `Valeur`
3. Écris une méthode `Lire()` avec un **receiver valeur** (`Compteur`) qui retourne `Valeur`
4. Dans `main()`, crée un `Compteur`, appelle `Incrementer()` trois fois, et affiche la valeur avec `Lire()`
5. **Bonus** : essaie de changer le receiver de `Incrementer` en valeur (`Compteur` au lieu de `*Compteur`), observe que la valeur ne change plus

---

### 4.3 — Correction

```go
package main

import "fmt"

type Compteur struct {
    Valeur int
}

// Receiver pointeur : modifie la struct originale
func (c *Compteur) Incrementer() {
    c.Valeur++
}

// Receiver valeur : lit seulement, pas besoin de pointeur
func (c Compteur) Lire() int {
    return c.Valeur
}

func main() {
    c := Compteur{Valeur: 0}

    c.Incrementer()
    c.Incrementer()
    c.Incrementer()

    fmt.Println("Valeur :", c.Lire()) // 3

    // Go fait le déréférencement automatiquement :
    // c.Incrementer() est équivalent à (&c).Incrementer()
}
```

**Bonus — avec receiver valeur (bug intentionnel) :**

```go
func (c Compteur) IncrementerBug() { // Receiver valeur = copie
    c.Valeur++ // Modifie la copie, l'original est intact
}

c2 := Compteur{Valeur: 0}
c2.IncrementerBug()
c2.IncrementerBug()
fmt.Println("Valeur (bug) :", c2.Lire()) // 0 — pas modifié !
```

---
---

## MODULE 5 — Compilation et mémoire

### 5.1 — Théorie

#### Python : interprété, garbage collecté, GIL

Python est **interprété** : le code source est lu et exécuté ligne par ligne par l'interpréteur. Avantage : développement rapide. Inconvénient : lent.

La mémoire est gérée par un **garbage collector** (GC) à comptage de références + cycle detector. Le **GIL** empêche deux threads Python de s'exécuter en même temps dans le même processus.

#### Go : compilé, GC concurrent, pas de GIL

Go est **compilé** : le code source est transformé en binaire natif avant exécution. `go build main.go` produit un exécutable autonome.

```bash
go build -o monprogramme main.go  # Produit un binaire natif
./monprogramme                     # Aucune dépendance, aucun runtime à installer
```

Go a un **garbage collector concurrent** : il tourne en arrière-plan en même temps que le programme, avec des pauses très courtes (quelques millisecondes). Tu n'as pas à libérer la mémoire manuellement.

Il n'y a **pas de GIL** : les goroutines peuvent vraiment s'exécuter en parallèle sur plusieurs cœurs CPU.

#### Ce que ça change concrètement

| | Python | Go |
|---|---|---|
| Déploiement | Interpréteur + pip install | Un seul binaire |
| Vitesse | ~50-100x plus lent que C | ~2-5x plus lent que C |
| Mémoire | GC à comptage de ref | GC concurrent, faibles pauses |
| Parallélisme | Limité par le GIL | Natif sur tous les cœurs |
| Cross-compilation | Difficile | `GOOS=linux GOARCH=amd64 go build` |

#### Le cycle de vie d'un programme Go

```
Code source (.go)
      ↓ go build
  Binaire natif
      ↓ exécution
  Goroutine principale (main)
      ↓ goroutines créées avec go
  Scheduler Go (M:N threading)
      ↓ GC en arrière-plan
  Mémoire libérée automatiquement
```

---

### 5.2 — Exercice

#### Exercice 5 — Observer la compilation

Cet exercice est différent : pas de code à écrire, mais des commandes à exécuter pour **toucher** la compilation.

1. Écris un fichier `main.go` minimal :

```go
package main

import "fmt"

func main() {
    fmt.Println("Hello from a compiled binary!")
}
```

2. Compile-le : `go build -o hello main.go`
3. Observe la taille du binaire : `ls -lh hello`
4. Exécute-le : `./hello`
5. **Cross-compile** pour Linux depuis macOS (ou inversement) :
   ```bash
   GOOS=linux GOARCH=amd64 go build -o hello-linux main.go
   file hello-linux  # Vérifier le type de binaire
   ```
6. Essaie d'introduire une erreur de type volontaire (assigner un string à un int) et observe le message du compilateur.

---

### 5.3 — Correction / Notes

Il n'y a pas de "correction" unique, mais voici les observations attendues :

- `ls -lh hello` donnera un binaire de ~1-2 Mo (Go embarque le runtime). C'est plus gros qu'un binaire C, mais le programme est autonome.
- `file hello-linux` indiquera `ELF 64-bit LSB executable, x86-64` — un vrai binaire Linux, compilé depuis un autre OS.
- L'erreur de type : Go refuse de compiler. Il ne dit pas "runtime error" mais `cannot use "hello" (untyped string constant) as int value in assignment`. L'erreur est détectée **avant** l'exécution, c'est exactement l'avantage du typage statique.

---
---

## ANNEXE — Syntaxe Go : référence rapide

### Déclarations

```go
// var : déclaration explicite (niveau package ou fonction)
var x int = 42
var nom string = "Alice"
var actif bool          // Valeur zéro : false

// := : déclaration courte (fonctions seulement)
y := 10
prenom := "Bob"

// Constantes
const Pi = 3.14159
const MaxRetries = 3
```

### Fonctions

```go
// Fonction simple
func saluer(nom string) string {
    return "Bonjour, " + nom
}

// Retours multiples (pattern courant)
func diviser(a, b float64) (float64, error) {
    if b == 0 {
        return 0, errors.New("division par zéro")
    }
    return a / b, nil
}

// Retours nommés (lisibilité)
func minMax(nums []int) (min, max int) {
    min, max = nums[0], nums[0]
    for _, n := range nums[1:] {
        if n < min { min = n }
        if n > max { max = n }
    }
    return // Retourne min et max implicitement
}
```

### Structures de contrôle

```go
// if / else
if x > 0 {
    fmt.Println("positif")
} else if x < 0 {
    fmt.Println("négatif")
} else {
    fmt.Println("zéro")
}

// if avec initialisation (idiome fréquent pour les erreurs)
if err := faireQuelqueChose(); err != nil {
    fmt.Println("Erreur :", err)
}

// for (seul mot-clé de boucle en Go — remplace for, while, do-while)
for i := 0; i < 5; i++ { ... }        // C-style
for condition { ... }                   // Equivalent while
for { ... }                             // Boucle infinie
for i, v := range maSlice { ... }      // Equivalent for x in liste Python
for cle, valeur := range maMap { ... } // Itération sur map

// switch (pas besoin de break, chaque case s'arrête automatiquement)
switch jour {
case "lundi", "mardi":
    fmt.Println("début de semaine")
case "vendredi":
    fmt.Println("TGIF")
default:
    fmt.Println("autre jour")
}
```

### Slices (équivalent des listes Python)

```go
// Création
var s []int                      // nil slice
s = []int{1, 2, 3}              // Littéral
s = make([]int, 5)              // 5 éléments, tous à 0
s = make([]int, 3, 10)         // len=3, cap=10

// Opérations
s = append(s, 4, 5)            // Ajouter des éléments
s2 := s[1:3]                   // Slice de s[1] à s[2] (comme Python)
len(s)                          // Longueur
cap(s)                          // Capacité

// Itération
for i, v := range s {
    fmt.Println(i, v)
}
```

### Maps (équivalent des dicts Python)

```go
// Création
m := map[string]int{"alice": 30, "bob": 25}
m = make(map[string]int)

// Accès
age := m["alice"]               // 30, ou 0 si absente
age, ok := m["charlie"]         // ok = false si absente (idiome Go)
if !ok {
    fmt.Println("charlie non trouvé")
}

// Modification
m["david"] = 28
delete(m, "bob")

// Itération
for nom, age := range m {
    fmt.Printf("%s : %d ans\n", nom, age)
}
```

### Defer

```go
func lireFichier(chemin string) error {
    f, err := os.Open(chemin)
    if err != nil {
        return err
    }
    defer f.Close() // Sera exécuté quand lireFichier() retourne, peu importe comment

    // ... lire le fichier ...
    return nil
}
```

### Goroutines et channels

```go
// Goroutine
go maFonction()
go func() { /* code anonyme */ }()

// Channel
ch := make(chan int)         // Non-bufferisé
ch := make(chan int, 10)     // Bufferisé (capacité 10)
ch <- valeur                 // Envoyer (bloquant si plein)
valeur := <-ch               // Recevoir (bloquant si vide)
close(ch)                    // Fermer (les receivers verront la valeur zéro ensuite)
for v := range ch { ... }    // Recevoir jusqu'à close()

// WaitGroup
var wg sync.WaitGroup
wg.Add(1)
go func() { defer wg.Done(); /* travail */ }()
wg.Wait()
```

### Pointeurs

```go
x := 42
p := &x          // p est un *int, contient l'adresse de x
fmt.Println(*p)  // Déréférencement : affiche 42
*p = 100         // Modifie x via le pointeur
fmt.Println(x)   // 100

// Receiver pointeur (méthode qui modifie la struct)
func (c *Compteur) Incrementer() {
    c.Valeur++
}
```

---

*Fin du guide — bonne pratique : après avoir lu chaque module, essaie de réécrire les exemples de mémoire, sans regarder la correction.*
