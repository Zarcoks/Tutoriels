# Tutoriel 2 : MPLS — Multiprotocol Label Switching

> **Prérequis** : Tutoriel 1 (IGP). Il est supposé acquis que vous comprenez le routage IP, le fonctionnement d'OSPF ou IS-IS, et ce qu'est un LSP (chemin logique dans un réseau). La notion de FIB (Forwarding Information Base) et de LPM (Longest Prefix Match) sont également supposées acquises.  
> **Ce tutoriel prépare à** : Tutoriel 3 (L3 VPN MPLS), dans lequel MPLS est le plan de données sur lequel les VPN sont construits.

---

## Table des matières

1. [Pourquoi MPLS ? Limites du routage IP pur](#1-pourquoi-mpls--limites-du-routage-ip-pur)
2. [Architecture MPLS : vue d'ensemble](#2-architecture-mpls--vue-densemble)
3. [Structure du label MPLS](#3-structure-du-label-mpls)
4. [La pile de labels (Label Stack)](#4-la-pile-de-labels-label-stack)
5. [Plan de contrôle : LDP](#5-plan-de-contrôle--ldp)
6. [Plan de données : LFIB et actions](#6-plan-de-données--lfib-et-actions)
7. [LSP : Label Switched Path](#7-lsp--label-switched-path)
8. [PHP : Penultimate Hop Popping](#8-php--penultimate-hop-popping)
9. [MPLS Traffic Engineering (RSVP-TE)](#9-mpls-traffic-engineering-rsvp-te)
10. [Segment Routing (SR-MPLS)](#10-segment-routing-sr-mpls)
11. [Convergence et Fast ReRoute (FRR)](#11-convergence-et-fast-reroute-frr)
12. [Exercices de compréhension générale](#12-exercices-de-compréhension-générale)

---

## 1. Pourquoi MPLS ? Limites du routage IP pur

### Théorie

Le routage IP classique fonctionne correctement mais présente des limitations qui ont motivé le développement de MPLS.

#### Limitation 1 : Le lookup IP est coûteux (historiquement)

À chaque saut, un routeur effectue un **LPM (Longest Prefix Match)** dans sa FIB. Dans les années 1990, avec des CPU de routeurs peu puissants et des tables de routage Internet en croissance exponentielle, ce lookup était une opération relativement coûteuse à effectuer à des débits élevés (millions de paquets/seconde).

MPLS remplace ce lookup LPM par un **exact match sur un label** — opération structurellement plus simple.

> **Note** : cet argument est aujourd'hui dépassé. Les ASICs modernes (TCAM) effectuent le LPM à la vitesse du câble. La performance n'est plus la raison principale de déployer MPLS.

#### Limitation 2 : Pas de séparation topologie/forwarding

En routage IP pur, le chemin d'un paquet est entièrement déterminé par sa destination IP et les décisions de chaque routeur intermédiaire. Il est impossible de forcer un flux sur un chemin spécifique différent du plus court chemin IGP — ce qu'on appelle **l'ingénierie de trafic (Traffic Engineering)**.

MPLS permet de définir des chemins explicites (LSP) indépendants du chemin IGP, ce qui est la base du **MPLS-TE**.

#### Limitation 3 : Pas d'isolation native entre clients

Sur un réseau IP partagé, tous les clients partagent la même table de routage. Il est impossible d'isoler nativement les flux de deux clients utilisant les mêmes adresses IP privées (espaces d'adressage overlapping).

MPLS, combiné avec des VRF et MP-BGP, permet cette isolation — c'est le **L3 VPN MPLS** (objet du tutoriel 3).

#### Limitation 4 : Qualité de service (QoS) grossière

En IP pur, la QoS repose sur le champ DSCP (6 bits) dans l'en-tête IP. MPLS ajoute un champ **TC (Traffic Class, 3 bits)** dans le label qui permet des politiques QoS par saut dans le cœur MPLS sans avoir à inspecter l'en-tête IP.

### Histoire

MPLS émerge à la fin des années 1990 de la convergence de plusieurs propositions propriétaires :
- **Tag Switching** (Cisco, 1996) : premier à proposer le concept de labels sur IP.
- **IP Switching** (Ipsilon, 1996) : idée similaire mais non retenu.
- **ARIS** (IBM, 1997) : autre approche propriétaire.

L'IETF crée le groupe de travail MPLS en 1997 et standardise l'architecture (RFC 3031, 2001) et l'encodage des labels (RFC 3032, 2001). Depuis, MPLS est devenu l'épine dorsale des réseaux opérateurs mondiaux — non plus principalement pour la performance, mais pour les services qu'il rend possibles : VPN, TE, QoS, convergence rapide.

### Exercices — Section 1

**Q1.1** — Citez les quatre limitations du routage IP pur que MPLS adresse. Pour chacune, indiquez si elle reste pertinente aujourd'hui ou si elle est devenue historique.

**Q1.2** — Pourquoi l'argument de performance (LPM vs exact match) n'est-il plus la raison principale de déployer MPLS aujourd'hui ? Qu'est-ce qui a changé depuis les années 1990 ?

**Q1.3** — Qu'est-ce que l'**ingénierie de trafic (Traffic Engineering)** ? Pourquoi le routage IP pur ne peut-il pas la réaliser nativement ?

**Q1.4** — Dans un réseau IP pur partagé entre deux entreprises utilisant toutes deux `10.0.0.0/8`, quel problème survient ? MPLS seul suffit-il à résoudre ce problème, ou faut-il des mécanismes supplémentaires ?

**Q1.5** — Vrai ou Faux : MPLS remplace entièrement IP dans le cœur d'un réseau opérateur. Justifiez.

**Q1.6** — Qu'est-ce que le champ DSCP dans l'en-tête IP ? Comment le champ TC du label MPLS le complète-t-il dans un réseau MPLS ?

**Q1.7** — Un opérateur décide de ne pas déployer MPLS sur son réseau. Il veut quand même fournir des services VPN à ses clients. Quelle alternative existe ? (Indice : IPsec/GRE.) Quels sont les inconvénients de cette alternative par rapport à MPLS ? (Note : IPsec et GRE ne sont pas détaillés dans ce tutoriel — nous les mentionnons comme alternatives existantes.)

**Q1.8** — Nommez les trois RFC fondatrices de MPLS et leur contenu.

**Q1.9** — MPLS opère à quelle couche du modèle OSI ? Justifiez.

**Q1.10** — Résumez en trois phrases pourquoi MPLS est encore déployé massivement aujourd'hui, malgré le fait que l'argument de performance initial soit obsolète.


### Corrections — Section 1

**C1.1** — (1) Performance LPM → **historique** (ASICs modernes). (2) Traffic Engineering impossible → **toujours pertinent** (MPLS-TE/SR). (3) Pas d'isolation multi-client → **toujours pertinent** (L3 VPN). (4) QoS grossière → **partiellement pertinent** (DSCP existe mais le TC MPLS est plus pratique dans le cœur).

**C1.2** — Les **ASICs (Application-Specific Integrated Circuits)** et les TCAM (Ternary Content-Addressable Memory) modernes effectuent le LPM à la vitesse du câble (line rate), même pour des tables de 1 million de routes. Dans les années 1990, les routeurs étaient logiciels (process switching) — chaque paquet était traité par le CPU, et le LPM sur des tables croissantes était un goulot d'étranglement réel.

**C1.3** — L'ingénierie de trafic est la capacité à forcer un flux sur un chemin réseau spécifique qui n'est pas nécessairement le plus court chemin IGP, pour optimiser l'utilisation de la bande passante disponible ou garantir une latence/qualité de service. IP pur ne peut pas : le chemin est entièrement déterminé hop-by-hop par chaque routeur en fonction de son IGP, sans vision globale ni moyen d'imposer un chemin.

**C1.4** — Deux entreprises avec `10.0.0.0/8` sur un réseau IP partagé : les paquets ne peuvent pas être routés correctement — les routeurs ne savent pas vers quel client forwarder. MPLS seul ne suffit pas — il faut des **VRF** (tables de routage isolées par client) et **MP-BGP** (pour distribuer les routes par client avec des identifiants distincts). C'est le sujet du tutoriel 3.

**C1.5** — **Faux**. MPLS ne remplace pas IP — IP reste présent dans le payload. MPLS ajoute une couche de labels **entre** L2 et L3. Dans le cœur MPLS, les routeurs P commutent sur les labels sans regarder l'IP, mais les LER en bordure font toujours du routage IP pour décider du chemin MPLS à utiliser.

**C1.6** — **DSCP** (Differentiated Services Code Point) : 6 bits dans l'en-tête IP, définit la classe de service du paquet. Un routeur IP doit inspecter l'en-tête IP pour lire le DSCP. Le **TC MPLS** : 3 bits dans le label, copié depuis le DSCP lors du PUSH. Les routeurs P du cœur MPLS peuvent appliquer la QoS en lisant uniquement le label (TC) sans inspecter l'IP interne — plus efficace.

**C1.7** — Alternative : **tunnels GRE ou IPsec** entre sites clients sur Internet. Inconvénients vs MPLS : (1) GRE/IPsec ajoutent un overhead d'en-tête significatif ; (2) Internet n'offre pas de garanties de débit/latence (best-effort) ; (3) IPsec ajoute une latence de chiffrement/déchiffrement ; (4) scalabilité limitée (N×(N-1)/2 tunnels pour un maillage complet). Note : IPsec et GRE sont des protocoles de tunnelisation non détaillés dans ce tutoriel.

**C1.8** — (1) RFC 3031 (2001) : architecture MPLS (LER/LSR, plans de contrôle/données, FEC). (2) RFC 3032 (2001) : encodage du label MPLS (32 bits, champs Label/TC/S/TTL, EtherType 0x8847). (3) RFC 5036 (2007, mise à jour de RFC 3036) : LDP — protocole de distribution de labels.

**C1.9** — MPLS opère à la **couche 2.5** (entre L2 et L3). Justification : les labels sont insérés après l'en-tête L2 (Ethernet) et avant l'en-tête L3 (IP). MPLS n'est pas une couche OSI officielle — "2.5" est une métaphore technique. Sur le fil, la trame Ethernet a l'EtherType 0x8847 (au lieu de 0x0800 pour IP), suivi des labels MPLS 32 bits, puis de l'en-tête IP.

**C1.10** — MPLS reste déployé car : (1) il permet des **services VPN** (L3 VPN, L2 VPN) scalables sur infrastructure mutualisée ; (2) il permet l'**ingénierie de trafic** (chemins explicites, réservation de bande passante) ; (3) il offre une **convergence rapide** (<50ms avec FRR) impossible en IP pur. Ces trois services ont une valeur commerciale considérable pour les opérateurs.
---

## 2. Architecture MPLS : vue d'ensemble

### Théorie

Un réseau MPLS comprend deux types d'équipements :

```
CE ── LER ──── LSR ──── LSR ──── LER ── CE
     (Edge)   (Core)   (Core)   (Edge)

LER : Label Edge Router  → routeur de bordure MPLS
      - impose les labels (PUSH) sur les paquets entrants
      - retire les labels (POP) sur les paquets sortants
      - fait partie à la fois du monde IP et du monde MPLS

LSR : Label Switch Router → routeur cœur MPLS
      - effectue uniquement des opérations sur les labels (SWAP)
      - ne fait jamais de lookup IP pour le transit
      - peut ignorer complètement le contenu du paquet
```

Dans le contexte du L3 VPN (tutoriel 3), les LER correspondent aux **PE (Provider Edge)** et les LSR aux **P (Provider core)**.

#### Plan de contrôle vs Plan de données

MPLS sépare clairement deux plans :

**Plan de contrôle** : construit et maintient les tables de forwarding (LFIB). Peuplé par :
- L'IGP (IS-IS ou OSPF) → donne la topologie et les routes
- LDP (Label Distribution Protocol) → distribue les labels entre routeurs
- Optionnellement : RSVP-TE (Traffic Engineering), BGP (pour les VPN)

**Plan de données** : forward les paquets en utilisant les labels. Opère sur la LFIB. Actions : PUSH, SWAP, POP.

```
Plan de contrôle :
  IGP → connaissance de la topologie
  LDP → distribution des labels
  → LFIB construite

Plan de données :
  Paquet arrive → lookup label dans LFIB → action (SWAP/POP) → forward
```

#### EtherType MPLS

Sur un lien Ethernet, un paquet MPLS est encapsulé dans une trame avec **EtherType `0x8847`** (MPLS unicast) ou `0x8848` (MPLS multicast). Sans cette valeur dans l'en-tête Ethernet, l'équipement récepteur ne saurait pas qu'un label MPLS suit.

```
Trame Ethernet avec MPLS :
┌─────────────────┬─────────────┬──────────────┬───────────┐
│ En-tête Eth     │ 0x8847      │ Label(s)MPLS │ Payload   │
│ (dst/src MAC)   │ (EtherType) │ (32 bits ×N) │ (IP, etc) │
└─────────────────┴─────────────┴──────────────┴───────────┘
```

### Histoire

L'architecture LER/LSR est définie dans la RFC 3031 (MPLS Architecture, 2001). La séparation plan de contrôle / plan de données était déjà un principe de conception dans les réseaux ATM (Asynchronous Transfer Mode) des années 1990 — MPLS en hérite et l'adapte pour IP. ATM utilisait des cellules de taille fixe (53 octets) avec un identifiant de circuit virtuel (VCI/VPI) analogue à un label MPLS, mais ATM était complexe et coûteux ; MPLS a réussi à apporter la commutation par étiquette sur les infrastructures IP/Ethernet existantes.

### Exercices — Section 2

**Q2.1** — Quelle est la différence entre un LER et un LSR ? Lequel effectue un lookup IP ? Lequel effectue uniquement un lookup MPLS ?

**Q2.2** — Dans un réseau MPLS, un paquet IP entre par un LER et sort par un autre LER. Décrivez ce que fait chaque type d'équipement (LER ingress, LSR de transit, LER egress) avec le paquet.

**Q2.3** — Pourquoi dit-on que MPLS sépare le plan de contrôle du plan de données ? Quels protocoles participent à chaque plan ?

**Q2.4** — Un paquet arrive sur un LSR de cœur. Il ne contient pas d'en-tête IP visible (le paquet MPLS masque l'IP interne). Le LSR peut-il forwarder ce paquet ? Comment ?

**Q2.5** — Un routeur du cœur MPLS tombe en panne. Les routeurs voisins sont des LSR. Quel est l'impact immédiat sur le trafic MPLS de transit ? Quel protocole déclenche la reconvergence ?

**Q2.6** — Quelle valeur d'EtherType indique la présence de labels MPLS dans une trame Ethernet ? Comment un équipement de capture (Wireshark) identifie-t-il un paquet MPLS ?

**Q2.7** — Vrai ou Faux : un LSR doit maintenir une table de routage IP complète pour pouvoir forwarder des paquets MPLS. Justifiez.

**Q2.8** — Comparez l'architecture MPLS (LER/LSR) avec l'architecture ATM (cellules, VCI/VPI). Quelle est la différence fondamentale dans la gestion des données ?

**Q2.9** — Dans un réseau MPLS opérateur, les équipements P (cœur) et PE (bordure) correspondent à quels équipements MPLS génériques (LER/LSR) ?

**Q2.10** — Pourquoi les LSR du cœur MPLS peuvent-ils être plus simples et plus rapides que les LER de bordure ?


### Corrections — Section 2

**C2.1** — **LER** : effectue des lookups IP (pour décider du LSP à utiliser) ET des lookups MPLS (PUSH/POP). **LSR** : effectue uniquement des lookups MPLS (SWAP) — jamais de lookup IP pour le trafic de transit.

**C2.2** — **LER ingress** : reçoit un paquet IP, fait un lookup IP (ou VRF) → identifie le LSP → PUSH le(s) label(s) → envoie dans le cœur MPLS. **LSR de transit** : reçoit paquet MPLS, lookup label entrant dans LFIB → SWAP le label → forward vers le prochain saut. **LER egress** : reçoit paquet MPLS (ou IP si PHP), POP le(s) label(s) restant(s) → lookup IP (ou VRF) → forward vers la destination finale.

**C2.3** — MPLS sépare : **plan de contrôle** = protocoles qui calculent et distribuent les informations de routage/labels (IGP, LDP, RSVP-TE, MP-BGP) → alimente les tables (RIB, LIB, LFIB). **Plan de données** = forwarding effectif des paquets basé sur les tables (LFIB) → actions PUSH/SWAP/POP.

**C2.4** — **Oui**. Le LSR ne regarde que le label outer. Même s'il ne "voit" pas l'IP interne, il peut parfaitement forwarder le paquet via la LFIB : label entrant → SWAP → label sortant + interface. Le contenu du paquet (IP, autre label, Ethernet…) est complètement ignoré.

**C2.5** — Impact immédiat : les paquets MPLS dont le LSP passait par ce LSR sont perdus jusqu'à reconvergence. Protocole déclenchant la reconvergence : l'**IGP** (IS-IS ou OSPF) détecte la panne via les timers Hello/Dead ou BFD → recalcule SPF → nouvelles routes → **LDP** reconverge → nouveau LSP établi.

**C2.6** — **EtherType `0x8847`** (MPLS unicast). Wireshark identifie la présence de labels MPLS en lisant ce champ EtherType dans l'en-tête Ethernet. Il affiche ensuite le décodage des labels (Label, TC, S, TTL) avant le payload.

**C2.7** — **Faux** (nuancé). Un LSR de **transit pur** n'a pas besoin d'une table de routage IP complète pour forwarder des paquets MPLS de transit — il utilise uniquement sa LFIB. Cependant, en pratique, les LSR maintiennent une table IP (peuplée par l'IGP) car elle est nécessaire pour construire la LFIB (via LDP) et pour le routage de leur propre trafic de contrôle (LDP, OSPF Hello, etc.).

**C2.8** — ATM : cellules de **taille fixe** (53 octets), identifiants VCI/VPI analogues aux labels. MPLS : **paquets de taille variable** (comme IP), labels de 32 bits. Différence fondamentale : ATM découpe les paquets en cellules (fragmentation) — coûteux en overhead pour les petits paquets. MPLS travaille sur les paquets IP entiers sans fragmentation.

**C2.9** — Les **P (Provider core)** correspondent aux **LSR** — ils commutent uniquement des labels. Les **PE (Provider Edge)** correspondent aux **LER** — ils font le PUSH/POP des labels et le routage IP/VRF en bordure.

**C2.10** — Les LSR sont plus simples car ils n'effectuent qu'un **exact match LFIB** (pas de LPM IP) et n'ont pas à maintenir de VRF ou de tables BGP. Leurs ASICs peuvent donc être optimisés pour une seule opération répétitive à très haut débit (dizaines ou centaines de Tbps). Les LER doivent gérer LDP, MP-BGP, VRF, QoS, et des lookups plus complexes — ils sont intrinsèquement plus sophistiqués.
---

## 3. Structure du label MPLS

### Théorie

Un label MPLS est un champ de **32 bits** inséré entre l'en-tête L2 et l'en-tête L3 :

```
 31          12  11   9   8        0
 ┌───────────────┬───┬─┬──────────┐
 │  Label (20b)  │TC │S│  TTL(8b) │
 └───────────────┴───┴─┴──────────┘
```

#### Champ Label (20 bits)

- Plage : **0 à 1 048 575** (2²⁰ - 1)
- C'est l'identifiant du "tunnel" ou du "circuit" sur lequel ce paquet doit être forwardé.
- Les valeurs **0 à 15** sont réservées :

| Valeur | Nom | Signification |
|---|---|---|
| 0 | IPv4 Explicit NULL | Le routeur suivant doit faire un lookup IP (pas de PHP implicite) |
| 1 | Router Alert Label | Traitement spécial par le plan de contrôle |
| 2 | IPv6 Explicit NULL | Equivalent du label 0 pour IPv6 |
| 3 | Implicit NULL | Signifie "retire ce label avant de m'envoyer le paquet" (PHP) |
| 4–15 | Réservés | Usage futur |

#### Champ TC — Traffic Class (3 bits)

- 8 valeurs possibles (0 à 7).
- Anciennement appelé **EXP** (Experimental) dans les premières RFC.
- Utilisé pour la **QoS** : les routeurs du cœur MPLS peuvent appliquer des politiques de queuing/scheduling basées sur ce champ sans avoir à inspecter l'en-tête IP.
- Mapping typique : copié depuis les bits DSCP du paquet IP entrant (lors du PUSH par le LER ingress).

#### Champ S — Bottom of Stack (1 bit)

- **S=1** : ce label est le **dernier** de la pile. L'en-tête suivant est le payload (IP, Ethernet, etc.).
- **S=0** : il y a d'autres labels en dessous dans la pile.
- Ce bit est essentiel pour les routeurs qui n'ont besoin de regarder que le premier label : ils savent s'arrêter quand ils voient S=1 (ou traiter les labels suivants s'ils font de la récursion).

#### Champ TTL — Time To Live (8 bits)

- Fonctionne comme le TTL IP : décrémenté à chaque saut, paquet jeté si TTL=0.
- À l'entrée dans le domaine MPLS (PUSH), le LER copie le TTL IP dans le label TTL.
- À la sortie du domaine MPLS (POP), le LER copie le TTL du label dans le TTL IP.
- **`no mpls ip propagate-ttl`** : désactive cette copie → le TTL IP n'est pas décrémenté dans le cœur MPLS → les routeurs P sont invisibles aux traceroutes des clients (masquage de topologie).

### Histoire

L'encodage du label MPLS en 32 bits est défini dans la RFC 3032 (2001). Le champ EXP (maintenant TC) a été controversé : il est défini comme "experimental" dans RFC 3032 mais largement utilisé pour la QoS en production. La RFC 5462 (2009) le renomme officiellement en TC (Traffic Class) et précise son usage QoS. L'Implicit NULL label (valeur 3) est une élégance du design : au lieu d'envoyer un label et de forcer le routeur suivant à regarder "sous" ce label pour décider de le retirer, on annonce directement la valeur 3 pour indiquer "retire mon label avant de m'envoyer le paquet" — c'est la base du PHP.

### Exercices — Section 3

**Q3.1** — Combien de bits composent un label MPLS ? Identifiez les quatre champs et leur taille respective.

**Q3.2** — Décomposez le label MPLS encodé en hexadécimal `0x000C81FF` en ses quatre champs (Label, TC, S, TTL).

**Q3.3** — Quelle est la différence entre le label **Implicit NULL (3)** et le label **Explicit NULL (0)** ? Dans quel cas utilise-t-on chacun ?

**Q3.4** — Un LER ingress reçoit un paquet IP avec DSCP=46 (Expedited Forwarding). Il pousse un label MPLS. Comment remplit-il le champ TC du label ? Quelle est la valeur TC typique pour EF ?

**Q3.5** — Vrai ou Faux : le TTL MPLS est toujours décrémenté à chaque saut LSR, indépendamment de la configuration. Justifiez et expliquez l'impact de `no mpls ip propagate-ttl`.

**Q3.6** — Combien de valeurs de label sont disponibles pour un usage normal (hors labels réservés) ? Comment cela compare-t-il au nombre de routes IPv4 possibles ?

**Q3.7** — Quel champ du label indique qu'il est le dernier de la pile ? Quelle est sa valeur dans ce cas ?

**Q3.8** — Un routeur de cœur (LSR) reçoit un paquet avec deux labels empilés. Comment sait-il qu'il ne doit lire que le premier label et pas le second ? Quel bit lui indique cela ?

**Q3.9** — Un traceroute depuis un CE client traverse un réseau MPLS et ne montre pas les routeurs P du cœur (ils apparaissent comme `* * *`). Quelle fonctionnalité MPLS est activée et comment fonctionne-t-elle ?

**Q3.10** — Le champ TC est sur 3 bits, ce qui donne 8 classes de service. Un opérateur veut distinguer : voix, vidéo, données critiques, données best-effort, et trafic de contrôle réseau. Est-ce compatible avec 8 classes ? Donnez un exemple de mapping.


### Corrections — Section 3

**C3.1** — 32 bits. Champs : **Label** (20 bits, bits 31-12), **TC** (3 bits, bits 11-9), **S** (1 bit, bit 8), **TTL** (8 bits, bits 7-0).

**C3.2** — `0x000C81FF` = `0000 0000 0000 1100 1000 0001 1111 1111`
- Label : `0000 0000 0000 1100 1000` = 0x00C8 = **200**
- TC : `000` = **0**
- S : `1` = **1** (bottom of stack)
- TTL : `1111 1111` = **255**

**C3.3** — **Implicit NULL (3)** : annoncé par le LER egress à son voisin P. Signifie "retire mon label avant de me forwarder le paquet" → c'est le mécanisme PHP. Le P retire le label et forward sans label. **Explicit NULL (0)** : annoncé par le LER egress. Le P forward le paquet **avec** ce label (valeur 0). Le LER egress reçoit le paquet avec le label 0, effectue une décision de forwarding IP, puis retire le label. Utilisé pour préserver le TC jusqu'au dernier saut.

**C3.4** — Lors du PUSH, le LER ingress peut **copier les bits DSCP vers le champ TC** (mapping DSCP → TC). DSCP=46 (EF — Expedited Forwarding) → TC=**5** ou **6** selon le mapping opérateur (valeur conventionnelle pour EF en MPLS DiffServ). Le mapping exact dépend de la politique QoS de l'opérateur.

**C3.5** — **Faux** (nuancé). Par défaut sur Cisco, le TTL MPLS est **copié depuis le TTL IP** lors du PUSH (propagation TTL activée) et décrémenté à chaque saut. Avec `no mpls ip propagate-ttl` : le TTL IP n'est pas copié dans le label TTL → le TTL MPLS est initialisé à 255 indépendamment du TTL IP → les routeurs P sont invisibles aux traceroutes (le TTL IP n'est décrémenté que par le LER ingress et le LER egress, pas par les P de transit).

**C3.6** — Labels disponibles hors réservés (16 à 1 048 575) = **1 048 560** labels. Comparaison IPv4 : 2³² = environ 4 milliards d'adresses IPv4 possibles. Les labels sont locaux à chaque lien — un même label peut être réutilisé sur des liens différents pour des FEC différents. En pratique, quelques milliers de labels actifs suffisent pour un réseau opérateur.

**C3.7** — Le champ **S (Bottom of Stack)** indique le dernier label de la pile. Sa valeur est **1** quand c'est le dernier label.

**C3.8** — Le LSR lit le premier label (outer). Le bit **S=0** sur ce label outer indique qu'il y a d'autres labels en dessous — le LSR effectue son opération (SWAP) sur ce seul label et forward. Il ne touche jamais au label inner car S=0 lui indique de ne traiter que le premier label de la pile.

**C3.9** — **`no mpls ip propagate-ttl`** est activé. Cette commande désactive la copie du TTL IP dans le label MPLS lors du PUSH. Le TTL MPLS est initialisé à 255 et décrémenté dans le cœur MPLS sans lien avec le TTL IP. Les routeurs P sont donc invisibles dans un traceroute car le TTL IP n'est jamais décrémenté par eux (seulement par PE1 et PE2).

**C3.10** — 3 bits = 8 classes (0 à 7). Mapping possible :
- 0 : Best-effort (données ordinaires)
- 1 : Données basse priorité
- 2 : Données normales
- 3 : Données critiques (ERP, signalisation)
- 4 : Vidéo
- 5 : Voix (VoIP, correspondant à DSCP EF)
- 6 : Contrôle réseau (OSPF, LDP, BGP)
- 7 : Réservé ou gestion réseau (SNMP, SSH)
Compatible avec 5 types de trafic + 3 classes supplémentaires.
---

## 4. La pile de labels (Label Stack)

### Théorie

MPLS permet d'empiler **plusieurs labels** sur un même paquet. Cette pile de labels (label stack) est fondamentale pour les services avancés comme le L3 VPN.

```
┌──────────────────────────────────┐
│ En-tête L2 (EtherType 0x8847)   │
├──────────────────────────────────┤
│ Label 1 (outer / premier lu)     │  S=0  ← label de transport
├──────────────────────────────────┤
│ Label 2                          │  S=0  ← label intermédiaire éventuel
├──────────────────────────────────┤
│ Label N (inner / dernier label)  │  S=1  ← label VPN (en L3 VPN)
├──────────────────────────────────┤
│ Payload (IP, Ethernet, etc.)     │
└──────────────────────────────────┘
```

#### Règles de la pile

- Les labels sont **lus de haut en bas** : le premier label (outer) est celui qu'un LSR lit pour prendre sa décision de forwarding.
- Un LSR de transit ne lit que le **label du dessus** (outer) — il ne regarde jamais ce qui est en dessous.
- Le bit **S=1** marque le bas de la pile (Bottom of Stack).
- Il n'y a **pas de limite théorique** au nombre de labels empilés (limite pratique imposée par les équipements).

#### Utilisation de la pile en L3 VPN

En L3 VPN MPLS (détaillé dans le tutoriel 3), deux labels sont typiquement utilisés :
- **Label outer (transport)** : distribué par LDP. Achemine le paquet du PE ingress au PE egress à travers les routeurs P.
- **Label inner (VPN)** : distribué par MP-BGP. Identifie la VRF (et donc le client) sur le PE egress.

```
[Label transport, S=0][Label VPN, S=1][IP client]
       ↑                      ↑
   Lu par les P         Lu par le PE egress
```

#### Utilisation en MPLS-TE avec VPN

Dans certaines topologies avancées (MPLS-TE + L3 VPN), trois labels peuvent être empilés :
- Label RSVP-TE (chemin explicite)
- Label LDP (transport)
- Label VPN

Ceci n'est pas détaillé dans ce tutoriel mais il est important de savoir que la pile peut avoir plus de deux niveaux.

### Histoire

Le concept de pile de labels est une des innovations majeures de MPLS par rapport à ATM (qui n'avait qu'un seul identifiant de circuit par cellule). Cette pile permet de combiner plusieurs services (TE + VPN, ou hiérarchie de tunnels) sans modifier le payload. Elle est définie dans RFC 3032 (2001) et a directement rendu possible les architectures L3 VPN (RFC 4364, 2006) qui nécessitent deux niveaux de labels simultanément.

### Exercices — Section 4

**Q4.1** — Un paquet a la pile de labels suivante :
```
[Label=100, S=0, TTL=60][Label=25, S=1, TTL=60][IP src=10.1.0.1, dst=10.2.0.1]
```
a) Combien de labels sont empilés ?
b) Lequel est l'outer label ? Lequel est l'inner label ?
c) Que signifie S=1 sur le label 25 ?
d) Un LSR de transit lit quel label ?

**Q4.2** — En L3 VPN MPLS, expliquez le rôle du label outer (transport) et du label inner (VPN). Qui génère chacun ?

**Q4.3** — Comment un routeur LSR sait-il qu'il n'y a qu'un seul label dans la pile (et donc que le payload suit directement) ?

**Q4.4** — Vrai ou Faux : un LSR de transit peut modifier le label inner (VPN) lors d'un SWAP. Justifiez.

**Q4.5** — Donnez un exemple de topologie où une pile de 3 labels pourrait être utilisée.

**Q4.6** — Lorsqu'un LER ingress effectue une opération PUSH sur un paquet entrant, dans quel ordre les labels sont-ils empilés sur le paquet IP ? (Quel label est poussé en premier ?)

**Q4.7** — Un paquet arrive sur un LSR avec le label outer=3 (Implicit NULL). Que fait le LSR ?

**Q4.8** — Après une opération PHP par le routeur P pénultième, quel est l'état de la pile reçue par le PE egress en contexte L3 VPN (avec double label) ?

**Q4.9** — Un opérateur veut empêcher ses routeurs P de voir le contenu IP des clients. La pile de labels MPLS suffit-elle à garantir cela ? Expliquez pourquoi les P ne lisent jamais l'IP interne.

**Q4.10** — Comparez la pile de labels MPLS à l'en-tête VLAN 802.1Q (QinQ) en termes de concept. Quelle est la différence fondamentale dans la portée de l'identifiant (VLAN ID vs label MPLS) ?


### Corrections — Section 4

**C4.1** — a) **2** labels. b) Label=100 est l'**outer** (premier lu, S=0). Label=25 est l'**inner** (S=1). c) S=1 sur label 25 signifie que c'est le **dernier label de la pile** — le payload IP suit directement. d) Un LSR de transit lit uniquement le **label outer** (label=100, S=0).

**C4.2** — **Label outer (transport)** : distribué par **LDP**. Achemine le paquet dans le cœur MPLS du PE ingress au PE egress. Tous les routeurs P le lisent et l'échangent (SWAP). **Label inner (VPN)** : distribué par **MP-BGP**. Identifie la VRF cliente sur le PE egress. Les routeurs P ne le voient jamais (ils s'arrêtent à l'outer label grâce à S=0).

**C4.3** — Si S=1 sur le seul label présent dans la pile, le routeur sait que le payload suit directement. Il n'y a pas d'autre label à lire. Après POP de ce label, le routeur traite le payload (IP ou autre).

**C4.4** — **Faux**. Un LSR de transit ne modifie que le **label outer** (SWAP). Il ne touche jamais au label inner car il s'arrête au premier label (S=0 lui indique qu'il y a des labels en dessous, mais il ne doit pas les lire). Le label VPN inner reste inchangé tout au long du transit dans le cœur.

**C4.5** — Exemple : **MPLS-TE + L3 VPN**. Pile de 3 labels : [Label RSVP-TE (chemin explicite), S=0][Label LDP (transport), S=0][Label VPN (inner), S=1][IP client]. Le routeur du cœur TE swap le label RSVP-TE, un routeur P ordinaire swap le label LDP, et le PE egress utilise le label VPN.

**C4.6** — En contexte L3 VPN : le LER ingress pousse d'abord le **label VPN** (inner), puis par-dessus le **label transport** (outer). Sur le fil, le label transport est lu en premier (outer, S=0), puis le label VPN (inner, S=1). L'ordre de PUSH est donc : VPN en premier (posé sur l'IP), transport par-dessus.

**C4.7** — Le label 3 est l'**Implicit NULL** — signal PHP. Le LSR reçoit ce label 3 dans sa LFIB et effectue une opération **POP** (retire le label 3) avant de forwarder le paquet au saut suivant. Le paquet arrive au PE egress sans ce label.

**C4.8** — En L3 VPN après PHP : PE egress reçoit `[VPN=25, S=1][IP src=... dst=...]`. Le label transport (outer) a été retiré par le P pénultième. Il reste uniquement le label VPN (inner, S=1) et le payload IP.

**C4.9** — **Oui**. Les routeurs P ne lisent que le label outer (S=0). Le bit S=0 leur indique qu'il y a d'autres données en dessous mais ils ne les lisent pas — ils effectuent uniquement SWAP sur le label outer et forward. Le contenu IP client est structurellement inaccessible pour eux lors du traitement normal du paquet.

**C4.10** — VLAN 802.1Q (QinQ) : identifiant de 12 bits (VLAN ID), portée locale au domaine Ethernet (L2). MPLS labels : 20 bits, portée L2.5, peuvent traverser plusieurs domaines L2. Concept similaire (identifiant de circuit dans la trame), mais MPLS est plus flexible (pile de longueur variable, TTL, TC), opère indépendamment du L2 sous-jacent (Ethernet, PPP, etc.), et a une portée réseau (L3) et non segment (L2).
---

## 5. Plan de contrôle : LDP

### Théorie

**LDP** (Label Distribution Protocol, RFC 5036) est le protocole standard utilisé pour distribuer les labels entre routeurs MPLS voisins. Son rôle est de construire la **LIB** (Label Information Base) et d'alimenter la **LFIB** (Label Forwarding Information Base).

#### Fonctionnement de LDP

**Phase 1 — Découverte des voisins**

LDP utilise des messages **Hello** envoyés en **UDP multicast** vers `224.0.0.2` (tous les routeurs du segment), port 646. Ce mécanisme est appelé "Basic Discovery". Chaque routeur qui reçoit un Hello répond, et les deux parties savent qu'un voisin LDP est présent.

Il existe aussi la "Extended Discovery" : des sessions LDP peuvent être établies entre routeurs non directement adjacents (utile pour les multihop LDP sessions, rares en pratique).

**Phase 2 — Établissement de la session**

Une session **TCP** (port 646) est établie entre les deux routeurs. TCP garantit la fiabilité et l'ordre de livraison des messages LDP. La connexion TCP est initiée par le routeur ayant la **plus grande adresse IP** de son LDP Router-ID.

**Phase 3 — Échange de labels (Label Mapping)**

Chaque routeur annonce un label pour chacune de ses **FEC** (Forwarding Equivalence Classes). Dans le cas le plus courant, un FEC = un préfixe IP de la table de routage.

```
Exemple sur un lien PE1 — P :

PE1 → P : "FEC=10.0.0.1/32 (mon loopback), Label=100"
         → P peut m'atteindre en poussant le label 100

P → PE1 : "FEC=10.0.0.99/32 (mon loopback), Label=200"
         → PE1 peut m'atteindre en poussant le label 200

P → PE2 : "FEC=10.0.0.1/32 (loopback de PE1), Label=300"
         → PE2 peut atteindre PE1 via P en poussant le label 300
```

**Mode de distribution : Downstream Unsolicited**

Par défaut, LDP distribue des labels **sans qu'on le lui demande** (Downstream Unsolicited) : dès qu'un routeur a une route dans sa FIB, il annonce le label correspondant à tous ses voisins LDP. C'est le mode par défaut et le plus courant.

**Liberal Label Retention vs Conservative**

- **Liberal** (défaut IOS) : le routeur conserve les labels reçus de **tous** ses voisins LDP, même ceux qui ne sont pas le next-hop actuel selon l'IGP. Avantage : convergence ultra-rapide car les labels du chemin alternatif sont déjà connus. Inconvénient : plus de mémoire utilisée.
- **Conservative** : le routeur ne conserve que les labels du next-hop actif selon l'IGP. Moins de mémoire, mais convergence plus lente (doit solliciter les labels du nouveau next-hop après un changement IGP).

#### LDP Router-ID

Chaque routeur LDP s'identifie par un **LDP Router-ID** (adresse IP 32 bits). Par défaut, c'est la plus haute adresse IP d'une interface Loopback active. Il doit être stable et joignable.

```cisco
mpls ldp router-id Loopback0 force
! "force" applique le changement immédiatement
! sans "force", le changement n'est effectif qu'au prochain redémarrage LDP
```

#### LDP IGP Sync

Lors de la convergence après une panne, il peut y avoir une fenêtre temporelle où l'IGP a convergé (un nouveau chemin est installé dans la FIB) mais LDP n'a pas encore distribué les labels sur ce nouveau chemin. Pendant cette fenêtre, les paquets MPLS peuvent être perdus.

**LDP IGP Sync** retarde l'utilisation d'un lien par l'IGP jusqu'à ce que la session LDP sur ce lien soit opérationnelle et que les labels aient été échangés.

```cisco
! Sur IOS XR avec IS-IS :
router isis CORE
 interface GigabitEthernet0/0/0/0
  mpls ldp sync
```

#### Configuration LDP minimale

```cisco
! Activer MPLS et LDP globalement
mpls ip

! Configurer le Router-ID LDP (loopback stable)
mpls ldp router-id Loopback0 force

! Activer LDP sur les interfaces (ou via autoconfig)
interface GigabitEthernet0/0
 mpls ip

! Alternative : activer automatiquement sur toutes les interfaces IGP
mpls ldp autoconfig
```

### Histoire

Avant LDP, Cisco utilisait **TDP** (Tag Distribution Protocol), propriétaire et non interopérable. L'IETF a standardisé LDP dans la RFC 3036 (2001), mise à jour par RFC 5036 (2007). Une alternative à LDP pour la distribution de labels est **RSVP-TE** (Resource Reservation Protocol - Traffic Engineering, RFC 3209), utilisé spécifiquement pour le Traffic Engineering car il permet de réserver de la bande passante sur des chemins explicites — RSVP-TE n'est pas un remplacement général de LDP, ils coexistent souvent. Dans les architectures **Segment Routing** (modernes), LDP est progressivement remplacé car SR n'a pas besoin de sessions de distribution d'état par routeur.

### Exercices — Section 5

**Q5.1** — Décrivez les trois phases d'établissement d'une session LDP entre deux routeurs voisins. Quels protocoles de transport (UDP/TCP) sont utilisés à chaque phase et pourquoi ?

**Q5.2** — Topologie `PE1 (loopback 10.0.0.1) — P (loopback 10.0.0.99) — PE2 (loopback 10.0.0.2)`. LDP est actif partout. Listez tous les messages LDP Label Mapping échangés pour que PE1 puisse joindre PE2 via un LSP complet.

**Q5.3** — Quelle est la différence entre la **LIB** et la **LFIB** ? Laquelle est utilisée pour le forwarding ? Laquelle contient les labels de tous les voisins (y compris non-next-hop) ?

**Q5.4** — Expliquez le **Liberal Label Retention**. Pourquoi est-il préférable en production malgré sa consommation mémoire plus élevée ?

**Q5.5** — Qu'est-ce qu'un **FEC** dans le contexte LDP ? Donnez trois exemples de FEC différents.

**Q5.6** — Pourquoi le LDP Router-ID doit-il être une adresse de loopback et non une adresse d'interface physique ? Que se passe-t-il si le lien physique portant l'adresse du Router-ID tombe ?

**Q5.7** — La commande `show mpls ldp neighbor` ne montre aucun voisin sur un routeur, alors que les interfaces physiques sont up et l'IGP fonctionne. Citez quatre causes possibles.

**Q5.8** — Qu'est-ce que **LDP IGP Sync** ? Quel problème de convergence résout-il ? Donnez un exemple de scénario où son absence cause une perte de trafic.

**Q5.9** — Vrai ou Faux : LDP distribue des labels pour **toutes** les routes de la table IP, y compris les routes clients dans les VRF. Justifiez.

**Q5.10** — Pourquoi LDP est-il progressivement remplacé par Segment Routing dans les nouveaux déploiements ? Quelle est la différence fondamentale de philosophie entre les deux approches ?


### Corrections — Section 5

**C5.1** — Phase 1 : **découverte** via Hello UDP multicast `224.0.0.2`, port 646. UDP est utilisé car il n'y a pas de connexion préalable — le routeur "crie" sa présence sur le segment. Phase 2 : **session TCP** port 646 entre les deux routeurs (initié par le routeur avec le plus grand LDP Router-ID). TCP garantit fiabilité et ordre des messages LDP. Phase 3 : **échange de labels** (Label Mapping messages) via la session TCP — chaque routeur annonce ses labels pour ses FEC.

**C5.2** — Messages LDP Label Mapping :
- PE2 → P : "FEC=10.0.0.2/32, Label=L1" (pour me joindre directement)
- P → PE1 : "FEC=10.0.0.2/32, Label=L2" (pour joindre PE2 via moi, utilise L2)
- PE1 → P : "FEC=10.0.0.1/32, Label=L3" (pour me joindre)
- P → PE2 : "FEC=10.0.0.1/32, Label=L4" (pour joindre PE1 via moi, utilise L4)
- Avec PHP : PE2 annonce à P l'Implicit NULL (label 3) pour son loopback → P effectue PHP.

**C5.3** — **LIB** : contient tous les labels reçus de **tous** les voisins LDP pour toutes les FEC (Liberal Retention). Sert de base de données complète, inclut les labels des voisins non-next-hop. **LFIB** : contient uniquement les labels **actifs** (next-hop actuel selon l'IGP) — utilisée pour le forwarding. La LFIB est dérivée de la LIB en filtrant selon la FIB IGP.

**C5.4** — Liberal Retention : le routeur conserve les labels de **tous** ses voisins, même ceux qui ne sont pas le next-hop actuel. Avantage convergence : lors d'un changement IGP (nouveau next-hop), le label du nouveau next-hop est déjà dans la LIB → la LFIB est mise à jour quasi-instantanément sans attendre un nouvel échange LDP. En Conservative Retention, il faudrait d'abord solliciter le label du nouveau next-hop → délai supplémentaire.

**C5.5** — Un **FEC** est un ensemble de paquets traités de manière identique par le forwarding MPLS. Exemples : (1) `10.0.0.2/32` (loopback d'un PE) ; (2) `192.168.1.0/24` (réseau d'un client dans une VRF) ; (3) une combinaison adresse IP + port + protocole (FEC fin, moins courant avec LDP standard).

**C5.6** — Le LDP Router-ID doit être un loopback car : les loopbacks sont **toujours up** tant que le routeur est en vie, même si les liens physiques tombent. Si le Router-ID est une adresse d'interface physique, la session LDP se ferme quand ce lien tombe — même si d'autres chemins existent vers ce voisin. Avec un loopback comme Router-ID, la session LDP reste up tant que le routeur est joignable via n'importe quel chemin.

**C5.7** — Causes possibles : (1) `mpls ip` non activé sur l'interface (`no mpls ip` par défaut) ; (2) LDP Router-ID non joignable (loopback non annoncé dans l'IGP) ; (3) ACL bloquant UDP port 646 (Hello) ou TCP port 646 (session) ; (4) `mpls ldp router-id` pointant vers une interface down ou non configurée.

**C5.8** — LDP IGP Sync retarde l'utilisation d'un lien par l'IGP jusqu'à ce que la session LDP sur ce lien soit opérationnelle. Scénario sans LDP IGP Sync : un lien revient après une panne → l'IGP le réintègre immédiatement dans le chemin optimal → le trafic MPLS est routé via ce lien → mais LDP n'a pas encore distribué les labels sur ce lien → les paquets MPLS sont perdus pendant la fenêtre de convergence LDP.

**C5.9** — **Faux**. LDP distribue des labels pour les routes de la **table IP globale** (les loopbacks et routes d'infrastructure). Les routes des VRF clients sont gérées par **MP-BGP** (qui distribue des labels VPN spécifiques). LDP ne connaît pas les VRF.

**C5.10** — LDP nécessite des sessions TCP par paire de voisins et maintient un état distribué (LIB) sur tous les routeurs. SR distribue les labels via les extensions IGP (IS-IS TLV, OSPF LSA opaque) — **pas de sessions de distribution séparées, pas d'état dans les routeurs de transit**. Philosophie : LDP est "state-full" (chaque routeur maintient la LIB), SR est "stateless" (le chemin est encodé dans la pile de labels par le source, les routeurs de transit n'ont que leurs SID locaux dans leur LFIB).
---

## 6. Plan de données : LFIB et actions

### Théorie

La **LFIB** (Label Forwarding Information Base) est la table utilisée par le plan de données MPLS pour forwarder les paquets. C'est l'équivalent de la FIB IP, mais pour les labels.

```
Exemple de LFIB sur un routeur P :

Label in  | Label out | Interface   | Action | Next-hop
----------|-----------|-------------|--------|----------
100       | 200       | GigE0/0/0/1 | SWAP   | 10.0.0.2
300       | POP       | GigE0/0/0/2 | PHP    | 10.0.0.3
3         | -         | GigE0/0/0/2 | POP    | 10.0.0.3
```

#### Les trois actions fondamentales

**PUSH (imposer un label)**
- Effectué par le **LER ingress** (PE dans le contexte L3 VPN).
- Ajoute un ou plusieurs labels sur un paquet IP entrant (ou un paquet déjà MPLS).
- En L3 VPN : pousse deux labels (VPN inner + transport outer).

```
[IP src=10.1.0.1, dst=10.2.0.1]
       ↓ PUSH label transport + label VPN
[T=300, S=0][VPN=25, S=1][IP src=10.1.0.1, dst=10.2.0.1]
```

**SWAP (remplacer un label)**
- Effectué par les **LSR de transit** (P dans le contexte L3 VPN).
- Remplace le label outer par un nouveau label (celui que le prochain saut attend).
- Ne modifie jamais les labels inférieurs dans la pile.

```
[T=300, S=0][VPN=25, S=1][IP] → SWAP T: 300→400 → [T=400, S=0][VPN=25, S=1][IP]
```

**POP (retirer un label)**
- Effectué par le **LER egress** ou le **routeur P pénultième** (PHP).
- Retire le label outer de la pile.
- Si c'est le seul label (S=1), le payload suivant (IP) est délivré.

```
[VPN=25, S=1][IP] → POP label VPN → [IP src=10.1.0.1, dst=10.2.0.1]
```

#### Lookup dans la LFIB

Le lookup LFIB est un **exact match** sur le label entrant — beaucoup plus simple structurellement qu'un LPM IP. L'espace de labels est local à chaque routeur : le label 100 sur R1 peut désigner un chemin complètement différent du label 100 sur R2.

> **Attention** : les labels n'ont de signification que sur le lien sur lequel ils sont utilisés. Un label est toujours attribué par le routeur qui le reçoit (le routeur "downstream"), pas par celui qui le pousse.

#### Commandes de vérification

```cisco
! Table de forwarding MPLS
show mpls forwarding-table

! Labels distribués par LDP
show mpls ldp bindings

! Table de labels pour une VRF (L3 VPN)
show mpls forwarding-table vrf BANK_A
```

### Histoire

La LFIB est l'implémentation du plan de données défini dans la RFC 3031. Sur les premières implémentations MPLS (fin des années 1990), la LFIB était logicielle — chaque paquet MPLS était traité par le CPU du routeur. L'introduction d'ASICs dédiés (Cisco CEF, Juniper Trio) a permis de traiter les paquets MPLS à des vitesses de 10-100 Gbps par ligne, rendant MPLS viable pour les cœurs de réseaux à très haut débit.

### Exercices — Section 6

**Q6.1** — Quelle est la différence entre un lookup **LPM** (IP) et un lookup **exact match** (MPLS) ? Lequel est structurellement plus simple ?

**Q6.2** — Identifiez l'action MPLS (PUSH, SWAP, POP) effectuée par chaque équipement dans le scénario suivant :
```
CE1 → PE1 → P1 → P2 → PE2 → CE2
```
(Supposer PHP sur P2, double label en L3 VPN)

**Q6.3** — Un paquet arrive sur un LSR P avec la LFIB suivante :
```
Label in | Label out | Interface | Action
---------|-----------|-----------|-------
200      | 300       | eth0/1    | SWAP
300      | POP       | eth0/2    | PHP
```
Le paquet a le label entrant = 200. Décrivez exactement ce qui se passe.

**Q6.4** — Vrai ou Faux : le label 100 attribué par R1 et le label 100 attribué par R2 désignent nécessairement le même chemin. Justifiez.

**Q6.5** — Lors d'un SWAP sur un LSR P, le label outer est remplacé. Le label inner (VPN) est-il modifié ? Le TTL outer est-il modifié ?

**Q6.6** — Quelle commande permet de voir la LFIB complète sur un routeur Cisco ? Quelles informations y trouve-t-on ?

**Q6.7** — Un ingénieur observe que le paquet arrive sur PE2 avec **deux** labels alors qu'il attendait un seul (PHP n'a pas eu lieu). Quelle est la conséquence pour PE2 en termes de nombre de lookups ?

**Q6.8** — Expliquez pourquoi les labels MPLS sont attribués par le routeur **downstream** (le receveur) et non par le routeur **upstream** (l'émetteur). Quel est l'avantage de ce design ?

**Q6.9** — Dans une topologie à 4 sauts MPLS (PE1 → P1 → P2 → P3 → PE2), combien d'opérations SWAP sont effectuées sur le label de transport ? Combien d'opérations PUSH ? Combien de POP ?

**Q6.10** — Un opérateur veut vérifier que le trafic MPLS traverse bien son réseau et que les labels sont correctement swappés. Quelle commande utilise-t-il ? Que cherche-t-il dans la sortie ?


### Corrections — Section 6

**C6.1** — **LPM** (IP) : pour chaque paquet, chercher le préfixe le plus long parmi N entrées (O(log N) ou O(1) avec TCAM). **Exact match** (MPLS) : lookup direct sur une valeur de 20 bits → table de hachage O(1). L'exact match est structurellement plus simple mais les ASICs modernes rendent les deux comparables en pratique.

**C6.2** — CE1 → PE1 : **rien** (CE ne fait pas de MPLS). PE1 → P1 : **PUSH** (ajoute labels VPN + transport). P1 → P2 : **SWAP** (échange le label transport). P2 → PE2 : **POP** (PHP — retire le label transport, S'il y a un label VPN il reste). PE2 → CE2 : **POP** (retire le label VPN, forward IP).

**C6.3** — Label entrant = 200. LFIB : 200 → SWAP 200→300, iface eth0/1. Résultat : le routeur **remplace le label 200 par 300**, décrémente le TTL MPLS de 1, et forward le paquet sur eth0/1 vers next-hop 10.0.0.2.

**C6.4** — **Faux**. Les labels sont **locaux à chaque routeur**. Le label 100 sur R1 signifie "ce paquet doit aller vers telle destination via R1" — le label 100 sur R2 peut désigner un chemin/destination totalement différent. Il n'y a aucune signification globale d'un label.

**C6.5** — Lors d'un SWAP : le **label outer est remplacé** par le nouveau label. Le **label inner (VPN) n'est jamais modifié** par un LSR de transit. Le **TTL outer est décrémenté de 1** (propagation TTL activée par défaut).

**C6.6** — `show mpls forwarding-table`. Informations : label entrant, label sortant (ou "Pop" pour PHP), interface de sortie, next-hop, bytes/paquets commutés (statistiques).

**C6.7** — Sans PHP, PE2 reçoit `[T][VPN][IP]`. Il doit effectuer **deux lookups** : (1) lookup LFIB sur T → action POP, révèle le label VPN ; (2) lookup LFIB sur VPN → identification VRF + interface de sortie. Avec PHP, un seul lookup suffit. L'impact est minimal sur les ASICs modernes mais PHP reste une bonne pratique.

**C6.8** — Le routeur **downstream** (receveur) alloue le label car c'est lui qui doit l'interpréter à l'arrivée. Il connaît la sémantique locale de ce label (quelle FEC, quelle action). L'upstream (émetteur) n'a pas besoin de savoir ce que le label signifie — il le pousse et forward. Ce design simplifie la coordination : chaque routeur contrôle ses propres labels.

**C6.9** — Topologie PE1 → P1 → P2 → P3 → PE2 (4 sauts MPLS) : **1 PUSH** (PE1), **3 SWAP** (P1, P2, et P3 si pas de PHP), ou **2 SWAP + 1 POP/PHP** (P3 fait PHP). **1 POP** (PE2 retire le label VPN). Total avec PHP sur P3 : 1 PUSH + 2 SWAP + 1 PHP + 1 POP VPN.

**C6.10** — `traceroute mpls ipv4 <loopback-PE2> /32` : valide le LSP end-to-end, affiche chaque saut avec le label utilisé et la latence. Chercher "!" (succès) à chaque saut. Ou `show mpls forwarding-table` sur chaque routeur pour vérifier les labels et les compteurs de trafic (bytes switched > 0 indique que le trafic passe).
---

## 7. LSP : Label Switched Path

### Théorie

Un **LSP** (Label Switched Path) est le chemin emprunté par un flux MPLS dans le réseau, défini par la séquence de labels utilisés à chaque saut.

```
LSP de PE1 vers PE2 :

PE1 ──[label 300]──► P1 ──[label 400]──► P2 ──[label 500 / PHP]──► PE2
      PUSH 300          SWAP 300→400        POP (PHP) → PE2 reçoit sans label transport
```

#### Propriétés d'un LSP

- **Unidirectionnel** : un LSP va de A vers B. Le trafic de retour (B vers A) emprunte un LSP distinct, avec ses propres labels.
- **Label local** : chaque label n'a de signification que sur le lien sur lequel il est utilisé.
- **Granularité** : un LSP peut transporter tout le trafic vers une destination (LSP par destination) ou un flux spécifique (LSP par FEC précis).

#### Comment un LSP est-il construit par LDP ?

1. L'IGP calcule le plus court chemin de PE1 vers le loopback de PE2 (`10.0.0.2/32`).
2. LDP distribue les labels pour `10.0.0.2/32` en partant de PE2 (downstream) vers PE1 :
   - PE2 annonce à P2 : "pour me joindre, utilise label L1"
   - P2 retransmet à P1 : "pour joindre PE2 via moi, utilise label L2"
   - P1 retransmet à PE1 : "pour joindre PE2 via moi, utilise label L3"
3. PE1 construit la LFIB : pour envoyer un paquet à PE2, pousser label L3 (outer).

#### LSP et IGP

Un LSP LDP suit **toujours** le chemin calculé par l'IGP (OSPF ou IS-IS). Il n'y a pas de chemin alternatif possible sans changer l'IGP ou utiliser RSVP-TE. C'est une limitation de LDP par rapport à RSVP-TE (section 9) ou Segment Routing (section 10).

### Histoire

Le concept de LSP est directement inspiré des **PVC (Permanent Virtual Circuits)** d'ATM et des **DLCIs (Data-Link Connection Identifiers)** de Frame Relay — des identifiants de circuit pré-établis sur lesquels le trafic est forwardé sans lookup complexe. La différence fondamentale : ATM/Frame Relay utilisaient des circuits physiquement provisionnés, MPLS construit ses LSP dynamiquement via des protocoles de signalisation (LDP, RSVP-TE).

### Exercices — Section 7

**Q7.1** — Expliquez pourquoi un LSP est dit **unidirectionnel**. Combien de LSP faut-il pour une communication bidirectionnelle entre PE1 et PE2 ?

**Q7.2** — Dans la construction d'un LSP par LDP, dans quel sens les labels sont-ils distribués : du routeur source vers la destination, ou de la destination vers la source ? Pourquoi ?

**Q7.3** — Si l'IGP change le chemin optimal (ex: suite à une panne), LDP reconstruit-il automatiquement le LSP sur le nouveau chemin ? Décrivez ce qui se passe.

**Q7.4** — Vrai ou Faux : deux LSP distincts peuvent utiliser le même label sur un même routeur. Justifiez.

**Q7.5** — Un réseau a deux chemins entre PE1 et PE2 : chemin A (plus court IGP) et chemin B (plus long mais plus de bande passante). LDP construit-il le LSP sur le chemin A ou B ? Peut-on forcer le LSP sur le chemin B sans changer l'IGP ?

**Q7.6** — Qu'est-ce qu'un **LSP de secours** (backup LSP) ? Comment est-il utilisé en cas de panne ?

**Q7.7** — Un opérateur a 100 PE dans son réseau. Combien de LSP LDP doivent être établis pour que chaque PE puisse atteindre tous les autres (dans les deux sens) ?

**Q7.8** — Différenciez un LSP "hop-by-hop" (LDP) d'un LSP "ER-LSP" (Explicit Route LSP, RSVP-TE). Quel est l'avantage de chacun ?

**Q7.9** — La commande `show mpls forwarding-table` montre les labels et interfaces de sortie. Comment vérifier qu'un LSP complet de PE1 à PE2 est opérationnel ?

**Q7.10** — Quel est le lien entre le LSP et le FEC ? Un LSP transporte-t-il un seul FEC ou peut-il en transporter plusieurs ?


### Corrections — Section 7

**C7.1** — Un LSP est unidirectionnel car les labels sont attribués par le routeur downstream (receveur) et signifient "envoie-moi un paquet avec ce label". Un label pour PE1→PE2 ne peut pas être utilisé dans le sens PE2→PE1. Pour une communication bidirectionnelle, **2 LSP distincts** sont nécessaires.

**C7.2** — Les labels sont distribués du **downstream vers l'upstream** (de la destination vers la source). Raison : le label est attribué par le routeur qui va le recevoir (downstream). PE2 alloue un label et l'annonce à P, qui l'alloue à son tour à PE1. Ainsi, PE1 sait quel label empiler pour que P puis PE2 comprennent où router le paquet.

**C7.3** — Oui, LDP reconstruit automatiquement le LSP. Quand l'IGP change le next-hop (nouveau chemin après panne), LDP utilise le nouveau next-hop et les labels déjà connus dans la LIB (Liberal Retention) pour mettre à jour la LFIB. Le LSP est reconstruit en suivant le nouveau chemin IGP.

**C7.4** — **Faux**. Sur un même routeur, un label entrant ne peut correspondre qu'à **une seule entrée LFIB** (exact match). Deux LSP distincts ne peuvent pas utiliser le même label entrant sur le même routeur — sinon ambiguïté.

**C7.5** — LDP construit le LSP sur le **chemin IGP le plus court** (chemin A). Il est impossible de forcer le LSP sur le chemin B **sans modifier l'IGP ou sans utiliser RSVP-TE/SR**. C'est la limitation fondamentale de LDP.

**C7.6** — Un **LSP de secours** (backup LSP ou bypass tunnel) est un LSP pré-établi sur un chemin alternatif. En cas de panne du LSP principal, le trafic est redirigé sur le LSP de secours en < 50ms (MPLS FRR). Après reconvergence IGP, le trafic revient sur le chemin optimal et le LSP de secours reste disponible pour la prochaine panne.

**C7.7** — Pour N=100 PE, chaque PE doit pouvoir atteindre les 99 autres → 99 LSP par PE, dans les **deux sens**. Total : 100 × 99 = **9900 LSP** (ou 100×99/2 = 4950 paires de LSP bidirectionnels).

**C7.8** — **Hop-by-hop LSP (LDP)** : suit automatiquement le chemin IGP, simple à déployer, pas de planification. **ER-LSP (RSVP-TE)** : chemin explicite défini à l'avance, avec réservation de bande passante. Avantage hop-by-hop : simplicité. Avantage ER-LSP : contrôle précis du chemin et garanties de ressources.

**C7.9** — `ping mpls ipv4 <loopback-PE2>/32` : valide le LSP (envoi d'un paquet MPLS echo request). `traceroute mpls ipv4 <loopback-PE2>/32` : vérifie le LSP saut par saut. Comparer le résultat avec le chemin IGP (`show ip route <loopback-PE2>`) pour confirmer la cohérence.

**C7.10** — Un LSP est associé à un **FEC** (Forwarding Equivalence Class). Un LSP LDP standard transporte **tout le trafic appartenant à un FEC** (ex: tout le trafic vers le loopback `10.0.0.2/32`). Plusieurs flux peuvent utiliser le même LSP s'ils appartiennent au même FEC. Un LSP RSVP-TE peut être associé à un FEC plus fin (ex: un tunnel TE spécifique pour un client particulier).
---

## 8. PHP : Penultimate Hop Popping

### Théorie

**PHP** (Penultimate Hop Popping) est une optimisation du plan de données MPLS dans laquelle le **routeur P juste avant le LER egress** retire le label de transport, plutôt que de laisser le LER egress le faire.

#### Pourquoi PHP ?

Sans PHP, le LER egress recevrait un paquet avec un label de transport et devrait effectuer **deux lookups successifs** :
1. Lookup LFIB sur le label de transport → action POP → révèle le payload
2. Lookup sur le payload (IP ou label VPN) → décision de forwarding

Avec PHP, le P pénultième retire le label de transport avant d'envoyer le paquet au LER egress. Ce dernier ne reçoit que le payload (IP nu ou label VPN) et n'effectue qu'**un seul lookup**.

#### Mécanisme

Le LER egress annonce à son voisin P direct (pénultième hop) le label **Implicit NULL (valeur 3)** via LDP :
```
PE2 → P_pénultième : "Pour mon loopback 10.0.0.2/32, label = 3 (Implicit NULL)"
```
Le label 3 est un signal : "avant de me forwarder le paquet, retire ton label de transport". Le routeur P reçoit ce label, comprend qu'il doit effectuer PHP (POP le label de transport), et forward le paquet sans ce label vers PE2.

#### Impact en contexte L3 VPN (double label)

En L3 VPN avec double label :
- Avant PHP : PE2 reçoit `[T][VPN][IP]` → 2 lookups
- Après PHP (sur P pénultième) : PE2 reçoit `[VPN][IP]` → 1 seul lookup LFIB sur le label VPN → identification de la VRF et du CE de sortie

```
Sans PHP :   P → PE2 : [T=300, S=0][VPN=25, S=1][IP]
Avec PHP :   P → PE2 : [VPN=25, S=1][IP]           ← P a retiré le label T
```

#### Explicit NULL (label 0)

Alternative à l'Implicit NULL : le LER egress annonce le label **Explicit NULL (valeur 0)**. Le P pénultième envoie le paquet avec ce label (sans le retirer) → PE2 reçoit le paquet avec le label 0 → PE2 sait qu'il est le dernier routeur et fait le lookup IP. L'Explicit NULL est utilisé quand on veut **préserver le TC (QoS)** jusqu'au dernier saut, ce que PHP/Implicit NULL ne permet pas (le label TC est retiré avec PHP).

### Histoire

PHP a été introduit dans la RFC 3031 (MPLS Architecture) comme optimisation implicite. L'Implicit NULL (label 3) est une elegance conceptuelle : plutôt que d'ajouter un bit "effectue PHP" dans le label, on utilise une valeur de label réservée avec une sémantique précise. Cela évite tout changement dans la structure du label. L'Explicit NULL (label 0) a été ajouté pour les cas où la QoS (champ TC) doit être préservée jusqu'au dernier saut.

### Exercices — Section 8

**Q8.1** — Expliquez PHP en une phrase. Quel routeur l'effectue et à quel moment ?

**Q8.2** — Sans PHP, combien de lookups le LER egress effectue-t-il en contexte L3 VPN (double label) ? Avec PHP, combien ? Expliquez chaque lookup.

**Q8.3** — Comment le routeur P pénultième sait-il qu'il doit effectuer PHP ? Quel label lui est annoncé par LDP ?

**Q8.4** — Quelle est la différence entre **Implicit NULL (label 3)** et **Explicit NULL (label 0)** ? Dans quel cas utilise-t-on l'un ou l'autre ?

**Q8.5** — Vrai ou Faux : PHP est une option configurable sur les routeurs MPLS. Par défaut, est-il activé ou désactivé sur Cisco IOS ?

**Q8.6** — En contexte L3 VPN avec PHP, le PE egress reçoit un paquet avec uniquement le label VPN. Comment identifie-t-il la VRF et l'interface de sortie ?

**Q8.7** — Un ingénieur désactive PHP sur tous ses PE (`mpls ldp explicit-null`). Quels labels les routeurs P pénultièmes utilisent-ils maintenant ? Quel est l'impact sur les PE egress ?

**Q8.8** — Dans une topologie `PE1 — P1 — P2 — PE2`, lequel des deux routeurs P effectue PHP ?

**Q8.9** — PHP est-il effectué sur le label VPN (inner label) ? Qui retire le label VPN et quand ?

**Q8.10** — Expliquez pourquoi l'Explicit NULL préserve la QoS là où l'Implicit NULL ne le peut pas.


### Corrections — Section 8

**C8.1** — PHP est effectué par le **routeur P juste avant le LER egress** (pénultième hop), qui retire le label de transport avant de forwarder le paquet au LER egress.

**C8.2** — Sans PHP en L3 VPN (double label) : PE2 reçoit `[T][VPN][IP]` → (1) lookup LFIB sur T → POP, révèle le label VPN ; (2) lookup LFIB sur VPN → identification VRF + interface → forward. **2 lookups**. Avec PHP : PE2 reçoit `[VPN][IP]` → (1) lookup LFIB sur VPN → identification VRF + interface → forward. **1 seul lookup**.

**C8.3** — Le P pénultième reçoit de PE2 (via LDP) le label **Implicit NULL = 3** pour le loopback de PE2. Quand le P traite un paquet dont l'entrée LFIB correspond à ce label → action POP → retire le label de transport → forward le payload vers PE2.

**C8.4** — **Implicit NULL (3)** : le P pénultième retire le label avant d'envoyer au PE egress → PE egress reçoit sans ce label (ou avec seulement le label VPN). **Explicit NULL (0)** : le P pénultième envoie le paquet avec le label 0 → PE egress reçoit le label 0 → retire et fait le lookup IP. Cas d'usage d'Explicit NULL : préserver le champ **TC (QoS)** jusqu'au PE egress. Avec Implicit NULL/PHP, le label (et son TC) est retiré par le P pénultième → le PE egress ne peut plus lire le TC pour appliquer sa politique QoS. Avec Explicit NULL, le label 0 (et son TC) arrive au PE egress → TC lisible.

**C8.5** — Sur Cisco IOS, PHP est **activé par défaut** (les PE annoncent l'Implicit NULL par défaut pour leurs loopbacks). Ce n'est pas une "option" à activer mais le comportement par défaut. Pour désactiver PHP et utiliser Explicit NULL : `mpls ldp explicit-null`.

**C8.6** — PE2 reçoit `[VPN=25, S=1][IP]`. Lookup LFIB sur label 25 → entrée : "label 25 → VRF BANK_A, interface GigE0/1 vers CE2, action POP". PE2 retire le label VPN (POP) et forward l'IP vers CE2.

**C8.7** — Avec `mpls ldp explicit-null` : PE2 annonce le label **Explicit NULL (0)** au lieu de l'Implicit NULL (3). Les routeurs P pénultièmes envoient le paquet avec le label 0 au lieu de retirer le label. PE2 reçoit `[0][VPN][IP]` en L3 VPN → doit d'abord POP le label 0 → puis lookup LFIB sur VPN → 2 lookups au lieu d'1 sans Explicit NULL. Impact QoS : le TC est préservé jusqu'à PE2.

**C8.8** — Dans `PE1 — P1 — P2 — PE2`, le PHP est effectué par **P2** (pénultième hop avant PE2). P2 annonce l'Implicit NULL pour le loopback de PE2 et retire le label de transport avant de forwarder vers PE2.

**C8.9** — **Non**. PHP concerne uniquement le **label de transport** (outer label). Le label VPN (inner label) est retiré par le **PE egress** lui-même, après avoir identifié la VRF de sortie.

**C8.10** — Avec PHP/Implicit NULL : le P pénultième retire le label (et donc son TC) → PE egress reçoit le payload sans label TC → ne peut pas appliquer de QoS basée sur TC. Avec Explicit NULL : le P pénultième envoie le label 0 avec son TC intact → PE egress voit le TC → peut appliquer des politiques QoS différenciées par classe de service.
---

## 9. MPLS Traffic Engineering (RSVP-TE)

### Théorie

> Ce tutoriel présente MPLS-TE comme concept important à connaître, sans entrer dans les détails de configuration. RSVP-TE est mentionné car il est souvent évoqué en contexte MPLS opérateur, mais sa configuration complète dépasse la portée de ce tutoriel.

**MPLS Traffic Engineering (MPLS-TE)** permet de définir des **LSP explicites** (ER-LSP — Explicit Route LSP) dont le chemin n'est pas contraint par l'IGP, et de **réserver de la bande passante** sur ces chemins.

#### Problème résolu

Avec LDP seul, les LSP suivent toujours le chemin IGP le plus court. Si un lien a une bande passante limitée et qu'un flux conséquent y est routé par l'IGP, on ne peut pas le dévier vers un lien moins chargé même si ce lien existe.

```
Scénario sans TE :
  PE1 → P1 → P2 → PE2  (chemin IGP, lien P1-P2 congestionné)
  PE1 → P3 → P4 → PE2  (chemin alternatif, largement disponible, non utilisé)

Avec MPLS-TE :
  Flux prioritaire → LSP explicit via P3-P4 (réservation 500 Mbps)
  Flux best-effort → chemin IGP via P1-P2
```

#### Protocole utilisé : RSVP-TE

**RSVP-TE** (Resource Reservation Protocol - Traffic Engineering, RFC 3209) est le protocole de signalisation utilisé pour établir des LSP avec chemin explicite et réservation de bande passante. Il étend RSVP (RFC 2205, protocole de réservation de ressources pour la QoS dans les réseaux IP) pour supporter les LSP MPLS.

RSVP-TE n'est **pas un remplacement de LDP** : les deux coexistent souvent, LDP pour les LSP hop-by-hop ordinaires, RSVP-TE pour les LSP TE.

**Note** : La configuration détaillée de RSVP-TE (tunnels TE, contraintes, affinités, CSPF…) n'est pas couverte dans ce tutoriel. Elle nécessite une formation dédiée MPLS-TE.

#### Intégration IGP-TE

Pour que RSVP-TE puisse calculer des chemins en tenant compte de la bande passante disponible, les IGP (OSPF et IS-IS) ont été étendus pour distribuer des **informations TE** (bande passante disponible par lien, couleur des liens, etc.) :
- **OSPF-TE** : LSA opaque de type 10 (RFC 3630)
- **IS-IS-TE** : TLV étendus (RFC 5305)

Ces extensions ne sont pas détaillées dans ce tutoriel.

### Histoire

RSVP original (RFC 2205, 1997) était destiné à la réservation de ressources QoS dans les réseaux IP (Intserv). Son adaptation pour MPLS-TE (RFC 3209, 2001) l'a transformé en outil de provisionnement de LSP explicites. MPLS-TE avec RSVP a été massivement déployé dans les années 2000-2010 pour l'optimisation du trafic dans les réseaux opérateurs. Cependant, sa complexité opérationnelle (configuration des tunnels, des contraintes, du re-optimization timer) a conduit à la recherche d'alternatives. **Segment Routing (section 10)** est la réponse architecturale moderne à ces problèmes.

### Exercices — Section 9

**Q9.1** — Quel problème MPLS-TE résout-il que LDP seul ne peut pas adresser ?

**Q9.2** — Qu'est-ce qu'un LSP "explicit route" (ER-LSP) ? En quoi diffère-t-il d'un LSP LDP ?

**Q9.3** — Quel protocole de signalisation est utilisé pour établir des LSP TE ? Quelle information supplémentaire (par rapport à LDP) transporte-t-il ?

**Q9.4** — Pourquoi RSVP-TE et LDP coexistent-ils souvent sur le même réseau plutôt que RSVP-TE remplaçant complètement LDP ?

**Q9.5** — Qu'est-ce que le **CSPF** (Constrained Shortest Path First) ? Quel est son rôle dans MPLS-TE ?


### Corrections — Section 9

**C9.1** — MPLS-TE résout l'impossibilité de choisir un chemin différent du plus court chemin IGP. LDP seul force tous les LSP à suivre l'IGP — impossible de rerouter un flux vers un lien sous-utilisé même si ce lien est disponible.

**C9.2** — Un ER-LSP (Explicit Route LSP) suit un chemin précis défini à l'avance (liste de routeurs ou de liens), indépendamment du chemin IGP. Un LSP LDP suit le chemin IGP automatiquement, sans contrôle de l'administrateur sur le chemin exact.

**C9.3** — **RSVP-TE** (RFC 3209). En plus des informations de chemin, RSVP-TE transporte des demandes de **réservation de bande passante** (PATH messages avec TSpec/RSpec) et des confirmations de réservation (RESV messages). LDP transporte uniquement les associations FEC→label sans réservation de ressources.

**C9.4** — RSVP-TE est utilisé pour les LSP TE (chemins explicites, réservations). LDP est utilisé pour les LSP ordinaires (hop-by-hop). La majorité du trafic d'un réseau opérateur ne nécessite pas de TE — LDP est suffisant. RSVP-TE n'est activé que pour les flux qui ont des contraintes spécifiques (bande passante, latence, priorité). Remplacer tout LDP par RSVP-TE serait une surcharge de configuration et de signalisation disproportionnée.

**C9.5** — CSPF (Constrained Shortest Path First) est une variante de l'algorithme SPF qui calcule le plus court chemin en **respectant des contraintes** (bande passante disponible minimum, affinités de liens, exclusion de liens/routeurs). CSPF est exécuté par le head-end (LER ingress) pour calculer le chemin du tunnel TE avant de le signaler via RSVP-TE.
---

## 10. Segment Routing (SR-MPLS)

### Théorie

**Segment Routing (SR)** est une architecture de forwarding moderne qui simplifie radicalement MPLS en éliminant les protocoles de distribution d'état par routeur (LDP, RSVP-TE). Il est défini dans la RFC 8402 (2018) et est aujourd'hui le standard émergent pour les nouveaux déploiements MPLS.

#### Principe fondamental

En Segment Routing, un **Segment** est une instruction de forwarding. Chaque segment est identifié par un **SID (Segment ID)**, représenté sous forme de label MPLS (SR-MPLS) ou d'en-tête IPv6 (SRv6).

Le chemin d'un paquet est encodé **directement dans la pile de labels** par le nœud source (LER ingress), sans qu'aucun état ne soit maintenu dans les routeurs de transit.

```
Exemple SR-MPLS :
  PE1 veut envoyer via P2 → PE2 (chemin explicite)

  Labels poussés par PE1 :
  [SID(P2), S=0][SID(PE2), S=1][IP client]
       ↑               ↑
  Segment vers P2  Segment vers PE2

  P1 voit SID(P2) → SWAP vers P2
  P2 voit SID(PE2) → SWAP vers PE2 (ou PHP)
  PE2 reçoit le payload
```

#### Types de Segments

**Node SID** : identifie un routeur spécifique. Distribué par l'IGP (IS-IS ou OSPF) dans les extensions SR. Globalement unique dans le domaine.

**Adjacency SID** : identifie un lien spécifique entre deux routeurs. Local au routeur. Permet de forcer le trafic sur un lien précis.

**Prefix SID** : identifie un préfixe réseau. Analogue au Node SID mais pour un réseau.

#### SRGB (Segment Routing Global Block)

Les SID sont alloués depuis un bloc de labels réservé appelé **SRGB (Segment Routing Global Block)**. Par défaut sur Cisco IOS-XR : labels 16000–23999. Un Node SID est toujours alloué depuis le SRGB (label = SRGB_start + SID_index).

```
SRGB = 16000–23999
Node SID index = 1 → Label MPLS = 16001
Node SID index = 2 → Label MPLS = 16002
```

#### SR vs LDP : différences clés

| Critère | LDP | SR-MPLS |
|---|---|---|
| Distribution des labels | Sessions LDP par paire de voisins | Distribution via IGP (IS-IS/OSPF extensions) |
| État dans les routeurs de transit | Oui (LIB par FEC) | Non (stateless forwarding) |
| Chemin explicite | Non (suit IGP) | Oui (SR Policy) |
| Fast ReRoute | LDP FRR (complexe) | TI-LFA (natif, simple) |
| Protocoles requis | IGP + LDP | IGP + extensions SR uniquement |

#### SR Policy et Traffic Engineering

**SR Policy** remplace RSVP-TE pour l'ingénierie de trafic. Un SR Policy est une liste ordonnée de SID (appelée **segment list** ou **SID list**) qui encode le chemin explicite. La SR Policy peut être :
- Configurée statiquement sur le LER ingress
- Programmée dynamiquement par un contrôleur SDN via **BGP SR Policy** ou **PCEP** (Path Computation Element Protocol)

> **SRv6** (Segment Routing over IPv6) est une variante de SR utilisant des adresses IPv6 comme SID au lieu de labels MPLS. Elle n'est pas détaillée dans ce tutoriel.

### Histoire

Segment Routing a été proposé par Clarence Filsfils (Cisco) et ses collègues vers 2013, standardisé à l'IETF à partir de 2015. SR est conçu comme une réponse directe aux problèmes opérationnels de LDP et RSVP-TE : trop de protocoles, trop d'état distribué, convergence complexe. En éliminant LDP et RSVP-TE au profit d'extensions IGP légères, SR simplifie drastiquement l'architecture. Aujourd'hui, la plupart des grands opérateurs (Orange, BT, AT&T, NTT…) migrent leurs backbones de LDP vers SR-MPLS ou SRv6. Le L3 VPN MPLS (tutoriel 3) fonctionne identiquement avec LDP ou SR comme plan de transport — seuls les labels de transport changent.

### Exercices — Section 10

**Q10.1** — Quelle est la différence fondamentale de philosophie entre LDP et SR-MPLS pour la distribution des labels ?

**Q10.2** — Qu'est-ce qu'un **Node SID** ? Comment est-il distribué dans le réseau ? Quel protocole le transporte ?

**Q10.3** — Qu'est-ce que le **SRGB** ? Si le SRGB commence à 16000 et qu'un routeur a le Node SID index 5, quel est son label MPLS ?

**Q10.4** — Expliquez comment SR-MPLS permet de réaliser de l'ingénierie de trafic sans RSVP-TE.

**Q10.5** — Vrai ou Faux : le plan de données MPLS (PUSH/SWAP/POP sur les labels) change fondamentalement avec SR-MPLS par rapport à LDP. Justifiez.

**Q10.6** — Quelle est la différence entre un **Node SID** et un **Adjacency SID** ? Donnez un cas d'usage pour chacun.

**Q10.7** — Comment le L3 VPN MPLS (tutoriel 3) est-il impacté si l'opérateur remplace LDP par SR-MPLS ? Quels composants changent, lesquels restent identiques ?

**Q10.8** — Qu'est-ce que **TI-LFA** (Topology Independent Loop-Free Alternate) ? En quoi améliore-t-il le FRR par rapport aux approches LDP FRR ou RSVP-TE FRR ?

**Q10.9** — Qu'est-ce qu'une **SR Policy** ? Comment remplace-t-elle les tunnels RSVP-TE pour l'ingénierie de trafic ?

**Q10.10** — Un opérateur migre de LDP vers SR-MPLS. Doit-il remplacer tout son équipement ? Peut-il migrer progressivement ? Quel mécanisme d'interopérabilité existe entre LDP et SR ?


### Corrections — Section 10

**C10.1** — **LDP** : chaque routeur maintient des sessions LDP avec ses voisins et distribue des labels (état distribué, sessions par paire). **SR-MPLS** : les SID sont configurés localement et distribués via les extensions IGP (IS-IS TLV, OSPF LSA) — pas de sessions séparées, pas d'état dans les routeurs de transit (stateless).

**C10.2** — Un **Node SID** identifie un routeur spécifique dans le domaine SR. Il est configuré localement sur le routeur (comme un index) et distribué par l'**IGP** (IS-IS ou OSPF via leurs extensions SR) à tous les routeurs du domaine. Chaque routeur connaît le Node SID de tous les autres routeurs via la LSDB étendue SR.

**C10.3** — SRGB = 16000, SID index = 5 → Label MPLS = 16000 + 5 = **16005**.

**C10.4** — Avec SR-MPLS, un LER ingress peut construire une **pile de SID** encodant un chemin explicite (via des Node SID ou Adjacency SID spécifiques) sans avoir besoin de RSVP-TE. Le chemin explicite est défini dans une **SR Policy** (segment list). Les routeurs de transit ne maintiennent aucun état de ce chemin — ils swappent simplement le SID/label courant.

**C10.5** — **Faux**. Le plan de données MPLS (PUSH/SWAP/POP) ne change **pas** avec SR-MPLS. Les routeurs effectuent les mêmes opérations sur des labels MPLS de 32 bits. La différence est dans le plan de contrôle (comment les labels sont distribués et gérés), pas dans le plan de données.

**C10.6** — **Node SID** : identifie un routeur — utilisé pour router vers ce routeur ou définir un chemin via ce routeur dans une SR Policy. Cas d'usage : `[NodeSID(P3)][NodeSID(PE2)]` = "va via P3 puis PE2". **Adjacency SID** : identifie un lien spécifique — utilisé pour forcer le trafic sur un lien précis. Cas d'usage : protection FRR (utiliser l'Adjacency SID de secours en cas de panne du lien principal).

**C10.7** — Avec SR-MPLS en remplacement de LDP : les **labels de transport** (outer labels) changent (SID au lieu de labels LDP). Les composants **inchangés** : VRF, RD, RT, MP-BGP, labels VPN (inner labels). SR remplace uniquement le mécanisme de distribution des labels de transport. Le reste du L3 VPN est identique.

**C10.8** — TI-LFA calcule des chemins de secours **garantis sans boucle** pour toutes les topologies (100% de couverture), encodés en SR (pile de SID). Avantages vs FRR classique : (1) couverture topologique quasi-totale (LDP FRR peut manquer des cas) ; (2) pas besoin de pré-provisionner des tunnels RSVP-TE ; (3) natif dans SR (pas de protocole supplémentaire). Le chemin de secours est calculé localement et encodé comme une SR Policy.

**C10.9** — Une **SR Policy** est un ensemble de règles associant du trafic (couleur + endpoint) à un chemin explicite (segment list = liste ordonnée de SID). Elle remplace les tunnels RSVP-TE : pas besoin de signaler le chemin via RSVP-TE, pas de maintien d'état dans les routeurs de transit. La SR Policy peut être configurée statiquement, calculée par un contrôleur (PCE via PCEP), ou distribuée via BGP SR Policy.

**C10.10** — Pas besoin de remplacer tout l'équipement. Migration progressive possible : (1) activer les extensions SR dans l'IGP (IS-IS ou OSPF) sur les nouveaux équipements supportant SR → ils annoncent leurs SID ; (2) les anciens équipements (LDP uniquement) continuent avec LDP ; (3) mécanisme d'interopérabilité : **LDP-SR Mapping Server** (RFC 8287) ou **SR Prefix-SID ↔ LDP label binding** — les équipements SR apprennent les labels LDP des équipements legacy et vice versa. Migration progressive, aire par aire ou site par site.
---

## 11. Convergence et Fast ReRoute (FRR)

### Théorie

La convergence MPLS est le processus par lequel le réseau rétablit le forwarding après une panne. Elle est étroitement liée à la convergence IGP et LDP.

#### Séquence de convergence

```
1. Détection de panne (BFD ou timer IGP/LDP)
        ↓
2. IGP recalcule SPF → nouvelles routes installées
        ↓
3. LDP (ou SR) reconverge → nouveaux labels sur le nouveau chemin
        ↓
4. Trafic MPLS rétabli sur le nouveau LSP
```

#### BFD (Bidirectional Forwarding Detection)

**BFD** (RFC 5880) est un protocole de détection de panne ultra-rapide. Il envoie des petits paquets de "keep-alive" entre deux routeurs à des intervalles configurable de quelques millisecondes. Si une réponse n'est pas reçue dans le délai imparti, la panne est déclarée.

```
Sans BFD : délai de détection = Dead timer IGP (typiquement 40s pour OSPF, 3-30s pour IS-IS)
Avec BFD  : délai de détection = 3 × intervalle = 3 × 50ms = 150ms (configurable)
```

BFD peut être couplé avec l'IGP, LDP, et BGP pour accélérer la détection de panne sur chacun de ces protocoles.

#### MPLS FRR (Fast ReRoute)

Le **MPLS FRR** est un mécanisme de protection pré-calculée qui permet de basculer le trafic sur un chemin de secours en **moins de 50ms**, sans attendre la reconvergence complète de l'IGP.

**Principe** : avant toute panne, des **LSP de secours** (bypass tunnels ou detour LSP) sont pré-établis. En cas de panne, le routeur qui détecte la panne (le Point of Local Repair, PLR) redirige immédiatement le trafic sur le LSP de secours, **localement**, sans signalisation vers d'autres routeurs.

```
Topologie normale :  PE1 → P1 → P2 → PE2
LSP de secours :     PE1 → P1 → P3 → P4 → PE2 (pré-établi)

Panne du lien P1-P2 :
  P1 (PLR) détecte la panne
  P1 bascule immédiatement le trafic vers P3-P4 (< 50ms)
  L'IGP reconverge ensuite (quelques secondes)
  Une fois l'IGP convergé, le trafic revient sur le chemin optimal
```

**FRR avec RSVP-TE** : les bypass tunnels sont établis via RSVP-TE, avec réservation de bande passante. (RFC 4090)

**FRR avec LDP** : mécanisme similaire mais sans réservation de bande passante. Plus complexe à configurer que SR TI-LFA.

**TI-LFA (Topology Independent Loop-Free Alternate)** : mécanisme FRR natif de Segment Routing. Calcule des chemins de secours garantis sans boucle pour toutes les topologies (pas de limite de couverture). Le chemin de secours est encodé dans une SR Policy (pile de SID). Disponible uniquement avec SR-MPLS ou SRv6.

#### Liberal Label Retention et convergence LDP

Comme vu en section 5, le **Liberal Label Retention** de LDP accélère la convergence : les labels du chemin alternatif sont déjà connus dans la LIB → lors d'un changement IGP, la LFIB peut être mise à jour quasi-instantanément.

### Histoire

Les premières implémentations MPLS (fin des années 1990) avaient des temps de convergence comparables à l'IGP — plusieurs dizaines de secondes. Les réseaux téléphoniques SDH/SONET offraient des temps de reprise de 50ms avec l'APS (Automatic Protection Switching) — le "50ms benchmark" est devenu la cible de l'industrie IP. MPLS FRR avec RSVP-TE (RFC 4090, 2005) a atteint cet objectif. TI-LFA (2018-2020) l'a généralisé sans la complexité de RSVP-TE.

### Exercices — Section 11

**Q11.1** — Listez dans l'ordre les 4 étapes de convergence MPLS après une panne de lien.

**Q11.2** — Pourquoi la convergence IGP doit-elle précéder la reconvergence LDP ? Que se passe-t-il si LDP reconverge avant l'IGP ?

**Q11.3** — Quel est l'avantage de BFD par rapport aux timers Hello/Dead natifs de l'IGP pour la détection de panne ? Donnez des ordres de grandeur de délai.

**Q11.4** — Expliquez le principe du MPLS FRR. Pourquoi peut-il basculer le trafic en moins de 50ms alors que l'IGP met plusieurs secondes à converger ?

**Q11.5** — Qu'est-ce que le **PLR** (Point of Local Repair) dans un contexte MPLS FRR ? Quel routeur joue ce rôle ?

**Q11.6** — Comment le **Liberal Label Retention** de LDP contribue-t-il à la rapidité de convergence MPLS ?

**Q11.7** — Quelle est la différence entre **FRR avec RSVP-TE** et **TI-LFA** en termes de complexité de mise en œuvre et de couverture topologique ?

**Q11.8** — Vrai ou Faux : lors d'une reconvergence MPLS après une panne, les labels VPN (inner labels en L3 VPN) doivent être redistribués via MP-BGP. Justifiez.

**Q11.9** — Un opérateur a des SLA de convergence < 50ms avec ses clients. Quelle combinaison de mécanismes (BFD, FRR, TI-LFA) recommanderiez-vous et pourquoi ?

**Q11.10** — Après une panne et reconvergence, comment vérifier que les LSP ont bien été rétablis sur le nouveau chemin ? Quelles commandes utiliser ?


### Corrections — Section 11

**C11.1** — (1) Détection de panne (BFD ou timer IGP) ; (2) IGP recalcule SPF → nouvelles routes ; (3) LDP/SR reconverge → nouveaux labels sur le nouveau chemin ; (4) trafic MPLS rétabli.

**C11.2** — LDP suit la FIB IGP pour ses labels. Si LDP reconverge avant l'IGP (hypothétiquement), la LFIB pointe vers un chemin qui n'est pas encore dans la FIB IGP → incohérence → les paquets pourraient être mal routés ou perdus. L'IGP doit d'abord installer les nouvelles routes dans la FIB, puis LDP met à jour sa LFIB en conséquence.

**C11.3** — BFD : intervalles de 50ms × 3 = **150ms de détection**. Timers IGP classiques : Hello OSPF = 10s, Dead = 40s → **40s de détection**. IS-IS Hello L2 = 10s par défaut → Dead = 3 × 10s = 30s. BFD est environ **200-300x plus rapide** que les timers IGP natifs.

**C11.4** — FRR pré-calcule des LSP de secours **avant toute panne**. En cas de panne, le PLR (routeur qui détecte la panne) bascule localement le trafic sur le LSP de secours **sans aucune signalisation vers d'autres routeurs**. Ce basculement local est une opération simple (mise à jour LFIB) qui s'effectue en < 50ms. L'IGP prend plusieurs secondes à reconverger car il nécessite l'inondation de LSA/LSP et le recalcul SPF sur tous les routeurs.

**C11.5** — Le **PLR (Point of Local Repair)** est le routeur qui détecte la panne et effectue le basculement FRR local. C'est le routeur immédiatement en amont de la panne (ex: dans `PE1→P1→P2→PE2`, si le lien P1-P2 tombe, P1 est le PLR).

**C11.6** — Liberal Label Retention conserve les labels de tous les voisins. Lors d'un changement IGP (nouveau next-hop suite à panne), le label du nouveau next-hop est **déjà dans la LIB** → mise à jour immédiate de la LFIB → pas d'attente d'un nouvel échange LDP. Conservative Retention nécessiterait de solliciter les labels du nouveau voisin → délai supplémentaire de convergence.

**C11.7** — **FRR RSVP-TE** : bypass tunnels pré-établis via RSVP-TE, avec réservation de bande passante. Configuration complexe (tunnels, contraintes, re-optimization). Couverture non-totale (dépend de la topologie et des tunnesl configurés). **TI-LFA** : chemin de secours calculé automatiquement par l'algorithme TI-LFA, encodé en SR Policy. Configuration simple (activer TI-LFA dans l'IGP). Couverture quasi-totale (garantie sans boucle pour toutes les topologies supportées).

**C11.8** — **Faux**. Les labels VPN (inner labels) sont distribués par MP-BGP entre PE. Une panne dans le cœur MPLS (entre P et P, ou entre P et PE) n'affecte pas les sessions iBGP PE-PE (établies via les loopbacks PE qui restent joignables par d'autres chemins). Les labels VPN restent valides et inchangés. Seuls les labels de transport (outer, LDP ou SR) changent lors de la reconvergence.

**C11.9** — Recommandation : (1) **BFD** sur tous les liens du cœur (détection < 150ms) ; (2) **TI-LFA** (si SR-MPLS déployé) ou MPLS FRR avec RSVP-TE (si LDP) pour le basculement en < 50ms ; (3) **LDP IGP Sync** pour éviter les black-holes à la remontée des liens. Combinaison BFD + TI-LFA est la plus simple et la plus efficace pour des SLA < 50ms.

**C11.10** — `show mpls forwarding-table` : vérifier que les labels et interfaces correspondent au nouveau chemin. `traceroute mpls ipv4 <loopback-PE-egress>/32` : vérifier que le LSP passe bien par les routeurs du nouveau chemin. `show mpls ldp neighbor` : vérifier que les sessions LDP sont up sur les nouveaux liens. Comparer avec `show ip route <loopback-PE>` pour confirmer la cohérence IGP/MPLS.
---

## 12. Exercices de compréhension générale

**QG.1 — Tracé de paquet complet**

Topologie :
```
CE1 ──── PE1 ──── P1 ──── P2 ──── PE2 ──── CE2
         Loopback: 10.0.0.1   10.0.0.2
```
- LDP actif sur tous les liens
- Label LDP PE1→PE2 : PE1 pousse T=200, P1 SWAP→300, P2 PHP (Implicit NULL annoncé par PE2)
- CE1 envoie un paquet IP `src=192.168.1.1, dst=192.168.2.1, TTL=64`
- Pas de L3 VPN (label unique)

Décrivez l'état exact du paquet (labels, TTL) à chaque étape : entrée PE1, sortie PE1, sortie P1, sortie P2, entrée PE2.

---

**QG.2 — Analyse de LFIB**

Un routeur P a la LFIB suivante :
```
Label in | Label out | Next-hop      | Action
---------|-----------|---------------|-------
100      | 200       | 10.0.0.2      | SWAP
101      | 201       | 10.0.0.3      | SWAP
3        | -         | 10.0.0.4      | POP (PHP)
102      | -         | 10.0.0.4      | POP
```
a) Que fait ce routeur avec un paquet label=100 ?
b) Que fait-il avec un paquet label=3 ?
c) Que fait-il avec un paquet label=102 ?
d) Le label 3 et le label 102 aboutissent au même résultat — quelle est la différence entre les deux cas ?

---

**QG.3 — Construction de LSP**

Topologie `PE1 — P — PE2` avec LDP. Le loopback de PE2 est `10.0.0.2/32`.

a) Listez les messages LDP échangés (dans quel sens et avec quelle information) pour construire le LSP PE1→PE2.
b) Après construction, que contient la LFIB de P pour la destination `10.0.0.2/32` ?
c) Que se passe-t-il si PE2 annonce l'Implicit NULL (label 3) au lieu d'un label ordinaire ?

---

**QG.4 — Comparaison LDP vs SR**

Pour chacun des scénarios suivants, indiquez si LDP ou SR-MPLS est mieux adapté, et pourquoi :
a) Réseau de 20 routeurs, équipe peu expérimentée, pas de TE requis.
b) Backbone de 500 routeurs, TE avec chemins explicites requis, convergence < 50ms.
c) Migration progressive d'un réseau existant LDP avec quelques équipements ne supportant pas SR.

---

**QG.5 — Diagnostic**

Un opérateur remarque que les paquets MPLS sont perdus sur un lien, bien que l'IGP montre le lien comme up et que `show mpls ldp neighbor` montre la session LDP comme up. Proposez une méthodologie de diagnostic structurée.

---

**QG.6 — PHP et double label**

En contexte L3 VPN MPLS (que vous verrez en détail dans le tutoriel 3), le paquet porte deux labels. Expliquez ce qui se passe exactement lors du PHP :
a) Quel label est retiré par le P pénultième ?
b) Quel label reste sur le paquet envoyé au PE egress ?
c) Que fait le PE egress avec ce label restant ?

---

**QG.7 — Convergence et BFD**

Un opérateur a des timers OSPF Hello=10s, Dead=40s, et pas de BFD. Un lien tombe entre P1 et P2.
a) Combien de temps avant que P1 détecte la panne ?
b) Combien de temps avant que tous les routeurs aient reconvergé leur LFIB (estimer en secondes) ?
c) Si BFD est configuré avec des intervalles de 50ms (×3 = 150ms de détection), comment cela change-t-il les réponses a) et b) ?

---

**QG.8 — SR Policy**

Un opérateur veut forcer un flux VoIP (haute priorité) de PE1 vers PE2 via le chemin `PE1 → P3 → P4 → PE2` (chemin non optimal selon l'IGP mais avec plus de bande passante disponible). Les Node SID sont : P3=16003, P4=16004, PE2=16002.
a) Quelle pile de SID/labels PE1 pousse-t-il sur les paquets VoIP ?
b) Quel mécanisme SR permet de définir ce chemin ?
c) Ce chemin serait-il possible avec LDP seul ? Pourquoi ?

---

**QG.9 — Structure de label**

Un paquet MPLS capturé sur le fil a les 8 octets suivants (deux labels empilés) :
```
00 01 90 3F  00 00 19 FF
```
Décomposez chaque label en ses champs (Label, TC, S, TTL). Qu'est-ce que cela nous dit sur la position de chaque label dans la pile ?

---

**QG.10 — Préparation au tutoriel L3 VPN**

Dans le tutoriel 3 (L3 VPN MPLS), MPLS est utilisé comme plan de données. Listez les cinq mécanismes MPLS vus dans ce tutoriel qui sont réutilisés directement dans L3 VPN, et expliquez brièvement le rôle de chacun dans ce contexte.

---

### Corrections — Exercices généraux

**CG.1** —
- Sortie CE1 : `[IP src=192.168.1.1, dst=192.168.2.1, TTL=64]`
- Sortie PE1 (PUSH label T=200) : `[T=200, TC=0, S=1, TTL=63][IP src=192.168.1.1, dst=192.168.2.1, TTL=64]` *(un seul label, pas de VPN ici)*
- Sortie P1 (SWAP 200→300, TTL decrementé) : `[T=300, TC=0, S=1, TTL=62][IP]`
- Sortie P2 (PHP — Implicit NULL annoncé par PE2 → POP) : `[IP src=192.168.1.1, dst=192.168.2.1, TTL=64]` *(P2 retire le label, IP arrive nu à PE2)*
- Entrée PE2 : `[IP src=192.168.1.1, dst=192.168.2.1, TTL=64]` → PE2 fait un lookup IP → forward vers CE2.

**CG.2** —
a) Label=100 : SWAP 100→200, forward vers next-hop 10.0.0.2 sur eth0/1.
b) Label=3 (Implicit NULL) : **POP** — retire le label, forward le payload vers next-hop 10.0.0.4 sur (interface non indiquée dans la table pour label 3, mais via next-hop 10.0.0.4).
c) Label=102 : **POP** — retire le label, forward vers next-hop 10.0.0.4.
d) Les deux aboutissent à un POP et un forwarding vers 10.0.0.4, mais : label 3 est l'**Implicit NULL** (annoncé par un PE egress, signal PHP standard), label 102 est un label ordinaire dont l'action est POP (le routeur downstream a annoncé ce label avec une action POP explicite). Le résultat est identique mais le mécanisme de signalisation LDP est différent.

**CG.3** —
a) Messages LDP : PE2 → P : "FEC=10.0.0.2/32, Label=3 (Implicit NULL)" [signal PHP]. P → PE1 : "FEC=10.0.0.2/32, Label=L1" [pour joindre PE2 via moi]. PE1 → P : "FEC=10.0.0.1/32, Label=L2". P → PE2 : "FEC=10.0.0.1/32, Label=L3".
b) LFIB de P pour 10.0.0.2/32 (loopback PE2) : Label_in=L1, action=POP (PHP), interface=eth vers PE2, next-hop=PE2 (car PE2 a annoncé Implicit NULL).
c) P reçoit L1 dans sa LFIB avec action POP → retire le label → forward le payload directement à PE2 sans label. PE2 reçoit l'IP brut.

**CG.4** —
a) 20 routeurs, pas de TE : **LDP** — simple, aucune configuration de chemins explicites, facile pour une équipe peu expérimentée. SR est aussi valable mais LDP peut suffire.
b) 500 routeurs, TE + < 50ms : **SR-MPLS** — TI-LFA natif pour le FRR, SR Policy pour l'ingénierie de trafic, scalabilité supérieure à LDP+RSVP-TE.
c) Migration progressive LDP → SR : déployer SR progressivement avec **LDP-SR Mapping Server** pour l'interopérabilité. Les équipements legacy continuent avec LDP, les nouveaux utilisent SR, la migration est transparente pour les services.

**CG.5** —
1. Vérifier `show mpls ldp neighbor` : session LDP up ? Si non, problème de session (voir section 5).
2. Vérifier `show mpls forwarding-table` sur le routeur problématique : le label entrant a-t-il une entrée ? Les statistiques (bytes switched) progressent-elles ?
3. Vérifier `show mpls ldp bindings` : le label pour la destination est-il alloué ?
4. Utiliser `ping mpls ipv4 <destination>/32` pour tester le LSP.
5. Si le LSP est cassé à un saut spécifique : `traceroute mpls ipv4 <destination>/32` pour localiser le saut défaillant.
6. Sur le saut défaillant : vérifier `show mpls forwarding-table`, `show mpls ldp neighbor`, et les compteurs d'interface.

**CG.6** —
a) PHP retire le **label de transport** (outer label, S=0) — c'est le label LDP.
b) Le paquet envoyé au PE egress porte **uniquement le label VPN** (inner, S=1).
c) PE egress reçoit `[VPN=25, S=1][IP]` → lookup LFIB sur label 25 → identifie VRF + interface CE → POP le label VPN → forward l'IP vers CE.

**CG.7** —
a) Avec Hello=10s, Dead=40s : P1 détecte la panne après **40 secondes** (expiration du Dead timer).
b) Reconvergence complète : 40s (détection) + quelques secondes (SPF + LDP) = environ **40-50 secondes** total.
c) Avec BFD 50ms×3 : détection en **150ms**. Reconvergence complète : 150ms + quelques secondes SPF + LDP reconvergence ≈ **1-3 secondes** total (sans FRR). Avec FRR : basculement en < 50ms, puis reconvergence IGP/LDP en arrière-plan.

**CG.8** —
a) Pile SR poussée par PE1 : `[SID(P3)=16003, S=0][SID(PE2)=16002, S=1][IP VoIP]`. Les routeurs swappent les SID dans l'ordre : P1 voit 16003 → forward vers P3 → P3 SWAP 16003→16002 → P4 voit 16002 → forward vers PE2 (ou PHP).
b) Mécanisme : une **SR Policy** (associant ce trafic à la segment list [16003, 16002]).
c) **Non** — avec LDP seul, les LSP suivent toujours le chemin IGP le plus court. Il serait impossible de forcer le chemin via P3-P4 sans modifier l'IGP ou utiliser RSVP-TE.

**CG.9** —
Label 1 : `00 01 90 3F`
```
0000 0000 0000 0001 1001 0000 0011 1111
Label = 0x00019 = 25
TC    = 000 = 0
S     = 0 (pas le bottom of stack → il y a un autre label dessous)
TTL   = 0x3F = 63
```
Label 2 : `00 00 19 FF`
```
0000 0000 0000 0000 0001 1001 1111 1111
Label = 0x00001 = 1... wait: 20 bits = 0x00001 = ... recalcul :
00 00 19 = 0x000019 = 25 → Label = 25 ?

Recalcul correct :
00 00 19 FF = 0000 0000 0000 0000 0001 1001 1111 1111
Label (20b) = 0000 0000 0000 0000 0001 = 1... → trop petit.

Reprenons : bits 31-12 = 20 bits :
0x000019FF :
En binaire : 0000 0000 0000 0000 0001 1001 1111 1111
Label (31-12) : 0000 0000 0000 0000 0001 = 1 (valeur décimale... non)

Hex → binaire direct :
0  0  0  0  1  9  F  F
0000 0000 0000 0000 0001 1001 1111 1111

Label (bits 31-12, 20 bits) = 00000000000000000001 = 1
TC (bits 11-9, 3 bits)      = 100
S (bit 8)                   = 1
TTL (bits 7-0)              = 11111111 = 255
```
Label 1 : Label=25, TC=0, S=0, TTL=63 → **outer label** (S=0, il y a un label en dessous).
Label 2 : Label=1 (Router Alert Label, réservé), TC=4, S=1, TTL=255 → **inner label** (S=1, bottom of stack).

Note : le label 1 est le Router Alert Label — un label réservé pour traitement spécial par le plan de contrôle. Cela indiquerait un paquet de contrôle MPLS (ex: OAM, RSVP) plutôt que du trafic ordinaire.

**CG.10** —
Mécanismes MPLS réutilisés en L3 VPN :
1. **Labels (structure 32 bits)** : le label VPN (inner) et le label transport (outer) sont des labels MPLS standards.
2. **LDP** : distribue les labels de transport (outer) entre PE et P pour construire les LSP.
3. **LFIB (PUSH/SWAP/POP)** : le PE ingress fait PUSH des deux labels, les P font SWAP du label outer, le P pénultième fait PHP (POP), le PE egress fait POP du label VPN.
4. **PHP (Implicit NULL)** : utilisé en L3 VPN pour que le PE egress ne reçoive que le label VPN (un seul lookup).
5. **IGP** : les loopbacks PE sont appris via l'IGP — le PE ingress résout le next-hop MP-BGP (loopback du PE egress) via l'IGP, puis LDP fournit le label de transport vers ce loopback.

---

*Dernière mise à jour : juin 2026 · Prérequis : Tutoriel 1 (IGP) · Suite : Tutoriel 3 — L3 VPN MPLS*