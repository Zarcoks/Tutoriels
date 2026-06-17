# Guide complet : L3 VPN MPLS

> Document de révision — couvre tous les concepts nécessaires à la compréhension et à la construction d'un L3 VPN sur backbone MPLS.  
> Chaque section contient : théorie · histoire · exercices ciblés. Les corrections de tous les exercices se trouvent en fin de document.

---

## Table des matières

1. [Vue d'ensemble : qu'est-ce qu'un L3 VPN ?](#1-vue-densemble--quest-ce-quun-l3-vpn-)
2. [MPLS : le plan de données](#2-mpls--le-plan-de-données)
3. [LDP : distribution des labels](#3-ldp--distribution-des-labels)
4. [IGP du cœur : OSPF ou IS-IS](#4-igp-du-cœur--ospf-ou-is-is)
5. [VRF : isolation des clients](#5-vrf--virtual-routing-and-forwarding)
6. [Route Distinguisher (RD)](#6-route-distinguisher-rd)
7. [Route Target (RT)](#7-route-target-rt)
8. [MP-BGP : le plan de contrôle VPN](#8-mp-bgp--le-plan-de-contrôle-vpn)
9. [Forwarding end-to-end : le double label](#9-forwarding-end-to-end--le-double-label)
10. [PE-CE : protocoles de routage côté client](#10-pe-ce--protocoles-de-routage-côté-client)
11. [Inter-AS L3 VPN (Options A, B, C)](#11-inter-as-l3-vpn-options-a-b-c)
12. [Convergence MPLS](#12-convergence-mpls)
13. [Exercices de compréhension générale](#13-exercices-de-compréhension-générale)
14. [Corrections](#14-corrections)

---

## 1. Vue d'ensemble : qu'est-ce qu'un L3 VPN ?

### Théorie

Un **L3 VPN MPLS** (aussi appelé RFC 4364 VPN ou MPLS/BGP VPN) permet à un opérateur de fournir à ses clients un service d'interconnexion de sites qui se comporte comme un réseau IP privé dédié — sans que l'opérateur ne déploie de liens physiques distincts pour chaque client.

Les acteurs du réseau sont :

```
CE ── PE ─── P ─── P ─── PE ── CE
     (Edge)  (Core)  (Core)  (Edge)

CE : Customer Edge  → routeur du client, ne voit pas MPLS
PE : Provider Edge  → routeur opérateur en bordure, fait tout le travail VPN
P  : Provider core  → routeur cœur, commute des labels, ignore les VRF
```

Ce que L3 VPN apporte, comparé à une liaison point-à-point dédiée ou à un VPN IPsec sur Internet :

| Critère | IPsec/Internet | L2 LL dédié | L3 VPN MPLS |
|---|---|---|---|
| Infrastructure | Publique | Privée (1 par client) | Privée mutualisée |
| SLA | Aucun | Garanti | Contractuel |
| IP privées | Avec NAT | Natives | Natives |
| Scalabilité | Bonne | Mauvaise | Excellente |
| QoS | Impossible | Facile | Via DSCP/MPLS TC |
| Coût | Faible | Très élevé | Moyen-élevé |

### Histoire

Dans les années 1990, les opérateurs interconnectaient les sites clients avec des circuits Frame Relay ou ATM — un PVC par paire de sites. Pour un client avec 10 sites en maillage complet, cela représentait 45 circuits. Ingérable à grande échelle.

MPLS émerge à la fin des années 1990 comme alternative. La RFC 2547 (1999, Rosen & Rekhter) pose les bases des VPN BGP/MPLS. Elle est mise à jour en RFC 4364 (2006) qui reste la référence actuelle. L'idée centrale : mutualiser l'infrastructure physique tout en isolant les clients par des mécanismes de plan de contrôle (VRF, RD, RT, MP-BGP).

### Exercices — Section 1

**Q1.1** — Un client a 8 sites qu'il veut interconnecter en maillage complet via des liaisons louées point-à-point. Combien de liens faut-il ? Même question avec 20 sites. Quelle formule générale s'applique ?

**Q1.2** — Listez les trois rôles distincts (CE, PE, P) et indiquez pour chacun : (a) à qui il appartient (client ou opérateur), (b) s'il a connaissance des VRF clients, (c) s'il fait de la commutation de labels.

**Q1.3** — Un client vous demande la différence entre un VPN IPsec site-à-site sur Internet et un L3 VPN MPLS. Rédigez une explication en 5 lignes maximum, sans jargon technique.

**Q1.4** — Vrai ou Faux : le CE d'un client L3 VPN MPLS doit implémenter MPLS. Justifiez.

**Q1.5** — Pourquoi dit-on que le L3 VPN MPLS est un service de **couche 3** et non de couche 2 ? Quelle est la conséquence pratique pour le client (que voit-il dans sa table de routage) ?

**Q1.6** — Un ingénieur junior affirme : "Le L3 VPN transporte les paquets IP des clients sur Internet via des tunnels chiffrés." Identifiez les deux erreurs dans cette affirmation.

**Q1.7** — Pour un opérateur gérant 500 clients avec en moyenne 5 sites chacun, expliquez pourquoi l'approche Frame Relay/ATM (un circuit par paire de sites) est impraticable, et comment L3 VPN MPLS résout ce problème.

**Q1.8** — Quel rôle joue le routeur P dans le plan de données ? Doit-il connaître les adresses IP des clients ? Pourquoi est-ce important pour la scalabilité ?

**Q1.9** — La RFC 4364 est la référence du L3 VPN MPLS. Quelle RFC l'a précédée, et quelle était son année de publication ?

**Q1.10** — Un opérateur veut offrir à ses clients un service « any-to-any » (chaque site peut communiquer avec tous les autres sans passer par un hub). Comparez brièvement la complexité de configuration entre Frame Relay et L3 VPN MPLS pour ce cas d'usage.

---

## 2. MPLS : le plan de données

### Théorie

MPLS (Multiprotocol Label Switching) est un mécanisme de **commutation par label** qui opère à la couche 2.5 (entre L2 et L3). Chaque paquet reçoit un ou plusieurs labels (pile de labels) qui dictent son chemin dans le réseau.

Structure d'un label MPLS (32 bits) :

```
 31          12  11   9   8        0
 ┌───────────────┬───┬─┬──────────┐
 │  Label (20b)  │TC │S│  TTL(8b) │
 └───────────────┴───┴─┴──────────┘

Label : valeur 0–1 048 575 (identifiant du chemin)
TC    : Traffic Class (3b) — pour la QoS, ex-EXP
S     : Bottom of Stack (1b) — 1 si c'est le dernier label
TTL   : Time To Live — fonctionne comme le TTL IP
```

Le paquet MPLS est encapsulé dans la trame L2 avec EtherType `0x8847`.

Un routeur qui commute des labels utilise sa **LFIB** (Label Forwarding Information Base) :

```
Entrée label  →  Sortie label  +  Interface  +  Action
100           →  200             eth0/1        SWAP
200           →  [POP]           eth0/2        PHP
```

Actions possibles :
- **PUSH** : ajouter un label (PE ingress)
- **SWAP** : remplacer le label (P cœur)
- **POP** : retirer le label (PE egress ou PHP)

**PHP (Penultimate Hop Popping)** : le routeur P situé juste avant le PE de sortie retire le label de transport, évitant au PE de faire une double lookup. Le PE de sortie reçoit le paquet avec seulement le label VPN (ou l'IP nue s'il n'y a qu'un seul label).

### Histoire

Avant MPLS, les routeurs effectuaient une **longest prefix match** dans la table IP à chaque saut — opération relativement coûteuse avec les CPU des années 1990. L'idée de commuter sur des labels à correspondance exacte est plus rapide.

MPLS émerge comme convergence entre plusieurs propriétaires (Tag Switching de Cisco, ARIS d'IBM, IP Switching de Ipsilon). L'IETF standardise MPLS avec les RFC 3031 (architecture) et 3032 (encodage) en 2001. Aujourd'hui, la vitesse n'est plus l'argument principal (les ASICs gèrent LPM à la vitesse du câble), mais MPLS reste incontournable pour ses capacités d'ingénierie de trafic et de VPN.

### Exercices — Section 2

**Q2.1** — Décomposez le label MPLS `0x000641FF` (32 bits en hex) en ses quatre champs : Label, TC, S, TTL. (Rappel : Label=20b, TC=3b, S=1b, TTL=8b)

**Q2.2** — Un paquet arrive sur un routeur P avec la pile de labels suivante :

```
[Label=500, TC=0, S=0, TTL=62]
[Label=25,  TC=0, S=1, TTL=62]
[IP src=10.1.0.1, dst=10.2.0.1]
```

La LFIB du routeur P contient : `label_in=500 → label_out=600, iface=eth1, action=SWAP`.

Décrivez exactement le paquet après traitement par ce routeur P.

**Q2.3** — Quelle est la différence entre la **FIB** (Forwarding Information Base) classique et la **LFIB** ? Quel type de lookup fait chacune ?

**Q2.4** — Expliquez PHP (Penultimate Hop Popping). Quel problème résout-il ? Quel routeur l'effectue ? Que reçoit le PE egress en conséquence dans un contexte L3 VPN (avec double label) ?

**Q2.5** — Un paquet MPLS est capturé sur le fil. Comment l'équipement de capture sait-il qu'il s'agit d'un paquet MPLS et non d'un paquet IP classique ? Donnez la valeur précise.

**Q2.6** — Vrai ou Faux : le champ TTL du label MPLS fonctionne de manière identique au TTL IP — il est décrémenté à chaque saut et le paquet est jeté si TTL=0. Justifiez.

**Q2.7** — À quel moment un routeur effectue-t-il une action PUSH ? Donnez un exemple concret dans le contexte d'un L3 VPN.

**Q2.8** — Un ingénieur observe la LFIB suivante sur un routeur :

```
Label in  | Label out | Interface | Action
----------|-----------|-----------|-------
3         | -         | eth0/2    | POP
100       | 200       | eth0/1    | SWAP
```

Le label 3 est le label MPLS réservé "Implicit Null". Que signifie son apparition dans la LFIB ? En quoi est-ce lié à PHP ?

**Q2.9** — Dans une pile MPLS à deux labels, comment distingue-t-on le label de transport (outer) du label VPN (inner) ? Quel est le bit qui marque la frontière ?

**Q2.10** — Le champ TC (Traffic Class, anciennement EXP) est sur 3 bits. Combien de valeurs distinctes peut-il prendre ? À quoi servent ces valeurs dans un réseau opérateur ?

---

## 3. LDP : distribution des labels

### Théorie

Pour que les routeurs P et PE connaissent les labels à utiliser, il faut un **protocole de distribution de labels**. LDP (Label Distribution Protocol, RFC 5036) est le plus courant.

LDP fonctionne ainsi :
1. Chaque routeur découvre ses voisins LDP via des messages Hello UDP multicast (`224.0.0.2`, port 646)
2. Une session TCP (port 646) est établie entre chaque paire de voisins
3. Chaque routeur annonce un label pour chacune de ses routes (en particulier ses **loopbacks**)

```
PE1 dit à P :  "Pour joindre 10.0.0.1/32 (mon loopback), utilise label 100"
P   dit à PE1: "Pour joindre 10.0.0.2/32 (loopback de P),   utilise label 200"
P   dit à PE2: "Pour joindre 10.0.0.1/32 (loopback de PE1), utilise label 300"
```

Le chemin de labels résultant forme un **LSP (Label Switched Path)**.

LDP utilise le mode **Liberal Label Retention** par défaut sur IOS : les labels reçus de tous les voisins sont conservés, même si ce voisin n'est pas le next-hop actuel. Cela accélère la convergence en cas de panne.

### Histoire

Avant LDP, Cisco utilisait **TDP** (Tag Distribution Protocol), propriétaire. L'IETF a standardisé LDP en 2001 pour assurer l'interopérabilité. Une alternative à LDP pour distribuer les labels est **RSVP-TE**, utilisé en ingénierie de trafic (Traffic Engineering) pour réserver de la bande passante sur des chemins explicites. Dans les architectures modernes, **Segment Routing** (SR-MPLS ou SRv6) tend à remplacer LDP en éliminant les sessions de distribution d'état par routeur.

### Exercices — Section 3

**Q3.1** — Décrivez les deux phases d'établissement d'une session LDP entre deux routeurs voisins. Quels protocoles de transport sont utilisés à chaque phase et pourquoi ?

**Q3.2** — Topologie : `PE1 — P — PE2`. LDP est actif partout. PE1 a le loopback `10.0.0.1/32`, P a `10.0.0.99/32`, PE2 a `10.0.0.2/32`. Listez tous les messages LDP Label Mapping qui sont échangés pour que PE1 puisse joindre PE2 via un LSP.

**Q3.3** — Quelle est la différence entre la **LIB** (Label Information Base) et la **LFIB** ? Laquelle est utilisée pour le forwarding ?

**Q3.4** — Expliquez la différence entre **Liberal Label Retention** et **Conservative Label Retention**. Quel mode est utilisé par défaut sur IOS et pourquoi est-il préférable en production ?

**Q3.5** — LDP distribue des labels pour quelles destinations prioritairement ? Pourquoi les loopbacks des PE sont-ils particulièrement importants ?

**Q3.6** — Un opérateur veut faire de l'ingénierie de trafic (forcer un flux sur un chemin spécifique différent du plus court chemin IGP). LDP peut-il répondre à ce besoin ? Quel protocole utiliser à la place ?

**Q3.7** — Vrai ou Faux : dans une topologie `PE1 — P1 — P2 — PE2`, le LSP de PE1 vers PE2 est unidirectionnel. Un LSP distinct est nécessaire pour le trafic de retour (PE2 → PE1). Justifiez.

**Q3.8** — Qu'est-ce qu'un **FEC** (Forwarding Equivalence Class) dans le contexte LDP ? Donnez un exemple concret.

**Q3.9** — La commande `show mpls ldp neighbor` ne montre aucun voisin sur PE1, alors que le lien physique vers P est up. Citez trois causes possibles.

**Q3.10** — Qu'est-ce que **LDP IGP Sync** ? Quel problème résout-il lors de la convergence après une panne ?

---

## 4. IGP du cœur : OSPF ou IS-IS

### Théorie

Le backbone MPLS a besoin d'un **IGP** pour que les PE se connaissent mutuellement (notamment via leurs loopbacks) et pour que LDP puisse distribuer les labels sur les bons chemins.

Deux IGP sont utilisés en pratique :

| | OSPF | IS-IS |
|---|---|---|
| Niveau OSI | L3 (IP) | L2 (indépendant IP) |
| Aire | Zones OSPF | Niveaux L1/L2 |
| Scalabilité | Bonne | Excellente |
| Convergence | Rapide | Très rapide |
| Adoption | Très répandue | Opérateurs, ISP |

Pour un backbone L3 VPN, les **loopbacks des PE** doivent absolument être annoncés dans l'IGP — c'est l'adresse utilisée comme next-hop iBGP.

Configuration IS-IS minimale :
```
router isis CORE
 net 49.0001.0000.0000.0001.00   ← NET : Area + System-ID + NSEL
 is-type level-2-only
 
interface Loopback0
 ip router isis CORE
 
interface GigabitEthernet0/0
 ip router isis CORE
```

### Histoire

OSPF (RFC 2328, 1998) et IS-IS (ISO 10589, 1992) ont tous deux été conçus pour des réseaux à grande échelle. IS-IS est historiquement privilégié par les grands opérateurs (notamment parce qu'il est indépendant d'IP et plus simple à étendre), tandis qu'OSPF domine les réseaux d'entreprise. Dans un contexte L3 VPN, le choix de l'IGP n'affecte pas le comportement VPN en lui-même.

### Exercices — Section 4

**Q4.1** — Pourquoi les loopbacks des PE doivent-ils être annoncés dans l'IGP du cœur ? Que se passe-t-il si un loopback PE n'est pas dans l'IGP ?

**Q4.2** — Décomposez l'adresse NET IS-IS suivante : `49.0001.0102.0304.0506.00`. Identifiez l'AFI, l'Area ID, le System-ID et le NSEL.

**Q4.3** — Quelle est la différence entre un routeur IS-IS `level-1`, `level-2-only` et `level-1-2` ? Lequel est recommandé pour les routeurs du cœur MPLS ?

**Q4.4** — Les routes des clients (contenues dans les VRF) sont-elles annoncées dans l'IGP du cœur ? Justifiez.

**Q4.5** — Un ingénieur oublie d'activer l'interface Loopback0 dans IS-IS sur PE1. Quelles sont les conséquences sur le plan de contrôle VPN (MP-BGP) ? Et sur le plan de données ?

**Q4.6** — Pourquoi IS-IS est-il dit "indépendant d'IP" et en quoi est-ce un avantage pour un opérateur backbone ?

**Q4.7** — Dans un backbone MPLS, est-ce que les routes des liens point-à-point entre P et PE (ex: `192.168.100.0/30`) doivent être annoncées dans l'IGP ? Sont-elles nécessaires pour le forwarding MPLS ?

**Q4.8** — Un opérateur utilise OSPF en Area 0 pour tout son backbone. Il a 500 routeurs. Citez deux problèmes potentiels de scalabilité et expliquez comment IS-IS les évite.

**Q4.9** — Quelle est la relation entre l'IGP et LDP ? Dans quel ordre doivent-ils converger pour que le trafic MPLS passe ?

**Q4.10** — Vrai ou Faux : on peut utiliser eBGP comme IGP du cœur MPLS à la place d'OSPF ou IS-IS. Justifiez votre réponse.

---

## 5. VRF : Virtual Routing and Forwarding

### Théorie

Une **VRF** est une instance de table de routage isolée sur un PE. Chaque client possède sa propre VRF sur chaque PE qui le dessert. Les interfaces CE-facing sont assignées à la VRF correspondante.

```
PE1
├── Table globale (IGP, iBGP avec autres PE)
├── VRF BANK_A     ← table de routage dédiée au client Banque
│     10.1.10.0/24 via CE1
│     10.2.10.0/24 via PE2 (appris via MP-BGP)
└── VRF HOSPITAL_C ← table de routage dédiée à l'hôpital
      10.1.0.0/24 via CE3
```

Conséquence importante : quand on affecte une interface à une VRF (`ip vrf forwarding BANK_A`), l'adresse IP de l'interface est supprimée — il faut la reconfigurer. C'est parce que l'interface bascule de la table globale vers la table VRF.

Configuration :
```cisco
ip vrf BANK_A
 rd 65000:100
 route-target export 65000:100
 route-target import 65000:100

interface GigabitEthernet0/1
 ip vrf forwarding BANK_A
 ip address 192.168.10.1 255.255.255.0
```

### Histoire

Avant les VRF, interconnecter plusieurs clients sur la même infrastructure impliquait soit des équipements physiques séparés, soit des VLAN avec des tables de routage communes — source de fuites entre clients. Les VRF (introduites dans IOS 12.0(5)T, fin des années 1990) permettent la **virtualisation du plan de routage** sur un seul équipement physique. Le concept a été étendu avec **VRF-Lite** (VRF sans MPLS, utilisable en entreprise) et est aujourd'hui omniprésent dans les architectures multi-tenant.

### Exercices — Section 5

**Q5.1** — Sur PE1, deux clients ont tous deux un réseau `10.0.0.0/8` dans leur VRF respective. Comment le routeur sait-il vers quel CE forwarder un paquet destiné à `10.5.0.1` ? Est-il possible d'avoir une collision ?

**Q5.2** — Un ingénieur tape `ip vrf forwarding BANK_A` sur l'interface `GigabitEthernet0/1` qui avait l'IP `192.168.10.1/24`. Que se passe-t-il immédiatement ? Que doit-il faire ensuite ?

**Q5.3** — Quelle est la différence entre **VRF** (dans le contexte L3 VPN MPLS) et **VRF-Lite** ? Dans quel cas utilise-t-on VRF-Lite ?

**Q5.4** — La table globale d'un PE contient-elle les routes des clients ? Pourquoi ou pourquoi pas ?

**Q5.5** — Vrai ou Faux : un PE peut avoir une interface physique appartenant à deux VRF différentes simultanément. Justifiez.

**Q5.6** — Un client a 3 sites connectés à 3 PE différents. Combien de VRF au minimum sont créées sur l'ensemble du réseau opérateur pour ce client ?

**Q5.7** — Décrivez la commande permettant de vérifier le contenu d'une VRF spécifique (`BANK_A`) sur un PE Cisco. Quelles informations s'y trouvent ?

**Q5.8** — Est-ce que le routeur CE a connaissance de l'existence des VRF sur le PE auquel il est connecté ? Justifiez.

**Q5.9** — Un opérateur veut permettre à deux clients (ALPHA et BETA) de partager un serveur DNS commun hébergé sur un troisième site. Les clients doivent rester isolés entre eux. Comment les VRF et les RT permettent-ils de modéliser cette situation ? (Répondre en termes d'import/export, pas besoin de la config complète.)

**Q5.10** — Expliquez pourquoi le next-hop MP-BGP d'une route VPN (loopback du PE distant) est résolu dans la **table globale** et non dans la VRF.

---

## 6. Route Distinguisher (RD)

### Théorie

**Problème** : deux clients peuvent utiliser les mêmes plages d'adresses IP privées (ex: `10.0.0.0/8`). Si PE1 annonce `10.1.0.0/24` pour BANK_A et `10.1.0.0/24` pour HOSPITAL_C via MP-BGP, PE2 reçoit deux préfixes identiques — BGP n'en garde qu'un.

**Solution** : le RD est un préfixe de 64 bits ajouté devant le préfixe IP pour créer une **route VPNv4 unique** dans BGP.

```
Sans RD : 10.1.0.0/24           ← ambiguë
Avec RD  : 65000:100:10.1.0.0/24  ← unique (BANK_A)
           65000:300:10.1.0.0/24  ← unique (HOSPITAL_C)
```

Format du RD : `AS:nn` ou `IP:nn`, sur 8 octets au total.

**Propriétés importantes :**
- Le RD n'a pas de sémantique de filtrage : il rend les routes uniques, rien de plus.
- Un même PE peut avoir des RD différents sur deux PE pour la même VRF client — c'est valide.
- Le RD est transporté dans MP-BGP mais n'est pas utilisé pour décider quelle VRF importe la route (c'est le rôle du RT).

### Histoire

La notion de Route Distinguisher a été introduite dans la RFC 2547bis (devenue RFC 4364). Elle répond directement au problème de l'overlapping address space entre clients, problème fondamental dans un contexte multi-tenant où l'opérateur n'a aucun contrôle sur le plan d'adressage de ses clients.

### Exercices — Section 6

**Q6.1** — Sans RD, que se passerait-il si deux clients (ALPHA et BETA) ont tous deux le réseau `172.16.0.0/16` et que PE1 annonce les deux préfixes via BGP à PE2 ?

**Q6.2** — Quelle est la taille en bits d'un RD ? Comment est-il structuré ? Donnez deux formats possibles avec un exemple de valeur pour chacun.

**Q6.3** — PE1 a la VRF CLIENT_A avec RD=`65000:1`. PE2 a aussi la VRF CLIENT_A mais avec RD=`65000:999`. Est-ce une erreur de configuration ? Expliquez pourquoi c'est valide ou non.

**Q6.4** — Une route VPNv4 est notée `65000:200:10.5.0.0/24`. Identifiez chaque composante. Quelle est la longueur totale du préfixe VPNv4 (en bits) ?

**Q6.5** — Le RD est-il utilisé pour filtrer les routes lors de l'import dans une VRF ? Quel mécanisme joue ce rôle ?

**Q6.6** — Pourquoi est-il conseillé d'utiliser des RD **différents par PE** pour un même client (ex: PE1 utilise `65000:101`, PE2 utilise `65000:102` pour la même VRF) ? Quel problème de BGP cela résout-il ?

**Q6.7** — Vrai ou Faux : le RD est visible par le routeur CE du client. Justifiez.

**Q6.8** — Un opérateur a 200 clients. Il décide d'utiliser le même RD `65000:1` pour tous les clients. Expliquez précisément pourquoi c'est une erreur critique.

**Q6.9** — Le format `IP:nn` d'un RD utilise une adresse IP (32 bits) + un numéro (16 bits). Donnez un exemple concret avec l'IP loopback du PE. Quand préfère-t-on ce format à `AS:nn` ?

**Q6.10** — Dans `show bgp vpnv4 unicast all` sur un PE, à quoi ressemble une entrée de la table BGP VPNv4 ? Identifiez où apparaît le RD dans la sortie.

---

## 7. Route Target (RT)

### Théorie

**Problème** : une fois les routes VPNv4 uniques grâce au RD, comment PE2 sait-il dans quelle VRF locale importer chaque route reçue via MP-BGP ?

**Solution** : le RT (Route Target) est une **Extended Community BGP** (attribut de 64 bits) attachée à chaque route VPNv4. Chaque VRF déclare quels RT elle exporte et importe.

```
PE1, VRF BANK_A :
  route-target export 65000:100   ← tag ajouté aux routes exportées
  route-target import 65000:100   ← importe les routes portant ce tag

PE2, VRF BANK_A :
  route-target export 65000:100
  route-target import 65000:100   ← importe les routes de PE1 ✅
  
PE2, VRF HOSPITAL_C :
  route-target export 65000:300
  route-target import 65000:300   ← n'importe PAS 65000:100 ❌
```

RT permet des topologies avancées :

**Extranet** (deux VPN qui partagent des routes communes) :
```
VRF BANK_A  : import 65000:100, import 65000:999  (999 = serveurs partagés)
VRF HOSP_C  : import 65000:300, import 65000:999
VRF SHARED  : export 65000:999
```

**Hub & Spoke** (le trafic inter-spokes passe obligatoirement par le hub) :
```
VRF HUB   : import 65000:spoke, export 65000:hub
VRF SPOKE : import 65000:hub,   export 65000:spoke
```

### Histoire

Le RT comme Extended Community BGP est défini dans la RFC 4360 (2006). Avant cela, le filtrage des routes VPN entre PE se faisait manuellement avec des route-maps — non scalable. Le RT automatise complètement ce filtrage et permet de décrire des topologies complexes dans la configuration VRF elle-même, sans toucher au BGP.

### Exercices — Section 7

**Q7.1** — Expliquez en une phrase la différence fondamentale de rôle entre RD et RT.

**Q7.2** — Configuration donnée :
```
VRF A : export RT=100:10, import RT=100:10
VRF B : export RT=100:20, import RT=100:20
VRF C : export RT=100:10, import RT=100:20
```
VRF C importe-t-elle les routes de VRF A ? Les routes de VRF B ? VRF A importe-t-elle les routes de VRF C ?

**Q7.3** — Qu'est-ce qu'une **Extended Community BGP** ? Pourquoi le RT est-il transporté sous cette forme plutôt que comme un attribut BGP ordinaire ?

**Q7.4** — Une entreprise a un site HQ qui doit pouvoir communiquer avec tous ses sites agences, et les agences doivent pouvoir communiquer entre elles directement (topologie any-to-any). Définissez les RT export/import pour HQ et pour les agences.

**Q7.5** — Même entreprise que Q7.4, mais cette fois le responsable sécurité exige que tout le trafic inter-agences passe par le HQ (topologie Hub & Spoke). Comment les RT changent-ils ?

**Q7.6** — Un opérateur héberge un service DNS partagé entre deux clients (CLIENT_A et CLIENT_B) qui doivent rester isolés entre eux. Proposez un schéma de RT (avec une VRF DNS_SHARED) permettant cette configuration.

**Q7.7** — Vrai ou Faux : un seul RT peut être exporté par une VRF. Justifiez et donnez un cas d'usage où exporter plusieurs RT est nécessaire.

**Q7.8** — Le RT est une Extended Community de 8 octets. Quel est son format précis ? En quoi diffère-t-il structurellement du RD, alors qu'ils ont la même notation `AS:nn` ?

**Q7.9** — PE2 reçoit une route VPNv4 de PE1 avec le RT `65000:100`. La VRF BANK_A de PE2 a `route-target import 65000:100`. La VRF CORP de PE2 a `route-target import 65000:999`. Dans quelle(s) VRF la route est-elle installée ?

**Q7.10** — Expliquez pourquoi, dans une topologie Hub & Spoke, les spokes apprennent les routes des autres spokes via le hub, alors qu'il n'y a pas de session BGP directe entre spokes.

---

## 8. MP-BGP : le plan de contrôle VPN

### Théorie

**MP-BGP** (Multiprotocol BGP, RFC 4760) est l'extension de BGP permettant de transporter des familles d'adresses autres qu'IPv4 unicast. Dans le contexte L3 VPN, il transporte la famille **VPNv4** (AFI=1, SAFI=128).

Les PE établissent des sessions **iBGP** entre eux (même AS opérateur), souvent via un **Route Reflector** pour éviter la règle full-mesh iBGP.

Ce que MP-BGP ajoute à BGP classique :

| Attribut BGP | Rôle dans L3 VPN |
|---|---|
| `MP_REACH_NLRI` | Annonce la route VPNv4 + label VPN (inner label) |
| `NEXT_HOP` | Loopback du PE source (résolu via IGP+LDP) |
| `EXTENDED_COMMUNITY` | Transporte le Route Target |
| `LOCAL_PREF` | Sélection du meilleur PE de sortie |

Configuration PE :
```cisco
router bgp 65000
 neighbor 10.0.0.2 remote-as 65000
 neighbor 10.0.0.2 update-source Loopback0
 neighbor 10.0.0.2 send-community extended   ← OBLIGATOIRE pour les RT
 
 address-family vpnv4
  neighbor 10.0.0.2 activate
 exit-address-family
 
 address-family ipv4 vrf BANK_A
  neighbor 192.168.10.2 remote-as 65001     ← session eBGP vers CE
  neighbor 192.168.10.2 activate
 exit-address-family
```

**`send-community extended` est obligatoire** : sans lui, les RT (Extended Community) ne sont pas transmis et l'import des routes en VRF ne fonctionne pas.

### Histoire

BGP-4 (RFC 1771, 1995) était conçu pour IPv4 unicast uniquement. L'extension multiprotocole (RFC 2283, 1998 — mise à jour RFC 4760, 2007) ajoute les attributs `MP_REACH_NLRI` et `MP_UNREACH_NLRI` pour annoter des NLRI de n'importe quelle famille. Cela a permis d'utiliser BGP comme protocole de signalisation universel : IPv6, L3 VPN, L2 VPN, EVPN, BGP-LS, Flowspec… Un seul protocole de contrôle pour tout le réseau.

### Exercices — Section 8

**Q8.1** — Qu'est-ce que l'AFI et le SAFI ? Quelle combinaison AFI/SAFI est utilisée pour le L3 VPN MPLS ?

**Q8.2** — Pourquoi les sessions MP-BGP entre PE sont-elles des sessions **iBGP** et non eBGP ? Quelle règle BGP cela implique-t-il sur la propagation des routes (règle du split-horizon iBGP) ?

**Q8.3** — Un réseau a 10 PE. Sans Route Reflector, combien de sessions iBGP full-mesh sont nécessaires ? Avec un Route Reflector central, combien ?

**Q8.4** — Quel est le rôle exact de `update-source Loopback0` dans la configuration BGP d'un PE ? Que se passe-t-il sans cette commande si le lien physique tombe ?

**Q8.5** — Un ingénieur configure MP-BGP entre PE1 et PE2 mais oublie `send-community extended`. Décrivez précisément le symptôme observé et la commande de vérification à utiliser.

**Q8.6** — Quelle est la différence entre `address-family vpnv4` et `address-family ipv4 vrf BANK_A` dans la configuration BGP d'un PE ? À quoi sert chacune ?

**Q8.7** — Dans un UPDATE MP-BGP VPNv4, que contient exactement le champ `MP_REACH_NLRI` ? Listez les éléments transportés.

**Q8.8** — Le NEXT_HOP d'une route VPNv4 reçue via iBGP est le loopback de PE1 (`10.0.0.1`). Dans quelle table PE2 résout-il ce next-hop ? Comment (quel protocole) ?

**Q8.9** — Vrai ou Faux : dans un déploiement avec Route Reflector, le RR doit obligatoirement être un PE (avoir des VRF). Justifiez.

**Q8.10** — MP-BGP est décrit comme un "protocole de signalisation universel". Citez trois autres familles d'adresses (au-delà du VPNv4) que MP-BGP peut transporter.

---

## 9. Forwarding end-to-end : le double label

### Théorie

C'est le mécanisme fondamental du L3 VPN : chaque paquet porte **deux labels empilés**.

```
┌──────────────────────────────────┐
│ L2 Header (EtherType 0x8847)     │
├──────────────────────────────────┤
│ Label TRANSPORT (outer)  S=0     │  ← Ajouté par PE ingress (LDP)
├──────────────────────────────────┤
│ Label VPN (inner)         S=1    │  ← Ajouté par PE ingress (MP-BGP)
├──────────────────────────────────┤
│ IP Header (paquet client)        │
├──────────────────────────────────┤
│ Payload                          │
└──────────────────────────────────┘
```

**Label transport** : permet d'acheminer le paquet du PE ingress jusqu'au PE egress à travers les routeurs P (qui ne connaissent pas les VRF).

**Label VPN** : permet au PE egress de savoir dans quelle VRF (et donc vers quel CE) forwarder le paquet IP interne.

**Parcours complet** d'un ping `10.1.10.1 → 10.2.10.1` (CE1→CE2) :

```
CE1                PE1               P                PE2               CE2
 │ IP brut          │                 │                 │                 │
 │─10.1.10.1──────▶│                 │                 │                 │
 │  →10.2.10.1      │ PUSH 2 labels   │                 │                 │
 │                  │[T=300][VPN=25]──▶ SWAP T label    │                 │
 │                  │                 │[T=400][VPN=25]──▶ PHP: retire T   │
 │                  │                 │                 │[VPN=25] IP brut │
 │                  │                 │                 │ lookup VRF BANK │
 │                  │                 │                 │──────────────────▶
 │                  │                 │                 │  IP brut         │
```

Les routeurs P ne voient jamais l'IP du client ni les VRF — ils font uniquement un **exact match** sur le label de transport.

### Histoire

L'idée du double label est au cœur de la RFC 4364. Elle résout élégamment le problème de scalabilité : les routeurs P, souvent très nombreux et à haut débit, n'ont pas besoin de connaître les routes VPN des clients (qui peuvent être des millions). Seuls les PE, en bordure, maintiennent les VRF. C'est le principe de **séparation des rôles** PE/P.

### Exercices — Section 9

**Q9.1** — À quel moment exact PE1 ajoute-t-il les deux labels sur un paquet entrant depuis CE1 ? Dans quel ordre sont-ils empilés (lequel est mis en premier) ?

**Q9.2** — Topologie : `CE1 — PE1 — P1 — P2 — PE2 — CE2`. PHP est actif sur P2. Décrivez l'état de la pile de labels à chaque étape : après PE1, après P1, après P2, après PE2.

**Q9.3** — Pourquoi les routeurs P n'ont-ils pas besoin de connaître le label VPN (inner label) ? Quel bit leur indique qu'il ne faut pas regarder plus loin dans la pile ?

**Q9.4** — Vrai ou Faux : si PHP est désactivé, PE2 doit effectuer deux lookups successifs pour forwarder le paquet. Expliquez lesquels.

**Q9.5** — D'où provient le **label VPN** (inner label) ? Quel protocole l'a distribué et à quel moment ?

**Q9.6** — D'où provient le **label transport** (outer label) ? Quel protocole l'a distribué ?

**Q9.7** — Un paquet arrive sur PE2 avec uniquement `[VPN=25][IP src=10.1.0.1, dst=10.2.0.1]` (PHP a déjà retiré le label de transport). Décrivez les opérations que PE2 effectue pour forwarder ce paquet vers CE2.

**Q9.8** — Pourquoi dit-on que MPLS opère à la couche 2.5 ? En quoi est-ce visible dans la structure du paquet sur le fil ?

**Q9.9** — Dans une topologie avec 3 routeurs P entre PE1 et PE2, combien d'opérations SWAP sont effectuées sur le label de transport ? Le label VPN est-il modifié pendant le transit dans le cœur ?

**Q9.10** — CE1 envoie un ping vers CE2. Le TTL IP du paquet est 64 à l'entrée de PE1. Quel est le TTL IP à l'arrivée sur CE2 (après traversée de PE1, 2 routeurs P, PE2) ? Le TTL MPLS joue-t-il un rôle visible pour CE2 ?

---

## 10. PE-CE : protocoles de routage côté client

### Théorie

Entre le PE et le CE (routeur du client), plusieurs protocoles peuvent être utilisés :

| Protocole | Avantages | Inconvénients |
|---|---|---|
| **eBGP** | Flexibilité, politique de routage | Complexité, AS client nécessaire |
| **OSPF** | Simple si client connaît OSPF | Redistribution dans MP-BGP, supe-backbone |
| **RIPv2** | Très simple | Peu scalable, peu utilisé |
| **Static** | Simplicité maximale | Pas de convergence automatique |

**eBGP PE-CE** est le plus courant en production. Le PE redistribue les routes apprises du CE dans la VRF, puis MP-BGP les propage aux autres PE.

Subtilité avec eBGP PE-CE : la session PE-CE est eBGP (AS différents), mais la session PE-PE est iBGP. Quand PE2 reçoit une route via iBGP dont le next-hop est PE1, il doit **résoudre ce next-hop dans la table globale** (pas dans la VRF) — c'est l'IGP + LDP qui s'en chargent.

### Histoire

Historiquement, les opérateurs proposaient OSPF ou RIP en PE-CE pour simplifier la vie des clients. Mais eBGP s'est imposé pour sa flexibilité : il permet de gérer facilement les AS clients overlapping (via AS-override ou allowas-in), et donne un contrôle fin sur les politiques de routage (local-pref, MED, communities).

### Exercices — Section 10

**Q10.1** — Expliquez la différence entre la session **PE-CE** et la session **PE-PE** en termes de type BGP (iBGP/eBGP) et de famille d'adresses transportée.

**Q10.2** — Un client utilise eBGP en PE-CE avec l'AS 65001 pour tous ses sites (même AS sur chaque CE). Par défaut, BGP rejette une route qui contient son propre AS dans l'AS-PATH (loop prevention). Quel problème cela pose-t-il pour le L3 VPN ? Quelles sont les deux solutions ?

**Q10.3** — Un client utilise OSPF en PE-CE. Expliquez le problème du "super-backbone" (OSPF supra-backbone via MPLS) et pourquoi les routes inter-sites apparaissent comme des routes de type O E2 côté CE.

**Q10.4** — Un client utilise des routes statiques en PE-CE. Il a deux sites. Que doit configurer l'ingénieur opérateur sur chaque PE pour que les routes des deux sites se retrouvent dans les VRF et soient propagées via MP-BGP ?

**Q10.5** — Dans une configuration PE-CE eBGP, le CE a l'AS 65001 et le PE a l'AS 65000. Le CE annonce `10.10.0.0/24` au PE. Décrivez le cheminement de cette route : comment passe-t-elle du CE vers PE1, puis de PE1 vers PE2 (via MP-BGP), puis de PE2 vers CE2 ?

**Q10.6** — Vrai ou Faux : le CE doit avoir une configuration MPLS pour participer au L3 VPN. Justifiez.

**Q10.7** — Quelle commande Cisco permet de vérifier les routes apprises d'un CE spécifique dans une VRF donnée ?

**Q10.8** — Quel est l'avantage d'eBGP PE-CE par rapport à une route statique PE-CE en termes de résilience ? Donnez un exemple concret.

**Q10.9** — Un client avec eBGP PE-CE souhaite influencer le chemin d'entrée dans son réseau (préférer PE1 par rapport à PE2). Quel attribut BGP peut-il utiliser depuis le CE, et comment le PE en tient-il compte ?

**Q10.10** — Pourquoi le next-hop d'une route VPNv4 reçue via iBGP (le loopback de PE1) est-il résolu dans la **table globale** du PE et non dans la VRF BANK_A ?

---

## 11. Inter-AS L3 VPN (Options A, B, C)

### Théorie

Quand un VPN doit s'étendre sur **deux AS opérateurs différents** (ex: deux opérateurs partenaires), trois options existent (RFC 4364, Section 10) :

**Option A — Back-to-Back VRF**
```
PE1 (AS1) ── ASBR1 ── ASBR2 ── PE2 (AS2)
              [VRF]    [VRF]
```
Les deux ASBR ont des VRF pour chaque client et échangent les routes via eBGP IPv4 standard. Simple mais ne scale pas (une interface physique par VRF entre ASBR).

**Option B — eBGP VPNv4 entre ASBR**
```
PE1 (AS1) ── ASBR1 ──VPNv4 eBGP── ASBR2 ── PE2 (AS2)
```
Les ASBR échangent les routes VPNv4 directement. Scale mieux mais les ASBR doivent maintenir toutes les routes VPN.

**Option C — Multihop MP-eBGP PE-PE**
```
PE1 (AS1) ───── MP-eBGP multihop ──── PE2 (AS2)
         (ASBR1 redistribue loopbacks)
```
Les PE échangent directement les routes VPNv4 via une session eBGP multihop. Les ASBR se contentent de redistribuer les loopbacks des PE. C'est l'option la plus scalable.

### Histoire

La problématique inter-AS est née de la nécessité pour des grandes entreprises multinationales d'obtenir un VPN de bout en bout passant par plusieurs opérateurs. Avant ces options, la solution était de superposer un tunnel GRE/IPsec entre les deux opérateurs — solution fragile et peu scalable. Les options A/B/C de la RFC 4364 ont formalisé des approches interopérables.

### Exercices — Section 11

**Q11.1** — Décrivez en une phrase le principe de chacune des trois options inter-AS (A, B, C).

**Q11.2** — Avec l'Option A, un opérateur a 80 VRF clients à gérer entre deux ASBR. Combien d'interfaces (ou sous-interfaces) faut-il au minimum de chaque côté ? Quel est l'impact opérationnel ?

**Q11.3** — Avec l'Option B, les ASBR échangent des routes VPNv4 via eBGP. Quel attribut BGP doit être réécrit sur l'ASBR pour que les labels de transport restent valides entre les deux AS ? (Indice : next-hop)

**Q11.4** — Avec l'Option C, comment les PE de l'AS1 connaissent-ils les loopbacks des PE de l'AS2 (nécessaires pour établir la session BGP multihop et pour le LSP) ?

**Q11.5** — Comparez les trois options sur le critère de la **charge sur les ASBR** (nombre d'états à maintenir).

**Q11.6** — Vrai ou Faux : avec l'Option A, les ASBR voient les routes clients sous forme VPNv4 (avec RD). Justifiez.

**Q11.7** — Pourquoi l'Option C est-elle dite "la plus scalable" mais aussi "la plus complexe à opérer" ?

**Q11.8** — Dans l'Option B, les ASBR doivent maintenir les routes VPN de tous les clients. Quel est le risque si un ASBR tombe ? Comment le mitiger ?

**Q11.9** — Un grand opérateur international fusionne avec un opérateur local. Ils ont chacun leur AS. Quelle option recommanderiez-vous si la priorité est la simplicité de déploiement avec peu de VRF clients (< 10) ? Et si la priorité est la scalabilité avec des centaines de VRF ?

**Q11.10** — Dans l'Option C, la session MP-eBGP entre PE1 (AS1) et PE2 (AS2) est de type multihop. Pourquoi le paramètre `ebgp-multihop` est-il nécessaire ? Quelle valeur TTL doit-il autoriser ?

---

## 12. Convergence MPLS

### Théorie

Lors d'une panne de lien dans le backbone MPLS, la convergence suit plusieurs étapes séquentielles :

```
1. Détection de panne (BFD ou timers IGP)
        ↓
2. IGP recalcule SPF → nouvelles routes
        ↓
3. LDP reconverge → nouveaux labels sur le nouveau chemin
        ↓
4. MP-BGP n'a pas besoin de reconverger (les loopbacks PE sont toujours joignables)
        ↓
5. Trafic VPN reprend sur le nouveau LSP
```

Grâce au **Liberal Label Retention** de LDP, les labels des chemins alternatifs sont déjà connus — la convergence LDP est quasi-instantanée une fois l'IGP reconvergé.

**BFD (Bidirectional Forwarding Detection)** peut réduire le temps de détection de panne à quelques millisecondes (contre 30-40s pour les timers OSPF/IS-IS classiques).

### Histoire

Les premières implémentations MPLS avaient une convergence comparable à l'IGP sous-jacent — plusieurs secondes en cas de panne. L'introduction de **FRR (Fast ReRoute)** avec RSVP-TE (RFC 4090, 2005) puis de **LDP FRR** et **Segment Routing TI-LFA** (Topology Independent Loop-Free Alternate) ont permis de descendre à des temps de convergence inférieurs à 50ms, comparables à ce que SDH/SONET offrait avec l'APS (Automatic Protection Switching).

### Exercices — Section 12

**Q12.1** — Listez dans l'ordre chronologique les cinq étapes de convergence lors d'une panne de lien dans le backbone MPLS.

**Q12.2** — Pourquoi MP-BGP n'a-t-il généralement pas besoin de reconverger lors d'une panne dans le cœur MPLS (entre P et P, ou entre P et PE) ?

**Q12.3** — Expliquez comment le **Liberal Label Retention** de LDP accélère la convergence par rapport au Conservative Label Retention.

**Q12.4** — Quel est le rôle de **BFD** dans la convergence MPLS ? Quelle est la différence de délai de détection entre BFD et les timers IGP classiques ?

**Q12.5** — Vrai ou Faux : lors d'une panne du lien PE1-P, les labels VPN (inner labels) changent. Justifiez.

**Q12.6** — Qu'est-ce que le **MPLS FRR (Fast ReRoute)** avec RSVP-TE ? Quel temps de convergence permet-il d'atteindre ?

**Q12.7** — Une panne se produit sur le lien entre P1 et P2 dans la topologie `PE1 — P1 — P2 — PE2`. Il existe un chemin alternatif `PE1 — P1 — P3 — PE2`. Décrivez ce que P1 doit faire pour rerouter le trafic.

**Q12.8** — Qu'est-ce que **TI-LFA (Topology Independent Loop-Free Alternate)** ? En quoi améliore-t-il le FRR classique ?

**Q12.9** — Lors d'une convergence suite à une panne, dans quel ordre les plans convergent-ils : plan de données ou plan de contrôle ? Expliquez la distinction.

**Q12.10** — Un ingénieur observe que le trafic VPN met 30 secondes à reprendre après une panne de lien dans le cœur. Il soupçonne les timers IGP. Quelle configuration peut-il ajuster pour réduire ce délai à moins d'une seconde ?

---

## 13. Exercices de compréhension générale

Ces exercices croisent plusieurs sections et évaluent la compréhension globale de l'architecture L3 VPN.

---

**QG.1 — Diagnostic complet**

`show bgp vpnv4 unicast vrf BANK_A` sur PE2 est vide, alors que PE1 a bien les routes dans sa VRF. Citez cinq causes possibles (dans différentes couches : L3 VPN, BGP, IGP/LDP).

---

**QG.2 — Tracé de paquet complet**

Topologie :
```
CE1 (AS65001) — PE1 — P1 — P2 — PE2 — CE2 (AS65002)
```
- VRF BANK_A sur PE1 et PE2
- Label transport : PE1 pousse T=300, P1 SWAP→350, P2 PHP (retire)
- Label VPN pour 10.2.0.0/24 annoncé par PE2 : VPN=42
- CE1 envoie un paquet IP `src=10.1.0.1, dst=10.2.0.5, TTL=64`

Décrivez l'état exact du paquet (tous les headers) à chaque étape : sortie CE1, sortie PE1, sortie P1, sortie P2, sortie PE2, arrivée CE2.

---

**QG.3 — Conception de schéma RT**

Une entreprise a trois catégories de sites :
- **HQ** (1 site) : doit communiquer avec tout le monde
- **Agences** (5 sites) : communiquent entre elles et avec HQ
- **Partenaires** (3 sites) : communiquent uniquement avec HQ, jamais entre eux ni avec les agences

Proposez un schéma RT complet (export/import pour chaque catégorie) garantissant ces contraintes.

---

**QG.4 — Analyse de configuration**

Un ingénieur présente la configuration suivante sur PE1 :

```cisco
ip vrf CLIENT_X
 rd 65000:50
 route-target export 65000:50
 route-target import 65000:60

ip vrf CLIENT_Y
 rd 65000:60
 route-target export 65000:60
 route-target import 65000:50
```

a) Les routes de CLIENT_X sont-elles importées dans CLIENT_Y ? Et inversement ?
b) Est-ce une topologie symétrique ou asymétrique ?
c) Donnez un cas d'usage réel correspondant à ce schéma.

---

**QG.5 — Rôles et tables**

Pour chaque table ci-dessous, indiquez sur quel équipement elle existe, ce qu'elle contient, et quel protocole la peuple :

| Table | Équipement(s) | Contenu | Peuplée par |
|---|---|---|---|
| RIB (table de routage globale) | ? | ? | ? |
| VRF routing table | ? | ? | ? |
| LFIB | ? | ? | ? |
| LIB | ? | ? | ? |
| BGP VPNv4 table | ? | ? | ? |

---

**QG.6 — Question ouverte : scalabilité**

Un opérateur a 1000 clients, chacun avec 10 sites. Expliquez pourquoi les routeurs P du cœur peuvent rester "légers" (peu d'état à maintenir) alors que le nombre de routes clients est potentiellement énorme. Quel mécanisme central rend cela possible ?

---

**QG.7 — Interopérabilité**

PE1 est un Cisco IOS-XR, PE2 est un Juniper MX. Les deux doivent établir une session MP-BGP VPNv4. Quels paramètres doivent impérativement être cohérents entre les deux équipements pour que la session fonctionne et les routes soient importées correctement ?

---

**QG.8 — Chronologie de mise en service**

Un ingénieur met en service un nouveau PE3 pour un client existant (VRF BANK_A déjà présente sur PE1 et PE2). Listez dans l'ordre les étapes de configuration et vérification nécessaires, depuis la connexion physique jusqu'au ping client fonctionnel.

---

**QG.9 — Analyse de panne**

Un client signale que son site C ne peut plus joindre son site A, mais peut encore joindre son site B. Les trois sites sont sur des PE différents (PE_A, PE_B, PE_C). Proposez une méthodologie de diagnostic structurée (plan de contrôle puis plan de données) pour isoler le problème.

---

**QG.10 — Question de conception**

Un opérateur envisage de remplacer LDP par **Segment Routing (SR-MPLS)** sur son backbone. Quels avantages SR-MPLS apporte-t-il par rapport à LDP pour un déploiement L3 VPN ? Quels composants du L3 VPN restent inchangés (VRF, RD, RT, MP-BGP) et pourquoi ?

---

## 14. Corrections

> Les corrections sont regroupées par section dans le même ordre que les exercices.

---

### Corrections — Section 1

**C1.1** — Pour N sites en maillage complet : `N×(N-1)/2` liens. Pour 8 sites : 28 liens. Pour 20 sites : 190 liens. La formule est celle des combinaisons C(N,2).

**C1.2** —

| Rôle | Appartient à | Connaît les VRF | Commute des labels |
|---|---|---|---|
| CE | Client | Non | Non |
| PE | Opérateur | Oui | Oui (push/pop) |
| P | Opérateur | Non | Oui (swap) |

**C1.3** — Un VPN IPsec passe par Internet public, sans garantie de débit ni de latence, et chiffre les données. Un L3 VPN MPLS utilise l'infrastructure privée de l'opérateur, offre des garanties contractuelles (SLA) de latence et de bande passante, et n'a pas besoin de chiffrement car le trafic est isolé sur un réseau privé.

**C1.4** — **Faux**. Le CE n'a pas connaissance de MPLS. Il voit simplement un routeur PE voisin avec lequel il échange des routes via un protocole standard (BGP, OSPF, statique). MPLS est entièrement transparent pour le CE.

**C1.5** — Le L3 VPN est un service de couche 3 car l'opérateur route les paquets IP entre les sites — il participe au plan de routage IP du client. Conséquence : le CE voit dans sa table de routage les réseaux des autres sites du client, comme s'ils étaient des voisins IP directs.

**C1.6** — Deux erreurs : (1) le L3 VPN MPLS ne passe **pas par Internet** — il utilise le réseau privé de l'opérateur ; (2) le L3 VPN MPLS **n'utilise pas de tunnel chiffré** — l'isolation est assurée par les VRF et MPLS, pas par du chiffrement.

**C1.7** — Avec Frame Relay, 500 clients × 5 sites × C(5,2) = 500 × 10 = 5000 circuits à gérer. Chaque circuit a sa propre configuration, supervision et SLA. Avec L3 VPN MPLS, l'infrastructure est mutualisée : l'opérateur configure une VRF par client sur chaque PE concerné, et MPLS gère le transport. Ajouter un site = configurer une VRF sur un PE et un protocole PE-CE, sans toucher aux autres circuits.

**C1.8** — Le routeur P commute des labels dans le plan de données : il reçoit un paquet avec un label, consulte sa LFIB, effectue un SWAP et redirige. Il ne connaît pas les adresses IP des clients. C'est fondamental pour la scalabilité : les P peuvent gérer des dizaines de millions de paquets par seconde sans maintenir l'état de milliers de VRF clients.

**C1.9** — La RFC 2547 (BGP/MPLS VPNs), publiée en **1999** par Rosen et Rekhter, a précédé la RFC 4364.

**C1.10** — Avec Frame Relay en any-to-any pour 10 sites : 45 PVC à configurer et superviser. Chaque nouveau site exige N-1 nouveaux circuits. Avec L3 VPN MPLS : un seul VPN avec RT any-to-any suffit. Ajouter un site = une VRF sur un PE. La complexité est O(N) au lieu de O(N²).

---

### Corrections — Section 2

**C2.1** — `0x000641FF` en binaire :
```
0000 0000 0000 0110 0100 0001 1111 1111
|←────── Label 20b ──────→|TC |S|←TTL→|
Label = 0x00064 = 100
TC    = 0b000   = 0
S     = 1
TTL   = 0xFF    = 255
```

**C2.2** — Après SWAP par le routeur P :
```
[Label=600, TC=0, S=0, TTL=61]   ← label swappé, TTL décrémenté
[Label=25,  TC=0, S=1, TTL=62]   ← inchangé (P ne touche pas l'inner label)
[IP src=10.1.0.1, dst=10.2.0.1]
```

**C2.3** — La **FIB** est la table de forwarding IP : elle mappe un préfixe IP (lookup par LPM — Longest Prefix Match) vers une interface et un next-hop. La **LFIB** mappe un label entrant (lookup par exact match) vers un label sortant + interface + action. Le lookup par exact match est plus simple et plus rapide que le LPM.

**C2.4** — PHP (Penultimate Hop Popping) : le routeur P immédiatement avant le PE egress retire le label de transport (outer label) avant de transmettre le paquet au PE. Cela évite au PE egress de faire deux lookups successifs (LFIB pour le label transport → LFIB pour le label VPN). Le PE egress reçoit directement le paquet avec uniquement le label VPN (S=1), et fait un seul lookup pour identifier la VRF et le CE de sortie.

**C2.5** — Dans la trame Ethernet, le champ EtherType vaut **`0x8847`** pour un paquet MPLS unicast (ou `0x8848` pour multicast). L'équipement de capture identifie la présence de labels MPLS grâce à cette valeur.

**C2.6** — **Vrai**, avec une nuance : le TTL MPLS fonctionne effectivement comme le TTL IP (décrémenté à chaque saut, jeté si 0). En pratique, les opérateurs peuvent désactiver la propagation du TTL IP dans le label MPLS (`no mpls ip propagate-ttl`) pour masquer la topologie interne du backbone aux traceroutes des clients.

**C2.7** — PUSH est effectué par le **PE ingress** : quand un paquet IP arrive depuis le CE, le PE effectue un lookup dans la VRF, détermine le label VPN (inner), puis détermine le label transport (outer) via la LFIB, et empile les deux labels sur le paquet avant de l'envoyer dans le cœur MPLS.

**C2.8** — Le label **3 (Implicit Null)** est un label réservé MPLS signifiant "retire le label avant de m'envoyer le paquet". Quand un PE egress annonce via LDP le label Implicit Null pour son propre loopback, le routeur voisin (P pénultième) comprend qu'il doit effectuer PHP : retirer ce label et envoyer le paquet sans label au PE. C'est la mise en œuvre standard de PHP.

**C2.9** — Dans une pile à deux labels, le **label outer** (transport) est celui dont le bit **S=0** (pas le fond de la pile). Le **label inner** (VPN) a le bit **S=1** (bottom of stack). C'est le bit S qui marque la frontière : le routeur sait qu'il a atteint le dernier label quand S=1.

**C2.10** — TC sur 3 bits = **8 valeurs distinctes** (0 à 7). Elles correspondent aux **classes de service** (CoS/QoS) : valeur 0 = best effort, valeurs croissantes = priorité croissante (7 = contrôle réseau). Les routeurs P peuvent utiliser ce champ pour appliquer des politiques de queuing/scheduling différenciées selon la priorité du flux.

---

### Corrections — Section 3

**C3.1** — Phase 1 : **découverte** via messages Hello LDP envoyés en UDP multicast (`224.0.0.2`, port 646). Phase 2 : **établissement de session** via TCP (port 646) entre les deux routeurs (connexion TCP initiée par le routeur avec la plus grande adresse IP de son Router-ID LDP). UDP est utilisé pour la découverte (pas de connexion préalable), TCP pour la session car il offre fiabilité et ordre de livraison pour les échanges de labels.

**C3.2** — Messages LDP Label Mapping échangés :
```
PE1 → P  : FEC=10.0.0.1/32, Label=L1  ("pour me joindre, utilise L1")
P  → PE1 : FEC=10.0.0.99/32, Label=L2
P  → PE1 : FEC=10.0.0.2/32,  Label=L3  ("pour joindre PE2 via moi, utilise L3")
PE2 → P  : FEC=10.0.0.2/32,  Label=L4  ("pour me joindre, utilise L4")
P  → PE2 : FEC=10.0.0.99/32, Label=L5
P  → PE2 : FEC=10.0.0.1/32,  Label=L6  ("pour joindre PE1 via moi, utilise L6")
```
Le LSP PE1→PE2 utilise : PE1 PUSH L3, P SWAP vers L4 (ou PHP), PE2 reçoit.

**C3.3** — La **LIB** (Label Information Base) stocke tous les labels reçus de tous les voisins LDP pour toutes les destinations, y compris ceux des voisins qui ne sont pas le next-hop actuel (Liberal Retention). La **LFIB** ne contient que les labels actifs utilisés pour le forwarding (le meilleur next-hop). Seule la LFIB est consultée lors du forwarding d'un paquet.

**C3.4** — **Liberal** : on conserve les labels reçus de tous les voisins même si ce n'est pas le next-hop actuel. Avantage : lors d'une panne, les labels du chemin alternatif sont déjà connus, la convergence est quasi-instantanée après la reconvergence IGP. **Conservative** : on ne conserve que les labels du next-hop actif. Avantage : moins de mémoire. Inconvénient : en cas de panne, il faut re-solliciter les labels du nouveau chemin. Liberal est le mode par défaut sur IOS, préférable en production pour sa convergence plus rapide.

**C3.5** — LDP distribue des labels pour **toutes les routes de la table IP globale**, mais les plus importantes sont les **loopbacks des PE** (routes /32). Ce sont ces adresses que MP-BGP utilise comme next-hop des routes VPNv4 — si le LSP vers un loopback PE n'existe pas, les routes VPNv4 de ce PE ne peuvent pas être forwardées.

**C3.6** — **Non**, LDP ne peut pas forcer un chemin différent du plus court chemin IGP — il distribue des labels en suivant la topologie IGP. Pour l'ingénierie de trafic avec chemin explicite, il faut utiliser **RSVP-TE** (Resource Reservation Protocol - Traffic Engineering), qui permet de signaler des LSP avec des chemins spécifiques et des réservations de bande passante.

**C3.7** — **Vrai**. Un LSP est unidirectionnel. Le LSP PE1→PE2 est distinct du LSP PE2→PE1. Chaque sens est établi indépendamment par LDP, avec ses propres labels.

**C3.8** — Un **FEC (Forwarding Equivalence Class)** est un ensemble de paquets traités de manière identique (même next-hop, même label). Dans LDP, un FEC correspond typiquement à un préfixe IP (`10.0.0.1/32`). Exemple concret : tous les paquets destinés à `10.0.0.2/32` (loopback de PE2) appartiennent au même FEC et reçoivent le même label de transport.

**C3.9** — Causes possibles :
1. LDP non activé sur l'interface (`no mpls ldp autoconfig` ou absence de `mpls ip` sur l'interface)
2. Le loopback LDP Router-ID n'est pas joignable / non configuré
3. ACL ou pare-feu bloquant UDP 646 (messages Hello) ou TCP 646

**C3.10** — **LDP IGP Sync** : mécanisme qui retarde l'utilisation d'un lien dans l'IGP jusqu'à ce que la session LDP sur ce lien soit opérationnelle et que les labels aient été échangés. Sans ce mécanisme, l'IGP peut commencer à router du trafic sur un lien avant que LDP ait distribué les labels, causant une perte de trafic MPLS pendant la fenêtre de convergence LDP.

---

### Corrections — Section 4

**C4.1** — Les loopbacks PE sont les adresses utilisées comme **next-hop des routes VPNv4 dans MP-BGP** (NEXT_HOP dans les UPDATE). Si un loopback PE n'est pas dans l'IGP, les autres PE ne peuvent pas le résoudre → le LSP n'existe pas → les routes VPNv4 de ce PE sont inaccessibles (next-hop unreachable) et ne sont pas installées dans les VRF.

**C4.2** — `49.0001.0102.0304.0506.00` :
- AFI : `49` (adressage privé ISO)
- Area ID : `0001`
- System-ID : `0102.0304.0506` (6 octets, identifiant unique du routeur)
- NSEL : `00` (toujours 00 pour un routeur)

**C4.3** — `level-1` : route uniquement dans son aire (comme OSPF intra-zone). `level-2-only` : route entre aires (comme OSPF backbone). `level-1-2` : participe aux deux niveaux. Pour les routeurs du cœur MPLS (P et PE), **`level-2-only`** est recommandé : tous dans le même domaine L2, pas de hiérarchie compliquée, convergence plus simple.

**C4.4** — **Non**. Les routes clients (contenues dans les VRF) ne sont **jamais annoncées dans l'IGP du cœur**. Elles sont transportées exclusivement par MP-BGP entre PE. L'IGP ne transporte que les routes d'infrastructure (loopbacks PE/P, liens entre routeurs). C'est ce qui rend l'architecture scalable : l'IGP reste léger même avec des milliers de VRF clients.

**C4.5** — Sans Loopback0 dans IS-IS sur PE1 : les autres PE ne connaissent pas l'adresse loopback de PE1 → ils ne peuvent pas établir de session iBGP avec PE1 (le next-hop MP-BGP de PE1 est son loopback) → aucune session MP-BGP fonctionnelle avec PE1. Plan de données : même si BGP était établi par un autre moyen, il n'y a pas de LSP vers PE1 → le trafic VPN entrant vers PE1 ne peut pas être forwardé.

**C4.6** — IS-IS utilise ses propres PDU (Protocol Data Units) au niveau L2 (Data Link), sans dépendre d'IP pour leur transport. Avantage opérateur : IS-IS continue de fonctionner même en cas de problème de configuration IP. Il est aussi plus simple d'ajouter des extensions (ex: SR extensions) sans contraintes IP. Enfin, IS-IS n'a pas de concept d'area backbone obligatoire comme l'Area 0 d'OSPF.

**C4.7** — Les liens point-à-point `/30` entre P et PE **doivent être dans l'IGP** pour que les routeurs s'atteignent et établissent LDP. En revanche, ils ne sont pas strictement nécessaires pour le forwarding MPLS (les labels suffisent). En pratique, on les annonce dans l'IGP mais on peut les filtrer de la table BGP pour ne pas les propager aux clients.

**C4.8** — Problèmes OSPF avec 500 routeurs en Area 0 unique : (1) la **base de données LSDB** devient énorme (un LSA par routeur + un par lien), le calcul SPF est lourd ; (2) toute **fluctuation de lien** déclenche une inondation d'un LSA dans tout le domaine et un SPF sur tous les routeurs. IS-IS résout ces problèmes avec les niveaux L1/L2 (hiérarchie naturelle) et une gestion plus efficace de l'inondation des LSP.

**C4.9** — L'IGP doit converger **avant** LDP, car LDP suit la topologie IGP pour construire les LSP. Et LDP doit converger avant que le trafic MPLS puisse transiter. Ordre : détection de panne → reconvergence IGP → reconvergence LDP → trafic MPLS rétabli.

**C4.10** — **Faux** (en pratique). Techniquement, on pourrait utiliser iBGP pour propager les loopbacks dans le cœur, mais BGP a une convergence beaucoup plus lente qu'OSPF ou IS-IS, et nécessite qu'un IGP soit déjà en place pour établir les sessions BGP. En production, iBGP ne remplace jamais un IGP pour le cœur MPLS.

---

### Corrections — Section 5

**C5.1** — Le routeur consulte la **VRF du client concerné**. Chaque paquet entrant est associé à une VRF via l'interface d'entrée (qui est assignée à une VRF spécifique). Une fois la VRF identifiée, le lookup se fait dans la table de routage de cette VRF uniquement. Deux clients avec le même `10.0.0.0/8` n'entrent en conflit que si un paquet pouvait appartenir aux deux VRF simultanément — ce qui est impossible car chaque interface appartient à une seule VRF.

**C5.2** — L'adresse IP `192.168.10.1/24` est **immédiatement supprimée** de l'interface. L'ingénieur doit reconfigurer l'adresse IP sur l'interface après avoir tapé `ip vrf forwarding BANK_A`. C'est parce que l'interface bascule de la table de routage globale vers la table VRF, et l'ancienne adresse IP était associée à la table globale.

**C5.3** — **VRF (L3 VPN MPLS)** : les VRF sont associées à des RD/RT et les routes sont échangées entre PE via MP-BGP avec MPLS comme plan de données. **VRF-Lite** : les VRF sont utilisées sans MPLS — les routes entre VRF de routeurs différents sont échangées via des sessions eBGP normales sur des interfaces dédiées. VRF-Lite est utilisé en entreprise pour séparer des zones de sécurité ou des clients sans déployer MPLS.

**C5.4** — **Non**. La table globale d'un PE contient uniquement les routes d'infrastructure (loopbacks, liens P-PE, routes IGP du cœur) et les routes BGP de signalisation. Les routes des clients restent dans leurs VRF respectives et sont **complètement invisibles** de la table globale. C'est le mécanisme d'isolation fondamental.

**C5.5** — **Faux**. Une interface physique (ou sous-interface) ne peut appartenir **qu'à une seule VRF** à la fois. Pour servir plusieurs clients sur le même lien physique, on utilise des sous-interfaces (802.1Q), chacune assignée à une VRF différente.

**C5.6** — Minimum **3 VRF** (une par PE qui dessert le client). Si les 3 sites sont sur 3 PE distincts, chaque PE a une VRF pour ce client. Le nombre de VRF = nombre de PE qui desservent au moins un site de ce client.

**C5.7** — `show ip route vrf BANK_A` — affiche la table de routage de la VRF BANK_A : toutes les routes connues pour ce client (apprises du CE local, et importées via MP-BGP depuis les autres PE). On y voit les préfixes, les next-hops, les métriques et les protocoles sources.

**C5.8** — **Non**. Le CE n'a aucune connaissance des VRF. Il voit simplement un routeur voisin (le PE) avec lequel il échange des routes via un protocole standard. La VRF est une abstraction entièrement côté PE.

**C5.9** — Modélisation : créer une VRF DNS_SHARED avec `export RT=999:999`. VRF CLIENT_A : `import RT=100:100, import RT=999:999`. VRF CLIENT_B : `import RT=200:200, import RT=999:999`. CLIENT_A n'importe pas le RT de CLIENT_B et vice versa → isolation entre clients. Les deux importent le RT de DNS_SHARED → accès au DNS commun.

**C5.10** — Le loopback du PE distant est une **adresse d'infrastructure opérateur**, pas une adresse du client. Elle est annoncée dans l'IGP du cœur et visible dans la table globale. La résolution dans la VRF serait impossible car la VRF ne contient que les routes du client — elle ne connaît pas les loopbacks des autres PE. C'est la séparation fondamentale entre l'infrastructure opérateur (table globale) et les données clients (VRF).

---

### Corrections — Section 6

**C6.1** — Sans RD, PE1 enverrait deux UPDATE BGP avec le même préfixe `172.16.0.0/16` à PE2. BGP applique sa règle de best-path selection et ne garde **qu'une seule route** par préfixe. L'une des deux routes client est perdue — les paquets d'un des deux clients sont forwardés vers le mauvais CE, ou la route est simplement absente.

**C6.2** — Le RD fait **64 bits** (8 octets). Deux formats :
- `AS:nn` : numéro AS 2 octets + valeur 4 octets. Ex : `65000:100`
- `IP:nn` : adresse IP 4 octets + valeur 2 octets. Ex : `10.0.0.1:100`

**C6.3** — **Valide**. Les RD peuvent différer entre PE pour la même VRF client. Le RD n'a aucune sémantique de filtrage ou d'identité VRF — c'est un simple préfixe d'unicité. La VRF est identifiée par son nom local sur chaque PE ; le filtrage se fait par RT. Des RD différents par PE sont même recommandés (voir C6.6).

**C6.4** — `65000:200:10.5.0.0/24` :
- RD : `65000:200` (64 bits)
- Préfixe IP : `10.5.0.0/24` (32 bits de réseau + 8 bits de longueur)
- Longueur totale du préfixe VPNv4 : **96 bits** (64 bits RD + 32 bits préfixe IP)

**C6.5** — **Non**. Le RD n'est pas utilisé pour le filtrage d'import. C'est le **RT (Route Target)** qui détermine dans quelle VRF une route est importée. Le RD sert uniquement à distinguer des préfixes identiques dans la table BGP VPNv4.

**C6.6** — Utiliser des RD différents par PE pour un même client résout un problème BGP : si deux PE annoncent la même route VPNv4 avec le même RD (ex : `65000:100:10.1.0.0/24`), BGP les traite comme deux chemins vers le même préfixe et peut n'en installer qu'un (add-path non activé). Avec des RD différents (`65000:101:10.1.0.0/24` depuis PE1 et `65000:102:10.1.0.0/24` depuis PE2), les deux routes sont distinctes dans la table VPNv4 → **meilleure résilience et load-balancing possibles**.

**C6.7** — **Faux**. Le RD est transparent au CE. Il existe uniquement dans les échanges MP-BGP entre PE et dans la table BGP VPNv4 des PE. Le CE voit uniquement des préfixes IP classiques.

**C6.8** — Erreur critique : si tous les clients ont le même RD `65000:1`, leurs routes VPNv4 ne sont plus distinguables dans la table BGP. Ex : CLIENT_A annonce `65000:1:10.0.0.0/24` et CLIENT_B annonce aussi `65000:1:10.0.0.0/24` → collision BGP, une des deux routes est écrasée → fuite ou perte de routage entre clients.

**C6.9** — Format IP:nn : ex `192.168.0.1:100` où `192.168.0.1` est le loopback du PE. On préfère ce format quand l'opérateur n'a pas de numéro AS 2 octets disponible (AS 4 octets ne tient pas dans le format AS:nn standard), ou pour garantir l'unicité par PE (l'IP loopback est naturellement unique par routeur).

**C6.10** — Dans `show bgp vpnv4 unicast all` :
```
Route Distinguisher: 65000:100 (VRF BANK_A)
*>i 10.1.0.0/24    192.168.1.1    0   100   0  65001 i
                   ↑ next-hop PE       ↑ local-pref
```
Le RD apparaît comme en-tête de section (`Route Distinguisher: 65000:100`), et chaque route listée dessous appartient à ce RD.

---

### Corrections — Section 7

**C7.1** — Le RD rend une route **unique** dans la table BGP VPNv4. Le RT décide dans quelle(s) VRF cette route est **importée**.

**C7.2** —
- VRF C importe-t-elle les routes de VRF A ? VRF C importe RT=100:20. VRF A exporte RT=100:10. `100:10 ≠ 100:20` → **Non**.
- VRF C importe-t-elle les routes de VRF B ? VRF C importe RT=100:20. VRF B exporte RT=100:20. → **Oui**.
- VRF A importe-t-elle les routes de VRF C ? VRF A importe RT=100:10. VRF C exporte RT=100:10. → **Oui**.
Topologie asymétrique : C reçoit les routes de A (via la réciproque), mais C ne reçoit pas les routes de B au sens attendu (B→C oui, mais pas dans le sens inverse sans vérification).

**C7.3** — Une **Extended Community BGP** est un attribut BGP optionnel transitionnel de 8 octets (RFC 4360), conçu pour transporter des informations supplémentaires attachées aux routes. Il est utilisé pour le RT car BGP doit pouvoir filtrer les routes selon ces valeurs sur les PE, et un attribut ordinaire ne dispose pas de la sémantique appropriée. De plus, les Extended Communities sont conçus pour être extensibles (plusieurs valeurs possibles) — une VRF peut avoir plusieurs RT export/import.

**C7.4** — Topologie any-to-any, tous communiquent avec tous :
```
HQ      : export RT=100:1, import RT=100:1, RT=100:2
Agences : export RT=100:2, import RT=100:1, RT=100:2
```
(Toutes les VRF importent tous les RT → tous se voient mutuellement.)

Plus simple :
```
HQ et Agences : export RT=65000:100, import RT=65000:100
```

**C7.5** — Topologie Hub & Spoke (agences ne se voient pas directement) :
```
HQ (Hub)  : export RT=65000:hub,   import RT=65000:spoke
Agences   : export RT=65000:spoke, import RT=65000:hub
```
Les agences exportent `spoke` → HUB importe `spoke` → HUB reçoit les routes de toutes les agences. HUB exporte `hub` → Agences importent `hub` → Agences reçoivent les routes du HUB (et donc indirectement des autres agences via le HUB qui les réexporte). Agences n'importent pas `spoke` → elles ne se voient pas directement.

**C7.6** — Schéma :
```
VRF CLIENT_A  : export RT=100:10, import RT=100:10, import RT=100:999
VRF CLIENT_B  : export RT=100:20, import RT=100:20, import RT=100:999
VRF DNS_SHARED: export RT=100:999, import RT=100:999
```
CLIENT_A et CLIENT_B n'importent pas leurs RT respectifs → isolation. Tous deux importent RT=100:999 → accès au DNS partagé.

**C7.7** — **Faux**. Une VRF peut exporter **plusieurs RT** simultanément. Cas d'usage : une VRF de services partagés qui doit être importée par plusieurs types de VRF clients. Ex : `VRF SHARED : export RT=100:999, export RT=100:888` pour deux communautés différentes d'importateurs.

**C7.8** — Le RT est une Extended Community BGP de 8 octets avec un type field en tête (2 octets) qui indique `Route Target` (type 0x0002 ou 0x0102). Les 6 octets restants portent la valeur `AS:nn`. Structurellement, RT et RD ont le même format numérique `AS:nn` mais leur rôle est radicalement différent : le RD est un préfixe BGP, le RT est un attribut de route. Ils sont transportés dans des champs BGP distincts.

**C7.9** — La route est installée **uniquement dans VRF BANK_A** (qui importe RT=65000:100). VRF CORP importe RT=65000:999 qui ne correspond pas → route non installée dans CORP.

**C7.10** — Dans Hub & Spoke, les spokes n'ont de session BGP qu'avec le Route Reflector (ou les PE en iBGP). Les routes des spokes arrivent sur le HUB PE via MP-BGP, sont importées dans la VRF HUB, puis **réexportées** par MP-BGP avec le RT=hub. Les PE des spokes importent ce RT et installent les routes dans leurs VRF SPOKE respectives. Le chemin des données passe par le HUB physiquement (car le HUB est le seul à avoir les routes de tous les spokes dans sa VRF, et les spokes routent vers le HUB pour atteindre n'importe quel autre spoke).

---

### Corrections — Section 8

**C8.1** — **AFI (Address Family Identifier)** : identifie la famille d'adresses (1 = IPv4, 2 = IPv6). **SAFI (Subsequent Address Family Identifier)** : sous-famille (1 = unicast, 128 = MPLS-labeled VPN). Pour le L3 VPN MPLS : **AFI=1, SAFI=128** (VPNv4).

**C8.2** — Les sessions PE-PE sont iBGP car tous les PE appartiennent au **même AS opérateur**. La règle iBGP implique le **split-horizon** : un routeur iBGP ne réannonce pas à un autre routeur iBGP une route apprise via iBGP. C'est pourquoi on utilise un **Route Reflector** (ou un full-mesh) pour éviter que les routes ne soient bloquées.

**C8.3** — Full-mesh iBGP pour N PE : `N×(N-1)/2`. Pour 10 PE : **45 sessions**. Avec un Route Reflector central : **10 sessions** (une par PE vers le RR). Le RR réfléchit les routes reçues d'un client vers tous les autres clients, contournant la règle du split-horizon.

**C8.4** — `update-source Loopback0` force BGP à utiliser l'adresse loopback comme source des paquets TCP BGP. Sans cette commande, BGP utilise l'adresse IP de l'interface de sortie physique. Si ce lien physique tombe, la session BGP tombe aussi — même si un chemin alternatif existe. Avec le loopback comme source (et le loopback annoncé dans l'IGP), la session BGP survit aux pannes de liens physiques car le loopback reste joignable via d'autres chemins.

**C8.5** — Symptôme : PE2 reçoit les routes VPNv4 de PE1 (elles apparaissent dans `show bgp vpnv4 unicast all`) mais **sans Extended Communities** (RT absent). Comme aucun RT ne correspond aux imports des VRF, **aucune route n'est installée** dans les VRF de PE2. `show bgp vpnv4 unicast vrf BANK_A` est vide. Commande de vérification : `show bgp vpnv4 unicast all` et inspecter la colonne Extended Community pour vérifier la présence du RT.

**C8.6** — `address-family vpnv4` : configure la session **PE-PE** (iBGP) pour échanger des routes VPNv4. C'est le canal de signalisation inter-PE qui transporte les routes de tous les clients avec leurs RD/RT/labels. `address-family ipv4 vrf BANK_A` : configure la session **PE-CE** pour un client spécifique — le PE participe au protocole de routage du client (eBGP, OSPF, etc.) dans le contexte de cette VRF.

**C8.7** — `MP_REACH_NLRI` contient :
- AFI/SAFI (1/128 pour VPNv4)
- NEXT_HOP : adresse loopback du PE source
- NLRI : liste de tuples `{RD + préfixe IP + longueur + label VPN}`

**C8.8** — PE2 résout le loopback de PE1 dans sa **table de routage globale** (table IP classique du PE, peuplée par l'IGP). La résolution se fait via l'IGP (IS-IS ou OSPF), qui a appris le loopback de PE1. Une fois l'adresse résolue en next-hop physique, LDP fournit le label de transport pour joindre ce next-hop.

**C8.9** — **Faux**. Le Route Reflector n'a pas besoin d'être un PE. Il peut être un routeur dédié sans VRF, qui se contente de réfléchir les routes VPNv4 entre clients BGP sans les installer. En pratique, les RR dédiés sont courants dans les grands réseaux opérateurs pour des raisons de scalabilité et de séparation des rôles.

**C8.10** — Exemples de familles MP-BGP : IPv6 unicast (AFI=2, SAFI=1), L2 VPN EVPN (AFI=25, SAFI=70), BGP-LS (Link State, pour distribuer la topologie IGP à des contrôleurs SDN), Flowspec (AFI=1, SAFI=133 pour distribuer des règles de filtrage), VPNv6 (AFI=2, SAFI=128).

---

### Corrections — Section 9

**C9.1** — PE1 ajoute les deux labels **au moment où le paquet IP entre depuis le CE**, après lookup dans la VRF. Ordre d'empilement (du bas vers le haut de la pile) : d'abord le **label VPN** (inner, S=1) est posé, puis le **label transport** (outer, S=0) est posé par-dessus. Sur le fil, le label transport est le premier rencontré (outer = premier lu).

**C9.2** —
```
Sortie CE1     : [IP src=10.1.0.1, dst=10.2.0.1, TTL=64]
Sortie PE1     : [T=300,S=0,TTL=63][VPN=42,S=1,TTL=64][IP TTL=64]
Sortie P1      : [T=350,S=0,TTL=62][VPN=42,S=1,TTL=64][IP TTL=64]  ← SWAP T
Sortie P2      : [VPN=42,S=1,TTL=64][IP TTL=64]                     ← PHP retire T
Sortie PE2     : [IP src=10.1.0.1, dst=10.2.0.5, TTL=64]            ← POP VPN, forward IP
Arrivée CE2    : [IP src=10.1.0.1, dst=10.2.0.5, TTL=64]
```
Note : le TTL MPLS est décrémenté par PE1, P1, P2 (sur le label transport). Le label VPN et le TTL IP peuvent rester inchangés pendant le transit cœur selon la configuration.

**C9.3** — Les routeurs P ne regardent que le **label outer** (premier de la pile). Le bit **S=0** sur le label outer indique qu'il y a d'autres labels en dessous — le routeur P effectue son SWAP sur ce seul label et ne cherche pas à lire ce qui suit. Il n'a donc structurellement aucun accès au label VPN (inner) ni à l'IP client.

**C9.4** — **Vrai**. Sans PHP, PE2 reçoit le paquet avec encore le label transport (outer). Il doit d'abord faire un lookup LFIB pour trouver que ce label correspond à lui-même (POP), puis faire un second lookup LFIB/VRF sur le label VPN (inner) pour déterminer la VRF et l'interface de sortie. Avec PHP, PE2 ne reçoit que le label VPN et fait un seul lookup.

**C9.5** — Le label VPN (inner) est distribué par **MP-BGP**. Il est annoncé par le PE egress (PE2) dans le champ `MP_REACH_NLRI` de l'UPDATE BGP VPNv4, au moment où PE2 exporte la route d'un CE vers les autres PE. Chaque PE alloue ses propres labels VPN pour chaque VRF locale.

**C9.6** — Le label transport (outer) est distribué par **LDP** (ou RSVP-TE). Il correspond au label que le routeur P a annoncé pour atteindre le loopback de PE2. PE1 apprend ce label via sa session LDP avec son voisin P direct.

**C9.7** — PE2 reçoit `[VPN=25][IP...]` :
1. Lookup dans la **LFIB** : label 25 → action POP, VRF BANK_A, interface `GigE0/1` vers CE2
2. PE2 retire le label VPN (POP)
3. Forward l'IP brut `dst=10.2.0.1` sur l'interface vers CE2 (éventuellement avec un lookup IP dans la VRF si le label seul ne suffit pas à identifier l'interface de sortie précise)

**C9.8** — MPLS opère à la couche 2.5 car les labels sont insérés **entre l'en-tête L2 (Ethernet) et l'en-tête L3 (IP)** dans la trame. Sur le fil, la trame Ethernet a EtherType `0x8847` (au lieu de `0x0800` pour IPv4), suivi des labels MPLS 32 bits, puis de l'en-tête IP. MPLS ne modifie ni le L2 ni le L3 — il s'intercale entre les deux.

**C9.9** — Avec 3 routeurs P entre PE1 et PE2 : **3 opérations SWAP** (une par routeur P sur le label transport), sauf le dernier P qui fait PHP (POP au lieu de SWAP). Le **label VPN n'est jamais modifié** pendant le transit dans le cœur — seul le label transport change à chaque saut.

**C9.10** — TTL IP de départ : 64. Traversée : PE1 (décrémente TTL IP de 1 si propagation TTL activée → 63), P1 (→62), P2 (→61), PE2 (→60). À l'arrivée sur CE2 : **TTL=60**. Si `no mpls ip propagate-ttl` est configuré sur le backbone, le TTL MPLS est initialisé indépendamment et le TTL IP n'est décrémenté que par PE1 et PE2 → CE2 reçoit TTL=62, et la topologie interne est masquée (les routeurs P sont invisibles au traceroute).

---

### Corrections — Section 10

**C10.1** — Session **PE-CE** : eBGP (AS différents — AS opérateur vs AS client), transporte des routes IPv4 unicast normales (préfixes IP du client). Session **PE-PE** : iBGP (même AS opérateur), transporte des routes VPNv4 (préfixes avec RD + labels MPLS) via MP-BGP avec AFI=1, SAFI=128.

**C10.2** — Problème : quand PE2 reçoit la route `10.1.0.0/24` (apprise de CE1 via PE1), elle contient l'AS 65001 dans son AS-PATH. PE2 l'annonce à CE2 (AS 65001). CE2 voit son propre AS dans l'AS-PATH → BGP rejette la route (loop prevention). Solutions : (1) **`as-override`** sur PE2 : remplace l'AS du CE source par l'AS du PE dans l'AS-PATH avant d'annoncer à CE2 ; (2) **`allowas-in`** sur CE2 : CE2 accepte les routes contenant son propre AS (moins sécurisé).

**C10.3** — Quand OSPF est utilisé en PE-CE, le PE redistribue les routes des autres sites (apprises via MP-BGP) dans le processus OSPF du client. Ces routes redistribuées apparaissent comme des routes **externes OSPF de type 2 (O E2)** côté CE, car elles franchissent la "frontière" OSPF que constitue le PE. Le "super-backbone" MPLS est traité comme un backbone OSPF de type spécial, ce qui peut poser des problèmes avec les routes inter-area si la topologie OSPF du client n'est pas bien conçue.

**C10.4** — Sur PE1 : route statique `ip route vrf BANK_A 10.2.0.0/24 <next-hop-CE2>` (ou simplement pointer vers l'interface CE-facing), puis redistribution dans MP-BGP via `redistribute static` dans l'address-family VRF correspondante. Même chose sur PE2 pour le réseau du site 1. Sans redistribution, les routes statiques ne sont pas propagées aux autres PE.

**C10.5** — (1) CE1 annonce `10.10.0.0/24` à PE1 via eBGP (AS-PATH: 65001). (2) PE1 importe la route dans VRF BANK_A, lui ajoute un RD et un label VPN, et l'annonce à PE2 via MP-BGP iBGP (AS-PATH conservé : 65001 ; next-hop = loopback PE1). (3) PE2 reçoit la route VPNv4, vérifie le RT, l'importe dans VRF BANK_A locale, résout le next-hop (loopback PE1) via IGP+LDP. (4) PE2 annonce `10.10.0.0/24` à CE2 via eBGP (AS-PATH: 65000 65001 — l'AS de PE est ajouté).

**C10.6** — **Faux**. Le CE n'a aucune configuration MPLS. MPLS est entièrement opéré dans le backbone opérateur (entre PE et P). Le CE voit simplement un voisin de routage classique.

**C10.7** — `show ip bgp vpnv4 vrf BANK_A neighbors <IP-CE> routes` — affiche les routes reçues du CE dans la VRF BANK_A. Ou `show ip route vrf BANK_A` pour voir toutes les routes installées dans la VRF.

**C10.8** — Avec eBGP PE-CE, si le lien vers CE tombe, BGP détecte la panne (via timers ou BFD) et retire automatiquement les routes du CE de la VRF → les autres PE ne reçoivent plus ces routes via MP-BGP → le trafic n'est plus routé vers un CE inaccessible. Avec une route statique, la route reste dans la VRF même si le CE est down → trafic envoyé vers un CE inaccessible, black-hole.

**C10.9** — Le CE peut utiliser l'attribut **MED (Multi-Exit Discriminator)** pour indiquer sa préférence d'entrée au PE. Le PE prend en compte le MED lors de la sélection du chemin BGP. Côté opérateur, si plusieurs PE desservent le même client, le LOCAL_PREF peut aussi être utilisé en interne pour influencer le PE préféré.

**C10.10** — Le loopback du PE distant est une adresse **d'infrastructure opérateur**, peuplée dans la table globale par l'IGP. La VRF ne contient que les routes du client — elle n'a aucune visibilité sur les loopbacks des PE. Si BGP tentait de résoudre le next-hop dans la VRF, il ne trouverait rien. C'est architecturalement la séparation entre plan d'infrastructure (table globale + IGP + LDP) et plan client (VRF + MP-BGP).

---

### Corrections — Section 11

**C11.1** —
- **Option A** : chaque ASBR a des VRF pour chaque client et échange les routes via eBGP IPv4 classique sur des interfaces dédiées par client.
- **Option B** : les ASBR échangent directement les routes VPNv4 via une session eBGP VPNv4, sans VRF sur les ASBR.
- **Option C** : les PE établissent des sessions MP-eBGP multihop directement entre eux (inter-AS), les ASBR se contentent de propager les loopbacks des PE.

**C11.2** — Option A : **1 interface (ou sous-interface) par VRF** de chaque côté. Pour 80 VRF : 80 interfaces sur ASBR1 + 80 interfaces sur ASBR2. Impact opérationnel : pour ajouter un client, il faut créer une sous-interface, une VRF, et une session eBGP sur chaque ASBR. Très laborieux et fragile à grande échelle.

**C11.3** — Avec Option B, le next-hop des routes VPNv4 reçues par ASBR2 est l'adresse d'ASBR1 (dans AS1). Ce next-hop n'est pas joignable depuis AS2. ASBR2 doit réécrire le **NEXT_HOP** avec sa propre adresse avant de propager les routes VPNv4 dans AS2 (`next-hop-self`). De même, les labels de transport doivent être réattribués entre les deux AS.

**C11.4** — Avec Option C, les PE de l'AS2 doivent connaître les loopbacks des PE de l'AS1 pour établir les sessions eBGP multihop et construire les LSP. Cela se fait par **redistribution des loopbacks PE de l'AS1 dans l'IGP de l'AS2** via les ASBR (ou par une session eBGP entre ASBR qui redistribue ces routes). Les ASBR servent de point de redistribution des routes d'infrastructure inter-AS.

**C11.5** —
- **Option A** : ASBR maintient une VRF + une session eBGP par client → charge proportionnelle au nombre de clients, très lourde.
- **Option B** : ASBR maintient toutes les routes VPNv4 (table BGP VPNv4 complète) + une seule session eBGP VPNv4 → charge moyenne, table VPNv4 potentiellement grande.
- **Option C** : ASBR ne maintient que les loopbacks PE (quelques routes) → charge minimale sur les ASBR ; la charge est sur les PE (sessions eBGP multihop directes).

**C11.6** — **Faux**. Avec Option A, les ASBR voient les routes sous forme **IPv4 classique** (sans RD) car ils les échangent via eBGP IPv4 standard sur des interfaces VRF dédiées par client. Le RD est supprimé lors du passage de la VRF ASBR1 à la session eBGP IPv4, puis reconstruit côté ASBR2.

**C11.7** — Plus scalable : les ASBR n'ont pas d'état client → peuvent être de simples routeurs P. Les PE communiquent directement, les routes VPN ne "touchent" pas les ASBR. Plus complexe à opérer : les loopbacks PE doivent être échangés entre AS (configuration et vérification délicates), les sessions eBGP multihop entre PE traversent plusieurs sauts, le TTL BGP doit être configuré explicitement, et le troubleshooting est plus difficile (le problème peut être dans l'IGP, LDP, ou BGP sur deux AS distincts).

**C11.8** — Si un ASBR tombe (Option B), toutes les routes VPNv4 qu'il propageait sont perdues → panne totale du service inter-AS. Mitigation : **deux ASBR en parallèle** avec une session eBGP VPNv4 sur chacun + redondance physique. Les routes sont propagées par les deux ASBR, et en cas de panne d'un ASBR, l'autre prend le relais.

**C11.9** — Peu de VRF (< 10), priorité simplicité → **Option A** : facile à comprendre, pas besoin de configuration BGP VPNv4 sur les ASBR, chaque client est isolé sur sa propre interface. Centaines de VRF, priorité scalabilité → **Option C** : ASBR légers, pas d'état client sur les ASBR, les PE gèrent directement leurs sessions BGP.

**C11.10** — `ebgp-multihop` est nécessaire car par défaut BGP eBGP n'accepte que les voisins directement adjacents (TTL=1). PE1 et PE2 sont séparés par plusieurs sauts (ASBR, routeurs P) → le TTL BGP serait épuisé avant d'atteindre le PE distant. La valeur de `ebgp-multihop` doit être égale ou supérieure au **nombre de sauts entre les deux PE** (ex: `ebgp-multihop 5` pour PE1 → ASBR1 → ASBR2 → PE2 = 3 sauts, une valeur de 5 laisse une marge).

---

### Corrections — Section 12

**C12.1** — Ordre chronologique :
1. Détection de panne (BFD ou expiration timer Hello IGP)
2. L'IGP inonde le réseau d'une mise à jour (LSA OSPF ou LSP IS-IS) et recalcule SPF
3. LDP reconverge sur le nouveau chemin (utilise les labels déjà connus via Liberal Retention)
4. Les LSP sont rétablis sur le nouveau chemin physique
5. Le trafic VPN reprend (MP-BGP n'a pas bougé, les labels VPN sont inchangés)

**C12.2** — MP-BGP n'a pas besoin de reconverger car les sessions iBGP sont établies entre les **loopbacks des PE**, qui restent joignables via d'autres chemins dans le cœur. Tant que les loopbacks PE sont atteignables (via l'IGP reconvergé), les sessions BGP restent up. Les routes VPNv4 et les labels VPN n'ont pas changé — seul le chemin de transport (labels LDP) a changé.

**C12.3** — Liberal Retention : LDP conserve les labels de **tous** les voisins, même ceux qui ne sont pas le next-hop actuel. Lors d'une panne, le nouveau next-hop IGP (chemin alternatif) correspond à un label déjà connu dans la LIB → la LFIB est mise à jour immédiatement sans attendre un nouvel échange LDP. Conservative Retention : les labels du chemin alternatif ne sont pas stockés → après la reconvergence IGP, il faut attendre que LDP sollicite et reçoive les labels du nouveau voisin next-hop, ajoutant un délai supplémentaire.

**C12.4** — BFD (Bidirectional Forwarding Detection) est un protocole de détection de panne ultra-rapide qui envoie des paquets de contrôle à intervalles de quelques millisecondes (typiquement 3×50ms = 150ms de détection). Comparé aux timers IGP classiques (Hello OSPF = 10s, Dead = 40s → détection en ~40s ; IS-IS Hello = 3-30s), BFD réduit le délai de détection de **30-40 secondes à moins de 1 seconde**.

**C12.5** — **Faux**. Les labels VPN (inner labels) sont alloués par les PE et annoncés via MP-BGP. Une panne dans le cœur (entre P ou entre P et PE) n'affecte pas les PE egress qui conservent leurs labels VPN. Seuls les **labels de transport** (outer labels, distribués par LDP) changent lors de la reconvergence sur un nouveau chemin.

**C12.6** — MPLS FRR avec RSVP-TE est un mécanisme de protection pré-calculée : avant toute panne, des **LSP de secours** (bypass tunnels ou detour paths) sont établis. En cas de panne, le trafic est reroué sur le LSP de secours en **moins de 50ms** (au niveau du plan de données, sans attendre la reconvergence IGP). Le routeur qui détecte la panne reroute localement, sans signalisation supplémentaire.

**C12.7** — P1 doit :
1. Détecter la panne du lien P1-P2 (BFD ou timer)
2. Consulter sa LFIB pour trouver un chemin alternatif vers PE2 (via P3 si un label alternatif est disponible — Liberal Retention)
3. Mettre à jour sa LFIB : le label entrant pour PE2 est désormais forwardé vers P3 au lieu de P2
4. Si le label pour P3→PE2 n'est pas encore connu, attendre la reconvergence LDP sur le nouveau chemin

**C12.8** — TI-LFA (Topology Independent Loop-Free Alternate) est une évolution du FRR qui calcule des chemins de secours **garantis sans boucle** pour n'importe quelle topologie, y compris les topologies où le FRR classique ne peut pas trouver de chemin alternatif sûr. TI-LFA utilise Segment Routing pour encoder le chemin de secours directement dans la pile de labels, sans avoir besoin de pré-signaler des tunnels RSVP-TE. Il offre une couverture proche de 100% de la topologie (vs FRR classique qui peut manquer certains scénarios).

**C12.9** — Le **plan de contrôle** converge en premier (IGP recalcule les routes, LDP redistribue les labels). Puis le **plan de données** bascule sur les nouvelles entrées LFIB. La distinction est importante : pendant la convergence du plan de contrôle, le plan de données peut continuer à utiliser les anciennes entrées LFIB (forwarding sur l'ancien chemin, qui peut être down) ou black-holer le trafic. FRR résout cela en basculant le plan de données **avant** la reconvergence complète du plan de contrôle.

**C12.10** — Actions possibles pour réduire le délai de convergence :
1. Activer **BFD** sur les interfaces du cœur (détection en < 1 seconde)
2. Réduire les timers Hello/Dead de l'IGP (`hello-interval 1, dead-interval 3` pour IS-IS ou OSPF)
3. Activer les **timers SPF rapides** (`timers throttle spf 10 100 5000` sur OSPF)
4. Déployer **LDP IGP Sync** pour éviter les micro-pannes au redémarrage d'un lien

---

### Corrections — Section 13 (Exercices généraux)

**CG.1** — Cinq causes possibles :

1. **`address-family vpnv4` non activée** : `neighbor X activate` manquant dans l'address-family vpnv4 → la famille VPNv4 n'est pas négociée.
2. **`send-community extended` absent** : les RT ne sont pas transmis → PE2 ne peut pas filtrer les routes dans les VRF.
3. **RT mismatch** : RT exporté par PE1 ≠ RT importé dans VRF BANK_A de PE2.
4. **Session BGP down** : loopback de PE1 non joignable (IGP ou LDP non convergé) → session iBGP down.
5. **VRF non configurée sur PE2** (ou RD absent) : PE2 n'a pas de VRF BANK_A → nulle part où importer les routes.

Causes bonus : interface CE-facing non assignée à la VRF, redistribution des routes CE non configurée dans l'address-family VRF.

**CG.2** —

```
Sortie CE1  : [IP src=10.1.0.1, dst=10.2.0.5, TTL=64]

Sortie PE1  : [T=300, S=0, TTL=63]
              [VPN=42, S=1, TTL=64]
              [IP src=10.1.0.1, dst=10.2.0.5, TTL=64]
              (PE1 PUSH T=300 outer, VPN=42 inner)

Sortie P1   : [T=350, S=0, TTL=62]      ← SWAP 300→350, TTL outer décrémenté
              [VPN=42, S=1, TTL=64]      ← inchangé
              [IP src=10.1.0.1, dst=10.2.0.5, TTL=64]

Sortie P2   : [VPN=42, S=1, TTL=64]     ← PHP : label T retiré par P2
              [IP src=10.1.0.1, dst=10.2.0.5, TTL=64]

Sortie PE2  : [IP src=10.1.0.1, dst=10.2.0.5, TTL=64]
              (PE2 POP VPN=42, lookup VRF BANK_A → forward vers CE2)

Arrivée CE2 : [IP src=10.1.0.1, dst=10.2.0.5, TTL=64]
```

**CG.3** — Schéma RT :

```
HQ         : export RT=200:1, import RT=200:1, RT=200:2, RT=200:3
Agences    : export RT=200:2, import RT=200:1, RT=200:2
Partenaires: export RT=200:3, import RT=200:1
```

Vérification des contraintes :
- HQ ↔ Agences : HQ importe 200:2 ✅, Agences importent 200:1 ✅
- HQ ↔ Partenaires : HQ importe 200:3 ✅, Partenaires importent 200:1 ✅
- Agences ↔ Agences : Agences importent 200:2 → elles se voient entre elles ✅
- Partenaires ↔ Agences : Partenaires importent uniquement 200:1 (HQ), pas 200:2 (Agences) ✅ isolation
- Partenaires ↔ Partenaires : Partenaires importent 200:1, pas 200:3 → aucun partenaire ne voit les autres ✅

**CG.4** —

a) CLIENT_X exporte RT=65000:50. CLIENT_Y importe RT=65000:60. `50 ≠ 60` → CLIENT_Y n'importe **pas** les routes de CLIENT_X. CLIENT_Y exporte RT=65000:60. CLIENT_X importe RT=65000:60. → CLIENT_X **importe** les routes de CLIENT_Y. La relation est **asymétrique**.

b) **Asymétrique** : X reçoit les routes de Y, mais Y ne reçoit pas les routes de X.

c) Cas d'usage réel : **fournisseur/client interne** — par exemple, une VRF de serveurs (CLIENT_Y) qui doit être joignable depuis une VRF de postes de travail (CLIENT_X), mais les serveurs n'ont pas besoin de connaître les routes des postes de travail. Ou un modèle de sécurité où les hôtes peuvent initier des connexions vers les serveurs, mais pas l'inverse au niveau routage.

**CG.5** —

| Table | Équipement(s) | Contenu | Peuplée par |
|---|---|---|---|
| RIB (table de routage globale) | PE, P | Routes d'infrastructure (loopbacks, liens) | IGP (OSPF/IS-IS) |
| VRF routing table | PE uniquement | Routes des clients (apprises du CE + importées via MP-BGP) | Protocol PE-CE (eBGP, OSPF…) + MP-BGP |
| LFIB | PE, P | Labels entrants → labels sortants + action | LDP (ou RSVP-TE) |
| LIB | PE, P | Tous les labels reçus de tous voisins LDP (Liberal Retention) | LDP |
| BGP VPNv4 table | PE (+ RR) | Routes VPNv4 (RD+préfixe+RT+label VPN+next-hop) | MP-BGP iBGP entre PE |

**CG.6** — Les routeurs P ne maintiennent que des **labels de transport** dans leur LFIB — un label par LSP, indépendamment du nombre de clients et de VRF. Même si l'opérateur a 1 million de routes clients réparties dans 1000 VRF, le routeur P voit uniquement : "label 300 → SWAP 400 → iface eth0". Le mécanisme central est la **séparation PE/P** rendue possible par le double label : les informations clients sont encapsulées derrière le label VPN (invisible pour P) et seuls les PE en bordure déchiffrent cette information. C'est le principe fondateur de la RFC 4364.

**CG.7** — Paramètres obligatoirement cohérents :
1. **AS numbers** : même AS opérateur (iBGP)
2. **AFI/SAFI** : VPNv4 activée des deux côtés (`address-family vpnv4`, `neighbor activate`)
3. **`send-community extended`** : activé des deux côtés pour transporter les RT
4. **RT values** : les RT export/import des VRF doivent correspondre pour que les routes soient importées
5. **Loopback joignable** : chaque PE doit pouvoir résoudre le loopback de l'autre via IGP et LDP
6. **`update-source Loopback0`** : sur les deux PE pour stabilité de la session

**CG.8** — Étapes dans l'ordre :
1. Connexion physique PE3 ↔ CE_nouveau, lien PE3 ↔ P (infrastructure)
2. Configuration IGP (IS-IS/OSPF) sur les liens P-PE3 et loopback PE3 → vérifier `show isis neighbors` / `show ospf neighbor`
3. Activation LDP sur les liens → vérifier `show mpls ldp neighbor`, `show mpls forwarding-table`
4. Configuration MP-BGP : session iBGP PE3 ↔ RR (ou PE1/PE2), `address-family vpnv4`, `send-community extended` → vérifier `show bgp vpnv4 unicast all summary`
5. Création VRF BANK_A sur PE3 (RD, RT identiques aux autres PE)
6. Assignation interface CE-facing à VRF BANK_A, configuration IP
7. Configuration protocole PE-CE (eBGP, OSPF ou statique) → vérifier `show ip route vrf BANK_A`
8. Vérification routes MP-BGP reçues des autres PE : `show bgp vpnv4 unicast vrf BANK_A`
9. Test ping end-to-end CE_nouveau ↔ CE1 / CE2

**CG.9** — Méthodologie de diagnostic :

**Étape 1 — Isoler le problème :** Le site B fonctionne mais pas le site A → problème spécifique à la relation PE_C ↔ PE_A, pas un problème global.

**Étape 2 — Plan de contrôle (MP-BGP) sur PE_C :**
- `show bgp vpnv4 unicast vrf BANK_A` : PE_C reçoit-il les routes du site A (depuis PE_A) ?
- Si non → problème BGP/VRF entre PE_C et PE_A : vérifier session BGP, RT, `send-community extended`

**Étape 3 — Si routes présentes, plan de contrôle sur PE_A :**
- `show bgp vpnv4 unicast vrf BANK_A` sur PE_A : PE_A a-t-il les routes du site C ?
- `show ip route vrf BANK_A` sur PE_A : la route vers CE_C est-elle installée ?

**Étape 4 — Plan de données :**
- `show mpls forwarding-table` sur PE_C : le label VPN de PE_A est-il présent ?
- `traceroute mpls ipv4 <loopback PE_A>` depuis PE_C : le LSP vers PE_A est-il fonctionnel ?
- Vérifier LDP entre PE_C et PE_A (via les routeurs P intermédiaires)

**Étape 5 — Vérification côté CE :**
- `show ip route` sur CE du site A : la route vers le site C est-elle présente ?
- Vérifier la session PE-CE sur PE_A pour la VRF BANK_A

**CG.10** — Avantages de SR-MPLS vs LDP pour L3 VPN :
1. **Pas de sessions LDP par routeur** : SR encode le chemin dans la pile de labels via des Segment IDs (SID) configurés statiquement — aucune signalisation d'état distribuée, plus simple à opérer.
2. **Source routing** : le PE ingress peut encoder un chemin explicite sans tunnel RSVP-TE → ingénierie de trafic sans état dans le cœur.
3. **TI-LFA natif** : protection FRR avec couverture ~100% sans configuration complexe.
4. **Interopérabilité avec SDN** : les contrôleurs peuvent programmer les chemins via BGP-LS + SR Policy.

Composants L3 VPN **inchangés** : VRF, RD, RT, MP-BGP. Ces mécanismes opèrent au niveau du plan de contrôle (distribution des routes clients et des labels VPN) indépendamment du protocole de distribution des labels de transport. SR remplace LDP uniquement pour les **labels de transport** (outer labels). Le label VPN (inner label), distribué par MP-BGP, reste exactement identique.
