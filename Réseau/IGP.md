# Tutoriel 1 : IGP — Interior Gateway Protocols

> **Prérequis supposés acquis** : modèle OSI (L1–L7), adressage IPv4 (notation CIDR, masques, calcul de sous-réseau), notion de routage statique (`ip route`), fonctionnement basique d'un routeur.  
> **Ce tutoriel prépare à** : Tutoriel 2 (MPLS), dans lequel l'IGP est le socle sur lequel LDP construit les LSP.

---

## Table des matières

1. [Le routage IP et ses limites](#1-le-routage-ip-et-ses-limites)
2. [Protocoles de routage dynamique : concepts communs](#2-protocoles-de-routage-dynamique--concepts-communs)
3. [Protocoles à vecteur de distance — RIP](#3-protocoles-à-vecteur-de-distance--rip)
4. [Protocoles à état de lien — fondamentaux](#4-protocoles-à-état-de-lien--fondamentaux)
5. [OSPF](#5-ospf)
6. [IS-IS](#6-is-is)
7. [Redistribution entre protocoles](#7-redistribution-entre-protocoles)
8. [Récapitulatif : quand utiliser quoi](#8-récapitulatif--quand-utiliser-quoi)
9. [Exercices de compréhension générale](#9-exercices-de-compréhension-générale)
10. [Corrections](#10-corrections)

---

## 1. Le routage IP et ses limites

### Théorie

Un **routeur** est un équipement de couche 3 qui prend des décisions de forwarding paquet par paquet, en consultant sa **table de routage** (RIB — Routing Information Base). Pour chaque paquet, il effectue un **Longest Prefix Match (LPM)** : parmi toutes les entrées qui correspondent à l'adresse destination, il choisit celle dont le masque est le plus long (le plus spécifique).

```
Table de routage d'un routeur :
  10.0.0.0/8      via 192.168.1.1   (match pour 10.5.3.1)
  10.5.0.0/16     via 192.168.1.2   (match plus spécifique pour 10.5.3.1)
  10.5.3.0/24     via 192.168.1.3   (match encore plus spécifique → choisi)
  0.0.0.0/0       via 192.168.0.1   (route par défaut)
```

**Le routage statique** consiste à renseigner manuellement chaque entrée de la table :
```cisco
ip route 10.2.0.0 255.255.0.0 192.168.1.1
ip route 0.0.0.0 0.0.0.0 192.168.0.1
```

Limites du routage statique :
- **Pas de convergence automatique** : si un lien tombe, la route statique reste dans la table → black-hole.
- **Scalabilité nulle** : pour N sites, chaque routeur doit avoir N-1 routes configurées manuellement.
- **Charge opérationnelle** : chaque modification topologique requiert une intervention humaine sur chaque routeur concerné.

**Le routage dynamique** résout ces problèmes : les routeurs échangent des informations de joignabilité entre eux et recalculent automatiquement les meilleures routes en cas de changement.

### Histoire

Les premiers réseaux ARPANET (ancêtre d'Internet, années 1970) utilisaient uniquement du routage statique. Avec la croissance du réseau, cela est vite devenu ingérable. Le premier protocole de routage dynamique standardisé pour les réseaux IP est **RIP** (Routing Information Protocol, 1988, RFC 1058), lui-même inspiré du protocole de routage de Xerox PARC (PARC Universal Protocol, 1970s). L'augmentation de la taille des réseaux dans les années 1990 a montré les limites des protocoles à vecteur de distance et conduit au développement des protocoles à état de lien (OSPF en 1989, IS-IS adapté pour IP en 1990).

### Exercices — Section 1

**Q1.1** — Un routeur a la table suivante :
```
192.168.0.0/16   via 10.0.0.1
192.168.1.0/24   via 10.0.0.2
192.168.1.128/25 via 10.0.0.3
0.0.0.0/0        via 10.0.0.4
```
Vers quel next-hop sont forwardés les paquets destinés à : (a) `192.168.1.200`, (b) `192.168.2.1`, (c) `10.5.0.1`, (d) `192.168.1.100` ?

**Q1.2** — Expliquez en une phrase ce qu'est le Longest Prefix Match et pourquoi ce mécanisme est nécessaire.

**Q1.3** — Un réseau de 20 routeurs entièrement maillé utilise du routage statique. Chaque routeur doit avoir une route vers les 19 autres. Combien de commandes `ip route` doit-on configurer au total (toutes les routes statiques de tous les routeurs) ?

**Q1.4** — Citez trois situations réelles où le routage statique reste approprié malgré ses limites.

**Q1.5** — Un routeur a une route statique `10.5.0.0/24 via 192.168.1.1` configurée. Le lien vers `192.168.1.1` tombe. Que se passe-t-il avec les paquets destinés à `10.5.0.3` ? Qu'aurait fait un protocole de routage dynamique ?

**Q1.6** — Quelle est la différence entre la **RIB** (Routing Information Base) et la **FIB** (Forwarding Information Base) ? Lequel des deux est consulté lors du forwarding d'un paquet ?

**Q1.7** — Vrai ou Faux : un routeur peut avoir plusieurs routes vers le même réseau dans sa table. Si oui, comment choisit-il laquelle utiliser pour le forwarding ?

**Q1.8** — Qu'est-ce qu'une **route par défaut** (`0.0.0.0/0`) ? Dans quel cas est-elle utilisée lors du LPM ?

**Q1.9** — Un ingénieur configure la route statique `ip route 0.0.0.0 0.0.0.0 192.168.0.1` sur un routeur de bordure. Quels paquets utilisent cette route ?

**Q1.10** — Quelle est la différence entre un **IGP** (Interior Gateway Protocol) et un **EGP** (Exterior Gateway Protocol) ? Donnez un exemple de chaque. Ce tutoriel couvre exclusivement les IGP — pourquoi BGP (un EGP) n'est-il pas abordé ici ?

---

## 2. Protocoles de routage dynamique : concepts communs

### Théorie

Tous les protocoles de routage dynamique partagent des concepts fondamentaux.

#### Distance administrative (AD)

Quand plusieurs protocoles de routage coexistent sur un routeur, ils peuvent proposer des routes vers le même réseau avec des métriques incomparables (un coût OSPF et un nombre de sauts RIP ne se comparent pas directement). La **distance administrative** est un entier (0–255) qui indique la **fiabilité** d'une source de routage. Plus l'AD est faible, plus la source est préférée.

| Source | AD par défaut (Cisco) |
|---|---|
| Interface connectée | 0 |
| Route statique | 1 |
| EIGRP résumé | 5 |
| eBGP | 20 |
| OSPF | 110 |
| IS-IS | 115 |
| RIP | 120 |
| iBGP | 200 |
| Non connu / inaccessible | 255 |

Quand deux protocoles proposent une route vers le même réseau, c'est la route avec l'AD la plus faible qui est installée dans la RIB.

#### Métrique

La **métrique** est la mesure de coût d'un chemin selon les critères propres à chaque protocole. Elle n'est pertinente que pour comparer des routes apprises par le **même protocole**.

| Protocole | Métrique |
|---|---|
| RIP | Nombre de sauts (hop count), max 15 |
| OSPF | Coût = référence bandwidth / bandwidth interface |
| IS-IS | Métrique configurable manuellement (défaut = 10 par interface) |
| EIGRP | Composite (bande passante + délai principalement) |

#### Convergence

La **convergence** désigne l'état où tous les routeurs d'un domaine ont une vue cohérente et à jour de la topologie. Le temps de convergence est le délai entre une panne et le moment où tous les routeurs ont mis à jour leurs tables de routage.

#### Classful vs Classless

Les protocoles **classful** (RIPv1, IGRP) n'incluent pas le masque dans leurs mises à jour → ils supposent des classes d'adresses (`/8`, `/16`, `/24`) et ne supportent pas CIDR ni VLSM. Les protocoles **classless** (RIPv2, OSPF, IS-IS, EIGRP) transportent les masques avec les préfixes → VLSM et CIDR supportés. Tout protocole moderne est classless.

### Histoire

Les premières versions de RIP et IGRP (années 1980) étaient classful — une contrainte héritée de l'époque où les adresses IP étaient divisées en classes fixes (A, B, C). La pénurie d'adresses IPv4 et l'introduction de CIDR (RFC 1519, 1993) ont rendu le classless indispensable. OSPF et IS-IS ont été conçus classless dès l'origine.

### Exercices — Section 2

**Q2.1** — Un routeur reçoit une route `10.0.0.0/8` d'OSPF et la même route `10.0.0.0/8` de RIP. Laquelle est installée dans la RIB ? Pourquoi ?

**Q2.2** — Quelle est la différence entre **distance administrative** et **métrique** ? Donnez un exemple où les deux jouent un rôle dans la sélection de route.

**Q2.3** — Un réseau converge après une panne. Expliquez ce que signifie "convergé" et pourquoi un état non-convergé est dangereux.

**Q2.4** — Vrai ou Faux : deux routes apprises par OSPF avec des métriques différentes peuvent coexister dans la RIB pour le même préfixe. Dans quel cas cela se produit-il ?

**Q2.5** — Qu'est-ce que le **VLSM** (Variable Length Subnet Masking) ? Pourquoi nécessite-t-il un protocole classless ?

**Q2.6** — Un routeur a deux interfaces avec OSPF actif. L'une est à 1 Gbps, l'autre à 100 Mbps. La référence bandwidth OSPF est 100 Mbps. Calculez le coût OSPF de chaque interface.

**Q2.7** — La distance administrative d'une route statique est 1 et celle d'OSPF est 110. Un ingénieur configure une route statique vers `192.168.10.0/24` ET active OSPF qui apprend la même route. Laquelle est dans la RIB ? Est-ce toujours le comportement souhaitable ?

**Q2.8** — Qu'est-ce qu'une **route flottante** (floating static route) ? Comment la configure-t-on et dans quel cas est-elle utile ?

**Q2.9** — Expliquez la différence entre un protocole IGP et un protocole EGP. Pourquoi un opérateur utilise-t-il un IGP à l'intérieur de son réseau et BGP vers les autres opérateurs ?

**Q2.10** — Un protocole de routage doit choisir entre deux chemins vers `10.5.0.0/24` : chemin A (3 sauts, lien 1 Gbps) et chemin B (2 sauts, lien 10 Mbps). RIP choisit B. OSPF choisit A. Expliquez pourquoi et lequel est objectivement meilleur pour du trafic réseau réel.

---

## 3. Protocoles à vecteur de distance — RIP

### Théorie

Un protocole à **vecteur de distance** fonctionne selon le principe suivant : chaque routeur connaît uniquement ses voisins directs et la distance (métrique) vers chaque destination. Il partage cette information avec ses voisins, qui la propagent à leur tour. Chaque routeur construit sa table de routage sans jamais avoir une vue globale de la topologie.

**Métaphore** : chaque routeur ne voit que ses panneaux de distance ("Lyon : 120 km") affichés par ses voisins directs, sans connaître les routes détaillées.

#### RIP (Routing Information Protocol)

RIPv2 (RFC 2453) est le représentant classique :

- **Métrique** : nombre de sauts (hop count). Maximum = **15 sauts** ; une route à 16 sauts est considérée inaccessible.
- **Mises à jour** : envoyées en **multicast** (`224.0.0.9`) toutes les **30 secondes**, que la topologie ait changé ou non.
- **Timers** :
  - Update timer : 30s (envoi périodique)
  - Invalid timer : 180s (si pas de mise à jour, route marquée inaccessible)
  - Flush timer : 240s (route supprimée de la table)
- **Convergence** : lente (plusieurs cycles de 30s peuvent être nécessaires).

**Problème du count-to-infinity** : lors d'une panne, les routeurs peuvent s'envoyer des informations contradictoires et incrémenter la métrique indéfiniment jusqu'à atteindre 16 (infini). Mécanismes de prévention :
- **Split horizon** : un routeur n'annonce pas une route vers le voisin depuis lequel il l'a apprise.
- **Poison reverse** : variante de split horizon qui annonce explicitement la route avec métrique 16 (infini) vers le voisin source.
- **Triggered updates** : envoi immédiat d'une mise à jour lors d'un changement (sans attendre 30s).

Configuration minimale RIPv2 :
```cisco
router rip
 version 2
 no auto-summary
 network 192.168.1.0
 network 10.0.0.0
```

### Histoire

RIPv1 (RFC 1058, 1988) est l'un des premiers IGP standardisés pour IP, basé sur l'algorithme de Bellman-Ford. Il était classful (pas de masque dans les mises à jour). RIPv2 (RFC 2453, 1998) ajoute le support CIDR et l'authentification. RIPng (RFC 2080) est la version pour IPv6. Aujourd'hui, RIP est quasi-absent des réseaux de production au profit d'OSPF et IS-IS, mais il reste enseigné pour comprendre les fondements des protocoles à vecteur de distance et les problèmes qu'ils posent (count-to-infinity, convergence lente). EIGRP (Cisco propriétaire) est un protocole à vecteur de distance avancé ("dual algorithm") qui corrige la plupart des problèmes de RIP — nous ne le détaillons pas dans ce tutoriel.

### Exercices — Section 3

**Q3.1** — Un réseau RIP a la topologie `A — B — C — D — E — F — G — H — I`. A annonce le réseau `10.1.0.0/24`. Quelle est la métrique de cette route sur le routeur I ? Peut-il l'installer dans sa table ?

**Q3.2** — Expliquez le problème du **count-to-infinity** avec un exemple simple à 3 routeurs : `A — B — C`, où C annonce `10.0.0.0/24` et le lien C-B tombe.

**Q3.3** — Quelle est la différence entre **split horizon** et **poison reverse** ? Lequel est plus agressif dans la prévention du count-to-infinity ?

**Q3.4** — RIP envoie des mises à jour toutes les 30 secondes. Dans le pire cas, combien de temps faut-il pour qu'une panne se propage dans un réseau de 5 routeurs en série avec RIP (sans triggered updates) ?

**Q3.5** — Vrai ou Faux : RIPv2 supporte le VLSM. Justifiez. Quelle commande est indispensable pour que RIPv2 fonctionne correctement avec VLSM sur Cisco ?

**Q3.6** — Pourquoi la métrique "nombre de sauts" est-elle une mauvaise métrique pour les réseaux modernes ? Donnez un exemple où RIP choisirait un chemin objectivement moins bon qu'OSPF.

**Q3.7** — Qu'est-ce qu'un **triggered update** dans RIP ? Quel problème résout-il par rapport au mécanisme d'update périodique ?

**Q3.8** — RIP a un invalid timer de 180s et un flush timer de 240s. Que signifient ces deux timers ? Que se passe-t-il à chaque échéance ?

**Q3.9** — Un routeur RIP reçoit une mise à jour avec une route `192.168.5.0/24`, métrique 16. Que fait-il avec cette route ?

**Q3.10** — Pourquoi RIP est-il inadapté à un réseau de plus de 15 sauts ? Quel est le vrai impact opérationnel de cette limite ?

---

## 4. Protocoles à état de lien — fondamentaux

### Théorie

Un protocole à **état de lien** adopte une approche radicalement différente : chaque routeur construit une **carte complète de la topologie** du réseau, puis calcule lui-même le meilleur chemin vers chaque destination.

#### Principe de fonctionnement

1. **Découverte des voisins** : chaque routeur identifie ses voisins directs via des messages Hello.
2. **Génération d'un LSA/LSP** : chaque routeur crée un paquet décrivant ses liens locaux (voisins, coûts). Ce paquet s'appelle **LSA** (Link State Advertisement) dans OSPF, ou **LSP** (Link State Packet) dans IS-IS.
3. **Inondation (flooding)** : chaque routeur envoie son LSA/LSP à **tous** les routeurs du domaine. Chaque routeur retransmet les LSA/LSP reçus, sauf sur l'interface par laquelle il les a reçus.
4. **Construction de la LSDB** : chaque routeur accumule tous les LSA/LSP dans sa **Link State Database (LSDB)**, qui représente la carte complète de la topologie.
5. **Calcul SPF** : l'algorithme de **Dijkstra (SPF — Shortest Path First)** est exécuté sur la LSDB pour calculer l'arbre des plus courts chemins depuis ce routeur vers toutes les destinations.

```
LSDB d'un routeur (exemple simplifié) :
  R1: liens vers R2 (coût 10), R3 (coût 5)
  R2: liens vers R1 (coût 10), R4 (coût 20)
  R3: liens vers R1 (coût 5),  R4 (coût 15)
  R4: liens vers R2 (coût 20), R3 (coût 15)

SPF depuis R1 :
  vers R2 : R1→R2, coût 10
  vers R3 : R1→R3, coût 5
  vers R4 : R1→R3→R4, coût 20 (vs R1→R2→R4 coût 30 → chemin via R3)
```

#### Avantages vs vecteur de distance

| Critère | Vecteur de distance | État de lien |
|---|---|---|
| Vue topologique | Partielle (voisins seulement) | Complète (toute la topologie) |
| Convergence | Lente (propagation hop-by-hop) | Rapide (inondation immédiate) |
| Count-to-infinity | Possible | Impossible |
| Charge CPU | Faible | Plus élevée (calcul SPF) |
| Charge réseau | Élevée (mises à jour périodiques complètes) | Faible (inondation uniquement sur changement) |
| Scalabilité | Limitée | Excellente (avec hiérarchie) |

#### Mécanisme d'inondation et numéros de séquence

Pour éviter les boucles d'inondation (un LSA pourrait circuler indéfiniment), chaque LSA/LSP porte un **numéro de séquence**. Si un routeur reçoit un LSA qu'il connaît déjà (même numéro de séquence ou plus ancien), il le jette sans le retransmettre.

### Histoire

L'algorithme de Dijkstra (1959) est le fondement mathématique des protocoles à état de lien. Son application aux protocoles de routage a été pionnière avec IS-IS (OSI, 1987) et OSPF (IETF, 1989). Le principe d'inondation fiable a été un défi technique majeur : il fallait garantir que tous les routeurs reçoivent exactement les mêmes informations sans boucle et sans perte, sur un réseau potentiellement partitionné ou en cours de convergence.

### Exercices — Section 4

**Q4.1** — Décrivez les 5 étapes du fonctionnement d'un protocole à état de lien, depuis la mise sous tension d'un routeur jusqu'à l'installation de routes dans sa table.

**Q4.2** — Pourquoi les protocoles à état de lien ne souffrent-ils pas du problème de count-to-infinity qui affecte RIP ?

**Q4.3** — Qu'est-ce que la **LSDB** ? Est-elle identique sur tous les routeurs d'un même domaine de routage ? Que se passe-t-il si deux routeurs ont des LSDB différentes ?

**Q4.4** — Expliquez le mécanisme d'**inondation** (flooding). Pourquoi ne crée-t-il pas de boucles infinies de LSA ?

**Q4.5** — Un réseau a 50 routeurs. Comparez la quantité d'informations de routage échangées lors d'un changement topologique entre RIP et OSPF.

**Q4.6** — Qu'est-ce que l'algorithme **SPF (Dijkstra)** ? Quelle est son entrée et sa sortie dans le contexte d'un IGP à état de lien ?

**Q4.7** — Pourquoi le calcul SPF est-il plus coûteux en CPU qu'un simple tri de vecteurs de distance ? Dans quelles situations ce surcoût est-il problématique ?

**Q4.8** — Vrai ou Faux : dans un réseau à état de lien, chaque routeur peut calculer indépendamment le chemin optimal sans avoir besoin de consulter ses voisins après la convergence initiale. Justifiez.

**Q4.9** — Deux routeurs R1 et R2 ont des LSDB légèrement différentes (R2 n'a pas encore reçu le LSA d'un lien récemment ajouté). R1 et R2 calculent-ils le même chemin optimal ? Quel est le risque ?

**Q4.10** — Comparez les protocoles à état de lien et à vecteur de distance sur le critère de la charge réseau lors d'un **changement topologique** (panne d'un lien). Lequel génère plus de trafic ? Pourquoi ?

---

## 5. OSPF

### Théorie

**OSPF** (Open Shortest Path First, RFC 2328 pour OSPFv2/IPv4, RFC 5340 pour OSPFv3/IPv6) est le protocole à état de lien le plus répandu dans les réseaux d'entreprise.

#### Architecture hiérarchique : les Aires (Areas)

Pour limiter le volume de la LSDB et la fréquence des calculs SPF, OSPF divise le réseau en **aires** (areas). Chaque aire a sa propre LSDB et effectue ses calculs SPF localement.

Règle fondamentale : **toutes les aires doivent être connectées à l'Area 0 (backbone)**.

```
Area 1 ── ABR ── Area 0 (Backbone) ── ABR ── Area 2
                      |
                     ABR
                      |
                   Area 3
```

Types de routeurs OSPF :
- **IR (Internal Router)** : tous ses liens dans une seule aire.
- **ABR (Area Border Router)** : connecté à plusieurs aires, dont l'Area 0. Résume les routes entre aires.
- **ASBR (Autonomous System Boundary Router)** : redistribue des routes depuis un autre protocole (RIP, statique, BGP) dans OSPF.
- **Backbone Router** : au moins un lien dans l'Area 0.

#### Types de LSA

| Type | Nom | Généré par | Portée |
|---|---|---|---|
| 1 | Router LSA | Chaque routeur | Intra-area |
| 2 | Network LSA | DR (Designated Router) | Intra-area |
| 3 | Summary LSA | ABR | Inter-area |
| 4 | ASBR Summary LSA | ABR | Inter-area |
| 5 | AS External LSA | ASBR | Tout le domaine OSPF |
| 7 | NSSA External LSA | ASBR dans une NSSA | Area NSSA uniquement |

#### DR et BDR sur les réseaux multi-accès

Sur un réseau Ethernet (multi-accès), si chaque routeur établissait une adjacence avec tous les autres, on aurait N×(N-1)/2 adjacences. Pour éviter cela, OSPF élit un **DR (Designated Router)** et un **BDR (Backup DR)** :
- Tous les routeurs établissent une adjacence uniquement avec DR et BDR.
- Le DR génère le LSA de type 2 représentant le réseau.
- Election : routeur avec la **priorité OSPF la plus élevée** (défaut = 1), puis à égalité le **Router-ID le plus élevé**.

```
R1, R2, R3, R4 sur un même segment Ethernet :
  R1 (priorité 100) → élu DR
  R2 (priorité 50)  → élu BDR
  R3, R4 (priorité 1) → DROther (adjacences avec DR et BDR seulement)
```

#### États d'adjacence OSPF

```
Down → Init → 2-Way → ExStart → Exchange → Loading → Full
```
- **2-Way** : les deux routeurs se voient mutuellement (échangé des Hello). DR/BDR non requis.
- **Full** : LSDB synchronisée. État final souhaité entre voisins.

#### Configuration OSPF minimale

```cisco
router ospf 1
 router-id 1.1.1.1
 
interface Loopback0
 ip ospf 1 area 0
 
interface GigabitEthernet0/0
 ip ospf 1 area 0
 ip ospf hello-interval 1
 ip ospf dead-interval 4
```

Timers OSPF par défaut :
- Hello interval : 10s (Ethernet), 30s (liens série)
- Dead interval : 4 × Hello interval

#### Coût OSPF

```
Coût = Référence bandwidth / Bande passante de l'interface
Référence par défaut = 100 Mbps

Interface 100 Mbps : coût = 100/100 = 1
Interface 10 Mbps  : coût = 100/10  = 10
Interface 1 Gbps   : coût = 100/1000 → arrondi à 1 (problème !)
```

Sur les réseaux modernes (liens Gbps ou plus), il faut augmenter la référence bandwidth (`auto-cost reference-bandwidth 10000` pour 10 Gbps) pour différencier les coûts.

### Histoire

OSPF version 1 est décrit dans la RFC 1131 (1989). La version 2 (RFC 2328, 1998) reste la référence pour IPv4. OSPFv3 (RFC 5340, 2008) supporte IPv6. OSPF a été conçu comme protocole **ouvert** (d'où "Open" dans son nom) pour succéder à IGRP (Cisco propriétaire). L'introduction des areas dans OSPFv2 est directement motivée par la nécessité de contenir la LSDB dans les très grands réseaux — un réseau OSPF en Area 0 unique avec des centaines de routeurs devient ingérable.

### Exercices — Section 5

**Q5.1** — Pourquoi OSPF nécessite-t-il une Area 0 (backbone) et impose-t-il que toutes les autres aires y soient connectées ? Que se passe-t-il si une aire est connectée uniquement à une autre aire non-backbone ?

**Q5.2** — Expliquez le rôle du DR sur un réseau multi-accès Ethernet. Sans DR, combien d'adjacences OSPF y aurait-il entre 8 routeurs sur le même segment ? Avec DR, combien d'adjacences full ?

**Q5.3** — Un routeur OSPF a le Router-ID `2.2.2.2`. Comment ce Router-ID est-il déterminé automatiquement si non configuré manuellement ? Quel problème peut survenir si le Router-ID change ?

**Q5.4** — Listez les 7 états d'adjacence OSPF dans l'ordre. Quel état indique que deux routeurs ont synchronisé leur LSDB ?

**Q5.5** — Quelle est la différence entre un LSA de type 3 et un LSA de type 5 ? Qui les génère et où sont-ils propagés ?

**Q5.6** — Un réseau a des liens à 1 Gbps et à 10 Gbps. La référence bandwidth OSPF est de 100 Mbps (défaut). Quels coûts OSPF sont calculés ? Quel problème cela pose-t-il ? Quelle est la solution ?

**Q5.7** — Expliquez la différence entre une **Stub Area** et une **NSSA (Not-So-Stubby Area)**. Dans quel cas utilise-t-on chacune ?

**Q5.8** — Deux routeurs OSPF R1 et R2 sont voisins sur un lien point-à-point. R1 a `hello-interval 10` et R2 a `hello-interval 5`. Que se passe-t-il ? Quelle est la règle OSPF sur ce paramètre ?

**Q5.9** — Vrai ou Faux : sur un lien point-à-point OSPF, un DR est élu. Justifiez.

**Q5.10** — Un ingénieur veut s'assurer que R1 est toujours élu DR sur un segment Ethernet, quels que soient les autres routeurs présents. Quelle commande utilise-t-il ? Que faire pour qu'un routeur ne soit jamais élu DR ?

---

## 6. IS-IS

### Théorie

**IS-IS** (Intermediate System to Intermediate System, ISO 10589) est un protocole à état de lien conçu initialement pour le protocole réseau OSI (CLNS/CLNP), puis étendu pour supporter IP (RFC 1195, IS-IS pour IP, 1990). Il est massivement utilisé par les opérateurs télécoms et les grands ISP.

#### Différences fondamentales avec OSPF

| Critère | OSPF | IS-IS |
|---|---|---|
| Couche transport | IP (protocole 89) | L2 directement (pas d'IP) |
| Hiérarchie | Areas + Area 0 obligatoire | Niveaux L1/L2 |
| Adressage | Router-ID (IP) | NET (adresse CLNS) |
| PDU | LSA (types 1–7) | LSP, CSNP, PSNP |
| Réseau multi-accès | DR + BDR | DIS (Designated IS) |

#### Adresse NET (Network Entity Title)

Chaque routeur IS-IS est identifié par une adresse NET au format CLNS :

```
49.0001.0102.0304.0506.00
│  │    │              │
│  │    │              └── NSEL : toujours 00 pour un routeur
│  │    └───────────────── System-ID : 6 octets, identifiant unique du routeur
│  └────────────────────── Area ID : longueur variable (ici 2 octets : 0001)
└───────────────────────── AFI : 49 = adressage privé (analogue à RFC 1918)
```

**Bonne pratique** : dériver le System-ID du loopback IP. Pour loopback `1.1.1.1` :
```
1.1.1.1 → 001.001.001.001 → 0010.0100.1001
NET : 49.0001.0010.0100.1001.00
```

#### Niveaux L1 et L2

IS-IS a deux niveaux hiérarchiques :
- **Level-1 (L1)** : routage intra-area. Un routeur L1 ne connaît pas la topologie hors de son aire. Il route vers un L1/L2 pour les destinations externes.
- **Level-2 (L2)** : routage inter-area (backbone IS-IS). Les routeurs L2 forment le backbone et connaissent la topologie inter-area.
- **Level-1-2** : routeur avec des adjacences L1 et L2.

Contrairement à OSPF, IS-IS n'impose pas une structure en étoile autour d'une area backbone. Les aires IS-IS forment naturellement un backbone L2 continu.

```
[Area 49.0001] L1/L2 ── L2 ── L2 ── L1/L2 [Area 49.0002]
     │                   ↑ Backbone L2               │
  L1 routeurs         L2 uniquement              L1 routeurs
```

#### PDU IS-IS

IS-IS utilise ses propres types de paquets (PDU — Protocol Data Units) transmis directement en L2 (EtherType `0xFEFE` pour CLNS) :
- **Hello PDU (IIH)** : découverte et maintien des adjacences. Types : LAN Hello (L1 et L2 distincts), P2P Hello.
- **LSP (Link State PDU)** : équivalent du LSA OSPF. Contient l'état des liens d'un routeur.
- **CSNP (Complete Sequence Number PDU)** : liste complète des LSP connus — permet de détecter les LSP manquants.
- **PSNP (Partial Sequence Number PDU)** : demande ou acquittement de LSP spécifiques.

#### DIS (Designated IS)

Sur un réseau multi-accès LAN, IS-IS élit un **DIS** (Designated IS), équivalent du DR OSPF. Différences importantes :
- Il n'y a **pas de backup DIS** (contrairement au BDR OSPF) — si le DIS tombe, une nouvelle élection a lieu rapidement.
- L'élection DIS est **préemptive** : si un routeur avec une priorité plus élevée apparaît, il remplace immédiatement le DIS actuel (contrairement à OSPF où le DR reste en place jusqu'à sa panne).
- Priorité par défaut : 64 (configurable).

#### Configuration IS-IS minimale

```cisco
router isis CORE
 net 49.0001.0010.0100.1001.00
 is-type level-2-only
 metric-style wide    ← active les métriques étendues (nécessaire pour TE, SR)

interface Loopback0
 ip router isis CORE
 isis circuit-type level-2-only

interface GigabitEthernet0/0
 ip router isis CORE
 isis circuit-type level-2-only
 isis metric 10
```

#### Métriques IS-IS

IS-IS supporte deux styles de métriques :
- **Narrow (défaut ancien)** : métrique max = 63 par lien, 1023 au total.
- **Wide** : métrique max = 16 777 215 par lien. Indispensable pour Segment Routing et Traffic Engineering.

`metric-style wide` est recommandé sur tous les déploiements modernes.

### Histoire

IS-IS est né du projet OSI (Open Systems Interconnection) de l'ISO dans les années 1980 comme protocole de routage pour le protocole réseau OSI (CLNP). La RFC 1195 (1990, "Use of OSI IS-IS for Routing in TCP/IP and Dual Environments") l'a étendu pour router IP en plus de CLNP. Les grands opérateurs (notamment en Europe) qui avaient investi dans OSI ont naturellement adopté IS-IS pour leurs réseaux IP. Aujourd'hui IS-IS est utilisé par la majorité des grands ISP (AT&T, BT, NTT, Orange…) parce qu'il est indépendant d'IP (plus robuste lors de problèmes de configuration IP), plus facilement extensible (les TLV IS-IS facilitent l'ajout d'extensions comme Segment Routing), et historiquement adopté avant OSPF dans beaucoup de backbones opérateurs.

### Exercices — Section 6

**Q6.1** — Décomposez l'adresse NET `49.0002.1921.6800.0001.00` : identifiez l'AFI, l'Area ID, le System-ID et le NSEL. De quelle adresse IP loopback le System-ID semble-t-il dérivé ?

**Q6.2** — Quelle est la différence fondamentale entre IS-IS et OSPF concernant la couche de transport de leurs PDU ? Quel avantage cela procure-t-il à IS-IS ?

**Q6.3** — Expliquez la hiérarchie L1/L2 d'IS-IS. Comment se compare-t-elle à la structure Area/Area 0 d'OSPF ?

**Q6.4** — Vrai ou Faux : IS-IS nécessite une area backbone (comme l'Area 0 d'OSPF) vers laquelle toutes les autres aires doivent être connectées. Justifiez.

**Q6.5** — Un routeur IS-IS est configuré `is-type level-2-only`. Peut-il former des adjacences avec un routeur `level-1-only` dans la même aire ? Justifiez.

**Q6.6** — Quelle est la différence entre le DIS d'IS-IS et le DR d'OSPF en termes d'élection et de préemption ?

**Q6.7** — Qu'est-ce que le `metric-style wide` dans IS-IS ? Pourquoi est-il indispensable dans les réseaux modernes supportant Segment Routing ?

**Q6.8** — Citez les quatre types de PDU IS-IS et décrivez brièvement le rôle de chacun.

**Q6.9** — Un routeur IS-IS P dans un backbone opérateur doit router à la fois des paquets CLNP (OSI) et IP. Quel `is-type` et quelle configuration de base sont nécessaires pour cela ? (Note : dans ce tutoriel, on suppose que votre environnement est IP uniquement — mentionner CLNP pour la culture.)

**Q6.10** — Un opérateur hésite entre OSPF et IS-IS pour son backbone. Il a 300 routeurs, des liens à 100 Gbps, et prévoit de déployer Segment Routing dans 6 mois. Quelle est votre recommandation et pourquoi ?

---

## 7. Redistribution entre protocoles

### Théorie

La **redistribution** est le mécanisme permettant d'injecter des routes apprises par un protocole dans un autre protocole. Elle est nécessaire lors de migrations (RIP vers OSPF), de réseaux hétérogènes, ou d'interconnexion de domaines de routage.

```
Domaine RIP ── ASBR ── Domaine OSPF
               │
         Redistribue les routes
         RIP → OSPF (et vice-versa si voulu)
```

#### Redistribution sur Cisco IOS

```cisco
router ospf 1
 redistribute rip metric 20 metric-type 2 subnets
 !                 ↑        ↑              ↑
 !           métrique OSPF  type externe   inclure les sous-réseaux

router rip
 version 2
 redistribute ospf 1 metric 5
 !                      ↑
 !               métrique RIP (hop count)
```

#### Types de routes externes OSPF

Routes redistribuées dans OSPF apparaissent comme **routes externes** :
- **Type 1 (E1/N1)** : la métrique externe + le coût interne OSPF sont additionnés. Plus précis.
- **Type 2 (E2/N2, défaut)** : seule la métrique externe compte. Le coût interne OSPF est ignoré.

#### Problèmes courants de redistribution

**Boucles de routage** : si on redistribue A→B ET B→A sans filtrage, des routes peuvent faire un aller-retour et créer des boucles. Solution : utiliser des **route-maps** et des **tags** pour identifier et filtrer les routes redistribuées.

**Métriques incompatibles** : une métrique OSPF (coût) n'a pas de sens dans RIP (hop count). Il faut toujours spécifier manuellement la métrique cible lors de la redistribution.

**Suboptimal routing** : la redistribution peut introduire des routes suboptimales si les métriques ne sont pas correctement calibrées.

### Histoire

La redistribution est apparue comme nécessité lors des migrations de RIP vers OSPF dans les années 1990. Elle est aujourd'hui courante dans les réseaux hybrides (entreprise OSPF + WAN opérateur BGP, ou migration IS-IS → SR-MPLS). Les pièges de redistribution (boucles, suboptimal routing) ont conduit à des recommandations de conception strictes : limiter les points de redistribution, filtrer systématiquement, et préférer des migrations "clean cutover" à la coexistence longue durée.

### Exercices — Section 7

**Q7.1** — Un réseau a un domaine OSPF et un domaine RIP interconnectés via un ASBR. L'ASBR redistribue RIP dans OSPF. Comment apparaissent les routes RIP dans OSPF ? Quel type de LSA les transporte ?

**Q7.2** — Expliquez le problème de **boucle de redistribution bidirectionnelle** (redistribution A→B et B→A simultanément). Comment les **tags BGP/OSPF** permettent-ils de l'éviter ?

**Q7.3** — Lors de la redistribution de routes statiques dans OSPF (`redistribute static`), quelle commande supplémentaire est souvent nécessaire et pourquoi ?

**Q7.4** — Quelle est la différence entre une route OSPF de type E1 et E2 ? Dans quel cas préfère-t-on E1 ?

**Q7.5** — Un routeur OSPF reçoit la même route `10.5.0.0/24` : une via OSPF (métrique 50) et une redistribuée depuis RIP (métrique externe 20, type E2). Laquelle est préférée ? Pourquoi ?

**Q7.6** — Vrai ou Faux : on peut redistribuer des routes BGP dans OSPF. Dans quel cas cela est-il dangereux ?

**Q7.7** — Lors d'une migration progressive d'un réseau RIP vers OSPF, comment gérer la coexistence des deux protocoles sans créer de boucles ?

**Q7.8** — La commande `redistribute rip metric 20 subnets` est configurée sur un ASBR OSPF. Que signifie le mot-clé `subnets` ? Que se passe-t-il sans lui ?

**Q7.9** — Citez deux scénarios réels où la redistribution est inévitable en production.

**Q7.10** — Un ingénieur redistribue toutes les routes d'un VRF dans OSPF par erreur (des milliers de routes clients). Quel impact cela a-t-il sur les routeurs OSPF ? Que doit-il faire pour corriger la situation ?

---

## 8. Récapitulatif : quand utiliser quoi

### Théorie

| Critère | RIP | OSPF | IS-IS |
|---|---|---|---|
| Taille du réseau | < 15 sauts, très petit | Moyenne à grande | Très grande (ISP) |
| Convergence | Lente | Rapide | Très rapide |
| Complexité config | Très simple | Moyenne | Moyenne-haute |
| Support VLSM/CIDR | Oui (v2) | Oui | Oui |
| Scalabilité | Très limitée | Bonne (avec areas) | Excellente |
| Indépendance IP | Non | Non | Oui |
| Support SR/TE | Non | Oui (OSPFv2 extensions) | Oui (natif, préféré) |
| Adoption typique | Legacy/lab | Entreprise | Opérateurs, ISP |
| RFC de référence | 2453 | 2328 (v2) | 10589 + 1195 |

**Règle empirique** :
- **RIP** : réseaux de lab, très petits réseaux legacy, formation.
- **OSPF** : réseaux d'entreprise, campus, datacenter enterprise.
- **IS-IS** : backbones opérateurs, ISP, tout réseau avec Segment Routing ou Traffic Engineering MPLS à grande échelle.

### Exercices — Section 8

**Q8.1** — Une PME a 5 routeurs dans un seul bâtiment. Quel IGP recommandez-vous ? Justifiez.

**Q8.2** — Un opérateur télécoms construit un nouveau backbone national de 200 routeurs avec déploiement SR-MPLS prévu. Quel IGP choisir et pourquoi ?

**Q8.3** — Un réseau existant tourne sur RIP. Il faut y ajouter un lien 10 Gbps en parallèle d'un lien 100 Mbps. RIP choisira-t-il le bon lien ? Que faire ?

**Q8.4** — Quand OSPF peut-il être préféré à IS-IS même dans un contexte opérateur ?

**Q8.5** — Un ingénieur conçoit un réseau hybride : l'entreprise utilise OSPF, le WAN opérateur utilise IS-IS. Comment les deux domaines communiquent-ils leurs routes ?

---

## 9. Exercices de compréhension générale

**QG.1 — Topologie complète**

Topologie OSPF :
```
R1 (Area 0) ── R2 (Area 0) ── R3 (ABR) ── R4 (Area 1) ── R5 (Area 1)
                                  │
                               R6 (ABR)
                                  │
                               R7 (Area 2) ── R8 (ASBR)
                                              │
                                           BGP/Internet
```
a) Quels routeurs génèrent des LSA de type 3 ?
b) Quels routeurs génèrent des LSA de type 5 ?
c) Les routeurs de l'Area 2 voient-ils les LSA de type 1 de l'Area 1 ?
d) R5 peut-il joindre Internet ? Via quel chemin (en termes de routeurs) ?

---

**QG.2 — Comparaison de convergence**

Un réseau de 10 routeurs en anneau perd un lien. Comparez le comportement de convergence de RIP vs OSPF :
- Combien de temps faut-il pour que tous les routeurs aient une vue cohérente dans chaque cas ?
- Quel type de messages est échangé dans chaque protocole lors de la convergence ?

---

**QG.3 — Diagnostic OSPF**

`show ip ospf neighbor` sur R1 montre R2 en état **2-Way** (et non Full) sur une interface Ethernet. Citez deux causes possibles et expliquez comment les identifier.

---

**QG.4 — IS-IS NET**

Un ingénieur doit assigner une adresse NET IS-IS à un routeur dont le loopback est `10.255.0.5`. L'area IS-IS est `49.0003`. Construisez l'adresse NET complète en suivant la convention de dérivation depuis l'IP loopback.

---

**QG.5 — Redistribution et distance administrative**

Un routeur reçoit `10.10.0.0/24` via OSPF (AD=110, métrique=50) et la même route via une route statique (AD=1). L'ingénieur veut qu'OSPF soit utilisé en priorité pour cette route (pour bénéficier de la convergence dynamique), mais garder la route statique comme secours. Comment configurer cela ?

---

**QG.6 — Coûts OSPF et chemin optimal**

Topologie :
```
R1 ──[1G]── R2 ──[100M]── R4
 │                          │
[100M]                    [1G]
 │                          │
R3 ──────────[10G]─────── R4
```
Référence bandwidth = 1000 Mbps (auto-cost reference-bandwidth 1000).
a) Calculez le coût de chaque lien.
b) Quel chemin R1→R4 est choisi par OSPF ? Justifiez.
c) Quel chemin serait choisi si la référence bandwidth était 100 Mbps (défaut) ?

---

**QG.7 — IS-IS vs OSPF : décision d'architecture**

Un opérateur migre son réseau de 400 routeurs d'OSPF vers IS-IS. Quels sont les enjeux techniques de cette migration ? Peut-on faire coexister OSPF et IS-IS pendant la migration ? Comment redistribuer les routes entre les deux domaines pendant la transition ?

---

**QG.8 — SPF et LSDB**

Donnez la LSDB (simplifiée) du réseau suivant et calculez manuellement l'arbre SPF depuis R1 :
```
R1 ──[coût 5]── R2 ──[coût 10]── R4
│                                  │
[coût 3]                        [coût 2]
│                                  │
R3 ──────────[coût 8]──────────── R4
```
Quel est le chemin le moins coûteux de R1 vers R4 ?

---

**QG.9 — Problème de DR**

Sur un segment Ethernet avec 6 routeurs OSPF (priorités : R1=100, R2=50, R3=1, R4=1, R5=0, R6=1) :
a) Quel routeur est élu DR ? Quel routeur est élu BDR ?
b) R5 peut-il être élu DR ? Pourquoi ?
c) Si R1 tombe, que se passe-t-il ? Quel routeur devient DR ?
d) Si R1 revient, reprend-il immédiatement le rôle de DR ?

---

**QG.10 — Préparation au tutoriel MPLS**

Dans le tutoriel MPLS (tutoriel 2), l'IGP sera utilisé pour deux rôles fondamentaux. Devinez ces rôles et expliquez pourquoi l'IGP doit converger **avant** que MPLS puisse fonctionner correctement.

---

## 10. Corrections

### Corrections — Section 1

**C1.1** — (a) `192.168.1.200` : correspond à `/25` (`192.168.1.128/25` couvre .128–.255) → **10.0.0.3**. (b) `192.168.2.1` : correspond à `/16` seulement → **10.0.0.1**. (c) `10.5.0.1` : aucune route spécifique → route par défaut → **10.0.0.4**. (d) `192.168.1.100` : correspond à `/24` et `/16`, mais `/24` est plus spécifique → **10.0.0.2**.

**C1.2** — Le Longest Prefix Match sélectionne la route dont le masque est le plus long (le plus spécifique) parmi toutes les routes correspondant à l'adresse destination. C'est nécessaire pour permettre à des routes plus générales (agrégats) et plus spécifiques (sous-réseaux) de coexister dans la même table.

**C1.3** — Pour 20 routeurs en maillage complet : chaque routeur a 19 routes statiques (une vers chaque autre). Total : 20 × 19 = **380 commandes `ip route`**.

**C1.4** — Cas appropriés pour le routage statique : (1) route par défaut sur un routeur de bordure Internet (simple et stable) ; (2) réseau à un seul routeur ou réseau stub (un seul chemin possible, pas de redondance) ; (3) route de secours (floating static) avec AD élevée, utilisée uniquement si le protocole dynamique perd la route.

**C1.5** — La route statique reste dans la table (elle ne détecte pas la panne) → les paquets sont envoyés vers le lien down → **black-hole**, les paquets sont perdus. Un protocole dynamique aurait détecté la panne via les mécanismes Hello/Dead, retiré la route, et calculé un chemin alternatif si disponible.

**C1.6** — La **RIB** (Routing Information Base) est la table de routage "brute" peuplée par les protocoles de routage (plusieurs protocoles peuvent proposer des routes). La **FIB** (Forwarding Information Base) est une copie optimisée de la RIB, compilée pour le forwarding rapide (sur Cisco, c'est CEF qui peuple la FIB). C'est la **FIB** qui est consultée lors du forwarding paquet par paquet.

**C1.7** — **Vrai**. Un routeur peut avoir plusieurs routes vers le même réseau, provenant de protocoles différents ou de chemins différents dans le même protocole. La sélection se fait en deux temps : (1) AD la plus faible (entre protocoles différents), (2) métrique la plus faible (au sein du même protocole). Si les deux chemins ont même AD et même métrique, les deux peuvent coexister → **ECMP (Equal-Cost Multi-Path)**.

**C1.8** — La route par défaut (`0.0.0.0/0`) correspond à **toutes les adresses** car tout préfixe IP correspond à un masque de longueur 0 (aucun bit ne doit correspondre). Elle est utilisée en dernier recours lors du LPM : si aucune route plus spécifique ne correspond, la route par défaut est utilisée. C'est le "gateway of last resort".

**C1.9** — Tous les paquets dont la destination ne correspond à **aucune** route plus spécifique dans la table de routage. En pratique, c'est tout le trafic Internet sortant sur un routeur de bordure.

**C1.10** — Un **IGP** route à l'intérieur d'un même domaine administratif (une entreprise, un opérateur). Un **EGP** route entre domaines administratifs distincts (entre opérateurs). Exemples : OSPF/IS-IS (IGP), BGP (EGP). BGP n'est pas abordé ici car il est de nature différente (politique de routage, AS-PATH, attributs complexes) et suppose que l'IGP est déjà en place à l'intérieur de chaque AS.

---

### Corrections — Section 2

**C2.1** — OSPF (AD=110) est préféré à RIP (AD=120) — **l'AD d'OSPF est plus faible**. La route OSPF est installée dans la RIB. La route RIP est connue mais non installée.

**C2.2** — La **distance administrative** compare des sources de routage **différentes** (quel protocole croire ?). La **métrique** compare des chemins **au sein du même protocole** (quel chemin est le moins coûteux ?). Exemple : deux routes `10.0.0.0/8` existent — une OSPF (AD=110, coût=20) et une IS-IS (AD=115, coût=5). L'AD est comparée en premier : OSPF gagne (110 < 115) même si IS-IS a un coût plus faible. La métrique IS-IS n'est jamais consultée dans ce cas.

**C2.3** — "Convergé" signifie que tous les routeurs ont une vue cohérente, identique et à jour de la topologie du réseau. En état non-convergé, différents routeurs peuvent avoir des informations contradictoires : R1 pense que le réseau `10.0.0.0/24` est joignable via R2, tandis que R2 pense qu'il est inaccessible → les paquets peuvent tourner en boucle ou être perdus.

**C2.4** — **Vrai**. Si deux chemins OSPF vers le même préfixe ont **la même métrique** (ECMP — Equal-Cost Multi-Path), les deux routes coexistent dans la RIB et la FIB, et le trafic est réparti entre elles (load-balancing). OSPF supporte jusqu'à 16 chemins ECMP par défaut sur Cisco.

**C2.5** — VLSM (Variable Length Subnet Masking) permet d'utiliser des masques de longueur variable sur différents sous-réseaux d'un même réseau (ex: `/30` pour les liens WAN, `/24` pour les LANs, `/32` pour les loopbacks). Un protocole classful suppose un masque fixe par classe d'adresse (classe A = /8, B = /16, C = /24) — il ne peut pas transporter des masques différents, donc ne peut pas gérer VLSM.

**C2.6** — Référence = 100 Mbps. Interface 1 Gbps : 100/1000 = 0.1 → arrondi à **1**. Interface 100 Mbps : 100/100 = **1**. Les deux interfaces ont le même coût → OSPF ne peut pas les différencier ! Solution : `auto-cost reference-bandwidth 10000` (10 Gbps), ce qui donne 10000/1000=10 pour 1G et 10000/100=100 pour 100M.

**C2.7** — La route **statique** (AD=1) est installée dans la RIB — elle est préférée à OSPF (AD=110). Ce comportement n'est **pas toujours souhaitable** : si le lien qui mène à la destination de la route statique tombe, la route statique reste (black-hole), tandis qu'OSPF aurait détecté la panne. Solution : utiliser une route statique "flottante" avec une AD supérieure à OSPF.

**C2.8** — Une **route flottante** est une route statique avec une AD **artificiellement élevée** (> celle du protocole dynamique). Elle n'est installée dans la RIB que si le protocole dynamique n'a plus de route pour cette destination. Exemple : `ip route 10.0.0.0 255.0.0.0 192.168.2.1 130` (AD=130 > OSPF=110) → n'est utilisée que si OSPF n'a plus la route. Cas d'usage : liaison de backup RNIS/4G qui ne s'active qu'en cas de panne du lien principal.

**C2.9** — Un **IGP** gère le routage à l'intérieur d'un seul domaine administratif (une entreprise, un AS). Il est optimisé pour la convergence rapide et la métrique de chemin. Un **EGP** (BGP) gère le routage entre domaines administratifs distincts — il prend des décisions basées sur des politiques (AS-PATH, communities) plutôt que sur des métriques pures. Un opérateur utilise un IGP en interne pour la convergence rapide et BGP vers les autres opérateurs pour la politique de routage inter-AS.

**C2.10** — RIP choisit le chemin à **2 sauts** (B) car sa métrique est le nombre de sauts. OSPF choisit le chemin à **3 sauts** (A) car sa métrique est le coût basé sur la bande passante — un lien 1 Gbps a un coût bien plus faible qu'un lien 10 Mbps. **OSPF a raison** : en réseau réel, la bande passante disponible est bien plus pertinente que le nombre de sauts pour déterminer la qualité d'un chemin.

---

### Corrections — Section 3

**C3.1** — 8 sauts entre A et I (`A→B→C→D→E→F→G→H→I`). Métrique sur I = **8**. Comme 8 < 16, la route est installée. Elle reste installable.

**C3.2** — Sans triggered updates ni poison reverse : B avait la route `10.0.0.0/24` via C (métrique 1). Quand le lien B-C tombe, B ne peut plus joindre C. Mais A a appris `10.0.0.0/24` via B (métrique 2). A annonce à B : "je peux joindre 10.0.0.0/24, métrique 2". B installe cette route (métrique 3, via A). A apprend de B que `10.0.0.0/24` est maintenant à métrique 3 → A met à jour à 4. Et ainsi de suite jusqu'à 16 (infini). Pendant ce temps, les paquets vers `10.0.0.0/24` bouclent entre A et B.

**C3.3** — **Split horizon** : R1 n'annonce pas à R2 une route apprise de R2. **Poison reverse** : R1 annonce à R2 la route apprise de R2 mais avec la métrique 16 (infini/inaccessible). Poison reverse est plus agressif car il force explicitement R2 à invalider sa route, alors que split horizon crée simplement un "silence" que R2 peut mal interpréter lors de la convergence.

**C3.4** — Pire cas avec 5 routeurs `R1-R2-R3-R4-R5` et panne du lien R4-R5 : R4 détecte la panne et invalide la route R5 dans son prochain update (jusqu'à 30s). R3 reçoit l'info et propage dans son prochain update (30s de plus). Idem pour R2 puis R1. Total pire cas : 4 × 30s = **120 secondes**. En pratique, avec les triggered updates, c'est quasi-immédiat.

**C3.5** — **Vrai**. RIPv2 inclut le masque de sous-réseau dans ses mises à jour (champ subnet mask dans l'entrée RTE). La commande indispensable sur Cisco est `no auto-summary` : sans elle, RIPv2 résume automatiquement les routes aux frontières classful, ce qui peut masquer des sous-réseaux et rendre le VLSM inopérant.

**C3.6** — Le hop count ignore la bande passante, la latence, la fiabilité. Exemple : chemin A via 3 liens 10 Gbps (3 sauts) et chemin B via 1 lien 56 Kbps (1 saut). RIP choisit B (1 saut) — résultat catastrophique en termes de débit. OSPF aurait choisi A car son coût serait bien plus faible.

**C3.7** — Un **triggered update** est une mise à jour envoyée immédiatement quand la topologie change, sans attendre le timer de 30s. Il réduit drastiquement le temps de convergence lors d'une panne (de 30s max à quelques secondes). Il est limité à l'information qui a changé (pas une mise à jour complète).

**C3.8** — **Invalid timer (180s)** : si un routeur ne reçoit aucune mise à jour pour une route pendant 180s, il la marque comme inaccessible (métrique 16) et envoie un triggered update. **Flush timer (240s)** : 60s après l'invalid timer, si toujours pas de mise à jour, la route est **supprimée** de la table de routage.

**C3.9** — Une route avec métrique 16 signifie **inaccessible** dans RIP. Le routeur ne l'installe pas dans sa table et la propage à ses voisins pour qu'ils invalident aussi cette destination.

**C3.10** — Un réseau de plus de 15 sauts a des destinations injoignables pour RIP (métrique ≥ 16 = infini). En pratique, même avec 15 sauts, la convergence est très lente et peu fiable. Impact opérationnel : impossible de déployer RIP dans un réseau géographiquement étendu avec de nombreux nœuds intermédiaires.

---

### Corrections — Section 4

**C4.1** — (1) Mise sous tension → envoi de messages **Hello** sur toutes les interfaces, découverte des voisins directs. (2) Établissement d'adjacences avec les voisins qui répondent. (3) Génération d'un **LSA/LSP** décrivant les liens locaux. (4) **Inondation** du LSA/LSP à tous les routeurs du domaine (retransmis par chaque routeur sauf sur l'interface source). (5) Accumulation dans la **LSDB**, puis exécution de l'algorithme **SPF (Dijkstra)** pour calculer les meilleurs chemins → installation dans la RIB.

**C4.2** — Les protocoles à état de lien ne souffrent pas du count-to-infinity car chaque routeur a une **vue complète et indépendante** de la topologie. Lors d'une panne, chaque routeur exécute SPF sur sa LSDB et calcule le nouveau chemin optimal sans avoir à "croire" ses voisins — il connaît la topologie globale et peut déduire lui-même que la route n'est plus valide.

**C4.3** — La LSDB est **identique** sur tous les routeurs d'un même domaine de routage (ou d'une même aire pour OSPF) — c'est une propriété fondamentale garantie par le mécanisme d'inondation fiable. Si deux routeurs ont des LSDB différentes (désynchronisation), ils calculent des arbres SPF différents → routes incohérentes → potentiellement des boucles de routage ou des destinations injoignables.

**C4.4** — Lors de l'inondation, chaque routeur retransmet un LSA/LSP reçu sur **toutes ses interfaces sauf celle par laquelle il l'a reçu**. Pour éviter les boucles infinies, chaque LSA porte un **numéro de séquence** : si un routeur reçoit un LSA avec un numéro de séquence qu'il connaît déjà (ou plus ancien), il le jette sans le retransmettre.

**C4.5** — **RIP** : lors d'un changement, chaque routeur envoie une mise à jour complète de sa table à ses voisins, qui à leur tour envoient leur table complète, et ainsi de suite — propagation hop-by-hop, O(N) messages de taille complète. **OSPF** : seul le LSA du lien modifié est inondé (petit message), une seule fois à travers tout le réseau — O(N) messages mais de taille minimale. OSPF est beaucoup plus efficace.

**C4.6** — L'algorithme SPF (Dijkstra) prend en entrée la **LSDB** (graphe de la topologie avec les coûts) et produit en sortie un **arbre des plus courts chemins** depuis le routeur local vers toutes les destinations. Il calcule itérativement la destination la moins coûteuse non encore traitée, puis met à jour les coûts de ses voisins.

**C4.7** — SPF est un algorithme O(N log N) ou O(N²) selon l'implémentation, où N est le nombre de nœuds dans la topologie. Avec des centaines de routeurs, le recalcul SPF peut prendre des dizaines à centaines de millisecondes et consommer du CPU. C'est problématique dans les réseaux très grands ou instables (flap de liens → SPF déclenché fréquemment → CPU saturé). Solution : **SPF throttling** (délais entre recalculs).

**C4.8** — **Vrai**. Une fois la LSDB construite et synchronisée, chaque routeur calcule SPF **indépendamment** sur sa propre copie de la LSDB. Il n'a pas besoin de consulter ses voisins pour cela. C'est précisément ce qui rend les protocoles à état de lien robustes et rapides à converger.

**C4.9** — Si les LSDB sont différentes (désynchronisées), les routeurs calculent des arbres SPF différents et peuvent obtenir des **routes inconsistantes**. Risque : R1 envoie un paquet vers R2 en pensant que R2 peut forwarder vers R3, mais R2 pense que ce chemin n'existe plus → le paquet est perdu ou tourne en boucle.

**C4.10** — Lors d'un changement : **RIP** génère des mises à jour périodiques complètes + triggered update → potentiellement lourd si les tables sont grandes. **OSPF/IS-IS** inondent uniquement le LSA/LSP du lien modifié (quelques centaines d'octets) → très efficace. Pendant la convergence initiale (synchronisation de la LSDB), les protocoles à état de lien peuvent générer plus de trafic, mais ce trafic est rapide et unique — contrairement aux cycles répétés de RIP.

---

### Corrections — Section 5

**C5.1** — L'Area 0 est obligatoire car elle est le **point de transit** entre toutes les aires. Sans Area 0, les routes inter-areas ne peuvent pas circuler (les LSA de type 3 sont générés par les ABR et propagés dans l'Area 0 vers les autres ABR). Si une aire est connectée uniquement à une autre aire non-backbone, elle est coupée du reste du réseau OSPF — ses routes ne sont pas redistribuées correctement. Solution : virtual link (connexion virtuelle via l'Area 0) pour les aires non directement connectées.

**C5.2** — Sans DR, 8 routeurs formeraient `8×7/2 = 28` adjacences full entre toutes les paires. Avec DR, chaque routeur (DROther) établit uniquement des adjacences **Full** avec DR et BDR = `2 adjacences par DROther`. Pour 6 DROthers : `6×2 + 1 (DR-BDR) = 13` adjacences Full. Réduction significative du nombre d'adjacences et des LSA de type 2 générés.

**C5.3** — Sans configuration manuelle, le Router-ID est l'adresse IP **la plus haute d'une interface Loopback active**, ou à défaut l'adresse IP la plus haute d'une interface physique active. Problème : si le Router-ID change (ajout d'une interface loopback plus haute, ou crash/redémarrage), le routeur doit reset ses adjacences OSPF et re-annoncer tous ses LSA avec le nouveau Router-ID → perturbation du réseau. C'est pourquoi le Router-ID doit toujours être **configuré manuellement** et statiquement.

**C5.4** — Les 7 états : **Down → Init → 2-Way → ExStart → Exchange → Loading → Full**. L'état **Full** indique que les deux routeurs ont échangé et synchronisé leurs LSDB complètes.

**C5.5** — **LSA type 3 (Summary LSA)** : généré par un **ABR**, transporte des routes agrégées d'une aire vers les autres aires via l'Area 0. Propagé dans le domaine OSPF (pas dans l'aire source). **LSA type 5 (AS External LSA)** : généré par un **ASBR**, transporte des routes externes (redistribuées depuis BGP, RIP, statique…). Propagé dans tout le domaine OSPF sauf les Stub Areas.

**C5.6** — Référence 100 Mbps. Lien 1 Gbps : 100/1000 = 0.1 → arrondi à **1**. Lien 10 Gbps : 100/10000 = 0.01 → arrondi à **1**. Problème : tous les liens >= 100 Mbps ont le même coût OSPF = 1 → OSPF ne peut pas différencier un lien 100 Mbps d'un lien 10 Gbps. Solution : `auto-cost reference-bandwidth 10000` (10 Gbps comme référence) → 1G donne coût 10, 10G donne coût 1, 100M donne coût 100.

**C5.7** — **Stub Area** : n'accepte pas les LSA de type 5 (routes externes). L'ABR injecte une route par défaut à la place. Utilisée quand les routeurs de l'aire n'ont pas besoin de connaître les routes externes. **NSSA (Not-So-Stubby Area)** : comme Stub, mais autorise un ASBR local à redistribuer des routes externes (via LSA de type 7, convertis en type 5 par l'ABR). Utilisée quand l'aire a un ASBR local mais qu'on veut quand même limiter les LSA de type 5 entrants.

**C5.8** — Les hello-intervals **doivent être identiques** sur les deux extrémités d'un lien OSPF pour qu'une adjacence s'établisse (de même que le dead-interval). Si R1 a hello-interval=10 et R2 a hello-interval=5, **aucune adjacence ne s'établit** — les paramètres de timer sont échangés dans les paquets Hello et vérifiés par chaque côté.

**C5.9** — **Faux**. Sur un lien **point-à-point** (deux routeurs seulement), il n'y a pas d'élection DR/BDR car la problématique de réduction des adjacences sur un réseau multi-accès ne se pose pas. Les deux routeurs passent directement en état Full sans élire de DR.

**C5.10** — Pour garantir l'élection de R1 comme DR : `ip ospf priority 255` sur l'interface de R1 (valeur max). Pour empêcher un routeur d'être élu DR : `ip ospf priority 0` (priorité 0 exclut de l'élection). Attention : l'élection OSPF n'est **pas préemptive** — si un routeur avec priorité plus élevée arrive après l'élection, le DR actuel reste en place jusqu'à sa panne.

---

### Corrections — Section 6

**C6.1** — `49.0002.1921.6800.0001.00` :
- AFI : `49`
- Area ID : `0002`
- System-ID : `1921.6800.0001`
- NSEL : `00`
Dérivation IP : `1921.6800.0001` → `192.1.168.0.00.01` → probablement `192.168.0.1` (avec padding).

**C6.2** — IS-IS envoie ses PDU directement en **couche 2** (L2), sans IP. OSPF utilise IP (protocole 89). Avantage IS-IS : il fonctionne même si la configuration IP d'une interface est erronée ou absente, ce qui facilite le troubleshooting et le bootstrapping. IS-IS est aussi plus facile à étendre (TLV) sans dépendre des contraintes IP.

**C6.3** — IS-IS L1 = routage intra-area (comme les areas OSPF non-backbone). IS-IS L2 = backbone inter-area (comme l'Area 0 OSPF). Différence majeure : IS-IS n'impose pas de topologie en étoile — le backbone L2 est un domaine plat, les aires L1 sont en périphérie. OSPF impose que toutes les aires soient physiquement connectées à l'Area 0.

**C6.4** — **Faux**. IS-IS n'a pas de "area backbone" obligatoire au sens OSPF. Le backbone IS-IS est naturellement formé par tous les routeurs L2 (level-2-only ou level-1-2). Aucune area spéciale n'est requise.

**C6.5** — **Non**. Un routeur `level-2-only` ne participe pas aux adjacences L1. Il ne peut former d'adjacences qu'avec d'autres routeurs L2 ou L1/L2 au niveau L2. Un routeur `level-1-only` n'a pas de capacité L2. Ils ne peuvent donc pas établir d'adjacence ensemble.

**C6.6** — **DIS IS-IS** : élection préemptive (un routeur avec une priorité plus haute remplace immédiatement le DIS actuel), pas de backup DIS. **DR OSPF** : élection non-préemptive (le DR reste en place jusqu'à sa panne, même si un routeur avec une priorité plus haute apparaît), BDR existe comme backup.

**C6.7** — `metric-style wide` active les **métriques étendues IS-IS** (32 bits par lien, jusqu'à 16 millions) au lieu des métriques narrow (6 bits, max 63 par lien). Indispensable pour Segment Routing (les SID nécessitent des métriques larges pour les calculs de chemins) et Traffic Engineering (RSVP-TE ou SR-TE utilisent des métriques TE qui ne tiennent pas dans le format narrow).

**C6.8** — (1) **IIH (IS-IS Hello PDU)** : découverte et maintien des adjacences. (2) **LSP (Link State PDU)** : contient l'état des liens d'un routeur, inondé dans tout le domaine. (3) **CSNP (Complete Sequence Number PDU)** : liste complète des LSP connus par un routeur (envoyé périodiquement par le DIS), permet de détecter les LSP manquants. (4) **PSNP (Partial Sequence Number PDU)** : demande un LSP spécifique manquant, ou acquitte la réception d'un LSP.

**C6.9** — Pour IP uniquement, la configuration IS-IS standard suffit (pas de commande CLNP à activer explicitement dans la plupart des implémentations modernes). Pour dual-stack CLNP+IP, il faudrait activer CLNS sur les interfaces. Dans ce tutoriel (IP uniquement), `ip router isis` sur les interfaces est suffisant. Le transport L2 de IS-IS fonctionne indépendamment d'IP.

**C6.10** — Recommandation : **IS-IS**. Arguments : (1) meilleure scalabilité à 300+ routeurs, (2) pas de contrainte d'Area 0 (topologie plus flexible), (3) support natif et éprouvé de Segment Routing via les extensions TLV IS-IS (RFC 8667), (4) convergence plus rapide, (5) indépendant d'IP (plus robuste). OSPF peut aussi supporter SR (RFC 8665) mais IS-IS est historiquement plus mature pour les déploiements opérateurs SR.

---

### Corrections — Section 7

**C7.1** — Les routes RIP redistribuées apparaissent dans OSPF comme des **routes externes de type E2** (par défaut) ou E1 selon la configuration. Elles sont transportées par des **LSA de type 5** (AS External LSA), générés par l'ASBR et propagés dans tout le domaine OSPF.

**C7.2** — Scénario : ASBR redistribue RIP→OSPF ET OSPF→RIP. Une route `10.0.0.0/8` apprise de RIP est redistribuée dans OSPF. Un routeur OSPF l'installe. Si la redistribution inverse (OSPF→RIP) est active, la route `10.0.0.0/8` est réinjectée dans RIP avec une métrique différente. RIP pense avoir une autre route vers `10.0.0.0/8`, la propage… boucle potentielle. **Solution avec tags** : lors de la redistribution RIP→OSPF, tagger les routes redistribuées (`tag 100`). Lors de la redistribution inverse OSPF→RIP, filtrer les routes avec ce tag (`route-map deny if tag=100`) pour éviter qu'elles repassent dans RIP.

**C7.3** — La commande `subnets` est souvent nécessaire. Sans elle, Cisco IOS ne redistribue que les routes **classful** (les routes avec un masque correspondant exactement à la classe de l'adresse — /8 pour A, /16 pour B, /24 pour C). Avec `subnets`, toutes les routes indépendamment du masque sont redistribuées.

**C7.4** — **E1** : métrique = coût externe + coût OSPF interne accumulé. Plus précis pour choisir le meilleur ASBR quand il y en a plusieurs. **E2** (défaut) : métrique = coût externe uniquement, le coût interne est ignoré. E1 est préféré quand il y a plusieurs ASBR redistribuant les mêmes routes externes et qu'on veut que les routeurs internes choisissent l'ASBR le plus proche.

**C7.5** — Les routes OSPF internes (AD=110, métrique=50) sont préférées aux routes externes E2 (même AD=110, mais les routes internes ont la priorité sur les externes dans le processus de sélection OSPF). La route OSPF interne `10.5.0.0/24` à métrique 50 est installée.

**C7.6** — **Vrai**, on peut redistribuer BGP dans OSPF. **Dangereux** si le BGP porte des milliers (ou millions) de routes Internet : injecter ces routes dans OSPF peut saturer la LSDB, provoquer des calculs SPF constants et crasher les routeurs OSPF. En production, on ne redistribue **jamais** la table BGP Internet complète dans un IGP.

**C7.7** — Migration progressive : activer OSPF sur les routeurs migrés, redistribuer OSPF→RIP (avec filtrage pour éviter les boucles) pour que les routeurs encore en RIP voient les routes OSPF, redistribuer RIP→OSPF pour que les routeurs OSPF voient les routes RIP. Migrer progressivement les routeurs de RIP vers OSPF. Supprimer la redistribution quand tous les routeurs sont sur OSPF. Utiliser des tags pour éviter les boucles de redistribution bidirectionnelle.

**C7.8** — Sans `subnets` : seules les routes classful sont redistribuées dans OSPF (ex: `192.168.0.0/24` est classful donc OK, mais `192.168.0.128/25` est un sous-réseau non-classful et serait ignoré). Avec `subnets` : toutes les routes indépendamment de leur masque sont redistribuées. Sur les réseaux modernes avec VLSM, `subnets` est **toujours nécessaire**.

**C7.9** — Scénarios réels : (1) **Bordure entreprise/opérateur** : redistribution entre OSPF (réseau interne) et BGP (routes reçues de l'opérateur) sur le routeur de bordure Internet. (2) **Migration de protocole** : passage de RIP vers OSPF — coexistence temporaire avec redistribution bidirectionnelle filtrée pendant la transition.

**C7.10** — Impact : des milliers de LSA de type 5 inondés dans tout le domaine OSPF → LSDB énorme → calculs SPF fréquents et lourds → CPU des routeurs OSPF saturé → potentiellement crash ou instabilité. Correction immédiate : supprimer la redistribution (`no redistribute`) sur l'ASBR → les LSA de type 5 expirent (MaxAge = 3600s ou flush immédiat) → LSDB revient à la normale.

---

### Corrections — Section 8

**C8.1** — **RIP** : 5 routeurs dans un même bâtiment, réseau simple, pas de contraintes de convergence strictes → RIP est suffisant et très simple à configurer. (Alternativement, OSPF reste toujours une meilleure pratique même pour les petits réseaux.)

**C8.2** — **IS-IS** : 200 routeurs + SR-MPLS. IS-IS est le choix naturel pour les backbones SR. Il supporte nativement les extensions Segment Routing (RFC 8667), a une scalabilité supérieure à OSPF, et est le standard de facto chez les opérateurs qui déploient SR.

**C8.3** — RIP utilise le hop count. Les deux liens (10 Gbps et 100 Mbps) ont le même nombre de sauts s'ils sont parallèles → RIP ferait du load-balancing entre les deux, ignorant complètement la différence de bande passante. Solution : migrer vers OSPF (métrique basée sur la bande passante) ou EIGRP.

**C8.4** — OSPF peut être préféré à IS-IS dans un contexte opérateur quand : (1) l'équipe a déjà une expertise OSPF et peu de budget formation, (2) le réseau est principalement enterprise avec des équipements qui ont une meilleure implémentation OSPF qu'IS-IS, (3) le réseau n'est pas trop grand (< 100-150 routeurs en Area 0 ou avec areas bien découpées).

**C8.5** — Les deux domaines (OSPF entreprise et IS-IS opérateur) communiquent via **redistribution** sur le routeur de bordure (qui participe aux deux protocoles). Les routes OSPF sont redistribuées dans IS-IS et vice versa, avec filtrage approprié pour éviter les boucles. En pratique, c'est souvent BGP qui sert d'intermédiaire entre les deux domaines plutôt qu'une redistribution directe IS-IS ↔ OSPF.

---

### Corrections — Section 9 (Exercices généraux)

**CG.1** —
a) **LSA de type 3** : générés par **R3 et R6** (les ABR) — ils résument les routes entre aires.
b) **LSA de type 5** : généré par **R8** (l'ASBR) — il redistribue des routes BGP/Internet dans OSPF.
c) **Non** : les LSA de type 1 (Router LSA) sont confinés à leur aire. Les routeurs de l'Area 2 ne voient pas les LSA de type 1 de l'Area 1 — ils ne voient que les LSA de type 3 résumant les routes de l'Area 1, générés par R3 (ABR).
d) **Oui**, R5 peut joindre Internet. Chemin : R5 → R4 → R3 (ABR) → Area 0 → R6 (ABR) → R7 → R8 (ASBR) → Internet. R5 utilise la route par défaut ou les routes externes propagées via les LSA de type 5.

**CG.2** —
**RIP** : après la panne, le premier routeur voisin du lien en panne attend jusqu'à 30s (update timer) avant d'envoyer une mise à jour, puis propagation hop-by-hop. Convergence totale : potentiellement plusieurs minutes (5-6 cycles de 30s pour un anneau de 10). Messages : RIP Response (mises à jour complètes de table) + triggered updates éventuels.
**OSPF** : détection via Dead timer (40s avec défaut, ou < 1s avec BFD). Immédiatement après détection, un LSA est inondé à tous les routeurs simultanément (en quelques secondes). Tous les routeurs recalculent SPF en parallèle. Convergence totale : quelques secondes. Messages : LSA de type 1 mis à jour, inondés via LSUpdate/LSAck.

**CG.3** —
État 2-Way sur un réseau Ethernet signifie que les deux routeurs se voient (échangé des Hello) mais n'ont pas atteint Full. Deux causes possibles :
(1) **L'un des routeurs est DROther** et l'autre aussi → sur un réseau Ethernet, les DROthers ne forment des adjacences Full qu'avec le DR et le BDR, pas entre eux. Vérifier avec `show ip ospf neighbor` si R1 et R2 sont tous deux DROthers.
(2) **Mismatch de paramètres Hello** (hello-interval, dead-interval, area ID, authentication, MTU) → l'adjacence ne progresse pas au-delà de 2-Way. Vérifier `debug ip ospf hello` ou `show ip ospf interface`.

**CG.4** —
Loopback `10.255.0.5`, area `49.0003`.
Dérivation : `10.255.0.5` → padder en 3 groupes de 3 chiffres : `010.255.000.005` → System-ID : `0102.5500.0005`.
NET : `49.0003.0102.5500.0005.00`

**CG.5** —
Solution : configurer la route statique avec une **AD supérieure à OSPF** (> 110) :
`ip route 10.10.0.0 255.255.255.0 <next-hop> 120`
Avec AD=120 > 110 (OSPF), la route statique n'est installée que si OSPF n'a plus de route vers `10.10.0.0/24` → c'est une route flottante qui sert de backup.

**CG.6** —
Référence = 1000 Mbps.
- Lien R1-R2 (1G) : 1000/1000 = **1**
- Lien R2-R4 (100M) : 1000/100 = **10**
- Lien R1-R3 (100M) : 1000/100 = **10**
- Lien R3-R4 (10G) : 1000/10000 = **0.1 → arrondi à 1**

b) Chemin R1→R4 via R2 : coût 1 + 10 = **11**. Chemin R1→R4 via R3 : coût 10 + 1 = **11**. Égalité → ECMP, les deux chemins sont utilisés.

c) Avec référence 100 Mbps : R1-R2 (1G) → coût 1, R2-R4 (100M) → coût 1, R1-R3 (100M) → coût 1, R3-R4 (10G) → coût 1 (arrondi). Tous les liens coût 1 → toutes les routes équivalentes → ECMP. OSPF ne différencie plus les liens — problème classique de la référence bandwidth trop basse.

**CG.7** —
Enjeux techniques : (1) coexistence OSPF et IS-IS pendant la migration — possible via redistribution bidirectionnelle filtrée (tags pour éviter boucles) ; (2) double configuration sur chaque routeur pendant la transition ; (3) tests de convergence à chaque étape ; (4) migration des métriques (les métriques OSPF et IS-IS ne sont pas directement comparables). Plan recommandé : migrer aire par aire (OSPF) ou zone par zone, avec redistribution au périmètre de migration. Cutover final : désactiver OSPF quand tous les routeurs tournent sur IS-IS.

**CG.8** —
LSDB :
- R1: R2 (coût 5), R3 (coût 3)
- R2: R1 (coût 5), R4 (coût 10)
- R3: R1 (coût 3), R4 (coût 8)
- R4: R2 (coût 10), R3 (coût 8)

SPF depuis R1 :
- Initialisation : R1=0, autres=∞
- Traiter R1 : voisins R2 (0+5=5), R3 (0+3=3)
- Traiter R3 (coût 3) : voisin R4 (3+8=11)
- Traiter R2 (coût 5) : voisin R4 (5+10=15 > 11, pas de mise à jour)
- Traiter R4 (coût 11)

Résultat : R1→R4 via **R1→R3→R4**, coût **11** (vs R1→R2→R4, coût 15).

**CG.9** —
a) Priorités : R1=100 (DR), R2=50 (BDR). R3, R4, R6 = priorité 1 (DROthers).
b) R5 a priorité **0** → ne peut jamais être élu DR ou BDR. Priorité 0 exclut de l'élection.
c) Si R1 tombe → R2 (BDR) devient DR. Nouvelle élection BDR parmi R3, R4, R6 → celui avec la plus haute priorité (tous à 1) → celui avec le plus grand Router-ID gagne.
d) **Non** : l'élection OSPF n'est pas préemptive. Quand R1 revient, il voit R2 comme DR et R3/R4/R6 comme BDR/DROther. R1 ne reprend pas le rôle de DR tant que R2 est en place. R1 deviendra DR seulement si R2 tombe.

**CG.10** —
Dans le tutoriel MPLS, l'IGP joue deux rôles fondamentaux :
(1) **Joignabilité des loopbacks PE** : l'IGP propage les adresses loopback de tous les routeurs PE dans le backbone. Ces loopbacks sont utilisés comme identifiants des sessions LDP et comme next-hop des sessions MP-BGP (tutoriel 3). Sans IGP convergé, les loopbacks sont inconnus → pas de session LDP possible.
(2) **Topologie pour LDP** : LDP distribue des labels en suivant les chemins IGP. LDP doit savoir vers quel voisin forwarder pour chaque destination — cette information vient de la FIB, qui est peuplée par l'IGP. L'IGP doit donc converger **avant** que LDP puisse construire les LSP (Label Switched Paths).

---

*Dernière mise à jour : juin 2026 · Suite : Tutoriel 2 — MPLS*