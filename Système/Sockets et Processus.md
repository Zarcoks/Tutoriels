# Tutoriel — Les sockets & la mémoire d'un processus

## Avant de commencer

**Objectif.** Comprendre en profondeur ce qu'est une *socket* (en programmation réseau et système) et, à travers elle, comment un processus gère ses ressources — sa mémoire et sa **table des descripteurs de fichiers** — et comment il dialogue avec le noyau.

**Méthode : la pyramide.** On bétonne d'abord les fondations (le processus, sa mémoire, les descripteurs, les objets noyau), puis on monte vers les sockets, puis vers le sommet (multiplexage, plongée noyau). Chaque étage s'appuie sur le précédent.

**Prérequis.**

- Une machine **Linux** (ta VM fait l'affaire).
- Les outils de base : `sudo apt install build-essential strace iproute2`.
- Un terminal et quelques bases de C.

**Compiler et lancer un programme C :**

```
gcc fichier.c -o prog
./prog
```

**Structure de chaque module :**

1. **Histoire** — d'où vient ce qu'on étudie.
2. **Théorie** — le fond. Le vocabulaire technique est introduit **en gras**, défini directement dans le texte au moment où il apparaît. Exemples en C à l'appui.
3. **Pratique** — à toi de jouer (du code, parfois juste des commandes pour constater).
4. **Correction** — les solutions, expliquées en détail (on prend le temps d'expliquer les *pourquoi*).

**Conseil :** tape le code toi-même plutôt que de tout copier-coller. C'est en faisant qu'on retient.

---

# Module 0 — Le processus, sa mémoire et la frontière avec le noyau

Avant de toucher aux sockets, il faut connaître le terrain de jeu : le **processus** et sa **mémoire**. Une socket vivra dans ce terrain, et chaque opération réseau franchira la frontière qu'on décrit ici.

## Histoire

Aux débuts de l'informatique, une machine n'exécutait qu'un seul programme à la fois, et celui-ci accédait directement à la mémoire physique. Partager l'ordinateur était risqué : un programme pouvait écraser un autre, ou le système lui-même. Dans les années 1960, le **temps partagé** (*time-sharing*) a permis à plusieurs programmes de tourner « en même temps », ce qui a forcé à les **isoler** les uns des autres. Deux idées majeures sont nées de là : le **processus** — un programme en cours d'exécution, doté de sa propre vue de la machine — et la **mémoire virtuelle** — l'illusion, pour chaque processus, de disposer seul de toute la mémoire (une idée déjà implémentée en 1962 sur l'ordinateur Atlas, à Manchester). À partir de 1969, **Unix** (Bell Labs) a fait du processus une brique centrale, à côté du principe « tout est fichier ». C'est précisément ce qui permettra, plus tard, aux sockets de se glisser naturellement dans le modèle des fichiers.

## Théorie

### Qu'est-ce qu'un processus ?

Quand tu lances un programme, le système en crée un **processus** : une instance de ce programme *en cours d'exécution*. Le fichier sur le disque est statique ; le processus, lui, est vivant. Le noyau lui attribue un **PID** (*Process IDentifier*), le numéro unique qui l'identifie — c'est lui qu'on retrouve dans le répertoire `/proc/<PID>/`.

Un processus réunit trois choses :

1. un **programme en exécution** (du code qui tourne) ;
2. un **espace d'adressage virtuel** privé (sa mémoire) ;
3. une **comptabilité tenue par le noyau** (son PID, ses fichiers ouverts, son état…).

Le point (2) nous occupe dans ce module ; le point (3) — et notamment la table des fichiers ouverts — sera le cœur du Module 1, et le pont vers les sockets.

### La mémoire virtuelle

Le noyau ne donne jamais à un processus un accès direct à la mémoire physique (la RAM). À la place, il lui présente un **espace d'adressage virtuel** : une carte mémoire *privée* dans laquelle le processus a l'illusion de disposer seul de toute la mémoire. Les adresses qu'il manipule sont **virtuelles** ; un composant matériel, la **MMU** (*Memory Management Unit*), les traduit à la volée en adresses **physiques**. Conséquence : deux processus peuvent utiliser « la même » adresse virtuelle sans se gêner, car elle pointe en réalité vers des cases physiques différentes. C'est la base de l'isolation entre processus.

Cet espace est découpé en **segments** (ou régions), chacun avec un rôle précis. Vue simplifiée (adresses hautes en haut) :

```
Adresses hautes
┌──────────────────────────────┐
│ pile (stack)         ↓ croît  │
├──────────────────────────────┤
│        ... espace libre ...   │
├──────────────────────────────┤
│ région mmap / bibliothèques   │
├──────────────────────────────┤
│ tas (heap)           ↑ croît  │
├──────────────────────────────┤
│ .bss   globales non init.     │
├──────────────────────────────┤
│ .data  globales initialisées  │
├──────────────────────────────┤
│ .text  code machine           │
└──────────────────────────────┘
Adresses basses   (page 0 non mappée → NULL = segfault)
```

- **`.text`** : le code machine du programme (lecture + exécution, pas d'écriture).
- **`.data`** : les variables globales/statiques **initialisées** (ex. `int x = 42;`).
- **`.bss`** : les variables globales/statiques **non initialisées** (ex. `int x;`), mises à zéro au démarrage.
- **Le tas** (*heap*) : la mémoire allouée dynamiquement (`malloc`). Croît vers les adresses **hautes**.
- **La pile** (*stack*) : les variables locales et la mécanique des appels de fonctions (chaque appel empile un *cadre*). Croît vers les adresses **basses**.
- **La zone mmap** : les **bibliothèques partagées** et les fichiers projetés en mémoire.

Deux choses à retenir : la **pile** et le **tas** croissent l'un vers l'autre depuis des extrémités opposées ; et chaque région a des **permissions** (lecture / écriture / exécution) que le noyau fait respecter.

### La frontière utilisateur / noyau

Ton code s'exécute en **mode utilisateur** (*user space*), avec des privilèges limités : il ne peut pas toucher directement au matériel ni à la mémoire des autres processus. Pour une opération sensible — lire un fichier, demander de la mémoire, envoyer des données sur une socket — il doit passer par un **appel système** (*syscall*) : une demande adressée au **noyau** (*kernel*), qui s'exécute, lui, en **mode noyau** (*kernel space*) avec tous les privilèges. L'appel système fait basculer le CPU en mode noyau, le noyau effectue le travail, puis rend la main au programme.

En pratique, on appelle rarement les syscalls « à la main » : on passe par la **libc**, la bibliothèque C standard, dont les fonctions enveloppent un ou plusieurs appels système (`printf`, par exemple, finit par appeler le syscall `write`).

**Pourquoi ça compte ici ?** Chaque opération sur une socket — `socket`, `bind`, `connect`, `send`, `recv` — est un appel système. Quand tu fais `recv`, le noyau **recopie** des octets depuis un tampon interne vers *ton* tampon : il franchit la frontière. Tout ce tutoriel se joue exactement là. Garde deux questions en tête pour la suite : *où vivent mes tampons* (pile ou tas ?) et *qu'est-ce qui traverse vers le noyau ?*

Dernier détail que tu vas constater : les adresses changent à chaque exécution. C'est l'**ASLR** (*Address Space Layout Randomization*), une protection qu'on examine à l'exercice 0.5.

### Exemple 1 — Où atterrit chaque variable ?

```c
#include <stdio.h>
#include <stdlib.h>

int initialise = 42;        /* -> segment .data */
int non_initialise;         /* -> segment .bss  */

int main(void) {
    int locale = 7;                          /* -> pile  */
    int *dynamique = malloc(sizeof(int));    /* -> tas   */

    printf("text  (code)     : %p\n", (void *)main);
    printf(".data (init)     : %p\n", (void *)&initialise);
    printf(".bss  (non-init) : %p\n", (void *)&non_initialise);
    printf("tas   (heap)     : %p\n", (void *)dynamique);
    printf("pile  (stack)    : %p\n", (void *)&locale);

    free(dynamique);
    return 0;
}
```

En affichant les adresses, on *voit* la carte mémoire se dessiner.

### Exemple 2 — Un programme n'est jamais « simple »

```c
#include <unistd.h>

int main(void) {
    write(1, "Salut le noyau\n", 15);   /* 1 = sortie standard */
    return 0;
}
```

M�me pour afficher une ligne, beaucoup de choses se passent sous le capot : le noyau **charge** ton programme (`execve`), la **libc est mappée** en mémoire, de la mémoire est **demandée** (`brk`/`mmap`), et seulement ensuite ton `write` s'exécute. L'outil `strace` permet d'observer tout ça (voir Pratique).

## Pratique

> Crée un dossier de travail : `mkdir ~/tuto-sockets && cd ~/tuto-sockets`, et range-y tes fichiers.

### Exercice 0.1 — Localiser les segments depuis le code

Crée `segments.c` (Exemple 1).

```
gcc segments.c -o segments
./segments
```

**Questions :** 1. Dans quel ordre apparaissent les adresses ? 2. Le tas et la pile sont-ils proches ou très éloignés, et pourquoi ? 3. Relance plusieurs fois : les adresses changent-elles ?

### Exercice 0.2 — Lire la vraie carte mémoire via `/proc`

Crée `pause.c` :

```c
#include <stdio.h>
#include <unistd.h>

int main(void) {
    printf("PID = %d — j'attends. (Ctrl-C pour quitter)\n", getpid());
    fflush(stdout);
    pause();        /* dort jusqu'à recevoir un signal */
    return 0;
}
```

```
gcc pause.c -o pause
./pause
```

**Dans un second terminal** (remplace `PID` par le numéro affiché) :

```
cat /proc/PID/maps
pmap PID
```

**Questions :** 1. Repère `[stack]`, `[heap]`, et le binaire `pause`. 2. Quelles permissions (`r`/`w`/`x`) porte le **code** ? Et la **pile** ? 3. Quels `.so` vois-tu, et qu'est-ce que c'est ? *(Quitter : `Ctrl-C`, ou `kill PID`.)*

### Exercice 0.3 — `.data` vs `.bss` : le secret du fichier binaire

```c
/* gros_bss.c */
#include <stdio.h>
int tab[1000000];              /* non initialisé -> .bss */
int main(void) { printf("%d\n", tab[0]); return 0; }
```

```c
/* gros_data.c */
#include <stdio.h>
int tab[1000000] = {1};        /* initialisé -> .data */
int main(void) { printf("%d\n", tab[0]); return 0; }
```

```
gcc gros_bss.c  -o gros_bss
gcc gros_data.c -o gros_data
size gros_bss   gros_data      # colonnes text / data / bss
ls -l gros_bss  gros_data      # taille des fichiers
```

**Questions :** 1. Lequel a un gros `.bss` ? un gros `.data` ? 2. Lequel produit le **fichier le plus gros** ? 3. Pourquoi ?

### Exercice 0.4 — Espionner les appels système avec `strace`

Crée `salut.c` (Exemple 2).

```
gcc salut.c -o salut
strace ./salut
strace -c ./salut          # résumé avec compteurs
```

**Questions :** 1. Retrouve `write(1, "Salut le noyau\n", 15)`. 2. Combien d'appels système *avant* ? 3. À quoi servent `execve`, `mmap`, `brk` ?

### Exercice 0.5 (bonus) — Constater l'ASLR

```
./segments                        # plusieurs fois : adresses qui bougent
setarch $(uname -m) -R ./segments # plusieurs fois : adresses stables
```

**Questions :** 1. Quelles adresses bougent le plus ? 2. Pourquoi randomiser ?

## Correction

### 0.1 — Localiser les segments

Ordre attendu, des adresses basses aux hautes : **`.text` < `.data` < `.bss` < tas**, puis, très loin au-dessus, la **pile**. Exemple réel (les valeurs varient à chaque exécution) :

```
text  (code)     : 0x55f17f1221a9
.data (init)     : 0x55f17f125010
.bss  (non-init) : 0x55f17f125018
tas   (heap)     : 0x55f1bdf262a0
pile  (stack)    : 0x7ffed818d53c
```

Le code, les données et le tas sont **regroupés** (zone `0x55...`) ; la pile est tout en **haut** (zone `0x7fff...`).

**Pourquoi un tel écart entre le tas (`0x55...`) et la pile (`0x7fff...`) ?** C'est *la* bonne question. Sur un processeur x86-64, l'espace d'adressage virtuel d'un processus est **gigantesque** : côté utilisateur, les adresses vont de `0x0` à `0x7fff_ffff_ffff`, soit 128 **Tio**. Le noyau range alors les choses **aux deux bouts** de ce vaste espace :

- tout en **bas**, le binaire (`.text` / `.data` / `.bss`) puis le **tas**, qui grandit vers le haut (↑) ;
- tout en **haut**, la **pile**, qui grandit vers le bas (↓), avec les bibliothèques (zone mmap) juste en dessous.

Entre les deux, un **immense trou** (plusieurs téraoctets) qui ne contient rien. Et c'est **voulu**, pour deux raisons :

1. en partant d'extrémités opposées et en croissant l'un vers l'autre, le tas et la pile ont chacun une marge de croissance énorme avant de risquer de se rencontrer ;
2. ce trou est **purement virtuel** : tant qu'on n'y écrit pas, il ne consomme **aucune** mémoire physique. Le noyau ne réserve de la RAM que pour les pages réellement utilisées. Un grand espace d'adressage ne coûte donc rien.

L'écart de valeur n'est pas de la place « gâchée » : c'est l'organisation naturelle qui donne à la pile et au tas tout l'espace dont ils pourraient avoir besoin. *(Et oui, les adresses changent à chaque lancement — c'est l'ASLR, exercice 0.5.)*

### 0.2 — La vraie carte mémoire

Dans `/proc/PID/maps`, chaque ligne décrit une région : `plage_d'adresses  permissions  offset  périphérique  inode  chemin`. Tu y trouves `[heap]` (le tas), `[stack]` (la pile, en haut), plusieurs lignes pour le binaire `pause`, et des `.so`.

Les **permissions** sont la partie intéressante. Le **code** apparaît en `r-xp` : **lisible et exécutable, mais pas inscriptible** — on ne réécrit pas son propre code en cours de route. La **pile** et le **tas** sont en `rw-p` : **lisibles et inscriptibles, mais pas exécutables**. Cette séparation stricte (on n'exécute pas des données, on ne modifie pas du code) porte un nom : **W^X** — *Write XOR eXecute*, « écrire **ou** exécuter, mais pas les deux ». Elle s'appuie sur le bit **NX** (*No-eXecute*) du processeur. Pourquoi ? Parce qu'une attaque classique consiste à injecter du code dans un tampon (sur la pile) puis à l'exécuter : interdire l'exécution sur la pile coupe court à cette technique.

Les `.so` sont des **bibliothèques partagées** chargées dynamiquement : au minimum la **libc** (`libc.so.6`) et le **chargeur dynamique** (`ld-linux-x86-64.so`), mappés dans la zone *mmap* (adresses `0x7f...`). « Partagées » signifie que leur code (en `r-x`) peut être mappé **une seule fois en RAM** et utilisé par tous les processus qui en ont besoin — d'où l'économie. `pmap PID` montre la même chose, en plus lisible, avec la taille de chaque région.

### 0.3 — `.data` vs `.bss`

`size` (valeurs réelles observées) :

```
   text     data      bss   filename
   1381      600  4000032   gros_bss     <- gros .bss, petit .data
   1381  4000616        8   gros_data    <- gros .data, petit .bss
```

`ls -l` :

```
   15992   gros_bss      (~16 Ko)
 4016008   gros_data     (~3,9 Mo)
```

`gros_bss` a un gros `.bss` ; `gros_data` a un gros `.data` ; et `gros_data` produit un fichier **~3,8 Mo plus gros**.

**Pourquoi ?** Il faut comprendre ce qu'est un fichier exécutable (un fichier **ELF**, sous Linux) : c'est un *plan de chargement*. Pour chaque section, il indique au noyau quoi mettre en mémoire au démarrage.

- Le `.bss` est *seulement non initialisé* (sa valeur de départ est zéro). Le fichier n'a donc **pas besoin de stocker** un million de zéros : il lui suffit de noter « réserve 4 Mo et mets-les à zéro ». En ELF, cette section est de type **NOBITS** : elle occupe de la place *en mémoire*, zéro octet *sur le disque*.
- Le `.data` contient des **valeurs explicites** non nulles. Le fichier doit donc **réellement stocker chaque octet** pour pouvoir les recharger à l'identique.

Conclusion : initialiser un gros tableau à des valeurs non nulles coûte de l'espace disque ; le laisser non initialisé (ou tout à zéro) n'en coûte pas. *(Détail : un tableau initialisé entièrement à zéro, `= {0}`, repart en `.bss` — c'est la présence de valeurs non nulles qui force le `.data`.)*

### 0.4 — Les appels système

1. `write(1, "Salut le noyau\n", 15)` est ton appel : écrire 15 octets sur le **descripteur 1**, la sortie standard (on creusera la notion de descripteur au Module 1).
2. Beaucoup d'appels ont lieu **avant**, typiquement :
   - `execve(...)` : l'appel qui a **lancé** ton programme (le shell l'a invoqué). C'est lui qui remplace l'image du shell par la tienne.
   - `brk(...)` / `mmap(...)` : des **demandes de mémoire** au noyau, pour le tas et pour projeter la libc en mémoire.
   - `openat` / `read` / `fstat` / `mprotect` : le **chargement dynamique** de la libc par `ld-linux` (ouvrir le `.so`, le lire, ajuster ses permissions).
   - `exit_group(0)` : la **terminaison** du programme.
3. En résumé : `execve` démarre le programme, `mmap` projette des fichiers/de la mémoire dans l'espace d'adressage (dont la libc), `brk` ajuste la frontière haute du tas.

**Leçon :** même un programme « trivial » dialogue énormément avec le noyau. Chaque ligne de `strace` = un franchissement de la frontière utilisateur → noyau → utilisateur. C'est exactement ce type d'échange qu'on observera, plus tard, pour `socket`, `connect`, `send`… *(La liste exacte dépend de la libc et du système ; l'essentiel est de retrouver ton `write` au milieu du reste.)*

### 0.5 — L'ASLR

Sans ASLR (`setarch ... -R`), les adresses sont **identiques** d'un lancement à l'autre. Avec ASLR (le défaut), trois choses sont décalées **aléatoirement** à chaque exécution : la base de la **pile**, celle de la **zone mmap** (bibliothèques), et — parce que le binaire est compilé en **PIE** (*Position-Independent Executable*) — celle du **code lui-même**. C'est très visible sur la pile (`0x7fff...`) et les bibliothèques, qui « sautent » de plusieurs Mo à chaque fois.

**Pourquoi ?** De nombreuses attaques supposent de **connaître à l'avance** l'adresse d'un bout de code ou de données (pour y détourner l'exécution, par exemple lors d'un débordement de tampon). En rendant ces adresses imprévisibles, l'ASLR fait échouer ces attaques « à l'aveugle ». C'est une défense peu coûteuse (un simple décalage au démarrage) et donc activée par défaut partout.

## À retenir

Un processus = un programme vivant, doté d'un **espace d'adressage privé** (`.text` / `.data` / `.bss` / tas / pile, avec un immense espace virtuel autour) et d'une **comptabilité tenue par le noyau**. Tout service du noyau passe par un **appel système**, qui franchit la frontière utilisateur/noyau — exactement le chemin qu'emprunteront toutes les opérations sur les sockets.

→ **Module 1** : on attaque la comptabilité côté noyau — la **table des descripteurs de fichiers** —, le pont direct vers les sockets.

---

# Module 1 — Les descripteurs de fichiers : l'abstraction universelle

On a vu que tout service du noyau passe par un appel système. Reste à savoir **comment un programme désigne** le fichier, le tube ou la socket sur lequel il veut agir. La réponse — d'une élégance redoutable — est le **descripteur de fichier**. C'est la pièce maîtresse de tout le tutoriel : une socket *est* un descripteur de fichier.

## Histoire

Quand Ken Thompson et Dennis Ritchie conçoivent Unix (1969-1971), ils prennent une décision radicale de simplicité : « **tout est fichier** ». Un fichier sur le disque, un terminal, une imprimante, un tube entre deux programmes… tout s'ouvre, se lit, s'écrit et se ferme avec **les mêmes quatre appels** : `open`, `read`, `write`, `close`. Pour désigner l'objet ouvert, le noyau renvoie un **petit entier** : le *file descriptor*. Plus besoin d'API différente par type de ressource. Cette uniformité est l'une des raisons de la longévité d'Unix — et c'est elle qui permettra, en 1983, d'ajouter les **sockets** comme un simple nouveau « type de fichier », sans bouleverser le modèle.

## Théorie

### Le descripteur, un simple entier

Un **descripteur de fichier** (*file descriptor*, souvent abrégé **fd**) est un **entier positif** que le noyau te remet quand tu ouvres une ressource. Tu le repasses ensuite à chaque opération (`read(fd, ...)`, `write(fd, ...)`). Ce n'est qu'un **numéro** : un **index** dans une table que le noyau tient **pour ton processus**.

Trois descripteurs sont ouverts d'office au démarrage de tout programme :

- **fd 0** : l'**entrée standard** (*stdin*) — d'où viennent les données saisies.
- **fd 1** : la **sortie standard** (*stdout*) — où va ce que tu affiches.
- **fd 2** : la **sortie d'erreur** (*stderr*) — où vont les messages d'erreur.

C'est pour ça que `write(1, ...)` écrit à l'écran : 1 = stdout. Le premier fichier que **tu** ouvres reçoit donc en général le numéro **3** (le plus petit libre).

### Les trois tables

Quand tu ouvres un fichier, le noyau ne se contente pas d'une table. Il en met **trois** en jeu — et comprendre cet empilement explique *tout* le comportement des fd (y compris des sockets) :

```
PROCESSUS  (une table par processus)        NOYAU  (partagé par tout le système)

Table des descripteurs              Table des fichiers ouverts          Table des inodes
                                    (position, flags, compteur)         (l'objet réel)

  fd 0  ──────────────────────────►  [ description #1 ]  ────────────►  ( inode X )
  fd 1  ──────────────────────────►  [ description #2 ]  ────────────►  ( inode Y )
  fd 2  ───────────────┐
  fd 3  ───────────────┴──────────►  [ description #3 ]  ────────────►  ( inode Z )
                                     (fd 2 et fd 3 pointent la même
                                      description → ils partagent l'offset)
```

1. La **table des descripteurs**, propre à **chaque processus** : un simple tableau où la case `fd` pointe vers une entrée de la table suivante. C'est elle qu'on voit dans `/proc/<PID>/fd`.
2. La **table des fichiers ouverts** (*open file descriptions*), **partagée à l'échelle du système**. Chaque entrée contient l'**offset** (la position de lecture/écriture courante, en octets), les **flags** d'ouverture (lecture seule, ajout…) et un **compteur de références** (on y revient au Module 2). 
3. La **table des inodes** : l'**inode** est l'objet qui représente réellement le fichier (ou le tube, ou la socket) dans le noyau.

L'**offset** vit donc dans la table du **milieu**, pas dans la table des descripteurs. Retiens bien ça : c'est la clé de `dup` et de `fork`.

### `open`, `read`, `write`, `lseek`, `close`

- `int fd = open("chemin", flags, mode);` ouvre/crée un fichier et renvoie un fd (ou `-1` en cas d'erreur).
- `ssize_t n = read(fd, buf, taille);` lit jusqu'à `taille` octets dans `buf`, **fait avancer l'offset**, renvoie le nombre d'octets lus (0 = fin de fichier).
- `ssize_t n = write(fd, buf, taille);` écrit, **fait avancer l'offset**.
- `off_t pos = lseek(fd, decalage, origine);` **déplace** l'offset sans lire ni écrire (`SEEK_SET` = depuis le début, `SEEK_CUR` = position actuelle).
- `close(fd);` libère le descripteur.

### `dup` : deux descripteurs, une seule description

`int fd2 = dup(fd1);` crée un **second** descripteur qui pointe vers **la même** entrée de la table des fichiers ouverts. Les deux fd partagent donc l'**offset** : si tu écris via `fd1`, la position avance aussi pour `fd2`. C'est précisément le mécanisme de la **redirection** du shell. Quand tu tapes `./prog > sortie.txt`, le shell ouvre le fichier, puis fait `dup2(fd_fichier, 1)` pour que le **descripteur 1** (stdout) pointe vers le fichier — et `printf` se retrouve écrit dans le fichier sans que `prog` n'en sache rien. (`dup2(ancien, nouveau)` est comme `dup`, mais en choisissant le numéro de destination.)

### Exemple — l'offset partagé par `dup`

```c
#include <fcntl.h>
#include <unistd.h>

int main(void) {
    int fd1 = open("dup.txt", O_CREAT | O_RDWR | O_TRUNC, 0644);
    int fd2 = dup(fd1);          /* même description → offset partagé */
    write(fd1, "ABC", 3);        /* offset : 0 → 3 */
    write(fd2, "DEF", 3);        /* écrit à partir de 3, pas de 0 ! */
    close(fd1);
    close(fd2);
    return 0;                    /* dup.txt contient "ABCDEF" */
}
```

## Pratique

### Exercice 1.1 — Quel numéro renvoie `open` ?

```c
/* fd.c */
#include <stdio.h>
#include <fcntl.h>
int main(void) {
    int a = open("/dev/null", O_RDONLY);
    int b = open("/dev/null", O_RDONLY);
    int c = open("/dev/null", O_RDONLY);
    printf("fd a = %d\nfd b = %d\nfd c = %d\n", a, b, c);
    return 0;
}
```

```
gcc fd.c -o fd && ./fd
```

**Question :** pourquoi les numéros commencent-ils à 3, et pas à 0 ?

### Exercice 1.2 — Voir la table des descripteurs (`/proc`)

Crée `attente_fd.c` : il ouvre deux fichiers puis attend.

```c
#include <stdio.h>
#include <fcntl.h>
#include <unistd.h>
int main(void) {
    open("/etc/hostname", O_RDONLY);
    open("/dev/null", O_RDONLY);
    printf("PID = %d (Ctrl-C pour quitter)\n", getpid());
    fflush(stdout);
    pause();
    return 0;
}
```

Lance-le, puis **dans un second terminal** :

```
ls -l /proc/PID/fd
```

**Questions :** 1. Que pointent les fd 0, 1, 2 ? 2. Retrouves-tu `/etc/hostname` et `/dev/null` ? Sur quels numéros ?

### Exercice 1.3 — L'offset se déplace

```c
/* offset.c */
#include <stdio.h>
#include <fcntl.h>
#include <unistd.h>
int main(void) {
    int fd = open("offset.txt", O_CREAT | O_RDWR | O_TRUNC, 0644);
    write(fd, "ABCDEF", 6);
    printf("offset après write       : %ld\n", (long)lseek(fd, 0, SEEK_CUR));
    lseek(fd, 0, SEEK_SET);
    printf("offset après lseek début : %ld\n", (long)lseek(fd, 0, SEEK_CUR));
    char buf[3];
    read(fd, buf, 3);
    printf("3 octets lus             : %.3s\n", buf);
    printf("offset après read        : %ld\n", (long)lseek(fd, 0, SEEK_CUR));
    close(fd);
    return 0;
}
```

**Question :** suis la valeur de l'offset à chaque étape. Pourquoi `read` ne relit-il pas « ABC » une seconde fois si on le rappelle ?

### Exercice 1.4 — La redirection à la main (`dup2`)

```c
/* redir.c */
#include <stdio.h>
#include <fcntl.h>
#include <unistd.h>
int main(void) {
    int fd = open("sortie.txt", O_CREAT | O_WRONLY | O_TRUNC, 0644);
    dup2(fd, 1);     /* le descripteur 1 (stdout) pointe vers le fichier */
    close(fd);
    printf("Ce texte part dans le fichier, pas à l'écran.\n");
    return 0;
}
```

```
gcc redir.c -o redir && ./redir
cat sortie.txt
```

**Question :** pourquoi `printf` finit-il dans le fichier alors qu'on ne lui a rien dit de spécial ?

## Correction

### 1.1 — Pourquoi 3, 4, 5 ?

Sortie :

```
fd a = 3
fd b = 4
fd c = 5
```

`open` renvoie toujours le **plus petit descripteur libre**. Or 0, 1 et 2 sont **déjà pris** au démarrage (stdin, stdout, stderr). Le premier libre est donc 3, puis 4, puis 5. Cette règle « plus petit libre » est ce qui rend la redirection possible : si on **ferme** le fd 1 puis qu'on ouvre un fichier, ce fichier récupère le numéro 1 — et devient la nouvelle sortie standard (c'est une autre façon de faire ce que `dup2` fait à l'exercice 1.4).

### 1.2 — La table des descripteurs en vrai

`ls -l /proc/PID/fd` montre chaque descripteur comme un **lien symbolique** vers ce qu'il désigne :

```
0 -> /dev/pts/0        (le terminal : stdin)
1 -> /dev/pts/0        (le terminal : stdout)
2 -> /dev/pts/0        (le terminal : stderr)
3 -> /etc/hostname
4 -> /dev/null
```

Les fd 0/1/2 pointent vers ton **terminal** (`/dev/pts/N`) : c'est pourquoi ce que tu tapes arrive au programme et ce qu'il affiche revient à l'écran. Tes deux `open` ont reçu 3 et 4, dans l'ordre des appels. Ce répertoire `/proc/PID/fd` **est** la visualisation directe de la table des descripteurs du processus — on s'en servira tout au long du tutoriel, y compris pour voir des sockets.

### 1.3 — L'offset qui avance

```
offset après write       : 6
offset après lseek début : 0
3 octets lus             : ABC
offset après read        : 3
```

Après avoir écrit 6 octets, l'offset est à **6** (fin du fichier). `lseek(..., SEEK_SET)` le ramène à **0**. Le `read` de 3 octets lit « ABC » **et fait avancer l'offset à 3**. Si tu rappelais `read`, il lirait « DEF », pas « ABC » : chaque lecture **consomme** la position. C'est exactement ce comportement (l'offset qui mémorise « où on en est ») qui, plus tard, fait qu'un `recv` sur une socket te donne la suite du flux et non toujours le début. L'offset vit dans la table du milieu (la description de fichier ouvert), pas dans la case `fd`.

### 1.4 — La redirection démystifiée

Après `./redir`, l'écran n'affiche rien ; `cat sortie.txt` montre :

```
Ce texte part dans le fichier, pas à l'écran.
```

`printf` écrit **toujours** sur le descripteur **1**, sans se poser de question. La ruse est ailleurs : `dup2(fd, 1)` a fait pointer la **case 1** de la table des descripteurs vers le fichier `sortie.txt` (à la place du terminal). Comme `printf` ne connaît que le **numéro** 1 et ignore *ce qu'il y a derrière*, son texte file dans le fichier. C'est **mot pour mot** ce que fait ton shell quand tu écris `./redir > sortie.txt`. Tu viens de réimplémenter la redirection à la main — et de toucher du doigt la force de l'abstraction « descripteur » : le code qui écrit ne dépend pas de la nature de la destination.

## À retenir

Un **descripteur de fichier** est un petit entier, **index** dans la table des descripteurs du processus. Cette table renvoie vers une **description de fichier ouvert** (qui porte l'**offset**, les flags, un compteur), elle-même reliée à un **inode** (l'objet réel). Les fd 0/1/2 sont stdin/stdout/stderr. `dup` crée un second fd sur la **même** description (offset partagé), ce qui explique la redirection.

→ **Module 2** : et si plusieurs descripteurs — voire plusieurs **processus** — pointent la même description ? On entre dans les **objets noyau**, le **compteur de références**, et l'héritage par `fork`.

---

# Module 2 — Objets noyau, compteur de références et héritage

Au Module 1, plusieurs descripteurs pouvaient pointer la même description de fichier ouvert. Que se passe-t-il alors quand on en ferme un ? Et lorsqu'un processus se **duplique** ? Pour répondre, il faut regarder **ce qu'il y a derrière** un descripteur : l'**objet noyau**, et la mécanique de **comptage** qui gère sa durée de vie.

## Histoire

Unix introduit très tôt deux idées qui font sa puissance. La première est `fork` : pour créer un nouveau processus, on **duplique** le processus courant à l'identique (l'enfant est une copie du parent). La seconde, due à Doug McIlroy en 1973, est le **tube** (*pipe*) : un canal entre deux programmes, qui permet d'écrire `ls | grep .c`. Tube et `fork` reposent sur le même socle : des **objets noyau** désignés par des descripteurs, dont la durée de vie est gérée par un **compteur de références**. C'est ce socle — et non un mécanisme propre aux fichiers — que les sockets réutiliseront.

## Théorie

### Derrière le descripteur : l'objet noyau

Un descripteur ne « contient » rien : il **pointe** (via la table du milieu) vers un **objet noyau**. Cet objet peut être de natures très variées : un fichier régulier, un **tube**, un périphérique (`/dev/null`, le terminal…), ou — bientôt — une **socket**. Le même quatuor `read`/`write`/`close` fonctionne sur tous, parce qu'ils exposent tous la même interface. C'est ça, « tout est fichier » : non pas que tout *soit* un fichier sur le disque, mais que tout se **manipule comme** un fichier.

### Le compteur de références

Chaque **description de fichier ouvert** (la table du milieu) porte un **compteur de références** (*refcount*) : le nombre de descripteurs qui pointent vers elle.

- `dup` ou `fork` **incrémentent** ce compteur (un descripteur de plus).
- `close` le **décrémente**.
- L'objet noyau n'est **réellement libéré** que lorsque le compteur tombe à **0**.

C'est pour ça que fermer **un** des deux fd obtenus par `dup` ne détruit pas l'objet : l'autre fd le maintient en vie. Cette comptabilité est invisible au quotidien, mais elle explique des comportements essentiels — par exemple pourquoi, sur un tube, le lecteur ne reçoit la « fin de fichier » que lorsque **tous** les écrivains ont fermé leur extrémité.

### Le tube (`pipe`)

`pipe(p)` crée **deux** descripteurs d'un coup, reliés par un **tampon en mémoire** dans le noyau :

- `p[0]` : l'extrémité de **lecture** ;
- `p[1]` : l'extrémité d'**écriture**.

Ce qu'on écrit dans `p[1]` ressort par `p[0]`. C'est notre premier objet noyau **qui n'est pas un fichier** — et un avant-goût de la socket, qui est elle aussi un canal de communication désigné par des descripteurs.

### `fork` : l'héritage des descripteurs

`fork()` crée un processus **enfant**, copie quasi conforme du parent. Point crucial pour nous : l'enfant reçoit une **copie de la table des descripteurs**, mais ces copies **pointent vers les mêmes descriptions de fichier ouvert** que le parent. Autrement dit, parent et enfant **partagent l'offset** (et le refcount augmente). Schéma :

```
       Parent                              Enfant (après fork)
  Table des descripteurs               Table des descripteurs
        fd 3  ───────────┐             ┌─────── fd 3
                         │             │
                         ▼             ▼
                  [ description de fichier ouvert ]   refcount = 2
                         │  (offset partagé)
                         ▼
                     ( inode )
```

C'est ce partage qui rend possible des montages comme `ls | grep` : le shell crée un tube, **fork** deux fois, et chaque enfant hérite de la bonne extrémité.

### `exec` et le drapeau *close-on-exec*

`exec` **remplace** le programme du processus par un autre, **mais conserve la table des descripteurs** : les fichiers ouverts restent ouverts à travers l'`exec`. C'est voulu (c'est ainsi qu'un shell passe une redirection au programme qu'il lance). Parfois on ne le veut pas — un descripteur sensible ne devrait pas « fuir » vers le programme exécuté. On pose alors le drapeau **FD_CLOEXEC** (*close-on-exec*) : le descripteur est **automatiquement fermé** au moment de l'`exec`.

### Exemple — tube entre parent et enfant

```c
#include <stdio.h>
#include <unistd.h>
#include <string.h>
#include <sys/wait.h>

int main(void) {
    int p[2];
    pipe(p);                       /* p[0]=lecture, p[1]=écriture */
    if (fork() == 0) {             /* enfant : lit */
        close(p[1]);
        char buf[64];
        ssize_t n = read(p[0], buf, sizeof buf - 1);
        buf[n] = '\0';
        printf("[enfant] reçu : %s\n", buf);
    } else {                       /* parent : écrit */
        close(p[0]);
        write(p[1], "message via tube", 16);
        close(p[1]);
        wait(NULL);
    }
    return 0;
}
```

## Pratique

### Exercice 2.1 — Un tube entre deux processus

Compile et lance l'exemple ci-dessus (`pipe.c`).

**Questions :** 1. Pourquoi le parent ferme-t-il `p[0]` et l'enfant `p[1]` ? 2. Que se passerait-il si le parent n'écrivait jamais et ne fermait jamais `p[1]` ?

### Exercice 2.2 — L'offset partagé par `fork`

```c
/* fork_offset.c */
#include <fcntl.h>
#include <unistd.h>
#include <sys/wait.h>
int main(void) {
    int fd = open("partage.txt", O_CREAT | O_RDWR | O_TRUNC, 0644);
    if (fork() == 0) {
        write(fd, "enfant\n", 7);
    } else {
        wait(NULL);                /* on attend l'enfant */
        write(fd, "parent\n", 7);
    }
    close(fd);
    return 0;
}
```

```
gcc fork_offset.c -o fork_offset && ./fork_offset
cat partage.txt
```

**Question :** le fichier contient-il `enfant` *puis* `parent`, ou bien `parent` a-t-il **écrasé** `enfant` ? Qu'est-ce que ça prouve sur l'offset ?

### Exercice 2.3 — Les descripteurs survivent à `exec` (sauf CLOEXEC)

```c
/* cloexec.c */
#include <stdio.h>
#include <fcntl.h>
#include <unistd.h>
int main(int argc, char **argv) {
    int fd = open("/dev/null", O_RDONLY);
    printf("Ouvert /dev/null sur le descripteur %d\n", fd);
    if (argc > 1)                          /* avec un argument : on active CLOEXEC */
        fcntl(fd, F_SETFD, FD_CLOEXEC);
    fflush(stdout);
    execlp("ls", "ls", "-l", "/proc/self/fd", (char *)NULL);  /* devient 'ls' */
    perror("execlp");
    return 1;
}
```

```
gcc cloexec.c -o cloexec
./cloexec        # sans argument
./cloexec x      # avec argument (CLOEXEC activé)
```

**Question :** dans quel cas le `ls` (qui a remplacé ton programme) voit-il encore le descripteur 3 vers `/dev/null` ? Pourquoi ?

## Correction

### 2.1 — Le tube

Affichage : `[enfant] reçu : message via tube`.

1. Un tube a deux extrémités, et **chaque processus ferme celle qu'il n'utilise pas**. Le parent n'écrit que : il ferme la lecture `p[0]`. L'enfant ne lit que : il ferme l'écriture `p[1]`. Ce n'est pas qu'une question de propreté : c'est **nécessaire** pour le point 2.
2. Le `read` de l'enfant ne se termine (en renvoyant 0, « fin de fichier ») que lorsque **plus aucun** descripteur d'écriture n'est ouvert — sur **tout le système**, grâce au compteur de références. Si le parent gardait `p[1]` ouvert, l'enfant resterait **bloqué** indéfiniment dans `read`, à attendre des données qui ne viendraient jamais. C'est une illustration directe du **refcount** : tant que le compteur des écrivains n'est pas à zéro, le tube est « encore susceptible » de recevoir des données.

### 2.2 — `fork` partage bien l'offset

`partage.txt` contient :

```
enfant
parent
```

`parent` n'a **pas** écrasé `enfant` : il s'est écrit **à la suite**. Voici pourquoi. Le `fork` a lieu **après** le `open` ; l'enfant hérite donc d'un descripteur qui pointe vers **la même** description de fichier ouvert que le parent — avec un **offset partagé**. L'enfant écrit « enfant\n » : l'offset commun passe à 7. Le parent (qui a attendu via `wait`) écrit ensuite « parent\n » : il démarre à 7, pas à 0. Résultat : les deux textes coexistent.

Si chacun avait eu son **propre** offset (ce qui *n'est pas* le cas après `fork`, mais le serait si chacun avait fait son propre `open`), le parent aurait redémarré à 0 et **écrasé** « enfant ». Cette expérience prouve donc, en une ligne de `cat`, que `fork` **duplique la table des descripteurs mais partage les descriptions** — exactement le schéma du module.

### 2.3 — `exec` et CLOEXEC

Le `ls` qui remplace ton programme affiche le contenu de `/proc/self/fd`, c'est-à-dire **sa propre** table de descripteurs (héritée, car `exec` ne remet pas la table à zéro).

Sans argument — le descripteur **survit** à l'`exec` :

```
0 -> /dev/pts/0
1 -> /dev/pts/0
2 -> /dev/pts/0
3 -> /dev/null          <-- toujours là après l'exec
```

Avec argument (`./cloexec x`) — le descripteur a été **fermé automatiquement** à l'`exec` :

```
0 -> /dev/pts/0
1 -> /dev/pts/0
2 -> /dev/pts/0
                        (plus de fd 3 : CLOEXEC l'a fermé)
```

Par défaut, un descripteur ouvert **traverse** l'`exec` : c'est ce qui permet, par exemple, de transmettre une redirection. En posant **FD_CLOEXEC**, on demande au noyau de le **fermer juste avant** de lancer le nouveau programme — utile pour ne pas laisser fuir un descripteur sensible (un fichier de configuration, une **socket**…) vers un sous-processus. Tu rencontreras souvent la variante `O_CLOEXEC`, qui pose ce drapeau dès l'ouverture, sans appel `fcntl` séparé.

## À retenir

Derrière chaque descripteur, un **objet noyau** (fichier, tube, périphérique, bientôt socket). Sa durée de vie est réglée par un **compteur de références** : `dup`/`fork` l'augmentent, `close` le diminue, libération à zéro. `fork` **copie la table des descripteurs** mais **partage les descriptions** (donc l'offset). `exec` **conserve** les descripteurs, sauf ceux marqués **CLOEXEC**.

→ **Module 3** : on crée enfin une **socket** — et on vérifie qu'elle se range, sans surprise, dans cette même table des descripteurs.

---

# Module 3 — La socket : un descripteur d'un nouveau genre

Tout est en place. Une **socket** n'est rien d'autre qu'un **objet noyau** d'un nouveau type, désigné par un **descripteur**, rangé dans la **table** qu'on connaît maintenant par cœur. Ce module est volontairement court : son but est de **constater** cette continuité avant de faire communiquer quoi que ce soit.

## Histoire

Au début des années 1980, l'ARPANET devient TCP/IP, et il faut une **interface de programmation** pour que les applications parlent au réseau. L'université de Berkeley la livre en 1983 avec la version **4.2BSD** d'Unix : l'**API socket**. Le coup de génie est de **réutiliser le descripteur de fichier** : une connexion réseau s'obtient, se lit et s'écrit presque comme un fichier. Cette API, à peine modifiée, est aujourd'hui le standard universel (Linux, macOS, Windows…). On parle d'ailleurs encore de « **Berkeley sockets** ».

## Théorie

### Une socket, c'est deux choses à la fois

Une **socket** est :

1. un **point de terminaison de communication** (*endpoint*) — l'objet par lequel un programme émet et reçoit ;
2. un **descripteur de fichier** — donc une entrée de plus dans la table du processus, manipulable avec les appels qu'on connaît.

On la crée avec :

```c
int s = socket(domaine, type, protocole);
```

L'appel renvoie un **fd** (ou `-1` en cas d'erreur). À partir de là, c'est un descripteur comme un autre.

### Domaine, type, protocole

- Le **domaine** (ou **famille d'adresses**) dit *dans quel monde* on communique :
  - **`AF_UNIX`** : communication **locale**, entre processus de la même machine (Module 4).
  - **`AF_INET`** : réseau **IPv4** (Modules 5 et 6).
  - **`AF_INET6`** : réseau IPv6.
- Le **type** dit *comment* :
  - **`SOCK_STREAM`** : un **flux** d'octets **fiable** et **ordonné**, orienté **connexion** (c'est TCP en `AF_INET`).
  - **`SOCK_DGRAM`** : des **datagrammes** indépendants, **sans connexion** ni garantie (c'est UDP en `AF_INET`).
- Le **protocole** précise lequel utiliser ; `0` laisse le noyau choisir le protocole par défaut du couple (par ex. TCP pour `AF_INET`+`SOCK_STREAM`).

### L'objet noyau derrière une socket

Côté noyau, une socket est représentée par une structure `struct socket`, doublée d'une `struct sock` qui porte l'état du protocole (tampons d'émission/réception, état de la connexion…). Tu n'as pas à les manipuler directement, mais il est utile de savoir qu'elles existent : c'est *là* que vivront les données en transit (Module 8). Du point de vue de **ta** table de descripteurs, en revanche, une socket apparaît exactement comme les autres entrées — avec une petite signature reconnaissable : dans `/proc/<PID>/fd`, elle pointe vers `socket:[<inode>]`.

### Exemple — créer une socket et l'observer

```c
#include <stdio.h>
#include <unistd.h>
#include <sys/socket.h>
#include <netinet/in.h>

int main(void) {
    int s = socket(AF_INET, SOCK_STREAM, 0);
    printf("socket() a renvoyé le descripteur : %d\n", s);
    printf("PID = %d — regarde /proc/%d/fd dans un autre terminal\n",
           getpid(), getpid());
    fflush(stdout);
    pause();
    close(s);
    return 0;
}
```

## Pratique

### Exercice 3.1 — Une socket prend un numéro de descripteur

Compile et lance l'exemple ci-dessus (`sock.c`).

**Question :** quel numéro `socket()` a-t-il renvoyé ? Pourquoi ce numéro précis (repense au Module 1) ?

### Exercice 3.2 — La signature d'une socket dans `/proc`

Le programme attend. **Dans un second terminal** :

```
ls -l /proc/PID/fd
```

**Question :** vers quoi pointe le descripteur de la socket ? En quoi est-ce différent d'un fichier régulier ?

### Exercice 3.3 — Plusieurs domaines, plusieurs types

```c
/* types.c */
#include <stdio.h>
#include <sys/socket.h>
int main(void) {
    int a = socket(AF_INET, SOCK_STREAM, 0);   /* TCP/IPv4   */
    int b = socket(AF_INET, SOCK_DGRAM,  0);   /* UDP/IPv4   */
    int c = socket(AF_UNIX, SOCK_STREAM, 0);   /* flux local */
    printf("AF_INET/STREAM -> fd %d\n", a);
    printf("AF_INET/DGRAM  -> fd %d\n", b);
    printf("AF_UNIX/STREAM -> fd %d\n", c);
    return 0;
}
```

**Question :** que remarques-tu sur les numéros renvoyés, quel que soit le type de socket ?

## Correction

### 3.1 — Un descripteur comme un autre

Sortie :

```
socket() a renvoyé le descripteur : 3
```

Le numéro **3**, et la raison est exactement celle du Module 1 : 0, 1 et 2 (stdin/stdout/stderr) sont pris, `socket()` reçoit donc le **plus petit descripteur libre**. Première leçon importante : `socket()` ne fait rien de « magique » côté table — il y ajoute une entrée, comme `open` l'aurait fait pour un fichier.

### 3.2 — `socket:[inode]`

`ls -l /proc/PID/fd` montre :

```
0 -> /dev/pts/0
1 -> /dev/pts/0
2 -> /dev/pts/0
3 -> socket:[123456]
```

Là où un fichier régulier afficherait un **chemin** (`/etc/hostname`), la socket affiche `socket:[<inode>]` : elle n'a pas (encore) de nom dans l'arborescence, juste un **numéro d'inode** interne au noyau. C'est la confirmation visuelle que la socket est bien un **objet noyau** rangé dans ta table de descripteurs — un objet d'un type particulier, mais logé au même endroit que les fichiers. Tout ce qu'on a appris aux Modules 1 et 2 (offset… enfin, état ; refcount ; héritage par `fork` ; CLOEXEC) **s'applique aussi aux sockets**.

### 3.3 — Toujours un descripteur

```
AF_INET/STREAM -> fd 3
AF_INET/DGRAM  -> fd 4
AF_UNIX/STREAM -> fd 5
```

Quel que soit le domaine (`AF_INET`, `AF_UNIX`) et le type (`SOCK_STREAM`, `SOCK_DGRAM`), `socket()` renvoie **un descripteur**, attribué selon la même règle du « plus petit libre » (3, 4, 5). Les différences (locale ou réseau, flux ou datagramme) ne changent **rien** à la place qu'occupe la socket dans ta table : elles changeront la façon dont on s'en sert dans les modules suivants. C'est toute la beauté de l'abstraction : un objet réseau se manipule avec les mêmes outils qu'un fichier.

## À retenir

`socket()` crée un **objet noyau** d'un nouveau type et te rend un **descripteur** ordinaire (règle du plus petit libre). Le **domaine** choisit le monde (`AF_UNIX` local, `AF_INET` réseau), le **type** choisit le mode (`SOCK_STREAM` flux fiable, `SOCK_DGRAM` datagramme). Dans `/proc/<PID>/fd`, une socket se reconnaît à `socket:[inode]`. Tout l'outillage des descripteurs s'y applique.

→ **Module 4** : on fait enfin **communiquer** deux programmes — en local d'abord, avec `AF_UNIX`, pour isoler le cycle de vie d'une socket sans la complexité du réseau.

---

# Module 4 — Sockets locales UNIX (`AF_UNIX`)

On fait enfin **dialoguer** deux programmes. On commence en **local** (`AF_UNIX`), sans réseau : pas d'adresse IP, pas de port, pas d'ordre des octets à gérer. Ça permet d'isoler ce qui compte ici — le **cycle de vie** d'une socket et la **danse des descripteurs** — avant d'ajouter le réseau au Module 5.

## Histoire

Les **sockets UNIX** (*Unix domain sockets*) servent à faire communiquer des processus **sur la même machine**. Elles sont partout sans qu'on les voie : le serveur d'affichage **X11**, le bus de messages **D-Bus**, le démon **Docker** (`/var/run/docker.sock`), ou encore **PostgreSQL** les utilisent pour leurs échanges locaux. Pourquoi pas simplement « du réseau en local » (127.0.0.1) ? Parce qu'une socket UNIX **court-circuite toute la pile réseau** (IP, TCP, calculs de sommes de contrôle) : c'est plus rapide, et ça permet en plus de s'appuyer sur les **permissions de fichiers** pour contrôler qui peut se connecter.

## Théorie

### Deux rôles : serveur et client

La communication est dissymétrique.

Le **serveur** suit ce cycle :

```
socket()  →  bind()  →  listen()  →  accept()  →  read()/write()  →  close()
```

- **`socket()`** : crée la socket (un descripteur).
- **`bind()`** : lui donne une **adresse** — ici, un **chemin de fichier** (ex. `/tmp/echo.sock`).
- **`listen()`** : la passe en mode **écoute** (elle accepte désormais des connexions).
- **`accept()`** : **bloque** jusqu'à l'arrivée d'un client, puis renvoie un **nouveau descripteur** dédié à cette connexion.
- ensuite on lit/écrit sur ce nouveau descripteur, et on le ferme.

Le **client**, lui, est plus simple :

```
socket()  →  connect()  →  read()/write()  →  close()
```

- **`connect()`** établit la liaison vers l'adresse du serveur.

### Le point clé : `accept()` fabrique un descripteur

C'est l'idée la plus importante du module, et elle découle directement de tout ce qu'on a construit. La socket sur laquelle on **écoute** (le *descripteur d'écoute*) ne sert **qu'à accepter** des connexions ; elle ne transporte aucune donnée. Chaque fois qu'un client se connecte, `accept()` crée une **nouvelle socket** — donc une **nouvelle entrée dans la table des descripteurs** — dédiée à **cette** connexion précise. Le descripteur d'écoute, lui, reste disponible pour accepter le client suivant. Un serveur qui gère trois clients aura donc, en même temps : **1** descripteur d'écoute **+ 3** descripteurs de connexion.

### L'adresse d'une socket UNIX

L'adresse se décrit avec une `struct sockaddr_un` :

```c
struct sockaddr_un addr;
memset(&addr, 0, sizeof addr);
addr.sun_family = AF_UNIX;                                  /* le domaine   */
strncpy(addr.sun_path, "/tmp/echo.sock", sizeof addr.sun_path - 1);  /* le chemin */
```

Les fonctions `bind`/`connect` attendent un `struct sockaddr *` générique ; on **transtype** (`(struct sockaddr *)&addr`) au moment de l'appel. C'est une habitude de l'API socket : une structure spécifique par famille, transtypée vers un type générique commun.

### Le serveur écho (`AF_UNIX`)

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <sys/socket.h>
#include <sys/un.h>

#define CHEMIN "/tmp/echo.sock"

int main(void) {
    int s = socket(AF_UNIX, SOCK_STREAM, 0);
    if (s < 0) { perror("socket"); exit(1); }

    struct sockaddr_un addr;
    memset(&addr, 0, sizeof addr);
    addr.sun_family = AF_UNIX;
    strncpy(addr.sun_path, CHEMIN, sizeof addr.sun_path - 1);

    unlink(CHEMIN);                       /* supprime un fichier socket résiduel */
    if (bind(s, (struct sockaddr *)&addr, sizeof addr) < 0) { perror("bind"); exit(1); }
    if (listen(s, 5) < 0) { perror("listen"); exit(1); }
    printf("Serveur en écoute sur %s (descripteur d'écoute = %d)\n", CHEMIN, s);

    for (;;) {
        int c = accept(s, NULL, NULL);    /* bloque, puis NOUVEAU descripteur */
        if (c < 0) { perror("accept"); continue; }
        printf("Nouvelle connexion sur le descripteur %d\n", c);

        char buf[256];
        ssize_t n;
        while ((n = read(c, buf, sizeof buf)) > 0)
            write(c, buf, n);             /* écho : on renvoie ce qu'on a reçu */
        close(c);                         /* le client est parti */
    }
}
```

### Le client

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <sys/socket.h>
#include <sys/un.h>

#define CHEMIN "/tmp/echo.sock"

int main(void) {
    int s = socket(AF_UNIX, SOCK_STREAM, 0);
    struct sockaddr_un addr;
    memset(&addr, 0, sizeof addr);
    addr.sun_family = AF_UNIX;
    strncpy(addr.sun_path, CHEMIN, sizeof addr.sun_path - 1);

    if (connect(s, (struct sockaddr *)&addr, sizeof addr) < 0) { perror("connect"); exit(1); }

    write(s, "Bonjour serveur", 15);
    char buf[256];
    ssize_t n = read(s, buf, sizeof buf);
    printf("Le serveur a renvoyé : %.*s\n", (int)n, buf);

    close(s);
    return 0;
}
```

## Pratique

### Exercice 4.1 — Faire tourner l'écho

Compile les deux (`serveur.c`, `client.c`), puis, dans **deux terminaux** :

```
# terminal 1
gcc serveur.c -o serveur && ./serveur

# terminal 2
gcc client.c -o client && ./client
```

**Question :** que renvoie le serveur ? Quels numéros de descripteurs vois-tu côté serveur (écoute vs connexion) ?

### Exercice 4.2 — La socket a un fichier

Le serveur tourne. Dans un autre terminal :

```
ls -l /tmp/echo.sock
```

**Question :** quel est le **tout premier caractère** de la ligne (celui qui indique le type du fichier) ? Que signifie-t-il ?

### Exercice 4.3 — Voir l'écoute et la connexion en même temps

Modifie le client pour qu'il **attende** avant de se fermer : ajoute `getchar();` juste avant `close(s);`. Lance serveur + client, laisse le client en attente, puis :

```
ls -l /proc/PID_DU_SERVEUR/fd
```

**Question :** combien de descripteurs de type `socket:[...]` vois-tu côté serveur, et à quoi correspondent-ils ?

### Exercice 4.4 — Pourquoi `unlink` avant `bind` ?

Retire la ligne `unlink(CHEMIN);` du serveur, recompile, et lance-le **deux fois de suite**.

**Question :** quelle erreur obtiens-tu au second lancement ? Pourquoi ?

## Correction

### 4.1 — L'écho fonctionne

Côté client : `Le serveur a renvoyé : Bonjour serveur`. Côté serveur :

```
Serveur en écoute sur /tmp/echo.sock (descripteur d'écoute = 3)
Nouvelle connexion sur le descripteur 4
```

Le **descripteur d'écoute** est **3** (plus petit libre après 0/1/2). À la connexion, `accept()` renvoie un **nouveau** descripteur, **4**, dédié à ce client. C'est sur le **4** qu'on lit et écrit ; le **3** reste prêt pour le prochain. Tu vois ici, en vrai, la distinction théorique : un descripteur pour *écouter*, un descripteur *par connexion*.

### 4.2 — Le `s` de *socket*

```
srwxr-xr-x 1 toi toi 0 ... /tmp/echo.sock
^
```

Le premier caractère est **`s`** : « socket ». (Pour rappel : `-` = fichier régulier, `d` = répertoire, `l` = lien, `c`/`b` = périphérique.) Le `bind()` d'une socket `AF_UNIX` crée donc une **entrée bien réelle dans le système de fichiers** : c'est ce nom que le client utilise pour la trouver via `connect()`. C'est aussi pour ça qu'on peut protéger l'accès avec des permissions Unix classiques.

### 4.3 — Un descripteur d'écoute + un par connexion

Tant que le client est en attente, `/proc/PID_DU_SERVEUR/fd` montre **deux** sockets :

```
3 -> socket:[...]      (le descripteur d'écoute)
4 -> socket:[...]      (la connexion avec ce client)
```

Si tu lançais un **deuxième** client (sans fermer le premier), tu verrais apparaître un **fd 5**, puis un **6**, etc. — un par connexion vivante, plus le descripteur d'écoute permanent. C'est la traduction concrète de « 1 écoute + N connexions ». (Avec ce serveur simple, le second client attendrait son tour : on est encore en mono-client tant qu'on ne sait pas surveiller plusieurs descripteurs — ce sera l'objet du Module 7.)

### 4.4 — `bind` veut un nom libre

Au second lancement :

```
bind: Address already in use
```

Le premier serveur a **créé** le fichier `/tmp/echo.sock`. Au second lancement, ce fichier **existe déjà**, et `bind()` refuse de réutiliser un nom occupé (erreur `EADDRINUSE`). D'où le `unlink(CHEMIN)` **avant** le `bind` : on efface une éventuelle socket résiduelle (par exemple laissée par un serveur précédent qui ne s'est pas arrêté proprement). C'est un réflexe à prendre avec `AF_UNIX`. On retrouvera une variante de ce problème en TCP au Module 5, résolue différemment (avec `SO_REUSEADDR`).

## À retenir

Le **serveur** fait `socket → bind → listen → accept → read/write → close` ; le **client** fait `socket → connect → read/write → close`. Le fait majeur : **`accept()` crée un nouveau descripteur par connexion**, distinct du descripteur d'écoute (1 écoute + N connexions). En `AF_UNIX`, `bind` crée un **vrai fichier** de type `s`, qu'il faut penser à `unlink` avant de réutiliser.

→ **Module 5** : même squelette, mais sur le **réseau** (TCP). On ajoute les adresses IP, les ports, et la fameuse question de l'**ordre des octets**.

---

# Module 5 — Sockets réseau TCP (`AF_INET`, `SOCK_STREAM`)

On reprend **exactement** le squelette du Module 4 (`socket → bind → listen → accept` côté serveur, `socket → connect` côté client), et on remplace le monde local par le **réseau**. Tout ce qui change tient à l'**adresse** : au lieu d'un chemin de fichier, on désigne une machine par une **adresse IP** et un **port** — et il faut gérer l'**ordre des octets**.

## Histoire

En 1974, Vint Cerf et Bob Kahn publient le protocole qui deviendra **TCP/IP** : un moyen de relier des réseaux hétérogènes en un « réseau de réseaux » (l'Internet). **IP** achemine des paquets d'une machine à l'autre ; **TCP** ajoute par-dessus un **flux fiable et ordonné** (les octets arrivent tous, dans l'ordre, sans doublon). Le modèle dominant est le **client-serveur** : un serveur attend à une adresse connue, des clients viennent s'y connecter. C'est ce modèle, via l'API socket de Berkeley, qu'on met en œuvre ici.

## Théorie

### Une adresse = une IP + un port

Sur le réseau, un point de terminaison se désigne par deux éléments : l'**adresse IP** (quelle machine) et le **port** (quel service/processus sur cette machine, un entier de 0 à 65535). On décrit ça avec une `struct sockaddr_in` (le `_in` = *internet*) :

```c
struct sockaddr_in addr;
memset(&addr, 0, sizeof addr);
addr.sin_family = AF_INET;                 /* le domaine : IPv4              */
addr.sin_port   = htons(8080);             /* le port, en ordre RÉSEAU      */
addr.sin_addr.s_addr = htonl(INADDR_ANY);  /* l'IP : ici, toutes les cartes */
```

- `sin_family` : la famille (`AF_INET`).
- `sin_port` : le port — mais passé par `htons()` (voir ci-dessous).
- `sin_addr` : l'adresse IP. `INADDR_ANY` signifie « écouter sur **toutes** les interfaces de la machine ».

### L'ordre des octets : `htons`, `htonl`, et compagnie

Voici le piège du réseau. Les processeurs ne stockent pas un entier multi-octets dans le même ordre : les x86/ARM courants sont **petit-boutistes** (*little-endian*, l'octet de poids faible en premier), alors que les protocoles réseau ont fixé un standard **gros-boutiste** (*big-endian*, dit « ordre réseau »). Si on envoyait le port `8080` tel quel depuis un x86, l'autre machine le lirait à l'envers.

D'où une famille de fonctions de conversion :

- **`htons`** (*host to network short*) : entier 16 bits (ports) de l'ordre **hôte** vers l'ordre **réseau**.
- **`htonl`** (*host to network long*) : entier 32 bits (adresses IPv4) hôte → réseau.
- **`ntohs`** / **`ntohl`** : la conversion **inverse** (réseau → hôte), pour relire ce qu'on reçoit.

Pour les adresses IP sous forme texte, on utilise plutôt :

- **`inet_pton`** (*presentation to network*) : `"127.0.0.1"` → forme binaire en ordre réseau.
- **`inet_ntop`** : l'inverse, binaire → texte (pour afficher l'IP d'un client).

Règle simple : **tout ce qui part sur le câble** (ports, adresses) passe par une conversion vers l'ordre réseau ; **tout ce qu'on reçoit** repasse en ordre hôte avant de l'afficher ou de le comparer.

### `SO_REUSEADDR`

Juste après la fermeture d'un serveur TCP, son port peut rester **bloqué** quelques dizaines de secondes (état `TIME_WAIT`, qu'on détaillera au Module 8). Un `bind` immédiat sur le même port échouerait alors avec `Address already in use`. L'option **`SO_REUSEADDR`**, posée avant le `bind`, autorise à réutiliser tout de suite ce port. C'est l'équivalent TCP du `unlink` qu'on faisait en `AF_UNIX` — un réflexe à avoir sur tout serveur.

### Le serveur TCP

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <sys/socket.h>
#include <netinet/in.h>
#include <arpa/inet.h>

#define PORT 8080

int main(void) {
    int s = socket(AF_INET, SOCK_STREAM, 0);
    if (s < 0) { perror("socket"); exit(1); }

    int oui = 1;
    setsockopt(s, SOL_SOCKET, SO_REUSEADDR, &oui, sizeof oui);  /* réutiliser le port */

    struct sockaddr_in addr;
    memset(&addr, 0, sizeof addr);
    addr.sin_family = AF_INET;
    addr.sin_addr.s_addr = htonl(INADDR_ANY);
    addr.sin_port = htons(PORT);

    if (bind(s, (struct sockaddr *)&addr, sizeof addr) < 0) { perror("bind"); exit(1); }
    if (listen(s, 5) < 0) { perror("listen"); exit(1); }
    printf("Serveur TCP en écoute sur le port %d (descripteur = %d)\n", PORT, s);

    for (;;) {
        struct sockaddr_in cli;
        socklen_t len = sizeof cli;
        int c = accept(s, (struct sockaddr *)&cli, &len);   /* récupère l'adresse du client */
        if (c < 0) { perror("accept"); continue; }

        char ip[INET_ADDRSTRLEN];
        inet_ntop(AF_INET, &cli.sin_addr, ip, sizeof ip);
        printf("Connexion de %s:%d sur le descripteur %d\n", ip, ntohs(cli.sin_port), c);

        char buf[256];
        ssize_t n;
        while ((n = read(c, buf, sizeof buf)) > 0)
            write(c, buf, n);
        close(c);
    }
}
```

### Le client TCP

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <sys/socket.h>
#include <netinet/in.h>
#include <arpa/inet.h>

#define PORT 8080

int main(int argc, char **argv) {
    const char *ip = (argc > 1) ? argv[1] : "127.0.0.1";

    int s = socket(AF_INET, SOCK_STREAM, 0);
    struct sockaddr_in addr;
    memset(&addr, 0, sizeof addr);
    addr.sin_family = AF_INET;
    addr.sin_port = htons(PORT);
    if (inet_pton(AF_INET, ip, &addr.sin_addr) != 1) { fprintf(stderr, "IP invalide\n"); exit(1); }

    if (connect(s, (struct sockaddr *)&addr, sizeof addr) < 0) { perror("connect"); exit(1); }

    write(s, "Bonjour serveur TCP", 19);
    char buf[256];
    ssize_t n = read(s, buf, sizeof buf);
    printf("Le serveur a renvoyé : %.*s\n", (int)n, buf);

    close(s);
    return 0;
}
```

## Pratique

### Exercice 5.1 — Le serveur, testé avec `nc`

Lance le serveur. Avant même d'écrire le client en C, teste-le avec **netcat** (un client réseau universel) :

```
# terminal 1
gcc serveur.c -o serveur && ./serveur

# terminal 2
nc 127.0.0.1 8080
# tape une ligne, valide : le serveur te la renvoie
```

**Question :** ce que tu tapes te revient-il ? Qu'affiche le serveur sur sa connexion ?

### Exercice 5.2 — Le client en C

```
gcc client.c -o client && ./client
```

**Question :** même résultat qu'avec `nc` ? (C'est le but : `nc` et ton client parlent le même TCP.)

### Exercice 5.3 — Observer les sockets avec `ss`

Le serveur tourne. Dans un autre terminal :

```
ss -tlnp        # sockets TCP en écoute (-t TCP, -l listen, -n numérique, -p processus)
```

Puis, **pendant** qu'un client est connecté (utilise le `nc` laissé ouvert) :

```
ss -tnp         # connexions TCP établies
```

**Question :** dans quel état est la socket d'écoute ? Et la connexion active ?

### Exercice 5.4 — Voir l'inversion d'octets

```c
/* byteorder.c */
#include <stdio.h>
#include <arpa/inet.h>
int main(void) {
    unsigned short x = 0x1234;
    printf("hôte   : 0x%04x\n", x);
    printf("réseau : 0x%04x\n", htons(x));
    return 0;
}
```

**Question :** les deux valeurs sont-elles identiques ? Que s'est-il passé, octet par octet ?

## Correction

### 5.1 / 5.2 — `nc` et le client C, même combat

Côté serveur :

```
Serveur TCP en écoute sur le port 8080 (descripteur = 3)
Connexion de 127.0.0.1:51058 sur le descripteur 4
```

On retrouve **trait pour trait** la structure du Module 4 : descripteur d'écoute **3**, descripteur de connexion **4** issu d'`accept()`. La seule nouveauté est l'**adresse du client** récupérée dans `cli` : `127.0.0.1` (l'IP de **loopback**, la machine qui se parle à elle-même) et un **port source** (ici `51058`) que le système attribue automatiquement au client. Que tu utilises `nc` ou ton client C ne change rien : tous deux ouvrent une socket TCP et parlent le même protocole — d'où le même écho. C'est l'intérêt de tester avec `nc` : il isole le serveur du client.

### 5.3 — Les états vus par `ss`

La socket d'écoute apparaît en état **`LISTEN`** :

```
State    Local Address:Port    Process
LISTEN   0.0.0.0:8080          users:(("serveur",pid=...,fd=3))
```

`0.0.0.0:8080` = « j'écoute sur **toutes** les interfaces, port 8080 » (c'est le `INADDR_ANY` du code), et `ss` indique même le **descripteur 3** côté processus. Pendant qu'un client est connecté, `ss -tnp` montre une ligne en état **`ESTAB`** (*established*) :

```
State   Local Address:Port      Peer Address:Port
ESTAB   127.0.0.1:8080          127.0.0.1:51058
```

Tu lis là, en clair, les **deux bouts** de la connexion : l'extrémité serveur (`:8080`) et l'extrémité client (`:51058`). Une connexion TCP est identifiée de façon unique par ce **quadruplet** (IP source, port source, IP destination, port destination) — c'est ce qui permet à un serveur de distinguer mille clients connectés sur le même port 8080. *(Si `ss` n'est pas installé : `sudo apt install iproute2`.)*

### 5.4 — L'octet de poids fort d'abord

```
hôte   : 0x1234
réseau : 0x3412
```

La valeur `0x1234` est devenue `0x3412` : les **deux octets ont été inversés**. Sur ton x86 (petit-boutiste), `0x1234` est rangé en mémoire comme `34 12` (poids faible d'abord). L'ordre réseau (gros-boutiste) impose `12 34` (poids fort d'abord) ; `htons` effectue donc l'échange. **Pourquoi tout ce cirque ?** Parce que deux machines d'architectures différentes doivent s'entendre : si chacune envoyait ses entiers « comme chez elle », un port ou une IP serait mal interprété à l'arrivée. En fixant **un** ordre commun sur le câble (le big-endian), le réseau garantit l'interopérabilité — et `htons`/`ntohs` sont les traducteurs. *(Sur une machine déjà big-endian, ces fonctions ne font simplement rien : le code reste correct partout.)*

## À retenir

Le squelette TCP est **identique** à celui d'`AF_UNIX` ; seule l'**adresse** change : `struct sockaddr_in` = famille `AF_INET` + **port** + **IP**. Tout ce qui part sur le réseau passe en **ordre réseau** (`htons`/`htonl`, `inet_pton`) ; tout ce qu'on reçoit revient en **ordre hôte** (`ntohs`/`ntohl`, `inet_ntop`). On pose **`SO_REUSEADDR`** avant `bind`. `ss -tlnp` (écoute) et `ss -tnp` (établies) donnent une vue directe des sockets, identifiées par le quadruplet IP:port des deux côtés.

→ **Module 6** : on passe au mode **sans connexion** avec UDP, et on mesure ce que `listen`/`accept`/`connect` faisaient vraiment pour nous.

---

# Module 6 — Sockets UDP (`SOCK_DGRAM`)

TCP nous donnait un **flux** fiable et ordonné, au prix d'une **connexion** à établir. UDP propose l'inverse : des **messages** indépendants, sans connexion, sans garantie — mais simples et rapides. En le programmant, on comprend **a contrario** tout ce que `listen`, `accept` et `connect` apportaient.

## Histoire

UDP (*User Datagram Protocol*) est le « petit frère minimaliste » de TCP. Là où TCP s'assure que tout arrive dans l'ordre, UDP se contente d'**envoyer un message et de passer à autre chose** : pas d'établissement de connexion, pas de retransmission, pas de garantie d'ordre. Ce dépouillement est un **atout** quand la rapidité prime sur la fiabilité parfaite : le **DNS** (une question, une réponse, terminé), les **jeux en ligne** (mieux vaut la position la plus récente qu'une vieille position retransmise), la **voix et la vidéo en direct**. C'est aussi la base de protocoles modernes comme **QUIC** (le transport de HTTP/3), qui rebâtissent leur propre fiabilité au-dessus d'UDP.

## Théorie

### Le datagramme

Un **datagramme** est un message **autonome** : il part avec son adresse de destination, et le réseau le livre… ou pas (sans prévenir). Il n'y a **pas de connexion**, donc pas de notion de « flux » ni d'offset qui progresse : chaque `recvfrom` te rend **un** datagramme entier, tel qu'il a été envoyé.

Conséquence directe sur l'API : on **n'utilise ni `listen`, ni `accept`, ni (en général) `connect`**. Il n'y a pas de connexion à mettre en écoute ni à accepter. À la place, deux appels portent **l'adresse à chaque message** :

- **`sendto(fd, buf, len, flags, dest, destlen)`** : envoie un datagramme **à** `dest`.
- **`recvfrom(fd, buf, len, flags, src, srclen)`** : reçoit un datagramme, et **remplit `src`** avec l'adresse de l'expéditeur (pratique pour lui répondre).

Le cycle est donc beaucoup plus court :

```
Serveur :  socket()  →  bind()  →  recvfrom()/sendto()  (en boucle)
Client  :  socket()  →  sendto()/recvfrom()
```

Le serveur fait toujours `bind` (il faut bien un port connu où le joindre), mais s'arrête là côté « mise en place » : pas de `listen`, pas d'`accept`, **pas de descripteur par client**. Un **unique** descripteur reçoit les datagrammes de **tout le monde** — et c'est `recvfrom` qui dit, à chaque message, qui l'a envoyé.

### Le serveur UDP

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <sys/socket.h>
#include <netinet/in.h>

#define PORT 9090

int main(void) {
    int s = socket(AF_INET, SOCK_DGRAM, 0);          /* DGRAM, pas STREAM */
    if (s < 0) { perror("socket"); exit(1); }

    struct sockaddr_in addr;
    memset(&addr, 0, sizeof addr);
    addr.sin_family = AF_INET;
    addr.sin_addr.s_addr = htonl(INADDR_ANY);
    addr.sin_port = htons(PORT);

    if (bind(s, (struct sockaddr *)&addr, sizeof addr) < 0) { perror("bind"); exit(1); }
    printf("Serveur UDP en attente sur le port %d\n", PORT);

    for (;;) {
        char buf[256];
        struct sockaddr_in cli;
        socklen_t len = sizeof cli;
        ssize_t n = recvfrom(s, buf, sizeof buf, 0, (struct sockaddr *)&cli, &len);
        if (n < 0) { perror("recvfrom"); continue; }
        sendto(s, buf, n, 0, (struct sockaddr *)&cli, len);   /* on répond à l'expéditeur */
    }
}
```

### Le client UDP

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <sys/socket.h>
#include <netinet/in.h>
#include <arpa/inet.h>

#define PORT 9090

int main(void) {
    int s = socket(AF_INET, SOCK_DGRAM, 0);
    struct sockaddr_in addr;
    memset(&addr, 0, sizeof addr);
    addr.sin_family = AF_INET;
    addr.sin_port = htons(PORT);
    inet_pton(AF_INET, "127.0.0.1", &addr.sin_addr);

    sendto(s, "Datagramme de test", 18, 0, (struct sockaddr *)&addr, sizeof addr);

    char buf[256];
    ssize_t n = recvfrom(s, buf, sizeof buf, 0, NULL, NULL);
    printf("Le serveur a renvoyé : %.*s\n", (int)n, buf);

    close(s);
    return 0;
}
```

## Pratique

### Exercice 6.1 — L'écho UDP

Dans deux terminaux :

```
gcc serveur_udp.c -o serveur_udp && ./serveur_udp
gcc client_udp.c  -o client_udp  && ./client_udp
```

**Question :** le datagramme revient-il ? Compare le code à celui du Module 5 : qu'est-ce qui a **disparu** ?

### Exercice 6.2 — Tester avec `nc -u`

```
nc -u 127.0.0.1 9090
# tape une ligne, valide : elle te revient
```

**Question :** l'option `-u` bascule `nc` en UDP. L'écho fonctionne-t-il comme en TCP ?

### Exercice 6.3 — Pas de connexion : la preuve

Observe d'abord la socket :

```
ss -ulnp        # -u = UDP (au lieu de -t)
```

Puis fais l'expérience suivante : **arrête le serveur** (Ctrl-C), et **ensuite** lance le client.

**Question :** en TCP, un client sans serveur reçoit `Connection refused` dès le `connect`. Que se passe-t-il ici côté client au `sendto` ? Pourquoi cette différence ?

## Correction

### 6.1 — Ce qui a disparu

Côté client : `Le serveur a renvoyé : Datagramme de test`. En comparant au Module 5, trois choses ont **disparu** côté serveur : `listen()`, `accept()`, et **le descripteur de connexion**. Il ne reste qu'**un seul** descripteur (celui du `socket()` initial), sur lequel transitent **tous** les datagrammes, de tous les clients. Là où TCP fabriquait un descripteur par client (via `accept`), UDP n'en a aucun à fabriquer : il n'y a pas de connexion à matérialiser. La contrepartie, c'est que le serveur doit **lui-même** noter qui lui a parlé — d'où l'adresse `cli` remplie par `recvfrom` et réutilisée dans `sendto`.

### 6.2 — Même écho, autre transport

Oui, `nc -u` obtient le même écho. Du point de vue de l'utilisateur, ça ressemble à TCP ; mais sous le capot, chaque ligne tapée part comme un **datagramme indépendant**, sans connexion préalable. Tu pourrais d'ailleurs envoyer un datagramme, attendre une heure, en envoyer un autre : il n'y a aucun « lien » à maintenir ouvert entre les deux.

### 6.3 — L'absence de connexion expliquée

`ss -ulnp` montre la socket en état **`UNCONN`** (*unconnected*) — et non `LISTEN`. Ce mot dit tout : il n'y a, par nature, **pas de connexion** côté UDP. La socket est juste « posée » sur le port 9090, prête à recevoir n'importe quel datagramme.

L'expérience « serveur arrêté » est la plus parlante. En **TCP**, `connect` négocie une connexion avec le serveur (la fameuse *poignée de main*) : si personne n'écoute, la machine distante répond par un refus, et le client obtient immédiatement `Connection refused`. En **UDP**, `sendto` ne négocie **rien** : il **dépose** son datagramme sur le réseau et rend la main aussitôt, **sans erreur**, que quelqu'un écoute ou non. Le datagramme part « dans le vide », et le client reste ensuite **bloqué** dans `recvfrom`, à attendre une réponse qui ne viendra jamais. *(En loopback strict, le noyau peut parfois renvoyer une erreur a posteriori, mais le principe tient : UDP ne garantit ni la connexion, ni la livraison, ni même de te prévenir d'un échec.)*

C'est la démonstration concrète de ce que TCP t'offrait gratuitement : `connect`/`accept` **créaient et vérifiaient** un canal (l'objet « connexion » côté noyau), avec un interlocuteur confirmé des deux côtés. UDP supprime cet objet — d'où l'absence d'`accept`, l'absence de descripteur par client, et l'absence d'erreur quand le destinataire est absent. Moins de garanties, mais moins de coût et plus de souplesse.

## À retenir

UDP = **datagrammes** sans connexion. On garde `socket` et (côté serveur) `bind`, mais on **abandonne** `listen`/`accept`/`connect` et le descripteur par client : un **unique** descripteur sert tout le monde. L'adresse voyage **dans chaque appel**, via `sendto`/`recvfrom`. Conséquence : pas d'objet « connexion » côté noyau, donc pas d'erreur si le destinataire est absent — l'exact opposé des garanties de TCP.

→ **Module 7** : revenons à TCP, mais avec un vrai problème de serveur — **gérer N clients à la fois** avec un seul processus, grâce au multiplexage (`epoll`).

---

# Module 7 — Multiplexage : gérer N descripteurs à la fois

Nos serveurs des modules 4 et 5 avaient un défaut : un `read` (ou un `accept`) **bloque** le processus tant que rien n'arrive. Ils ne savent donc servir qu'**un client à la fois**. Comment, avec **un seul** processus, surveiller des **dizaines de milliers** de descripteurs et réagir à celui qui devient prêt ? C'est le **multiplexage d'entrées/sorties**, et c'est le couronnement de toute la pyramide : il n'est possible *que* parce que les descripteurs sont des objets de première classe, rangés dans une table.

## Histoire

À la fin des années 1990, les serveurs web font face au **problème C10k** : tenir **10 000 connexions** simultanées sur une seule machine. La solution naïve — un processus (ou un fil) par client — s'effondre sous le coût mémoire et les changements de contexte. La voie qui passe à l'échelle est l'**approche événementielle** : un seul fil qui surveille tous les descripteurs et ne traite que ceux qui sont prêts. L'outillage a évolué : **`select`** (issu de BSD, 1983), puis **`poll`**, puis enfin **`epoll`** (Linux 2.6, vers 2002), conçu précisément pour rester efficace avec des dizaines de milliers de descripteurs. C'est cette mécanique qui fait tourner **nginx**, **Node.js** ou **Redis**.

## Théorie

### Le problème : l'appel bloquant

Quand le serveur du Module 5 fait `read(c, ...)`, il **s'endort** jusqu'à ce que *ce* client envoie quelque chose. Pendant ce sommeil, il est incapable d'accepter un nouveau client ou de servir les autres. Lancer un processus par client (`fork`) répartit le problème mais coûte cher. L'idée du multiplexage est différente : **ne jamais bloquer sur un seul descripteur**, mais demander au noyau « parmi **tout cet ensemble** de descripteurs, **lesquels** sont prêts ? », puis n'agir que sur ceux-là.

### `select`, `poll`, `epoll` : trois générations

- **`select(nfds, &readfds, ...)`** : on remplit un ensemble (**`fd_set`**) avec les descripteurs à surveiller (`FD_SET`), on appelle `select`, et au retour on teste lesquels sont prêts (`FD_ISSET`). Simple, mais deux limites : un plafond (`FD_SETSIZE`, souvent 1024 descripteurs) et un coût qui **croît avec le nombre** de descripteurs (le noyau les reparcourt tous à chaque appel).
- **`poll(fds[], nfds, timeout)`** : même idée avec un **tableau** de `struct pollfd` (plus de plafond fixe), mais toujours un balayage **linéaire** à chaque appel.
- **`epoll`** (Linux) : on **enregistre une fois** les descripteurs dans un objet noyau (l'*instance epoll*), et le noyau te rend directement **la liste de ceux qui sont prêts**, sans tout reparcourir. Le coût ne dépend plus du nombre total surveillé : c'est ce qui passe à l'échelle.

### Les trois appels d'`epoll`

- **`int ep = epoll_create1(0);`** crée l'instance epoll — **et te rend un descripteur** (l'instance epoll est, elle aussi, un objet noyau dans ta table !).
- **`epoll_ctl(ep, EPOLL_CTL_ADD, fd, &ev);`** ajoute un descripteur `fd` à surveiller (et `EPOLL_CTL_MOD` / `EPOLL_CTL_DEL` pour modifier / retirer).
- **`int n = epoll_wait(ep, events, max, timeout);`** **bloque** jusqu'à ce qu'au moins un descripteur surveillé soit prêt, puis remplit `events` avec **uniquement** les descripteurs prêts.

*(En passant : `epoll` propose deux modes de réveil, le mode **niveau** — par défaut, « réveille-moi tant qu'il reste des données » — et le mode **front** — `EPOLLET`, « réveille-moi une seule fois à l'arrivée ». On reste ici en mode niveau, plus simple.)*

### La boucle d'événements

Le schéma mental est limpide : on tient **un ensemble de descripteurs** (le descripteur d'écoute + un par client connecté), on demande au noyau lesquels sont prêts, et on distingue deux cas :

```
        ┌─────────────────────────────────────────────┐
        │  epoll_wait : « qui est prêt ? »             │
        └───────────────┬─────────────────────────────┘
                        │
         ┌──────────────┴───────────────┐
         ▼                              ▼
  c'est le descripteur          c'est un descripteur
  d'ÉCOUTE qui est prêt         de CLIENT qui est prêt
  → accept() : nouveau          → read() : on lit, on
    client, on l'AJOUTE           renvoie l'écho (et si
    à l'ensemble surveillé        read=0, on le RETIRE)
```

### Le serveur écho `epoll` (multi-clients, un seul processus)

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <sys/socket.h>
#include <netinet/in.h>
#include <sys/epoll.h>

#define PORT 8080
#define MAX_EVENTS 64

int main(void) {
    int s = socket(AF_INET, SOCK_STREAM, 0);
    int oui = 1;
    setsockopt(s, SOL_SOCKET, SO_REUSEADDR, &oui, sizeof oui);

    struct sockaddr_in addr;
    memset(&addr, 0, sizeof addr);
    addr.sin_family = AF_INET;
    addr.sin_addr.s_addr = htonl(INADDR_ANY);
    addr.sin_port = htons(PORT);
    bind(s, (struct sockaddr *)&addr, sizeof addr);
    listen(s, 16);
    printf("Serveur epoll en écoute sur le port %d\n", PORT);

    int ep = epoll_create1(0);
    struct epoll_event ev;
    ev.events = EPOLLIN;
    ev.data.fd = s;                                  /* on surveille le descripteur d'écoute */
    epoll_ctl(ep, EPOLL_CTL_ADD, s, &ev);

    struct epoll_event events[MAX_EVENTS];
    for (;;) {
        int n = epoll_wait(ep, events, MAX_EVENTS, -1);   /* attend qu'au moins un soit prêt */
        for (int i = 0; i < n; i++) {
            int fd = events[i].data.fd;

            if (fd == s) {                           /* le descripteur d'écoute est prêt */
                int c = accept(s, NULL, NULL);       /* → nouvelle connexion */
                ev.events = EPOLLIN;
                ev.data.fd = c;
                epoll_ctl(ep, EPOLL_CTL_ADD, c, &ev);/* on l'ajoute à la surveillance */
                printf("Nouveau client sur le descripteur %d\n", c);
            } else {                                 /* un client a envoyé des données */
                char buf[256];
                ssize_t r = read(fd, buf, sizeof buf);
                if (r <= 0) {                        /* déconnexion : on le retire */
                    epoll_ctl(ep, EPOLL_CTL_DEL, fd, NULL);
                    close(fd);
                    printf("Client %d déconnecté\n", fd);
                } else {
                    write(fd, buf, r);               /* écho */
                }
            }
        }
    }
}
```

## Pratique

### Exercice 7.1 — Plusieurs clients à la fois

Lance le serveur, puis ouvre **plusieurs** clients **simultanément** dans des terminaux séparés :

```
# terminal 1
gcc serveur_epoll.c -o serveur_epoll && ./serveur_epoll

# terminaux 2, 3, 4 ...
nc 127.0.0.1 8080
```

Tape dans chacun : chaque ligne te revient, **indépendamment**, sans qu'un client n'en bloque un autre.

**Question :** combien de **processus** le serveur lance-t-il pour gérer tous ces clients ? Vérifie :

```
ps -C serveur_epoll          # combien de lignes ?
```

### Exercice 7.2 — Les descripteurs qui s'accumulent

Pendant que plusieurs clients sont connectés :

```
ls -l /proc/PID_DU_SERVEUR/fd
```

**Question :** combien de `socket:[...]` vois-tu, et comment ce nombre évolue-t-il quand un client se connecte ou se déconnecte ?

### Exercice 7.3 (bonus) — La comparaison qui fait mal

Reprends le serveur **bloquant** du Module 5. Connecte un premier `nc` et **ne tape rien**. Connecte un second `nc` et essaie d'envoyer une ligne.

**Question :** le second client est-il servi ? Compare avec le serveur `epoll`. Que fait le serveur du Module 5 pendant que le premier client se tait ?

## Correction

### 7.1 — Un seul processus pour tous

Le serveur affiche, au fil des connexions :

```
Serveur epoll en écoute sur le port 8080
Nouveau client sur le descripteur 5
Nouveau client sur le descripteur 6
```

Et `ps -C serveur_epoll` ne montre qu'**une seule** ligne : **un unique processus** (donc un seul fil d'exécution) sert tous les clients. C'est tout l'intérêt par rapport à l'approche « un `fork` par client » : aucune multiplication de processus, une empreinte mémoire minime, et pourtant des clients servis **en parallèle** du point de vue de l'utilisateur. Le secret n'est pas le parallélisme matériel, mais le fait de **ne jamais bloquer** sur un seul descripteur : `epoll_wait` rend la main dès que *n'importe lequel* est prêt, et la boucle traite chacun à tour de rôle, très vite.

**Pourquoi les clients démarrent-ils au descripteur 5, et pas 3 ?** Excellente occasion de relier au Module 1. Comptons les descripteurs ouverts par le serveur : 0/1/2 (stdin/out/err), **3** = la socket d'écoute (`socket()`), **4** = … l'**instance epoll** elle-même ! Car `epoll_create1` **consomme un descripteur** (l'instance epoll est un objet noyau, logé dans la table comme les autres). Le premier client accepté reçoit donc le plus petit libre suivant : **5**, puis **6**, etc. Le fait que l'objet qui *surveille* les descripteurs soit lui-même *un* descripteur n'est pas une bizarrerie — c'est la cohérence du modèle « tout est descripteur » poussée jusqu'au bout.

### 7.2 — L'ensemble surveillé, rendu visible

`/proc/PID/fd` reflète **en direct** l'ensemble que gère `epoll`. Au repos, tu vois 2 sockets : le descripteur d'écoute (3) et l'instance epoll (4, qui apparaît comme `anon_inode:[eventpoll]`). Chaque client connecté **ajoute** un `socket:[...]` (5, 6, 7…) ; chaque déconnexion le fait **disparaître** (grâce au `EPOLL_CTL_DEL` + `close` de la boucle). Tu observes donc, fichier par fichier, la **table des descripteurs** grandir et rétrécir au rythme des connexions — la concrétisation visuelle de « 1 écoute + 1 instance epoll + N connexions ».

### 7.3 — Ce que le multiplexage résout

Avec le serveur **bloquant** du Module 5 : le premier `nc` se connecte, le serveur entre dans son `read(c, ...)` et **s'y endort** puisque ce client ne dit rien. Le second `nc` peut bien se connecter (le noyau met sa connexion en file d'attente), mais le serveur, **coincé dans `read`**, n'appellera pas `accept` pour lui : le second client reste **ignoré** tant que le premier n'a pas parlé puis quitté. Avec `epoll`, ce blocage est impossible : le serveur n'attend jamais *un* client en particulier, il attend *l'ensemble*, et traite immédiatement celui qui devient prêt. C'est précisément ce qui sépare un serveur jouet d'un serveur capable d'encaisser des milliers de connexions.

## À retenir

Le multiplexage permet à **un seul fil** de gérer **N descripteurs** sans bloquer sur aucun : on tient un **ensemble** de descripteurs, le noyau dit lesquels sont **prêts**, on agit. `select`/`poll` reparcourent tout à chaque appel ; **`epoll`** enregistre une fois et passe à l'échelle. Ses trois appels : `epoll_create1` (qui **consomme lui-même un descripteur**), `epoll_ctl` (ADD/MOD/DEL), `epoll_wait`. Sans la table des descripteurs des modules 1-3, rien de tout cela n'aurait de sens : multiplexer, c'est manipuler un **ensemble de descripteurs** comme une donnée.

→ **Module 8** (optionnel) : on descend sous l'API, dans les **tampons du noyau** et la **machine à états de TCP**, pour comprendre `TIME_WAIT`, `SO_REUSEADDR` et le mode non bloquant.

---

# Module 8 (optionnel) — Plongée dans le noyau

L'API socket cache une machinerie. Ce module ouvre le capot pour relier ce qu'on a manipulé « par-dessus » à ce qui se passe « en dessous » : les **tampons** où vivent les données, la **machine à états** de TCP, et les détails qui expliquent enfin `TIME_WAIT`, `SO_REUSEADDR` et le mode non bloquant.

## Histoire

Quand tu fais `send`, les octets ne partent pas instantanément sur le câble. Le noyau les **recopie** dans un tampon interne et les transmet à son rythme, en respectant le **protocole TCP** — une véritable **machine à états** (établissement, transfert, fermeture ordonnée) standardisée dès 1981 (RFC 793). Les outils `strace`, `ss` et `/proc/net/tcp` permettent d'observer cette mécanique en vrai. C'est le bon moment : on a tous les concepts pour la lire.

## Théorie

### Les tampons : où vivent les données

Chaque socket possède, côté noyau, un **tampon d'émission** (`SO_SNDBUF`) et un **tampon de réception** (`SO_RCVBUF`). Le trajet d'un octet est donc :

```
  ton tampon (espace utilisateur)
        │  write()/send()  → RECOPIE vers le noyau
        ▼
  tampon d'émission du noyau  ──(TCP, à son rythme)──►  réseau
                                                          │
  tampon de réception du noyau ◄────────────────────────┘
        │  read()/recv()   → RECOPIE vers l'utilisateur
        ▼
  ton tampon (espace utilisateur)
```

C'est la frontière du Module 0 qui réapparaît : `send` ne « met pas sur le réseau », il **recopie dans le noyau**, qui se charge du reste. À l'intérieur, le noyau range les données dans des structures appelées **`sk_buff`**. Deux conséquences pratiques : `send` peut **réussir** alors que rien n'est encore arrivé à destination (les octets sont juste dans le tampon noyau) ; et si le tampon d'émission est plein (récepteur lent), `send` **bloque** (ou échoue avec `EAGAIN` en mode non bloquant, voir plus bas).

### La machine à états de TCP

Une connexion TCP traverse une série d'**états**, qu'on peut tous voir avec `ss -tan`. Les principaux :

- **`LISTEN`** : le serveur attend des connexions (après `listen()`).
- **`SYN-SENT` / `SYN-RECV`** : la *poignée de main* en trois temps (SYN → SYN-ACK → ACK) est en cours — c'est ce que `connect`/`accept` orchestrent.
- **`ESTABLISHED`** : la connexion est ouverte, les données circulent.
- **`FIN-WAIT` / `CLOSE-WAIT` / `LAST-ACK`** : la fermeture ordonnée (chaque camp annonce qu'il a fini).
- **`TIME-WAIT`** : l'état « d'après fermeture », détaillé juste en dessous.

### `TIME_WAIT` et `SO_REUSEADDR`, enfin élucidés

Quand le camp qui **ferme en premier** a terminé, sa socket ne disparaît pas tout de suite : elle reste en **`TIME_WAIT`** pendant un délai (typiquement quelques dizaines de secondes). **Pourquoi ?** Pour deux raisons : (1) s'assurer que le dernier accusé de réception est bien arrivé en face, et (2) laisser « mourir » sur le réseau d'éventuels paquets retardataires de cette connexion, afin qu'ils ne soient pas pris par erreur pour une **nouvelle** connexion réutilisant le même couple d'adresses/ports.

C'est précisément ce qui explique le `Address already in use` au redémarrage d'un serveur : son ancien port est encore retenu en `TIME_WAIT`. L'option **`SO_REUSEADDR`** dit au noyau « autorise-moi à me lier à ce port même s'il traîne un `TIME_WAIT` » — ce qui est sûr pour un serveur qu'on relance. On boucle ainsi la boucle ouverte au Module 5 : `SO_REUSEADDR` n'est pas une formule magique, c'est la réponse à un comportement précis de la machine à états.

### Bloquant vs non bloquant

Par défaut, une socket est **bloquante** : `recv` sans donnée **endort** le processus jusqu'à ce que des octets arrivent. En posant le drapeau **`O_NONBLOCK`** (`fcntl(fd, F_SETFL, O_NONBLOCK)`), on la rend **non bloquante** : `recv` qui n'a rien à lire **revient immédiatement** avec `-1` et `errno == EAGAIN` (alias `EWOULDBLOCK`), au lieu d'attendre. Ce mode est le **compagnon naturel d'`epoll`** : la boucle d'événements ne doit **jamais** se bloquer, donc ses sockets sont en non bloquant, et `EAGAIN` signifie simplement « plus rien à lire pour l'instant, repasse plus tard ».

### Deux petits programmes d'observation

```c
/* bufsize.c — taille des tampons d'une socket */
#include <stdio.h>
#include <sys/socket.h>
#include <netinet/in.h>
int main(void) {
    int s = socket(AF_INET, SOCK_STREAM, 0);
    int val; socklen_t len = sizeof val;
    getsockopt(s, SOL_SOCKET, SO_RCVBUF, &val, &len);
    printf("Tampon de réception (SO_RCVBUF) : %d octets\n", val);
    getsockopt(s, SOL_SOCKET, SO_SNDBUF, &val, &len);
    printf("Tampon d'émission   (SO_SNDBUF) : %d octets\n", val);
    return 0;
}
```

```c
/* nonblock.c — recv non bloquant qui ne trouve rien */
#include <stdio.h>
#include <errno.h>
#include <string.h>
#include <fcntl.h>
#include <unistd.h>
#include <sys/socket.h>
#include <netinet/in.h>
int main(void) {
    int s = socket(AF_INET, SOCK_DGRAM, 0);
    struct sockaddr_in addr;
    memset(&addr, 0, sizeof addr);
    addr.sin_family = AF_INET;
    addr.sin_addr.s_addr = htonl(INADDR_LOOPBACK);
    addr.sin_port = htons(9099);
    bind(s, (struct sockaddr *)&addr, sizeof addr);

    fcntl(s, F_SETFL, O_NONBLOCK);                 /* mode non bloquant */
    char buf[64];
    ssize_t n = recvfrom(s, buf, sizeof buf, 0, NULL, NULL);
    if (n < 0)
        printf("recvfrom : rien à lire, errno=%d (%s)\n", errno, strerror(errno));
    close(s);
    return 0;
}
```

## Pratique

### Exercice 8.1 — `strace` d'une session TCP complète

Relance le serveur TCP du Module 5 **sous strace**, connecte un client, puis observe :

```
strace -e trace=network,read,write,close ./serveur
```

**Question :** retrouves-tu, dans l'ordre, `socket` → `bind` → `listen` → `accept` → `read` → `write` → `close` ? C'est toute la pyramide en une trace.

### Exercice 8.2 — Voir `TIME_WAIT`

Lance le serveur TCP, connecte puis ferme un client, et **tout de suite** :

```
ss -tan | grep 8080
```

**Question :** vois-tu une ligne en état `TIME-WAIT` ? Que devient-elle après ~30-60 s ? Et que se passe-t-il si tu relances un serveur **sans** `SO_REUSEADDR` pendant ce délai ?

### Exercice 8.3 — La taille des tampons

```
gcc bufsize.c -o bufsize && ./bufsize
```

**Question :** quelles tailles obtiens-tu ? (Indice : elles dépendent du réglage système, et Linux **double** souvent la valeur en interne pour sa propre comptabilité.)

### Exercice 8.4 — Le mode non bloquant

```
gcc nonblock.c -o nonblock && ./nonblock
```

**Question :** quel `errno` obtiens-tu, et que signifie-t-il ?

### Exercice 8.5 (bonus) — Décoder `/proc/net/tcp`

```
cat /proc/net/tcp | head
```

**Question :** les colonnes `local_address` et `rem_address` sont en **hexadécimal** (`IP:port`). Repère ton serveur sur le port 8080 (`0x1F90`). Quel est le code d'état dans la colonne `st` ? *(Indice : `0A` = LISTEN, `01` = ESTABLISHED, `06` = TIME_WAIT.)*

## Correction

### 8.1 — La pyramide dans une trace

`strace` déroule très exactement la séquence du tutoriel : `socket(AF_INET, SOCK_STREAM, ...)` renvoie le fd 3 ; `bind` et `listen` le préparent ; `accept` rend le fd 4 à la connexion ; puis les `read`/`write` font la navette (chacun étant une **recopie** à travers la frontière du Module 0), et `close` libère la connexion. Tu **vois** ainsi que les jolis appels C de haut niveau ne sont rien d'autre que des **appels système** franchissant la frontière utilisateur/noyau — exactement ce qu'on observait dès le Module 0 avec un simple `write`.

### 8.2 — `TIME_WAIT` en action

Juste après la fermeture, `ss -tan` montre une ligne :

```
State       Local Address:Port     Peer Address:Port
TIME-WAIT   127.0.0.1:8080         127.0.0.1:51058
```

Elle **disparaît** d'elle-même après le délai (quelques dizaines de secondes). Pendant ce temps, relancer le serveur **sans** `SO_REUSEADDR` échoue avec `bind: Address already in use` : le port est encore « retenu » par le `TIME_WAIT`. **Avec** `SO_REUSEADDR` (comme dans notre code des modules 5 et 7), le `bind` réussit immédiatement. C'est l'explication complète, promise au Module 5, du réflexe `SO_REUSEADDR`.

### 8.3 — Les tampons

Valeurs typiques observées (elles **varient** selon le système) :

```
Tampon de réception (SO_RCVBUF) : 131072 octets
Tampon d'émission   (SO_SNDBUF) : 16384 octets
```

Ces nombres sont les tailles que le noyau réserve, **par socket**, pour mettre en file les données entrantes et sortantes. Ils expliquent des comportements concrets : un `send` de quelques Ko « réussit » instantanément parce qu'il ne fait que **remplir le tampon d'émission** (le réseau videra ce tampon ensuite) ; et si un récepteur lent laisse le tampon d'émission se remplir, les `send` suivants finissent par bloquer. *(Linux renvoie souvent une valeur doublée par rapport à ce que tu fixerais, car il compte aussi son propre surcoût de gestion.)*

### 8.4 — `EAGAIN`

```
recvfrom : rien à lire, errno=11 (Resource temporarily unavailable)
```

`errno == 11` est **`EAGAIN`** (≡ `EWOULDBLOCK`). En mode **bloquant**, ce `recvfrom` aurait **attendu** un datagramme. En mode **non bloquant** (`O_NONBLOCK`), il **refuse d'attendre** et signale aussitôt « rien pour l'instant ». Ce n'est pas une vraie erreur : c'est le signal normal qui dit à une boucle événementielle « repasse plus tard ». C'est exactement le contrat sur lequel reposent les serveurs `epoll` du Module 7 : ne jamais bloquer, traiter ce qui est prêt, ignorer poliment les `EAGAIN`.

### 8.5 — Le quadruplet en hexadécimal

Dans `/proc/net/tcp`, chaque ligne est une socket. `local_address` se lit `IP:port` en hexa : le port 8080 s'écrit `0x1F90`. La colonne `st` donne l'état : tu verras `0A` (LISTEN) pour ton serveur au repos, `01` (ESTABLISHED) pour une connexion active, `06` (TIME_WAIT) juste après une fermeture. C'est la **source brute** que des outils comme `ss` se contentent de mettre en forme. Savoir la lire, c'est toucher le fond de la pyramide : la table des connexions, telle que le noyau la tient.

## À retenir

Sous l'API, le noyau gère des **tampons** (`SO_SNDBUF`/`SO_RCVBUF`) : `send`/`recv` **recopient** les données à travers la frontière utilisateur/noyau (rappel du Module 0), le réseau étant servi de façon asynchrone. TCP est une **machine à états** (`LISTEN`, `ESTABLISHED`, `TIME_WAIT`…) qu'on lit avec `ss -tan` ou `/proc/net/tcp`. **`TIME_WAIT`** retient brièvement un port après fermeture, ce que **`SO_REUSEADDR`** permet de contourner. Le mode **`O_NONBLOCK`** fait revenir `recv` immédiatement avec **`EAGAIN`** quand il n'y a rien — fondation du modèle `epoll`.

---

# Conclusion — La pyramide, du sommet

Reparcourons la montée, maintenant qu'on est en haut :

- **M0** — Un **processus** a une mémoire privée et dialogue avec le noyau par **appels système**, à travers une **frontière**.
- **M1** — Le noyau lui tient une **table de descripteurs** ; un descripteur est un simple **entier** qui désigne un objet, via une description portant l'**offset**.
- **M2** — Derrière chaque descripteur, un **objet noyau** à durée de vie comptée (**refcount**), hérité par **`fork`**, conservé par **`exec`** (sauf `CLOEXEC`).
- **M3** — Une **socket** n'est qu'un **objet noyau de plus**, rangé dans la même table (`socket:[inode]`).
- **M4** — En local (`AF_UNIX`), on apprend le **cycle** serveur/client, et qu'**`accept` crée un descripteur par connexion**.
- **M5** — Sur le réseau (TCP), on ajoute **IP + port** et l'**ordre des octets**.
- **M6** — En **UDP**, l'absence de connexion révèle, par contraste, ce que TCP nous offrait.
- **M7** — Le **multiplexage** (`epoll`) gère **N descripteurs** dans un seul fil — possible *parce que* les descripteurs sont des objets manipulables.
- **M8** — Sous le capot : **tampons**, **machine à états**, et l'explication des détails (`TIME_WAIT`, `EAGAIN`).

Le fil conducteur tient en une phrase : **une socket est un descripteur de fichier, et un descripteur est une entrée dans une table que le noyau tient pour ton processus.** Tout le reste — domaines, types, `accept`, `epoll`, tampons — n'est que la déclinaison de cette idée. Bonne pratique, et n'hésite pas à modifier les ports, casser les serveurs et observer : c'est en bricolant qu'on solidifie la pyramide.