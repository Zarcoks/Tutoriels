# Tutoriel 3 : L3 VPN MPLS

> **Prérequis** : Tutoriel 1 (IGP) et Tutoriel 2 (MPLS). Il est supposé acquis que vous comprenez le fonctionnement d'OSPF ou IS-IS, la structure d'un label MPLS (32 bits, champs Label/TC/S/TTL), les actions PUSH/SWAP/POP, LDP, la LFIB, et le mécanisme PHP. Les notions de session BGP iBGP/eBGP et d'attributs BGP de base (AS-PATH, NEXT_HOP, LOCAL_PREF, MED) sont supposées connues — ce tutoriel utilise BGP mais ne le détaille pas comme protocole de routage inter-AS général.
>
> **Ce tutoriel couvre** : l'architecture L3 VPN MPLS (RFC 4364), du plan de contrôle (VRF, RD, RT, MP-BGP) au plan de données (double label, forwarding end-to-end), en passant par les topologies avancées (Hub & Spoke, Extranet, Inter-AS) et la convergence.

---

## Table des matières

1. [Vue d'ensemble : qu'est-ce qu'un L3 VPN ?](#1-vue-densemble--quest-ce-quun-l3-vpn-)
2. [VRF : Virtual Routing and Forwarding](#2-vrf--virtual-routing-and-forwarding)
3. [Route Distinguisher (RD)](#3-route-distinguisher-rd)
4. [Route Target (RT)](#4-route-target-rt)
5. [MP-BGP : le plan de contrôle VPN](#5-mp-bgp--le-plan-de-contrôle-vpn)
6. [Forwarding end-to-end : le double label](#6-forwarding-end-to-end--le-double-label)
7. [PE-CE : protocoles de routage côté client](#7-pe-ce--protocoles-de-routage-côté-client)
8. [Inter-AS L3 VPN (Options A, B, C)](#8-inter-as-l3-vpn-options-a-b-c)
9. [Convergence spécifique L3 VPN](#9-convergence-spécifique-l3-vpn)
10. [Exercices de compréhension générale](#10-exercices-de-compréhension-générale)

---

## 1. Vue d'ensemble : qu'est-ce qu'un L3 VPN ?

### Théorie

Un **L3 VPN MPLS** (RFC 4364, aussi appelé BGP/MPLS IP VPN) est un service fourni par un opérateur permettant d'interconnecter plusieurs sites d'un même client comme s'ils formaient un réseau IP privé — sans que l'opérateur ne déploie d'infrastructure physique dédiée par client.

#### Les acteurs du réseau

```
Site A                                              Site B
  │                                                   │
 CE ──── PE ─────── P ─────── P ─────── PE ──── CE
         │        (cœur)    (cœur)       │
      VRF client                      VRF client
      loopback                        loopback
      MP-BGP                          MP-BGP

CE : Customer Edge  → routeur appartenant au client
PE : Provider Edge  → routeur opérateur en bordure, porte les VRF
P  : Provider core  → routeur cœur, commute des labels, ignore les VRF
```

- Le **CE** ne fait aucun MPLS. Il échange des routes avec le PE via un protocole classique (eBGP, OSPF, statique). Il ne sait pas qu'il est dans un VPN MPLS.
- Le **PE** maintient une table de routage séparée (VRF) par client. C'est lui qui fait tout le travail VPN : isolation, distribution des routes via MP-BGP, imposition des labels.
- Le **P** ne connaît pas les clients. Il commute uniquement le label de transport. Il ne voit jamais les adresses IP des clients.

#### Comparaison avec les alternatives

Trois grandes familles de solutions permettent d'interconnecter des sites distants :

| Critère | IPsec/Internet | Liaisons louées (LL) | L3 VPN MPLS |
|---|---|---|---|
| Infrastructure | Internet public | Dédiée par client | Mutualisée opérateur |
| SLA bande passante | Aucun | Garanti | Contractuel |
| SLA latence | Aucun | Garanti | Contractuel |
| Adresses IP privées | Via NAT ou tunnel | Natives | Natives |
| Scalabilité | Bonne | Mauvaise (N² liens) | Excellente |
| QoS | Très limitée | Facile | Via DSCP/TC MPLS |
| Chiffrement natif | Oui (IPsec) | Non | Non (optionnel en surcouche) |
| Coût | Faible | Très élevé | Moyen à élevé |

> **IPsec** est un protocole de sécurité qui chiffre et authentifie les communications IP. Il est utilisé en VPN site-à-site sur Internet (tunnels IPsec/GRE). Il n'est pas détaillé dans ce tutoriel — nous le mentionnons comme alternative au L3 VPN MPLS. Sa principale limitation en contexte multi-sites est la scalabilité (N×(N-1)/2 tunnels pour un maillage complet) et l'absence de garanties de qualité de service.
>
> **GRE** (Generic Routing Encapsulation) est un protocole de tunnelisation qui peut encapsuler n'importe quel protocole réseau dans IP. Il est souvent combiné avec IPsec. Comme IPsec, il n'est pas détaillé ici.

#### Ce que L3 VPN résout

Le L3 VPN répond à la problématique opérateur suivante : comment faire coexister des centaines de clients avec des espaces d'adressage potentiellement identiques (tous utilisent `10.0.0.0/8`) sur une infrastructure physique mutualisée, tout en garantissant l'isolation totale entre clients ?

La réponse tient en quatre mécanismes qui font l'objet des sections suivantes :
1. **VRF** : isolation des tables de routage par client sur les PE
2. **RD** : unicité des préfixes dans MP-BGP malgré les overlaps
3. **RT** : filtrage d'import/export entre VRF
4. **MP-BGP + double label** : distribution des routes et forwarding

### Histoire

Dans les années 1990, les opérateurs interconnectaient les sites clients avec des circuits **Frame Relay** (PVC — Permanent Virtual Circuits) ou **ATM**. Pour un client avec N sites en maillage complet, il fallait N×(N-1)/2 circuits. Pour 10 sites : 45 circuits, chacun provisionné manuellement. Cette approche ne scalait pas.

MPLS émerge fin des années 1990. La **RFC 2547** (1999, Rosen & Rekhter, "BGP/MPLS VPNs") pose les bases des VPN BGP/MPLS. Elle est mise à jour par la **RFC 4364** (2006, "BGP/MPLS IP Virtual Private Networks") qui reste la référence actuelle. L'idée centrale : mutualiser l'infrastructure physique tout en isolant les clients par des mécanismes de plan de contrôle (VRF, RD, RT, MP-BGP), sans imposer aux clients de changer quoi que ce soit à leurs équipements.

### Exercices — Section 1

**Q1.1** — Un client a 8 sites à interconnecter en maillage complet. (a) Avec des liaisons louées point-à-point, combien de liens faut-il ? (b) Avec L3 VPN MPLS, combien de connexions physiques PE-CE faut-il ? (c) Quelle est la formule générale pour le maillage complet ?

**Q1.2** — Listez les trois rôles CE, PE, P. Pour chacun : (a) à qui appartient-il (client ou opérateur), (b) fait-il du MPLS, (c) maintient-il des VRF clients, (d) connaît-il les adresses IP des autres clients ?

**Q1.3** — Un client demande pourquoi ses paquets ne sont pas chiffrés dans le L3 VPN MPLS de son opérateur. Que lui répondez-vous ? Quelle solution existe s'il veut du chiffrement ?

**Q1.4** — Vrai ou Faux : le CE d'un client L3 VPN MPLS doit configurer MPLS. Justifiez.

**Q1.5** — Expliquez en quoi le L3 VPN est un service de **couche 3** et non de couche 2. Quelle en est la conséquence pour le client (que voit-il dans sa table de routage) ?

**Q1.6** — Pourquoi dit-on que les routeurs P peuvent être "légers" (peu d'état à maintenir) même dans un réseau avec des centaines de clients et des millions de routes VPN ?

**Q1.7** — Un ingénieur junior affirme : "Le L3 VPN MPLS est un VPN chiffré qui passe par Internet via des tunnels sécurisés." Identifiez les trois erreurs dans cette affirmation.

**Q1.8** — La RFC 4364 remplace la RFC 2547. Quelle était l'année de publication de chacune ? Quel est l'apport principal de la RFC 4364 par rapport à la RFC 2547 ?

**Q1.9** — Un opérateur a 500 clients avec en moyenne 6 sites chacun. Comparez le nombre de circuits à gérer entre l'approche Frame Relay et l'approche L3 VPN MPLS pour un service any-to-any.

**Q1.10** — Quels sont les quatre mécanismes fondamentaux du L3 VPN MPLS ? Donnez le rôle de chacun en une phrase.


### Corrections — Section 1

**C1.1** — (a) Maillage complet 8 sites : `8×7/2 = 28` liaisons. (b) L3 VPN MPLS : **8 connexions PE-CE** (une par site). (c) Formule maillage complet : `N×(N-1)/2`.

**C1.2** —

| Rôle | Appartient à | Fait du MPLS | VRF clients | Connaît IP clients autres |
|---|---|---|---|---|
| CE | Client | Non | Non | Non |
| PE | Opérateur | Oui (PUSH/POP) | Oui | Oui (via VRF) |
| P | Opérateur | Oui (SWAP) | Non | Non |

**C1.3** — Le L3 VPN MPLS n'est pas chiffré par défaut : l'isolation repose sur des mécanismes de plan de contrôle (VRF, labels) sur l'infrastructure privée de l'opérateur, pas sur du chiffrement. Le trafic voyage sur le réseau privé de l'opérateur (pas sur Internet) — le niveau de confiance est contractuel. S'il veut du chiffrement, le client peut superposer **IPsec** entre ses CE (tunnels IPsec CE-CE indépendants du VPN MPLS). IPsec n'est pas détaillé dans ce tutoriel.

**C1.4** — **Faux**. Le CE ne fait aucun MPLS. MPLS est entièrement opéré dans le backbone opérateur (entre PE et P). Le CE voit simplement un voisin de routage standard (eBGP, OSPF ou route statique vers le PE).

**C1.5** — L3 VPN = service de couche 3 car l'opérateur **route les paquets IP** du client entre les sites — il participe au plan de routage IP. Conséquence : le CE voit dans sa table de routage les réseaux des autres sites du client, comme s'ils étaient des voisins IP directs (via le PE). Comparer avec L2 VPN où l'opérateur transporte des trames Ethernet et le client gère lui-même le routage IP.

**C1.6** — Les P ne maintiennent que des labels de transport dans leur LFIB — une entrée par LSP, indépendamment du nombre de clients et de routes VPN. Un réseau de 2000 clients avec des millions de routes VPN se résume pour le P à quelques milliers d'entrées LFIB (une par loopback PE connu). La séparation PE/P via le double label rend cela possible : les routes VPN sont cachées derrière le label VPN, opaque pour les P.

**C1.7** — Trois erreurs : (1) le L3 VPN MPLS n'est **pas chiffré** (il n'utilise pas IPsec) ; (2) il ne **passe pas par Internet** (il utilise le réseau privé de l'opérateur) ; (3) ce ne sont **pas des tunnels** au sens classique (pas de chiffrement, pas d'encapsulation IPsec — c'est un mécanisme de commutation de labels sur infrastructure mutualisée).

**C1.8** — RFC 2547 : **1999**. RFC 4364 : **2006**. La RFC 4364 clarifie et formalise les mécanismes de la RFC 2547 (notamment les options inter-AS A/B/C, la gestion des RT avec plusieurs valeurs, et les recommandations de déploiement). Elle reste rétrocompatible avec les implémentations RFC 2547.

**C1.9** — Frame Relay : 500 clients × (moyenne 6 sites) = 3000 sites. Maillage complet par client : `5×6/2=15` circuits × 500 clients = **7500 circuits** à provisionner et gérer. L3 VPN : **3000 connexions PE-CE** (une par site). Ajouter un site = configurer une VRF sur un PE et une session PE-CE.

**C1.10** — (1) **VRF** : isolation des tables de routage par client sur les PE. (2) **RD** : rend les préfixes IP clients uniques dans la table BGP VPNv4 malgré les overlaps. (3) **RT** : filtre l'import/export des routes entre VRF (quel client reçoit quelles routes). (4) **MP-BGP + double label** : distribue les routes VPN entre PE et fournit le label VPN pour le forwarding.

---

## 2. VRF : Virtual Routing and Forwarding

### Théorie

Une **VRF** (Virtual Routing and Forwarding) est une instance de table de routage **complètement isolée** sur un PE. Chaque client a sa propre VRF sur chaque PE qui le dessert. Les interfaces côté CE sont assignées à la VRF correspondante.

```
PE1
├── Table globale
│     10.0.0.1/32  → Loopback0 (propre loopback PE)
│     10.0.0.2/32  → via IGP (loopback PE2)
│     10.0.0.99/32 → via IGP (loopback P)
│     [routes d'infrastructure opérateur uniquement]
│
├── VRF CLIENT_A
│     10.1.10.0/24 → GigE0/1 (CE du site A1)
│     10.2.10.0/24 → via PE2 (appris via MP-BGP)
│     [routes du client A uniquement]
│
└── VRF CLIENT_B
      192.168.0.0/24 → GigE0/2 (CE du site B1)
      192.168.1.0/24 → via PE3 (appris via MP-BGP)
      [routes du client B uniquement]
```

Les VRF CLIENT_A et CLIENT_B sont **totalement isolées** l'une de l'autre et de la table globale. Même si CLIENT_A et CLIENT_B utilisent tous deux le réseau `10.0.0.0/8`, il n'y a aucune collision — chaque VRF a sa propre instance de table de routage.

#### Impact sur les interfaces

Quand on assigne une interface à une VRF (`ip vrf forwarding NOM_VRF` sur Cisco IOS), **l'adresse IP de l'interface est immédiatement supprimée**. Il faut la reconfigurer. C'est parce que l'interface bascule de la table globale vers la table VRF.

```cisco
ip vrf CLIENT_A
 rd 65000:100
 route-target export 65000:100
 route-target import 65000:100

interface GigabitEthernet0/1
 ip vrf forwarding CLIENT_A   ! ← l'IP est supprimée ici
 ip address 192.168.10.1 255.255.255.0  ! ← reconfigurer l'IP
```

#### Table globale vs VRF

La **table globale** du PE contient uniquement les routes d'infrastructure opérateur :
- Loopbacks des PE (pour les sessions MP-BGP et LDP)
- Routes des liens entre P et PE
- Routes apprises via l'IGP du cœur

Les routes des clients ne sont **jamais** dans la table globale. L'isolation est totale.

#### VRF-Lite

**VRF-Lite** est une utilisation des VRF **sans MPLS**. Les VRF sont locales à un seul routeur ; pour interconnecter des VRF entre routeurs, on utilise des interfaces dédiées (physiques ou sous-interfaces) avec une session eBGP ou des routes statiques par VRF. VRF-Lite est utilisé en entreprise pour la segmentation (ex : séparer les réseaux de différentes zones de sécurité sur un seul routeur) sans déployer MPLS. Ce tutoriel se concentre sur les VRF dans le contexte L3 VPN MPLS.

#### Résolution du next-hop MP-BGP

Point important : quand le PE egress (PE2) reçoit une route VPNv4 via MP-BGP dont le NEXT_HOP est le loopback de PE1 (`10.0.0.1`), il résout ce next-hop dans la **table globale** (pas dans la VRF). Car l'adresse `10.0.0.1` est une adresse d'infrastructure opérateur, connue via l'IGP. C'est grâce à cette résolution que LDP peut fournir le label de transport vers PE1.

### Histoire

Les VRF ont été introduites dans Cisco IOS 12.0(5)T (fin des années 1990) comme composant fondateur du L3 VPN MPLS. Avant les VRF, la seule façon d'isoler le routage de plusieurs clients sur un même routeur physique était de déployer des routeurs physiques séparés. Les VRF ont permis la **virtualisation du plan de routage**, précurseur des techniques de virtualisation réseau modernes (VRF-Lite en entreprise, Network Slicing dans les réseaux 5G). Le concept est aujourd'hui omniprésent : tout équipement réseau moderne supporte les VRF.

### Exercices — Section 2

**Q2.1** — Sur PE1, deux clients ont tous deux le réseau `172.16.0.0/16` dans leur VRF respective (VRF_A et VRF_B). Un paquet destiné à `172.16.5.1` arrive sur l'interface assignée à VRF_A. Vers quel CE est-il forwardé ? Y a-t-il un risque de confusion avec VRF_B ?

**Q2.2** — Un ingénieur tape `ip vrf forwarding CLIENT_A` sur l'interface `GigabitEthernet0/1` qui avait l'IP `192.168.10.1/24`. Que se passe-t-il immédiatement ? Que doit-il faire ensuite pour rétablir la connectivité ?

**Q2.3** — Quelle est la différence entre VRF (dans le contexte L3 VPN MPLS) et VRF-Lite ? Dans quel contexte utilise-t-on VRF-Lite ?

**Q2.4** — La table globale d'un PE contient-elle les routes des clients ? Justifiez. Quelles routes y trouve-t-on ?

**Q2.5** — Vrai ou Faux : une interface physique d'un PE peut appartenir à deux VRF différentes simultanément. Justifiez. Comment dessert-on deux clients sur un même lien physique ?

**Q2.6** — Un client a 4 sites connectés à 4 PE différents. Combien de VRF au minimum sont créées sur l'ensemble du réseau opérateur pour ce client ?

**Q2.7** — Quelles commandes Cisco permettent de : (a) voir les VRF configurées sur un PE, (b) voir la table de routage d'une VRF spécifique, (c) voir les interfaces assignées à une VRF ?

**Q2.8** — Le CE a-t-il connaissance de l'existence des VRF sur le PE ? Justifiez. Que voit-il de son côté ?

**Q2.9** — Expliquez pourquoi le next-hop MP-BGP d'une route VPN (loopback du PE distant) est résolu dans la table globale et non dans la VRF du client.

**Q2.10** — Un opérateur veut permettre à deux clients (ALPHA et BETA) d'accéder à un serveur DNS commun hébergé chez l'opérateur, tout en restant isolés l'un de l'autre. Comment les VRF et les RT (section 4) permettent-ils de modéliser cette situation ? (Répondez en termes de tables de routage, sans la config complète.)


### Corrections — Section 2

**C2.1** — Le routeur consulte la **VRF du client** déterminée par l'interface d'entrée (assignée à VRF_A). Il fait un lookup dans VRF_A uniquement → trouve la route vers `172.16.5.1` via le CE de VRF_A. **Aucun risque de confusion** avec VRF_B : chaque paquet est associé à une VRF dès son entrée (via l'interface), et les tables de routage VRF sont complètement séparées.

**C2.2** — L'adresse IP `192.168.10.1/24` est **immédiatement supprimée** de l'interface. L'ingénieur doit **reconfigurer l'IP** sur l'interface après la commande `ip vrf forwarding`. C'est parce que l'interface bascule de la table globale vers la table VRF, et l'ancienne IP était dans la table globale.

**C2.3** — **VRF L3 VPN** : associées à RD/RT, les routes sont échangées entre PE via MP-BGP avec MPLS comme plan de données. **VRF-Lite** : VRF sans MPLS — les routes entre VRF de routeurs différents sont échangées via des interfaces dédiées (physiques ou sous-interfaces) avec eBGP ou routes statiques. VRF-Lite est utilisé en entreprise pour la segmentation réseau sans MPLS.

**C2.4** — **Non**. La table globale d'un PE contient uniquement les routes d'infrastructure opérateur (loopbacks PE/P, liens entre routeurs, routes IGP du cœur). Les routes clients restent dans leurs VRF respectives — isolées de la table globale et des autres VRF. C'est le mécanisme d'isolation fondamental.

**C2.5** — **Faux**. Une interface physique ne peut appartenir qu'à **une seule VRF**. Pour servir deux clients sur le même lien physique, on utilise des **sous-interfaces 802.1Q** (trunk), chacune assignée à une VRF différente.

**C2.6** — Minimum **4 VRF** (une par PE desservant le client). Si les 4 sites sont sur 4 PE distincts, chaque PE crée une VRF pour ce client. Si deux sites partagent un PE, ce PE a une VRF mais deux interfaces CE-facing dans cette VRF.

**C2.7** — (a) `show ip vrf` : liste toutes les VRF et leurs interfaces. (b) `show ip route vrf CLIENT_A` : table de routage de la VRF CLIENT_A. (c) `show ip vrf interfaces` ou regarder la colonne "Interfaces" dans `show ip vrf`.

**C2.8** — **Non**. Le CE ne sait pas qu'il est dans une VRF. Il voit simplement un routeur voisin (le PE) avec lequel il échange des routes via un protocole standard. La VRF est une abstraction entièrement côté PE, transparente pour le CE.

**C2.9** — Le loopback du PE distant est une **adresse d'infrastructure opérateur**, annoncée dans l'IGP du cœur et visible dans la table globale. La VRF ne contient que les routes du client — elle ne connaît pas les loopbacks des PE. Si BGP tentait de résoudre le next-hop dans la VRF, il ne trouverait rien. La table globale (peuplée par l'IGP) est le seul endroit où ces adresses sont connues.

**C2.10** — Modélisation : VRF_ALPHA avec import du RT de DNS_SHARED, VRF_BETA avec import du RT de DNS_SHARED, VRF_DNS avec export d'un RT distinct (`DNS:999`). ALPHA et BETA n'importent pas leurs RT respectifs → isolation entre eux. Les deux importent le RT de DNS → accès au DNS commun. Le DNS voit les deux clients (importe leurs RT ou est accessible via route statique). C'est la topologie **Extranet** (section 4).
---

## 3. Route Distinguisher (RD)

### Théorie

#### Le problème : les espaces d'adressage overlapping

Deux clients peuvent utiliser les mêmes plages d'adresses IP privées. Si PE1 annonce `10.1.0.0/24` pour CLIENT_A et `10.1.0.0/24` pour CLIENT_B via MP-BGP, PE2 reçoit deux préfixes identiques. BGP ne garde qu'un seul chemin par préfixe — l'un des deux clients est perdu.

#### La solution : le Route Distinguisher

Le **RD** est un préfixe de **64 bits** ajouté devant le préfixe IP pour créer une **adresse VPNv4 unique** dans la table BGP :

```
Sans RD :  10.1.0.0/24              ← ambiguë (quel client ?)
Avec RD :  65000:100:10.1.0.0/24   ← CLIENT_A (unique)
           65000:200:10.1.0.0/24   ← CLIENT_B (unique)
```

Une adresse VPNv4 est donc `RD (64 bits) + préfixe IPv4 (32 bits)` = **96 bits** au total. C'est ce qui est stocké dans la table BGP VPNv4.

#### Format du RD

Deux formats sont définis (RFC 4364) :

| Format | Structure | Exemple |
|---|---|---|
| Type 0 : AS:nn | AS 2 octets + valeur 4 octets | `65000:100` |
| Type 1 : IP:nn | IP 4 octets + valeur 2 octets | `10.0.0.1:100` |
| Type 2 : AS4:nn | AS 4 octets + valeur 2 octets | `65000:100` (AS 32 bits) |

Le format `IP:nn` utilise généralement le loopback du PE comme partie IP — il garantit l'unicité par PE sans coordination globale.

#### Propriétés importantes du RD

- Le RD **n'a pas de sémantique de filtrage** : il rend les routes uniques dans BGP, rien de plus. Ce n'est pas lui qui décide quelle VRF importe quelle route — c'est le rôle du **RT** (section 4).
- Deux PE peuvent utiliser des RD différents pour la même VRF d'un même client : c'est valide et même recommandé (voir correction Q3.6).
- Le RD est **transparent pour le CE** : il n'existe que dans les échanges MP-BGP entre PE.
- Le RD est **configurable par VRF** sur chaque PE.

```cisco
ip vrf CLIENT_A
 rd 65000:100        ! ← RD de cette VRF
 route-target export 65000:100
 route-target import 65000:100
```

### Histoire

La notion de Route Distinguisher a été introduite dans la RFC 2547 (1999). Elle répond directement au problème de l'overlapping address space entre clients, problème fondamental dans un contexte multi-tenant où l'opérateur n'a aucun contrôle sur le plan d'adressage de ses clients (RFC 1918 — adresses privées — est disponible pour tous). Le RD est une solution élégante : plutôt que d'obliger les clients à utiliser des adresses uniques, on ajoute un préfixe opérateur pour créer l'unicité dans le plan de contrôle, tout en transportant les adresses réelles dans le plan de données.

### Exercices — Section 3

**Q3.1** — Sans RD, que se passerait-il si CLIENT_A et CLIENT_B ont tous deux le réseau `192.168.1.0/24` et que PE1 annonce les deux via MP-BGP à PE2 ?

**Q3.2** — Quelle est la taille totale (en bits) d'une adresse VPNv4 ? Comment est-elle composée ?

**Q3.3** — Décomposez la route VPNv4 `65000:200:10.5.0.0/24` : identifiez le RD et le préfixe IP. Quelle est la longueur totale en bits de cette route VPNv4 ?

**Q3.4** — Vrai ou Faux : le RD est utilisé pour filtrer les routes lors de l'import dans une VRF. Quel mécanisme joue ce rôle à la place ?

**Q3.5** — Un opérateur utilise le format RD `IP:nn` avec l'IP loopback du PE. PE1 a le loopback `10.0.0.1`, PE2 a le loopback `10.0.0.2`. Les deux desservent le client GAMMA. Proposez un RD pour chaque PE, et expliquez l'avantage de cette approche.

**Q3.6** — Pourquoi est-il recommandé d'utiliser des RD **différents par PE** pour un même client (ex: PE1 utilise `65000:101`, PE2 utilise `65000:102` pour la même VRF) ? Quel problème BGP cela résout-il ?

**Q3.7** — Vrai ou Faux : le CE du client voit le RD dans les routes qu'il reçoit du PE. Justifiez.

**Q3.8** — Un opérateur décide d'utiliser le même RD `65000:1` pour tous ses clients. Expliquez précisément pourquoi c'est une erreur critique.

**Q3.9** — Dans la commande `show bgp vpnv4 unicast all` sur un PE Cisco, à quoi ressemble une entrée de la table BGP VPNv4 ? Identifiez où apparaît le RD.

**Q3.10** — Quelle est la différence entre un RD et un RT, en une phrase chacun ? (Anticipation de la section 4.)


### Corrections — Section 3

**C3.1** — Sans RD, PE1 enverrait deux UPDATE BGP avec le préfixe identique `192.168.1.0/24`. BGP applique sa règle de best-path selection et ne conserve qu'**un seul chemin**. L'une des deux routes clients est perdue — PE2 n'installe qu'une route `192.168.1.0/24` dans une seule VRF (ou la mauvaise). L'autre client n'est plus joignable, ou son trafic est routé vers le mauvais CE.

**C3.2** — Une adresse VPNv4 = **RD (64 bits) + préfixe IPv4 (variable)**. Taille totale : 64 bits (RD) + 32 bits (adresse IP) = **96 bits** pour une route hôte (/32). La longueur du préfixe s'ajoute comme metadata.

**C3.3** — RD : `65000:200` (64 bits). Préfixe IP : `10.5.0.0/24` (réseau 32 bits, longueur 24 bits). Longueur totale de la route VPNv4 : **64 + 32 = 96 bits** (pour la partie adresse).

**C3.4** — **Faux**. Le RD n'a aucun rôle de filtrage — il rend les routes uniques dans BGP, rien de plus. C'est le **RT (Route Target)** qui détermine dans quelle VRF une route est importée.

**C3.5** — PE1 (loopback `10.0.0.1`) : RD = `10.0.0.1:1` (ou `10.0.0.1:100`). PE2 (loopback `10.0.0.2`) : RD = `10.0.0.2:1`. Avantage : l'unicité est garantie naturellement par le loopback du PE (déjà unique dans le réseau), sans coordination globale. Deux PE n'auront jamais le même RD pour une même VRF.

**C3.6** — Avec des RD identiques par VRF sur tous les PE (ex: `65000:100` sur PE1 et PE2 pour CLIENT_A), les deux PE annoncent `65000:100:10.1.0.0/24` à PE3. BGP les traite comme deux chemins vers le **même préfixe VPNv4** et n'en installe qu'un (add-path non activé). Avec des RD différents (`65000:101` sur PE1 et `65000:102` sur PE2), les deux routes sont **distinctes** dans la table VPNv4 → PE3 peut les installer toutes les deux (résilience, load-balancing).

**C3.7** — **Faux**. Le RD est ajouté par le PE lors de l'export dans MP-BGP et retiré par le PE lors de l'import dans la VRF. Le CE ne voit que des préfixes IPv4 classiques — il ne connaît jamais le RD.

**C3.8** — Si tous les clients ont le même RD `65000:1`, leurs routes VPNv4 ne sont plus distinguables : CLIENT_A annonce `65000:1:10.0.0.0/24` et CLIENT_B annonce `65000:1:10.0.0.0/24` → **collision BGP** identique au cas sans RD. Une route écrase l'autre. La raison d'être du RD est perdue.

**C3.9** — Dans `show bgp vpnv4 unicast all` :
```
Route Distinguisher: 65000:100 (default for vrf CLIENT_A)
BGP table version is 5, local router ID is 10.0.0.2
*>i 10.1.0.0/24  10.0.0.1   0   100  0  65001 i
                 ↑ NEXT_HOP       ↑ LOCAL_PREF
```
Le RD apparaît en tête de section (`Route Distinguisher: 65000:100`). Chaque route listée en dessous appartient à ce RD.

**C3.10** — **RD** : préfixe 64 bits qui rend une route VPN **unique** dans la table BGP. **RT** : tag Extended Community BGP qui **filtre** dans quelle VRF une route est importée.
---

## 4. Route Target (RT)

### Théorie

#### Le problème : comment savoir dans quelle VRF importer une route ?

Une fois les routes VPNv4 uniques grâce au RD, PE2 reçoit potentiellement des routes pour des dizaines de clients différents. Comment sait-il dans quelle VRF locale importer chaque route ?

#### La solution : le Route Target

Le **RT** (Route Target) est une **Extended Community BGP** (attribut de 8 octets) attachée à chaque route VPNv4. Il fonctionne comme un **tag** : chaque VRF déclare quels RT elle exporte (tags ajoutés aux routes lors de l'export dans MP-BGP) et quels RT elle importe (routes portant ces tags sont importées dans la VRF).

```
PE1, VRF CLIENT_A :
  route-target export 65000:100   ← toutes les routes exportées portent ce tag
  route-target import 65000:100   ← toutes les routes portant ce tag sont importées

PE2, VRF CLIENT_A :
  route-target export 65000:100
  route-target import 65000:100   ← reçoit les routes de PE1 ✅

PE2, VRF CLIENT_B :
  route-target export 65000:200
  route-target import 65000:200   ← ne reçoit PAS les routes de CLIENT_A ❌
```

#### RT et Extended Community BGP

Le RT est transporté comme une **Extended Community BGP** (RFC 4360). Une Extended Community est un attribut BGP optionnel de 8 octets, conçu pour porter des informations supplémentaires attachées aux routes. Elle est transportée dans l'attribut `EXTENDED_COMMUNITIES` du message UPDATE BGP.

Contrairement aux attributs BGP ordinaires, les Extended Communities sont **transitives** (propagées entre AS) et **extensibles** (plusieurs valeurs possibles sur une même route). Une VRF peut exporter plusieurs RT simultanément.

> **BGP** (Border Gateway Protocol) est le protocole de routage inter-AS d'Internet. Il est supposé connu à un niveau de base (sessions iBGP/eBGP, attributs principaux). Ce tutoriel utilise MP-BGP (extension multiprotocole de BGP) mais ne détaille pas BGP dans sa globalité.

#### Topologies avancées avec les RT

**Topologie any-to-any** (tous les sites se voient mutuellement) :
```
Tous les sites :
  export RT=65000:100
  import RT=65000:100
```

**Topologie Hub & Spoke** (tout le trafic inter-sites passe par le HUB) :
```
VRF HUB   : export RT=65000:hub,   import RT=65000:spoke
VRF SPOKE : export RT=65000:spoke, import RT=65000:hub
```
Les spokes n'importent pas `65000:spoke` → ils ne se voient pas directement. Ils importent `65000:hub` → ils voient les routes du HUB. Le HUB importe `65000:spoke` → il voit toutes les routes des spokes et les réexporte vers les autres spokes via son RT hub.

**Topologie Extranet** (partage d'un service commun entre plusieurs clients) :
```
VRF CLIENT_A : export RT=65000:100, import RT=65000:100, import RT=65000:999
VRF CLIENT_B : export RT=65000:200, import RT=65000:200, import RT=65000:999
VRF SHARED   : export RT=65000:999, import RT=65000:999
             ! SHARED exporte 65000:999 → CLIENT_A et CLIENT_B l'importent
             ! CLIENT_A et CLIENT_B n'importent pas leurs RT respectifs → isolation
```

### Histoire

Le RT comme Extended Community BGP est défini dans la RFC 4360 (2006). L'idée de filtrer automatiquement les routes VPN par tag avait été ébauché dans la RFC 2547 mais sans mécanisme standard. Avant les RT, le filtrage des routes VPN entre PE se faisait manuellement avec des route-maps BGP — non scalable au-delà de quelques clients. Les RT ont permis de réduire la configuration d'un nouveau site VPN à la déclaration de quelques lignes de RT dans la VRF, les PE redistribuant automatiquement les routes selon les règles d'import/export.

### Exercices — Section 4

**Q4.1** — Expliquez en une phrase la différence fondamentale de rôle entre RD et RT.

**Q4.2** — Configuration donnée :
```
VRF A : export RT=100:10, import RT=100:10
VRF B : export RT=100:20, import RT=100:20
VRF C : export RT=100:10, import RT=100:20
```
(a) VRF C importe-t-elle les routes de VRF A ? (b) VRF C importe-t-elle les routes de VRF B ? (c) VRF A importe-t-elle les routes de VRF C ?

**Q4.3** — Qu'est-ce qu'une **Extended Community BGP** ? En quoi diffère-t-elle structurellement d'un attribut BGP ordinaire ?

**Q4.4** — Une entreprise a un site HQ et 5 sites agences. Elle veut que les agences communiquent entre elles ET avec le HQ (any-to-any). Définissez les RT export/import pour HQ et pour les agences.

**Q4.5** — Même entreprise, mais le responsable sécurité exige que tout le trafic inter-agences passe par le HQ (Hub & Spoke). Modifiez les RT de Q4.4 pour satisfaire cette contrainte.

**Q4.6** — Un opérateur héberge un serveur DNS partagé entre CLIENT_A et CLIENT_B, qui doivent rester isolés l'un de l'autre. Proposez un schéma de RT complet (avec une VRF DNS_SHARED).

**Q4.7** — Vrai ou Faux : une VRF ne peut exporter qu'un seul RT. Justifiez et donnez un cas d'usage où exporter plusieurs RT est nécessaire.

**Q4.8** — PE2 reçoit une route VPNv4 de PE1 avec le RT `65000:100`. La VRF CLIENT_A de PE2 a `import RT=65000:100`. La VRF CLIENT_B de PE2 a `import RT=65000:999`. Dans quelle(s) VRF la route est-elle installée ?

**Q4.9** — Dans une topologie Hub & Spoke, les spokes apprennent les routes des autres spokes **via le HUB**. Expliquez le mécanisme précis : comment une route du SPOKE1 arrive-t-elle dans la table de routage du SPOKE2 ?

**Q4.10** — Pourquoi le RT est-il transporté comme une Extended Community BGP plutôt que comme un attribut BGP ordinaire (comme LOCAL_PREF ou MED) ?


### Corrections — Section 4

**C4.1** — Le RD rend une route **unique** dans BGP. Le RT décide dans quelle VRF cette route est **importée**.

**C4.2** — (a) VRF C importe RT=100:20. VRF A exporte RT=100:10. `100:10 ≠ 100:20` → **Non**. (b) VRF C importe RT=100:20. VRF B exporte RT=100:20. → **Oui**. (c) VRF A importe RT=100:10. VRF C exporte RT=100:10. → **Oui**. Topologie asymétrique : C reçoit les routes de B, A reçoit les routes de C, mais C ne reçoit pas les routes de A.

**C4.3** — Une Extended Community BGP est un attribut de **8 octets** (RFC 4360), transporté dans l'attribut `EXTENDED_COMMUNITIES` d'un UPDATE BGP. Elle diffère d'un attribut ordinaire car : (1) elle est **extensible** (plusieurs valeurs possibles sur une même route) ; (2) elle est **transitive** entre AS ; (3) elle a un champ de type explicite permettant aux routeurs qui ne la comprennent pas de la propager sans l'interpréter.

**C4.4** — Any-to-any : tous importent et exportent le même RT. Exemple :
```
HQ     : export RT=65000:100, import RT=65000:100
Agences: export RT=65000:100, import RT=65000:100
```
Tous se voient mutuellement.

**C4.5** — Hub & Spoke :
```
HQ (Hub)  : export RT=65000:hub,   import RT=65000:spoke
Agences   : export RT=65000:spoke, import RT=65000:hub
```
Les agences n'importent pas `65000:spoke` → elles ne se voient pas directement. Elles importent `65000:hub` → elles voient les routes du HQ (qui réexporte les routes des autres agences via son RT hub).

**C4.6** — Schéma Extranet :
```
VRF CLIENT_A  : export RT=65000:10, import RT=65000:10, import RT=65000:999
VRF CLIENT_B  : export RT=65000:20, import RT=65000:20, import RT=65000:999
VRF DNS_SHARED: export RT=65000:999
```
CLIENT_A et CLIENT_B n'importent pas leurs RT respectifs → isolation. Tous deux importent RT=65000:999 → accès au DNS partagé. Le DNS exporte uniquement 65000:999 → les clients ne voient que le DNS, pas les routes de l'autre client.

**C4.7** — **Faux**. Une VRF peut exporter **plusieurs RT** simultanément. Exemple d'usage : une VRF de services partagés (DNS, NTP) qui doit être accessible par deux communautés différentes de clients : `export RT=65000:999, export RT=65000:888`. Les deux communautés importent leur RT respectif et obtiennent l'accès au service.

**C4.8** — La route est installée **uniquement dans VRF CLIENT_A** (qui importe RT=65000:100). VRF CLIENT_B n'importe que RT=65000:999 → route non installée.

**C4.9** — Route de SPOKE1 : PE_SPOKE1 exporte avec RT=65000:spoke. Le RR réfléchit cette route vers PE_HUB. PE_HUB importe RT=65000:spoke → la route de SPOKE1 est dans VRF_HUB. PE_HUB réexporte cette route (dans ses propres UPDATE BGP) avec son RT=65000:hub. Le RR réfléchit vers PE_SPOKE2. PE_SPOKE2 importe RT=65000:hub → la route de SPOKE1 est maintenant dans VRF_SPOKE2. Le trajet données physiques pour SPOKE2→SPOKE1 passera par PE_SPOKE2 → PE_HUB (car PE_SPOKE2 a comme next-hop le loopback de PE_HUB) → PE_SPOKE1.

**C4.10** — Un attribut BGP ordinaire (comme LOCAL_PREF) ne peut avoir qu'**une seule valeur** par route et n'est pas conçu pour être filtré par des routeurs tiers. Les Extended Communities : (1) permettent **plusieurs valeurs** sur une même route (une VRF peut avoir plusieurs RT) ; (2) ont un type field qui permet à des routeurs ne les comprenant pas de les **propager sans les interpréter** ; (3) sont **transitives** entre AS (utile pour l'inter-AS Option B/C).
---

## 5. MP-BGP : le plan de contrôle VPN

### Théorie

**MP-BGP** (Multiprotocol BGP, RFC 4760) est l'extension de BGP qui permet de transporter des familles d'adresses autres qu'IPv4 unicast. Dans le contexte L3 VPN, il transporte la famille **VPNv4** identifiée par :
- **AFI** (Address Family Identifier) = 1 (IPv4)
- **SAFI** (Subsequent AFI) = 128 (MPLS-labeled VPN addresses)

#### Sessions MP-BGP entre PE

Les PE établissent des sessions **iBGP** (même AS opérateur) entre eux pour échanger les routes VPNv4. La règle du split-horizon iBGP (un routeur iBGP ne réannonce pas une route reçue d'un pair iBGP à un autre pair iBGP) impose l'utilisation d'un **Route Reflector (RR)** pour éviter le full-mesh.

```
Topologie avec Route Reflector :

        Route Reflector
           /    |    \
         PE1   PE2   PE3

PE1 envoie ses routes VPNv4 au RR → RR les réfléchit à PE2 et PE3
(sans RR, PE1 devrait avoir des sessions directes avec PE2 ET PE3)
```

> **Route Reflector** : mécanisme BGP (RFC 4456) permettant à un routeur désigné (le RR) de réannoncer les routes iBGP reçues d'un client iBGP vers d'autres clients iBGP, contournant la règle du split-horizon. Il n'a pas besoin d'être un PE — un RR dédié sans VRF est courant dans les grands réseaux. Ce mécanisme BGP est supposé connu à un niveau de base.

#### Ce que MP-BGP transporte pour le L3 VPN

| Attribut BGP | Contenu dans L3 VPN |
|---|---|
| `MP_REACH_NLRI` | Route VPNv4 (RD + préfixe) + label VPN (inner label) |
| `NEXT_HOP` | Loopback du PE source (résolu via IGP+LDP) |
| `EXTENDED_COMMUNITIES` | Route Target(s) |
| `LOCAL_PREF` | Préférence de chemin (pour sélectionner le meilleur PE egress) |

Le label VPN (inner label) est transporté dans le champ `MP_REACH_NLRI` du message UPDATE. C'est ce label que le PE egress utilisera pour identifier la VRF et le CE de sortie.

#### Configuration MP-BGP sur un PE

```cisco
router bgp 65000
 ! Session avec le Route Reflector
 neighbor 10.0.0.100 remote-as 65000
 neighbor 10.0.0.100 update-source Loopback0
 neighbor 10.0.0.100 send-community extended   ! ← OBLIGATOIRE pour les RT

 address-family vpnv4
  neighbor 10.0.0.100 activate               ! ← active la famille VPNv4
 exit-address-family

 address-family ipv4 vrf CLIENT_A
  neighbor 192.168.10.2 remote-as 65001      ! ← session eBGP vers CE
  neighbor 192.168.10.2 activate
 exit-address-family
```

#### `send-community extended` : commande critique

Sans `send-community extended`, les **Extended Communities** (dont les RT) ne sont **pas transmises** dans les UPDATE BGP. PE2 reçoit les routes VPNv4 mais sans RT — il ne sait pas dans quelle VRF les importer. Résultat : aucune route n'est installée dans les VRF de PE2, même si la session BGP est établie.

#### `update-source Loopback0` : stabilité de la session

Sans cette commande, BGP utilise l'adresse IP de l'interface physique de sortie comme source de la session TCP. Si ce lien physique tombe, la session BGP tombe aussi — même si un chemin alternatif vers le voisin existe. Avec `update-source Loopback0`, la session BGP utilise le loopback (toujours actif tant que le routeur est en vie) → la session survit aux pannes de liens individuels.

#### Route Reflector et L3 VPN

Un RR dans un réseau L3 VPN reçoit les routes VPNv4 de tous les PE et les réfléchit à tous les autres PE. Il n'a pas besoin de connaître les VRF — il se contente de propager les attributs (RD, RT, label VPN, NEXT_HOP) sans les interpréter. Cela lui permet d'être un routeur simple (voire un serveur dédié) sans VRF configurées.

### Histoire

BGP-4 (RFC 1771, 1995) ne supportait que IPv4 unicast. L'extension multiprotocole MP-BGP (RFC 2283, 1998, mise à jour RFC 4760, 2007) ajoute les attributs `MP_REACH_NLRI` et `MP_UNREACH_NLRI` pour annoncer des NLRI (Network Layer Reachability Information) de n'importe quelle famille (AFI/SAFI). Cette extension a transformé BGP en protocole de signalisation universel : VPNv4, VPNv6, L2 VPN, EVPN (Ethernet VPN — non détaillé dans ce tutoriel), BGP-LS (Link State), Flowspec… Un seul protocole de contrôle pour tous les services réseau.

> **EVPN** (Ethernet VPN, RFC 7432) est une extension MP-BGP pour les VPN de couche 2 (L2 VPN). Il utilise la même infrastructure MP-BGP et MPLS mais avec des SAFI différents (SAFI 70). EVPN n'est pas détaillé dans ce tutoriel — il ferait l'objet d'un tutoriel séparé.

### Exercices — Section 5

**Q5.1** — Qu'est-ce que l'AFI et le SAFI ? Quelle combinaison AFI/SAFI est utilisée pour le L3 VPN MPLS ? Et pour IPv6 unicast ?

**Q5.2** — Pourquoi les sessions MP-BGP entre PE sont-elles des sessions **iBGP** et non eBGP ? Quelle règle BGP cela implique-t-il sur la propagation des routes (split-horizon iBGP) ?

**Q5.3** — Un réseau a 12 PE. (a) Sans Route Reflector, combien de sessions iBGP full-mesh sont nécessaires ? (b) Avec un RR central, combien ? (c) Le RR doit-il avoir des VRF configurées ?

**Q5.4** — Quel est le rôle exact de `update-source Loopback0` dans la configuration BGP d'un PE ? Que se passe-t-il si un lien physique tombe sur le PE et que cette commande n'est pas configurée ?

**Q5.5** — Un ingénieur configure MP-BGP entre PE1 et PE2 mais oublie `send-community extended`. Décrivez le symptôme observé et la commande de vérification à utiliser.

**Q5.6** — Quelle est la différence entre `address-family vpnv4` et `address-family ipv4 vrf CLIENT_A` dans la configuration BGP d'un PE ? À quoi sert chacune ?

**Q5.7** — Que contient exactement le champ `MP_REACH_NLRI` d'un UPDATE MP-BGP VPNv4 ? Listez tous les éléments transportés.

**Q5.8** — Le NEXT_HOP d'une route VPNv4 reçue via iBGP est `10.0.0.1` (loopback de PE1). Dans quelle table PE2 résout-il ce next-hop ? Via quel protocole ?

**Q5.9** — Vrai ou Faux : le Route Reflector doit obligatoirement être un PE (avoir des VRF). Justifiez.

**Q5.10** — Citez trois autres familles d'adresses (AFI/SAFI) que MP-BGP peut transporter, au-delà du VPNv4.


### Corrections — Section 5

**C5.1** — **AFI** = Address Family Identifier : identifie la famille d'adresses (1=IPv4, 2=IPv6, 25=L2 VPN…). **SAFI** = Subsequent AFI : sous-famille (1=unicast, 128=MPLS-labeled VPN). L3 VPN MPLS : **AFI=1, SAFI=128** (VPNv4). IPv6 unicast : **AFI=2, SAFI=1**.

**C5.2** — Les sessions PE-PE sont iBGP car tous les PE appartiennent au **même AS opérateur**. La règle iBGP implique le **split-horizon** : un routeur iBGP ne réannonce pas à un autre pair iBGP une route apprise via iBGP → nécessité d'un full-mesh ou d'un Route Reflector.

**C5.3** — (a) Full-mesh pour 12 PE : `12×11/2 = 66 sessions`. (b) Avec RR central : **12 sessions** (une par PE). (c) **Non**, le RR n'a pas besoin de VRF — il propage les attributs VPN (RD, RT, label VPN, NEXT_HOP) sans les interpréter ni les installer.

**C5.4** — `update-source Loopback0` force la session BGP à utiliser l'adresse loopback comme source TCP. Sans cette commande, la source est l'IP de l'interface physique de sortie. Si ce lien physique tombe, la session BGP tombe aussi — même si un chemin alternatif vers le voisin existe. Avec le loopback, la session survit aux pannes de liens individuels.

**C5.5** — Sans `send-community extended` : PE2 reçoit les routes VPNv4 de PE1 mais **sans Extended Communities** (RT absent). PE2 ne peut pas filtrer les routes dans ses VRF (aucun RT ne correspond). **Résultat** : `show bgp vpnv4 unicast vrf CLIENT_A` sur PE2 est vide. Vérification : `show bgp vpnv4 unicast all` sur PE2 → les routes apparaissent sans RT dans la colonne Extended Community.

**C5.6** — `address-family vpnv4` : configure la session **PE-PE** (iBGP) pour échanger les routes VPNv4 — canal de signalisation inter-PE pour toutes les VRF. `address-family ipv4 vrf CLIENT_A` : configure la session **PE-CE** pour un client spécifique, dans le contexte de la VRF CLIENT_A (eBGP ou OSPF avec le CE).

**C5.7** — `MP_REACH_NLRI` contient : (1) AFI/SAFI (1/128) ; (2) NEXT_HOP : loopback du PE source ; (3) NLRI : liste de tuples `{RD (64b) + préfixe IP + longueur + label VPN (3 octets)}`. C'est dans ce champ que le label VPN (inner label) voyage entre PE.

**C5.8** — PE2 résout le loopback de PE1 (`10.0.0.1`) dans sa **table de routage globale**, via l'**IGP** (IS-IS ou OSPF). L'IGP a appris ce loopback comme route d'infrastructure. LDP fournit ensuite le label de transport pour joindre ce loopback.

**C5.9** — **Faux**. Le Route Reflector peut être un routeur dédié sans VRF. Il reçoit les routes VPNv4 de ses clients iBGP (les PE) et les réfléchit à tous les autres clients, sans interpréter les RT ni installer les routes dans des VRF. Des RR dédiés (parfois des serveurs Linux avec un démon BGP) sont courants dans les grands réseaux pour des raisons de scalabilité.

**C5.10** — Exemples : (1) **IPv6 unicast** (AFI=2, SAFI=1) ; (2) **L2 VPN EVPN** (AFI=25, SAFI=70) — pour les VPN Ethernet ; (3) **BGP-LS** (AFI=16388, SAFI=71) — pour distribuer la topologie IGP à des contrôleurs SDN ; (4) **BGP Flowspec** (AFI=1, SAFI=133) — pour distribuer des règles de filtrage de trafic ; (5) **VPNv6** (AFI=2, SAFI=128).
---

## 6. Forwarding end-to-end : le double label

### Théorie

C'est le mécanisme central du L3 VPN MPLS : chaque paquet client traverse le cœur avec **deux labels empilés**.

```
Structure d'un paquet L3 VPN dans le cœur MPLS :

┌──────────────────────────────────────┐
│ En-tête L2 (EtherType 0x8847)       │
├──────────────────────────────────────┤
│ Label TRANSPORT (outer)   S=0       │  ← distribué par LDP ou SR
├──────────────────────────────────────┤
│ Label VPN        (inner)   S=1      │  ← distribué par MP-BGP
├──────────────────────────────────────┤
│ En-tête IP client                    │
├──────────────────────────────────────┤
│ Payload client                       │
└──────────────────────────────────────┘
```

- **Label de transport** (outer, S=0) : permet d'acheminer le paquet du PE ingress au PE egress à travers les P. Distribué par LDP (ou SR, tutoriel 2). Seul ce label est lu et swappé par les routeurs P — ils ne savent rien du client.
- **Label VPN** (inner, S=1) : permet au PE egress d'identifier la VRF (et donc le client et le CE de sortie). Distribué par MP-BGP dans le champ `MP_REACH_NLRI`. Ce label est alloué par le PE egress et annoncé aux autres PE lors de l'export de ses routes VRF.

#### Parcours complet d'un paquet

Topologie : `CE1 → PE1 → P1 → P2 → PE2 → CE2`, PHP sur P2.

```
CE1 → PE1 :
  Paquet IP : [src=10.1.10.1, dst=10.2.10.1, TTL=64]
  PE1 fait un lookup dans VRF CLIENT_A :
    next-hop = PE2, label VPN = 42
  PE1 résout PE2 via IGP → label transport LDP = 300
  PE1 PUSH les deux labels (VPN d'abord, transport par-dessus) :
  [T=300, S=0, TTL=63][VPN=42, S=1, TTL=64][IP src=10.1.10.1 dst=10.2.10.1]

PE1 → P1 :
  P1 regarde uniquement le label outer (T=300)
  LFIB : 300 → SWAP 400, iface eth1
  [T=400, S=0, TTL=62][VPN=42, S=1, TTL=64][IP ...]

P1 → P2 :
  P2 regarde T=400
  LFIB : 400 → POP (PHP, Implicit NULL annoncé par PE2)
  P2 retire le label de transport
  [VPN=42, S=1, TTL=64][IP src=10.1.10.1 dst=10.2.10.1]

P2 → PE2 :
  PE2 reçoit le paquet avec uniquement le label VPN
  LFIB : label 42 → VRF CLIENT_A, interface GigE0/1, action POP
  PE2 retire le label VPN, forward l'IP vers CE2

PE2 → CE2 :
  [IP src=10.1.10.1, dst=10.2.10.1, TTL=64]  ← IP brut, comme à l'entrée
```

#### Pourquoi les P ne connaissent-ils pas les clients ?

Les routeurs P ne lisent que le label outer (S=0). Le bit S=0 indique qu'il y a d'autres labels en dessous — ils n'ont pas besoin (et ne doivent pas) regarder le label VPN ni l'IP cliente. C'est la séparation fondamentale PE/P : seuls les PE maintiennent les VRF, les P restent simples et scalables.

#### Scalabilité

Dans un réseau avec 1000 clients et des millions de routes VPN :
- Les P n'ont que quelques milliers d'entrées LFIB (une par LSP de transport, indépendamment du nombre de clients).
- Seuls les PE ont les VRF et les routes VPN.
- C'est le **principe de séparation des rôles** qui rend L3 VPN scalable.

### Histoire

L'idée du double label est au cœur de la RFC 4364 (2006). Elle résout élégamment le problème de scalabilité des routeurs P grâce à la séparation entre label de transport (infrastructure opérateur) et label VPN (service client). Ce modèle a inspiré de nombreuses architectures de services MPLS ultérieures : L2 VPN (VPWS, VPLS), EVPN, segment routing avec VPN, etc.

### Exercices — Section 6

**Q6.1** — À quel moment exact PE1 impose-t-il les deux labels sur un paquet entrant depuis CE1 ? Dans quel ordre les labels sont-ils empilés (lequel est posé en premier sur le paquet IP) ?

**Q6.2** — Topologie `CE1 — PE1 — P1 — P2 — P3 — PE2 — CE2`. PHP est actif sur P3. Label transport : PE1 pousse T=100, P1 SWAP→200, P2 SWAP→300, P3 PHP. Label VPN : VPN=55. Décrivez l'état exact de la pile de labels à chaque étape : après PE1, après P1, après P2, après P3, entrée PE2.

**Q6.3** — Pourquoi les routeurs P n'ont-ils pas besoin de connaître le label VPN (inner) ? Quel bit du label outer leur indique de ne pas regarder plus loin dans la pile ?

**Q6.4** — D'où provient le label VPN ? Quel protocole le distribue et à quel moment ?

**Q6.5** — D'où provient le label de transport ? Quel protocole le distribue ?

**Q6.6** — Vrai ou Faux : si PHP est désactivé, PE2 doit effectuer deux lookups successifs pour forwarder le paquet. Expliquez lesquels.

**Q6.7** — Après PHP, PE2 reçoit `[VPN=42, S=1][IP src=10.1.10.1, dst=10.2.10.1]`. Décrivez les opérations exactes que PE2 effectue pour forwarder ce paquet vers CE2.

**Q6.8** — Un paquet IP entre sur PE1 avec TTL=128. Il traverse PE1, P1, P2 (PHP), PE2. Quel est le TTL IP à l'arrivée sur CE2, avec la propagation TTL MPLS activée par défaut ? Et avec `no mpls ip propagate-ttl` ?

**Q6.9** — Dans une topologie avec 3 routeurs P entre PE1 et PE2, combien d'opérations SWAP sont effectuées sur le label de transport ? Le label VPN est-il modifié pendant le transit ?

**Q6.10** — Un ingénieur fait un traceroute depuis CE1 vers CE2. Il voit CE1, PE1, puis directement PE2, puis CE2 (les P n'apparaissent pas). Quelle fonctionnalité MPLS explique l'invisibilité des P ? Comment la désactiver si besoin de déboguer ?


### Corrections — Section 6

**C6.1** — PE1 impose les labels **au moment où le paquet IP entre depuis le CE**, après lookup dans la VRF. Ordre d'empilement : d'abord le **label VPN** (inner, S=1) est posé sur l'IP, puis le **label transport** (outer, S=0) est posé par-dessus. Sur le fil, le label transport est le premier lu (outer).

**C6.2** —
```
Sortie CE1  : [IP src=10.1.0.1, dst=10.2.0.5, TTL=64]
Sortie PE1  : [T=100, S=0, TTL=63][VPN=55, S=1, TTL=64][IP TTL=64]
Sortie P1   : [T=200, S=0, TTL=62][VPN=55, S=1, TTL=64][IP TTL=64]  ← SWAP
Sortie P2   : [T=300, S=0, TTL=61][VPN=55, S=1, TTL=64][IP TTL=64]  ← SWAP
Sortie P3   : [VPN=55, S=1, TTL=64][IP TTL=64]                       ← PHP retire T
Entrée PE2  : [VPN=55, S=1, TTL=64][IP TTL=64]
Sortie PE2  : [IP src=10.1.0.1, dst=10.2.0.5, TTL=64]               ← POP VPN
Arrivée CE2 : [IP TTL=64]
```

**C6.3** — Les P ne lisent que le label outer. Le bit **S=0** sur le label outer indique qu'il y a d'autres labels en dessous — le P effectue SWAP sur ce seul label et ne lit pas ce qui suit. Structurellement, il est **impossible** pour un P de voir le label VPN ou l'IP client lors du traitement normal.

**C6.4** — Le label VPN est distribué par **MP-BGP** dans le champ `MP_REACH_NLRI` de l'UPDATE VPNv4. Il est alloué par le **PE egress** (PE2) et annoncé aux autres PE lors de l'export de ses routes VRF. Chaque route dans la VRF de PE2 a son propre label VPN.

**C6.5** — Le label de transport est distribué par **LDP** (ou SR-MPLS). LDP l'alloue en suivant le chemin IGP entre PE ingress et PE egress — c'est le label que le voisin P direct a annoncé au PE ingress pour atteindre le loopback du PE egress.

**C6.6** — **Vrai**. Sans PHP, PE2 reçoit `[T][VPN][IP]` et doit effectuer deux lookups : (1) lookup LFIB sur T → action POP, révèle le label VPN ; (2) lookup LFIB sur VPN → identification de la VRF et de l'interface de sortie. Avec PHP, PE2 ne reçoit que `[VPN][IP]` et n'effectue qu'un seul lookup LFIB.

**C6.7** — PE2 reçoit `[VPN=42, S=1][IP]`. (1) Lookup LFIB sur label 42 → VRF CLIENT_A, interface GigE0/1 vers CE2, action POP. (2) PE2 retire le label VPN (POP). (3) Forward l'IP vers CE2 sur GigE0/1.

**C6.8** — Avec propagation TTL activée (défaut) : PE1 décrémente TTL IP (64→63) lors du PUSH, P1 décrémente le TTL MPLS outer, P2 PHP (retire le label, TTL MPLS disparaît), PE2 décrémente TTL IP (63→62). CE2 reçoit TTL=**62**. Avec `no mpls ip propagate-ttl` : le TTL IP n'est pas copié dans le label MPLS et n'est décrémenté que par PE1 et PE2 → CE2 reçoit TTL=**62** aussi dans ce cas simple (PE1: 64→63, PE2: 63→62). Les P sont invisibles au traceroute.

**C6.9** — 3 routeurs P entre PE1 et PE2 : selon la topologie, **2 SWAP** (sur P1 et P2) + **1 PHP** (sur P3, le pénultième). Le label VPN n'est **jamais modifié** pendant le transit dans le cœur — il arrive intact sur PE2.

**C6.10** — **`no mpls ip propagate-ttl`** (ou la configuration MPLS TTL par défaut qui ne propage pas le TTL dans le cœur). Les P sont invisibles car le TTL MPLS est initialisé à 255 indépendamment du TTL IP — les P le décrément mais le TTL IP n'est jamais épuisé par eux. Pour voir les P dans le traceroute : activer `mpls ip propagate-ttl` (défaut sur certaines plateformes) ou utiliser `traceroute mpls ipv4 <loopback>` directement.
---

## 7. PE-CE : protocoles de routage côté client

### Théorie

Entre le PE et le CE (routeur du client), plusieurs protocoles peuvent être utilisés pour échanger les routes :

| Protocole | Avantages | Inconvénients |
|---|---|---|
| **eBGP** | Flexibilité, politique de routage précise, gestion des AS overlapping | Complexité de configuration, nécessite un AS côté client |
| **OSPF** | Familier pour les équipes réseau d'entreprise | Redistribution dans MP-BGP, problème du super-backbone |
| **IS-IS** | Rare en PE-CE | Très peu utilisé |
| **RIPv2** | Très simple | Peu scalable, peu recommandé |
| **Routes statiques** | Simplicité maximale | Pas de convergence automatique, black-hole en cas de panne CE |

**eBGP PE-CE** est le plus courant en production opérateur.

#### eBGP PE-CE

La session PE-CE est **eBGP** (AS différents : AS opérateur ≠ AS client). Le PE redistribue les routes apprises du CE dans la VRF, puis MP-BGP les propage aux autres PE.

```
CE1 (AS 65001) ─── eBGP ─── PE1 (AS 65000) ─── iBGP ─── PE2 (AS 65000) ─── eBGP ─── CE2 (AS 65001)
```

**Problème de la même AS côté client** : si CE1 et CE2 ont le même AS (65001), quand PE2 annonce à CE2 la route `10.1.0.0/24` (apprise de CE1 via PE1), l'AS-PATH contient `65000 65001`. CE2 voit son propre AS (65001) dans l'AS-PATH → loop prevention BGP → CE2 rejette la route.

Solutions :
- **`as-override`** sur PE2 : PE2 remplace l'AS source (65001) par son propre AS (65000) dans l'AS-PATH avant d'annoncer à CE2. CE2 ne voit plus son propre AS.
- **`allowas-in`** sur CE2 : CE2 accepte les routes contenant son propre AS dans l'AS-PATH (moins sécurisé).

#### OSPF PE-CE et le super-backbone

Quand OSPF est utilisé entre PE et CE, le PE redistribue les routes des autres sites (apprises via MP-BGP) dans le processus OSPF local. Ces routes redistribuées apparaissent comme des **routes externes OSPF (O E2)** côté CE, car elles traversent une "frontière" OSPF au niveau du PE.

Le problème du **"super-backbone MPLS"** survient quand les sites utilisent OSPF avec des area IDs identiques : OSPF traite le backbone MPLS comme un Area 0 virtuel, mais les mécanismes de redistribution peuvent créer des incohérences de type de route (intra-area vs inter-area). C'est une complexité opérationnelle supplémentaire — eBGP PE-CE évite ce problème.

**Note** : RSVP-TE (Traffic Engineering) et sa configuration détaillée en contexte PE-CE dépassent la portée de ce tutoriel.

#### Routes statiques PE-CE

La solution la plus simple : configurer des routes statiques sur le PE pour les réseaux du CE, puis les redistribuer dans MP-BGP.

```cisco
ip route vrf CLIENT_A 10.10.0.0 255.255.0.0 192.168.1.2
!
router bgp 65000
 address-family ipv4 vrf CLIENT_A
  redistribute static
 exit-address-family
```

Inconvénient majeur : si le CE tombe, la route statique reste dans la VRF → les paquets sont envoyés vers un CE inaccessible (black-hole). eBGP détecte la panne et retire automatiquement la route.

### Histoire

Historiquement, les opérateurs proposaient OSPF ou RIP en PE-CE pour simplifier la vie des clients (pas besoin de configurer BGP côté CE). Mais eBGP s'est imposé pour sa flexibilité : gestion simple des AS overlapping (as-override, allowas-in), politiques de routage précises (communities, prepend), et meilleure intégration avec le plan de contrôle VPN. Aujourd'hui, eBGP PE-CE est le standard de facto dans les réseaux L3 VPN opérateurs, OSPF PE-CE étant réservé aux cas où le client a une forte contrainte OSPF existante.

### Exercices — Section 7

**Q7.1** — Expliquez la différence entre la session **PE-CE** et la session **PE-PE** en termes de : (a) type BGP (iBGP/eBGP), (b) famille d'adresses transportée, (c) AS impliqués.

**Q7.2** — Un client utilise eBGP PE-CE avec le même AS 65001 sur tous ses CE (5 sites). Expliquez le problème qui survient et les deux solutions possibles (`as-override` et `allowas-in`). Laquelle préférez-vous et pourquoi ?

**Q7.3** — Un client utilise OSPF PE-CE. Les routes des sites distants apparaissent comme O E2 (OSPF External type 2) sur le CE. Pourquoi ? Quel problème cela pose-t-il si le client a des politiques de routage basées sur le type de route OSPF ?

**Q7.4** — Un client utilise des routes statiques PE-CE. Son CE tombe en panne. Que se passe-t-il avec les paquets destinés à ce site ? Qu'aurait fait eBGP PE-CE dans la même situation ?

**Q7.5** — Décrivez le cheminement complet d'une route `10.10.0.0/24` depuis CE1 (qui l'annonce via eBGP) jusqu'à CE2, en passant par PE1, MP-BGP, PE2.

**Q7.6** — Vrai ou Faux : le CE doit avoir une configuration MPLS pour participer au L3 VPN. Justifiez.

**Q7.7** — Quelle commande Cisco permet de vérifier les routes reçues d'un CE spécifique dans une VRF donnée ?

**Q7.8** — Quel est l'avantage d'eBGP PE-CE par rapport aux routes statiques en termes de **résilience** ? Donnez un exemple concret avec les timers BFD.

**Q7.9** — Un client avec eBGP PE-CE souhaite que l'ensemble de son trafic sortant préfère PE1 plutôt que PE2 (PE1 est mieux connecté). Quel attribut BGP peut-il utiliser depuis ses CE pour influencer ce choix ?

**Q7.10** — Pourquoi le next-hop d'une route VPNv4 reçue via iBGP (le loopback de PE1) est-il résolu dans la **table globale** du PE egress et non dans la VRF CLIENT_A ?


### Corrections — Section 7

**C7.1** — (a) PE-CE : **eBGP** (AS opérateur ≠ AS client). PE-PE : **iBGP** (même AS opérateur). (b) PE-CE : IPv4 unicast classique (préfixes du client). PE-PE : VPNv4 (AFI=1, SAFI=128, avec RD, RT, label VPN). (c) PE-CE : AS opérateur (65000) ↔ AS client (65001). PE-PE : AS opérateur (65000) ↔ AS opérateur (65000).

**C7.2** — Problème : PE2 annonce à CE2 la route `10.1.0.0/24` avec AS-PATH `65000 65001`. CE2 voit son propre AS (65001) → loop prevention BGP → rejette. **`as-override`** sur PE2 : PE2 remplace 65001 par 65000 dans l'AS-PATH → CE2 ne voit plus son AS → accepte. **`allowas-in`** sur CE2 : CE2 accepte même si son AS est dans l'AS-PATH. Préférence : **`as-override`** côté PE — plus sécurisé (pas de modification de la politique côté CE, et évite les boucles réelles entre sites du même client).

**C7.3** — Les routes des autres sites sont redistribuées depuis MP-BGP dans le processus OSPF du PE. Elles franchissent la "frontière" OSPF au niveau du PE → apparaissent comme O E2 (routes externes de type 2). Problème : si le client utilise des politiques basées sur le type OSPF (ex: préférer les routes intra-area), les routes inter-sites seront systématiquement moins préférées même si elles sont proches. Cela peut causer du routage suboptimal ou des comportements inattendus.

**C7.4** — Avec routes statiques : le PE conserve la route dans la VRF → les paquets sont envoyés vers le CE tombé → **black-hole** (paquets perdus silencieusement). eBGP PE-CE : la session eBGP expire (Hold timer, 90-180s par défaut, ou < 1s avec BFD) → le PE retire la route de la VRF → envoie un WITHDRAW MP-BGP → les autres PE retirent aussi cette destination → le trafic ne part plus vers ce site.

**C7.5** — (1) CE1 annonce `10.10.0.0/24` à PE1 via eBGP (AS-PATH: 65001). (2) PE1 importe dans VRF CLIENT_A, ajoute RD+label VPN, annonce à PE2 via MP-BGP iBGP (AS-PATH: 65001, NEXT_HOP=loopback PE1, RT=65000:100). (3) PE2 vérifie le RT, importe dans VRF CLIENT_A locale. (4) PE2 annonce à CE2 via eBGP (AS-PATH: 65000 65001, ou 65000 si as-override).

**C7.6** — **Faux**. Le CE ne fait aucun MPLS. Il échange des routes IP avec le PE via eBGP/OSPF/statique, exactement comme avec n'importe quel routeur voisin classique. MPLS est entièrement transparent pour le CE.

**C7.7** — `show bgp vpnv4 unicast vrf CLIENT_A neighbors <IP-CE> routes` — routes reçues du CE. Ou `show ip route vrf CLIENT_A` pour voir toutes les routes installées dans la VRF.

**C7.8** — eBGP PE-CE : quand le CE tombe, BFD détecte la panne en 3×50ms = 150ms (ou < 1s selon la config). La session eBGP tombe → le PE retire immédiatement les routes du CE de la VRF → WITHDRAW MP-BGP → les autres PE convergent. Avec routes statiques, la panne n'est jamais détectée automatiquement → black-hole permanent jusqu'à intervention humaine.

**C7.9** — Le CE peut utiliser l'attribut **MED (Multi-Exit Discriminator)** pour signaler sa préférence d'entrée (valeur MED basse = chemin préféré). Le PE prend en compte le MED lors de la sélection best-path eBGP. Alternativement, côté opérateur, le **LOCAL_PREF** peut être configuré sur PE1 pour qu'il préfère PE1 comme point de sortie pour ce client.

**C7.10** — Le loopback du PE distant est une **adresse d'infrastructure opérateur** dans la table globale (peuplée par l'IGP). La VRF CLIENT_A ne contient que les routes du client — elle ne "voit" pas les loopbacks des PE. Si BGP cherchait dans la VRF, il ne trouverait rien. La résolution dans la table globale est architecturalement nécessaire pour accéder à LDP (qui connaît le label de transport vers ce loopback).
---

## 8. Inter-AS L3 VPN (Options A, B, C)

### Théorie

Quand un client veut un VPN s'étendant sur **deux AS opérateurs différents** (ex: un opérateur national et un opérateur international partenaire), trois options existent (RFC 4364, Section 10) :

#### Option A — Back-to-Back VRF (la plus simple)

```
PE1 (AS1) ──── ASBR1 ──── ASBR2 ──── PE2 (AS2)
               [VRF]      [VRF]
               par        par
               client     client
```

Les deux ASBR (AS Boundary Routers) ont des VRF pour chaque client. Ils échangent les routes via des sessions **eBGP IPv4 classiques** (ou des routes statiques) sur des interfaces dédiées par VRF (une sous-interface par client).

- **Avantage** : simple, chaque AS est complètement indépendant, pas besoin de modifier le plan de contrôle VPN.
- **Inconvénient** : ne scale pas. Pour N clients, il faut N interfaces (ou sous-interfaces) entre les ASBR, plus N sessions eBGP. Pour 50 clients : 50 interfaces de chaque côté.

#### Option B — eBGP VPNv4 entre ASBR

```
PE1 (AS1) ──── ASBR1 ════VPNv4 eBGP════ ASBR2 ──── PE2 (AS2)
```

Les ASBR échangent directement des routes **VPNv4** via une session eBGP VPNv4 (avec AFI=1, SAFI=128). Les ASBR maintiennent toutes les routes VPN de tous les clients dans leur table BGP VPNv4.

- **Avantage** : une seule session eBGP entre les ASBR, indépendamment du nombre de clients.
- **Inconvénient** : les ASBR doivent maintenir l'intégralité des routes VPN → plus lourds. Le NEXT_HOP doit être réécrit par l'ASBR (`next-hop-self`), et les labels de transport doivent être réalloués entre les deux AS.

#### Option C — MP-eBGP multihop PE-PE

```
PE1 (AS1) ═══════════ MP-eBGP multihop ═══════════ PE2 (AS2)
         (ASBR1 redistribue les loopbacks PE de AS1 dans AS2)
```

Les PE établissent directement des sessions **MP-eBGP multihop** entre eux (eBGP car AS différents, multihop car non directement adjacents). Les ASBR se contentent de redistribuer les loopbacks des PE de l'autre AS dans l'IGP local.

- **Avantage** : les ASBR sont très légers (pas de routes VPN). C'est l'option la plus scalable.
- **Inconvénient** : la plus complexe à opérer. Les sessions multihop entre PE nécessitent `ebgp-multihop` avec un TTL suffisant, et la redistribution des loopbacks PE entre AS doit être gérée soigneusement. Le troubleshooting est plus difficile (le problème peut être dans l'IGP, LDP ou BGP de l'un ou l'autre AS).

#### Tableau comparatif

| Critère | Option A | Option B | Option C |
|---|---|---|---|
| Complexité config | Faible | Moyenne | Élevée |
| Scalabilité clients | Très limitée | Bonne | Excellente |
| Charge ASBR | Élevée (VRF/interface par client) | Moyenne (table VPNv4) | Minimale (loopbacks uniquement) |
| Isolation AS | Totale | Partielle (routes VPN aux ASBR) | Partielle |
| Cas d'usage typique | < 10 clients, migration | Quelques dizaines de clients | Centaines de clients |

### Histoire

La problématique inter-AS est née du besoin des grandes entreprises multinationales d'obtenir des VPN de bout en bout passant par plusieurs opérateurs régionaux. Avant les options A/B/C de la RFC 4364, la solution était de superposer des tunnels GRE/IPsec entre les domaines MPLS — solution fragile et non scalable. Les trois options formalisent des approches interopérables avec des compromis explicites. En pratique, l'Option A domine les petits déploiements, l'Option C est la référence pour les grands opérateurs internationaux.

### Exercices — Section 8

**Q8.1** — Décrivez le principe de chacune des trois options inter-AS en une phrase.

**Q8.2** — Un opérateur a 60 clients VPN à faire passer entre deux AS. Il envisage l'Option A. Combien d'interfaces (ou sous-interfaces) faut-il au minimum entre les deux ASBR ? Quel est l'impact opérationnel si un nouveau client est ajouté ?

**Q8.3** — Avec l'Option B, quel attribut BGP doit être réécrit par l'ASBR lors de la propagation des routes VPNv4 entre les deux AS ? Pourquoi ?

**Q8.4** — Avec l'Option C, comment PE1 (AS1) connaît-il l'adresse loopback de PE2 (AS2), nécessaire pour établir la session MP-eBGP multihop ?

**Q8.5** — Comparez les trois options sur le critère de la charge sur les ASBR.

**Q8.6** — Vrai ou Faux : avec l'Option A, les ASBR voient les routes VPN sous forme VPNv4 (avec RD). Justifiez.

**Q8.7** — Pourquoi l'Option C est-elle la plus scalable mais aussi la plus difficile à déboguer ?

**Q8.8** — Dans l'Option B, si un ASBR tombe, quel est l'impact sur tous les VPN inter-AS ? Comment mitiger ce risque ?

**Q8.9** — Un client a 5 VPN inter-AS entre deux opérateurs partenaires. Les deux opérateurs ont des équipes peu expérimentées en MPLS-VPN. Quelle option recommandez-vous et pourquoi ?

**Q8.10** — Dans l'Option C, pourquoi le paramètre `ebgp-multihop` est-il nécessaire ? Quelle valeur TTL doit-il autoriser au minimum ?


### Corrections — Section 8

**C8.1** — **Option A** : chaque ASBR a des VRF par client et échange les routes via eBGP IPv4 classique sur des interfaces dédiées par VRF. **Option B** : les ASBR échangent directement des routes VPNv4 via une session eBGP VPNv4. **Option C** : les PE de chaque AS établissent des sessions MP-eBGP multihop directement entre eux, les ASBR redistribuant uniquement les loopbacks PE.

**C8.2** — Option A, 60 clients : **60 sous-interfaces** par ASBR (une par VRF cliente), soit 60 de chaque côté. Ajouter un client = créer une nouvelle sous-interface, une VRF, et une session eBGP sur chaque ASBR → **impact opérationnel élevé** et risque d'erreurs croissant avec le nombre de clients.

**C8.3** — Le **NEXT_HOP** doit être réécrit par l'ASBR. Les routes VPNv4 reçues de PE1 (AS1) ont comme NEXT_HOP le loopback de PE1 — adresse non joignable depuis AS2. ASBR2 doit changer le NEXT_HOP avec sa propre adresse (`next-hop-self`) avant de propager dans AS2. Les labels de transport doivent aussi être réalloués entre les deux AS.

**C8.4** — Avec Option C, ASBR1 redistribue les loopbacks des PE de AS1 dans l'IGP de AS2 (via redistribution ou BGP entre ASBR). PE2 apprend les loopbacks de PE1 via l'IGP de son AS. LDP construit des LSP vers ces loopbacks. La session MP-eBGP multihop peut alors s'établir entre PE1 (AS1) et PE2 (AS2) en utilisant ces loopbacks comme endpoints.

**C8.5** — **Option A** : très lourde (VRF + interface + session eBGP par client). **Option B** : moyenne (une session eBGP VPNv4, mais toutes les routes VPN dans la table ASBR). **Option C** : minimale (les ASBR ne maintiennent que quelques routes loopback PE, pas de routes VPN).

**C8.6** — **Faux**. Avec Option A, les ASBR voient les routes sous forme **IPv4 classique** (sans RD). Le RD est retiré lors de l'export depuis la VRF de ASBR1, et les routes sont annoncées comme des préfixes IPv4 ordinaires sur l'interface dédiée par client. ASBR2 les réimporte dans sa VRF correspondante et reconstruit le RD.

**C8.7** — Plus scalable : les ASBR n'ont pas d'état client → peuvent être de simples routeurs P. Les PE gèrent directement leurs routes VPN. Plus difficile à déboguer : le problème peut être dans l'IGP d'un AS, dans la redistribution des loopbacks entre AS, dans la session eBGP multihop, ou dans les labels de transport inter-AS. La chaîne de dépendances est longue et traverse deux domaines administratifs distincts.

**C8.8** — Si l'ASBR unique tombe en Option B : **toutes** les routes VPNv4 inter-AS sont perdues → panne totale pour tous les clients inter-AS. Mitigation : **deux ASBR en parallèle** (ASBR1a et ASBR1b), chacun avec une session eBGP VPNv4 vers l'AS partenaire. Les routes VPNv4 sont annoncées par les deux ASBR → redondance.

**C8.9** — 5 clients, équipes peu expérimentées : **Option A**. Justification : simple à comprendre et à configurer (interfaces dédiées, eBGP IPv4 classique), facile à déboguer, aucune connaissance VPNv4 inter-AS requise. Les 5 clients = 5 sous-interfaces par ASBR — gérable manuellement. L'Option C serait trop complexe pour des équipes peu expérimentées.

**C8.10** — `ebgp-multihop` est nécessaire car par défaut, eBGP n'accepte que les voisins directement adjacents (TTL=1 dans le paquet BGP). PE1 et PE2 sont séparés par plusieurs sauts (ASBR1, ASBR2, routeurs P) → le TTL serait épuisé. La valeur doit être ≥ **nombre de sauts entre PE1 et PE2** (ex: `ebgp-multihop 5` pour PE1→ASBR1→ASBR2→PE2 = 3 sauts, valeur 5 avec marge).
---

## 9. Convergence spécifique L3 VPN

### Théorie

La convergence L3 VPN a des caractéristiques spécifiques qui la distinguent de la convergence MPLS générale (vue dans le tutoriel 2).

#### Séquence de convergence

```
Panne d'un lien dans le cœur MPLS :

1. Détection (BFD ou timer IGP)
        ↓
2. IGP recalcule SPF → nouvelles routes dans la FIB
        ↓
3. LDP/SR reconverge → nouveaux labels de transport sur le nouveau chemin
        ↓
4. MP-BGP n'a PAS besoin de reconverger
   (les sessions iBGP sont établies entre loopbacks PE, toujours joignables)
        ↓
5. Trafic VPN reprend sur le nouveau chemin de transport
   (les labels VPN sont inchangés)
```

**Observation clé** : MP-BGP ne reconverge pas lors d'une panne dans le cœur. Les sessions iBGP PE-PE sont établies sur les loopbacks des PE, qui restent joignables via l'IGP reconvergé. Les routes VPN et les labels VPN ne changent pas — seul le chemin de transport (labels LDP) change.

#### Stabilité des labels VPN

Les labels VPN sont alloués par les PE egress et annoncés via MP-BGP. Ils ne changent que si :
- Le PE egress redémarre
- La VRF est supprimée et recréée
- Un changement de configuration force une réallocation

En pratique, les labels VPN sont **très stables** — ils survivent aux pannes du cœur, aux reconvergences IGP, et aux changements de chemin MPLS.

#### Convergence PE-CE

Si le lien **PE-CE** tombe (panne côté client) :
1. PE détecte la panne (BFD sur la session eBGP PE-CE, ou expiration du timer BGP Hold)
2. PE retire les routes du CE de la VRF locale
3. PE envoie un UPDATE MP-BGP `WITHDRAW` aux autres PE (via le RR)
4. Les autres PE retirent ces routes de leurs VRF
5. Les CE distants ne reçoivent plus ces routes → convergence complète

Avec BFD sur la session eBGP PE-CE, la détection peut se faire en < 1 seconde au lieu de 90-180 secondes avec les timers BGP Hold natifs.

#### MPLS FRR et L3 VPN

Le **MPLS FRR** (Fast ReRoute, tutoriel 2) s'applique au label de transport. En cas de panne, le PLR bascule le trafic sur le LSP de secours en < 50ms. Les labels VPN ne sont pas affectés — le paquet avec son label VPN intact est redirigé sur un nouveau chemin de transport.

```
Avant panne :  [T=300][VPN=42][IP]  via chemin PE1→P1→P2→PE2
Après FRR   :  [T=500][VPN=42][IP]  via chemin PE1→P1→P3→P4→PE2
                       ↑
               label VPN inchangé
```

#### BGP PIC (Prefix Independent Convergence)

**BGP PIC** est un mécanisme qui pré-calcule des chemins de secours BGP pour les routes VPN. Si le PE egress préféré devient indisponible, le trafic est redirigé sur un PE egress alternatif (si le client a deux PE de sortie) sans attendre la reconvergence BGP complète.

> **BGP PIC** n'est pas détaillé dans ce tutoriel — il fait partie des mécanismes avancés de haute disponibilité L3 VPN. Nous le mentionnons comme mécanisme existant pour la convergence côté PE egress.

### Histoire

La convergence des réseaux VPN MPLS a été un sujet de recherche intensif dans les années 2000-2010. Les opérateurs réalisaient des SLA de convergence < 50ms pour leurs clients VPN critiques (finance, santé), comparables aux réseaux téléphoniques SDH/SONET. L'introduction de BFD (RFC 5880, 2010), de MPLS FRR (RFC 4090, 2005) et de BGP PIC a permis d'atteindre ces objectifs. Aujourd'hui, TI-LFA (Segment Routing) offre une couverture FRR quasi-totale sans la complexité de RSVP-TE.

### Exercices — Section 9

**Q9.1** — Listez dans l'ordre les 5 étapes de convergence lors d'une panne de lien dans le cœur MPLS, dans le contexte d'un L3 VPN.

**Q9.2** — Pourquoi MP-BGP n'a-t-il généralement pas besoin de reconverger lors d'une panne dans le cœur MPLS ?

**Q9.3** — Vrai ou Faux : lors d'une reconvergence MPLS suite à une panne dans le cœur, les labels VPN (inner labels) changent. Justifiez.

**Q9.4** — Quelle est la différence de comportement lors d'une panne du **lien PE-CE** (côté client) par rapport à une panne dans le **cœur MPLS** (entre P) en termes de protocoles impliqués dans la convergence ?

**Q9.5** — Comment BFD accélère-t-il la détection d'une panne sur la session eBGP PE-CE ? Donnez des ordres de grandeur de délai (avec et sans BFD).

**Q9.6** — Expliquez comment le MPLS FRR (Fast ReRoute) s'applique au trafic L3 VPN. Les labels VPN sont-ils affectés par le basculement FRR ?

**Q9.7** — Un opérateur a un SLA de convergence de 50ms pour ses clients VPN. Quelle combinaison de mécanismes (BFD, FRR/TI-LFA, BGP PIC) recommandez-vous pour atteindre cet objectif ?

**Q9.8** — Un lien entre P1 et P2 tombe. Listez les événements dans l'ordre, de la détection jusqu'au moment où le trafic VPN reprend, en précisant quels protocoles sont impliqués à chaque étape.

**Q9.9** — Dans quelle situation les labels VPN **changent-ils** ? Donnez deux exemples.

**Q9.10** — Vrai ou Faux : si le Route Reflector tombe, les sessions MP-BGP PE-PE existantes sont immédiatement coupées et toutes les routes VPN sont perdues. Justifiez.


### Corrections — Section 9

**C9.1** — (1) Détection de panne (BFD ou timer IGP) ; (2) IGP recalcule SPF → nouvelles routes dans la FIB ; (3) LDP/SR reconverge → nouveaux labels de transport sur le nouveau chemin ; (4) MP-BGP n'a pas besoin de reconverger ; (5) trafic VPN reprend sur le nouveau LSP avec les mêmes labels VPN.

**C9.2** — MP-BGP n'a pas besoin de reconverger car les sessions iBGP PE-PE sont établies entre les **loopbacks des PE**, qui restent joignables via d'autres chemins dans le cœur (l'IGP a reconvergé). Les sessions iBGP restent up. Les routes VPNv4 et les labels VPN n'ont pas changé — seul le chemin de transport (LSP LDP) a changé.

**C9.3** — **Faux**. Les labels VPN sont alloués par les PE egress et annoncés via MP-BGP. Une panne dans le cœur MPLS ne touche pas les PE egress → leurs labels VPN restent inchangés. Seuls les **labels de transport** (outer, LDP) changent lors de la reconvergence sur un nouveau chemin.

**C9.4** — Panne **cœur MPLS** : IGP + LDP reconvergent (plan de transport). MP-BGP n'est pas impliqué. Panne **PE-CE** : c'est MP-BGP qui est impliqué — la session eBGP PE-CE tombe → le PE envoie un WITHDRAW MP-BGP aux autres PE → les VRF des autres PE retirent les routes de ce site. L'IGP et LDP du cœur ne sont pas impliqués.

**C9.5** — Sans BFD : la session eBGP expire après **Hold timer** (défaut 90s ou 180s selon config). Avec BFD sur la session eBGP PE-CE : détection en **3 × intervalle BFD** (typiquement 3 × 50ms = 150ms, ou 3 × 300ms = 900ms selon la config). Gain : de 90-180s à moins d'1 seconde.

**C9.6** — MPLS FRR s'applique au **label de transport** (outer). En cas de panne, le PLR bascule le trafic sur le LSP de secours : le paquet `[T_nouveau][VPN=42][IP]` est forwardé sur un nouveau chemin. Le label VPN (42) est **inchangé** — il arrive intact sur PE2 qui l'utilise pour identifier la VRF et le CE de sortie.

**C9.7** — Pour SLA < 50ms : (1) **BFD** sur les liens du cœur (détection < 150ms de panne physique) ; (2) **TI-LFA** (avec SR-MPLS, couverture ~100%) ou **MPLS FRR RSVP-TE** pour le basculement en < 50ms ; (3) **BFD sur les sessions eBGP PE-CE** pour une convergence rapide en cas de panne côté client. BGP PIC optionnel pour la convergence PE egress.

**C9.8** — (1) Panne du lien P1-P2. (2) **BFD** détecte la panne en 150ms (ou timers IGP en 30-40s). (3) **IS-IS/OSPF** inonde un LSA/LSP de mise à jour, recalcule SPF sur tous les routeurs. (4) **LDP** utilise les labels alternatifs déjà connus (Liberal Label Retention) pour mettre à jour la LFIB. (5) Les **paquets VPN** (avec leurs labels VPN inchangés) reprennent sur le nouveau chemin de transport. **MP-BGP** n'est pas impliqué.

**C9.9** — Les labels VPN changent quand : (1) le **PE egress redémarre** (les labels sont réalloués après le redémarrage, les autres PE reçoivent de nouveaux labels via MP-BGP) ; (2) la **VRF est supprimée et recréée** sur le PE egress (la commande `no ip vrf` puis `ip vrf` réinitialise les labels).

**C9.10** — **Faux** (partiellement). Si le RR tombe : les sessions iBGP PE-RR tombent, mais les **sessions TCP BGP existantes entre PE et RR** cessent d'exister. Cependant, les **routes VPNv4 déjà installées dans les tables BGP des PE** restent présentes pendant un certain temps (BGP Graceful Restart peut prolonger cela). En pratique, sans RR, les PE ne peuvent plus s'échanger de nouvelles routes VPN ni retirer des routes existantes. Les routes en place continuent à fonctionner jusqu'à expiration des timers BGP. Solution : déployer **deux RR redondants**.
---

## 10. Exercices de compréhension générale

**QG.1 — Diagnostic : table VPN vide**

`show bgp vpnv4 unicast vrf CLIENT_A` sur PE2 est vide, alors que PE1 a bien les routes dans sa VRF CLIENT_A. Citez **cinq causes possibles**, en couvrant les différentes couches (BGP, VRF, IGP/LDP).

---

**QG.2 — Tracé de paquet complet**

Topologie :
```
CE1 (AS 65001) — PE1 — P1 — P2 — PE2 — CE2 (AS 65001)
```
- VRF CLIENT_A sur PE1 et PE2
- PE1 PUSH : T=100 (transport), VPN=42 (VPN)
- P1 SWAP T : 100→200
- P2 PHP (retire T)
- CE1 envoie : `[IP src=10.1.0.1, dst=10.2.0.5, TTL=64]`

Décrivez l'état exact du paquet (tous les headers, TTL) à chaque étape : sortie CE1, sortie PE1, sortie P1, sortie P2, sortie PE2, arrivée CE2.

---

**QG.3 — Conception d'un schéma RT**

Une entreprise a trois catégories de sites :
- **HQ** (1 site) : doit communiquer avec tout le monde.
- **Agences** (6 sites) : communiquent entre elles et avec HQ.
- **Partenaires** (3 sites) : communiquent uniquement avec HQ, jamais entre eux ni avec les agences.

Proposez un schéma RT complet (export/import pour chaque catégorie) garantissant ces contraintes. Vérifiez votre solution pour chaque paire de catégories.

---

**QG.4 — Analyse de configuration**

Un ingénieur présente la configuration suivante sur un PE :

```cisco
ip vrf SERVICE_X
 rd 65000:50
 route-target export 65000:50
 route-target import 65000:60

ip vrf SERVICE_Y
 rd 65000:60
 route-target export 65000:60
 route-target import 65000:50
```

(a) Les routes de SERVICE_X sont-elles importées dans SERVICE_Y ? Et inversement ?
(b) La topologie est-elle symétrique ?
(c) Donnez un cas d'usage réel correspondant à ce schéma.
(d) Quel est le problème si SERVICE_X et SERVICE_Y ont toutes deux le réseau `192.168.0.0/24` ?

---

**QG.5 — Rôles et tables**

Pour chaque table ci-dessous, indiquez sur quel(s) équipement(s) elle existe, ce qu'elle contient, et quel(s) protocole(s) la peuple(nt) :

| Table | Équipement(s) | Contenu | Peuplée par |
|---|---|---|---|
| Table de routage globale (RIB) | ? | ? | ? |
| Table de routage VRF | ? | ? | ? |
| LFIB | ? | ? | ? |
| LIB | ? | ? | ? |
| Table BGP VPNv4 | ? | ? | ? |

---

**QG.6 — Scalabilité**

Un opérateur a 2000 clients, chacun avec 5 sites en moyenne. Expliquez pourquoi les routeurs P du cœur peuvent rester "légers" (peu d'état à maintenir) alors que le nombre de routes VPN est potentiellement immense. Quel mécanisme central rend cela possible ?

---

**QG.7 — Mise en service d'un nouveau PE**

Un opérateur ajoute un nouveau PE4 pour un client existant (VRF CLIENT_A déjà présente sur PE1, PE2, PE3). Listez dans l'ordre toutes les étapes de configuration et de vérification nécessaires, depuis la connexion physique jusqu'au ping client fonctionnel.

---

**QG.8 — Analyse de panne**

Un client signale que son site C (sur PE_C) ne peut plus joindre son site A (sur PE_A), mais peut encore joindre son site B (sur PE_B). Proposez une méthodologie de diagnostic structurée en distinguant le plan de contrôle et le plan de données.

---

**QG.9 — Inter-AS et options**

Un opérateur français (AS 1000) et un opérateur allemand (AS 2000) veulent offrir un VPN inter-AS à 150 clients communs. L'opérateur français a une équipe expérimentée, l'opérateur allemand moins. Quelle option inter-AS recommandez-vous et pourquoi ? Quels seraient les risques de choisir l'Option A dans ce cas ?

---

**QG.10 — SR-MPLS et L3 VPN**

Un opérateur migre son backbone de LDP vers SR-MPLS. Listez les composants du L3 VPN qui **changent** et ceux qui **restent identiques**. Pourquoi peut-on dire que SR-MPLS est "transparent" pour le plan de contrôle VPN ?

---

### Corrections — Exercices généraux

**CG.1** — Cinq causes possibles :
1. **`address-family vpnv4` non activée** : `neighbor X activate` manquant → la famille VPNv4 n'est pas négociée.
2. **`send-community extended` absent** sur la session PE1→PE2 (ou PE1→RR) : les RT ne sont pas transmis → PE2 ne peut pas importer les routes.
3. **RT mismatch** : RT exporté par PE1 ≠ RT importé dans VRF CLIENT_A de PE2.
4. **Session BGP down** : loopback de PE1 non joignable depuis PE2 (IGP ou LDP non convergé, ou `update-source Loopback0` absent).
5. **VRF CLIENT_A non configurée sur PE2** (ou RD absent) : nulle part où importer les routes.

Causes supplémentaires : interface CE-facing non assignée à la VRF, redistribution des routes CE non configurée, RR ne réfléchit pas les routes (erreur de configuration client/serveur BGP).

**CG.2** —
```
Sortie CE1  : [IP src=10.1.0.1, dst=10.2.0.5, TTL=64]

Sortie PE1  : [T=100, TC=0, S=0, TTL=63]
              [VPN=42, TC=0, S=1, TTL=64]
              [IP src=10.1.0.1, dst=10.2.0.5, TTL=64]
              (PE1 PUSH VPN puis T)

Sortie P1   : [T=200, TC=0, S=0, TTL=62]   ← SWAP 100→200, TTL outer décrementé
              [VPN=42, TC=0, S=1, TTL=64]   ← inchangé
              [IP TTL=64]

Sortie P2   : [VPN=42, TC=0, S=1, TTL=64]  ← PHP : P2 retire label T
              [IP src=10.1.0.1, dst=10.2.0.5, TTL=64]

Sortie PE2  : [IP src=10.1.0.1, dst=10.2.0.5, TTL=64]
              (PE2 POP label VPN=42, lookup VRF CLIENT_A → forward vers CE2)

Arrivée CE2 : [IP src=10.1.0.1, dst=10.2.0.5, TTL=64]
```

**CG.3** — Schéma RT :
```
HQ          : export RT=200:1, import RT=200:1, RT=200:2, RT=200:3
Agences     : export RT=200:2, import RT=200:1, RT=200:2
Partenaires : export RT=200:3, import RT=200:1
```
Vérification :
- HQ ↔ Agences : HQ importe 200:2 ✅, Agences importent 200:1 ✅
- HQ ↔ Partenaires : HQ importe 200:3 ✅, Partenaires importent 200:1 ✅
- Agences ↔ Agences : Agences importent 200:2 → se voient entre elles ✅
- Partenaires ↔ Agences : Partenaires importent 200:1 seulement (pas 200:2) ✅ isolation
- Partenaires ↔ Partenaires : Partenaires importent 200:1 (pas 200:3) ✅ isolation

**CG.4** —
a) SERVICE_X exporte RT=65000:50. SERVICE_Y importe RT=65000:60. `50≠60` → SERVICE_Y n'importe **pas** les routes de X. SERVICE_Y exporte RT=65000:60. SERVICE_X importe RT=65000:60. → SERVICE_X **importe** les routes de Y. Asymétrique.
b) **Asymétrique** : X reçoit les routes de Y, mais Y ne reçoit pas les routes de X.
c) Cas d'usage : **serveurs → clients**. Y = VRF de serveurs (DNS, NTP, services internes). X = VRF de postes de travail ou clients. Les postes (X) peuvent joindre les serveurs (Y), mais les serveurs ne connaissent pas les routes des postes.
d) Problème de RD : si les deux VRF ont des réseau identiques `192.168.0.0/24` ET que des routes sont importées, il y aurait un conflit dans la VRF importatrice. Ici, seule X importe les routes de Y → si Y annonce `192.168.0.0/24`, elle est importée dans X. Si X a aussi `192.168.0.0/24` localement, la route locale (connectée) prime généralement sur la route importée — potentiel problème de routage asymétrique.

**CG.5** —

| Table | Équipement(s) | Contenu | Peuplée par |
|---|---|---|---|
| RIB (table globale) | PE, P | Routes d'infrastructure : loopbacks PE/P, liens entre routeurs | IGP (OSPF/IS-IS) |
| Table de routage VRF | PE uniquement | Routes des clients : réseaux CE locaux + routes importées d'autres sites | Protocole PE-CE (eBGP/OSPF/statique) + MP-BGP (import RT) |
| LFIB | PE, P | Labels entrants → labels sortants + interface + action | LDP (ou SR) |
| LIB | PE, P | Tous les labels reçus de tous les voisins LDP (Liberal Retention) | LDP |
| Table BGP VPNv4 | PE (+ RR) | Routes VPNv4 (RD+préfixe) avec RT, label VPN, NEXT_HOP | MP-BGP iBGP entre PE |

**CG.6** — Les routeurs P ne maintiennent que des entrées LFIB de **labels de transport** — une entrée par LSP vers un loopback PE. Même avec 2000 clients et des millions de routes VPN, le nombre de loopbacks PE est de l'ordre de quelques dizaines ou centaines → la LFIB des P reste petite. Le mécanisme central : le **double label**. Le label de transport (outer) donne aux P toute l'information nécessaire (vers quel PE egress forwarder). Le label VPN (inner), opaque pour les P, transporte l'identité du client. Séparation PE/P = scalabilité.

**CG.7** — Étapes dans l'ordre :
1. Connexion physique PE4 ↔ CE_nouveau + lien PE4 ↔ P
2. Configuration IGP (IS-IS/OSPF) sur les liens P-PE4 + loopback PE4 → `show isis neighbors`
3. Activation LDP sur les liens → `show mpls ldp neighbor`, `show mpls forwarding-table`
4. Configuration MP-BGP : session PE4 ↔ RR, `address-family vpnv4`, `send-community extended`, `update-source Loopback0` → `show bgp vpnv4 unicast all summary`
5. Création VRF CLIENT_A sur PE4 (RD unique, RT identiques aux autres PE)
6. Assignation de l'interface CE-facing à VRF CLIENT_A, configuration IP
7. Configuration protocole PE-CE (eBGP ou autre) → `show ip route vrf CLIENT_A`
8. Vérification routes MP-BGP reçues : `show bgp vpnv4 unicast vrf CLIENT_A`
9. Test ping end-to-end : CE_nouveau ↔ CE1 / CE2

**CG.8** — Méthodologie structurée :

**Étape 1 — Isoler** : le problème est spécifique à la relation PE_C ↔ PE_A. PE_B fonctionne → pas un problème global de PE_C ni de RR.

**Étape 2 — Plan de contrôle sur PE_C** :
- `show bgp vpnv4 unicast vrf CLIENT_A` : PE_C reçoit-il les routes du site A (from PE_A) ?
- Si **non** : vérifier session BGP PE_C ↔ RR, RT, `send-community extended`.
- Si **oui** : les routes sont dans la table BGP mais pas installées dans la VRF ?

**Étape 3 — Plan de contrôle sur PE_A** :
- `show bgp vpnv4 unicast vrf CLIENT_A` sur PE_A : PE_A a-t-il les routes du site C ?
- `show ip route vrf CLIENT_A` sur PE_A : routes CE_A correctement redistribuées ?

**Étape 4 — Plan de données** :
- `show mpls forwarding-table` sur PE_C : label VPN de PE_A présent ?
- `traceroute mpls ipv4 <loopback PE_A>/32` depuis PE_C : LSP fonctionnel ?
- Vérifier LDP entre PE_C et PE_A via les P intermédiaires.

**Étape 5 — Vérification CE** :
- `show ip route` sur CE_A : la route vers le site C est-elle présente ?
- Session PE-CE sur PE_A pour la VRF CLIENT_A.

**CG.9** — Recommandation : **Option C** pour 150 clients. Justification : l'Option A nécessiterait 150 sous-interfaces + sessions eBGP par ASBR → impraticable. L'Option B est un bon compromis mais les ASBR maintiennent 150 × N routes VPNv4. L'Option C est la plus scalable pour ce volume. Pour l'équipe allemande moins expérimentée : investir dans la formation Option C est plus rentable que de gérer 150 sous-interfaces avec Option A. Risques de l'Option A à 150 clients : erreurs de configuration par VRF, saturation des interfaces physiques ASBR, temps de provisionnement d'un nouveau client très long.

**CG.10** —
**Changent** avec SR-MPLS vs LDP : (1) les **labels de transport** (outer labels) — ce sont des SID (Node SID, Adjacency SID) distribués via l'IGP au lieu de labels LDP ; (2) le **protocole de distribution des labels de transport** — extensions IGP (IS-IS TLV, OSPF LSA) au lieu de sessions LDP ; (3) le mécanisme de **FRR** — TI-LFA au lieu de LDP FRR.

**Restent identiques** : (1) **VRF** (isolation des clients sur les PE) ; (2) **RD** (unicité des préfixes dans MP-BGP) ; (3) **RT** (filtrage import/export) ; (4) **MP-BGP** (distribution des routes VPN et des labels VPN entre PE) ; (5) **labels VPN** (inner labels, distribués par MP-BGP) ; (6) **plan de données** (PUSH/SWAP/POP — les P swappent des SID exactement comme ils swappaient des labels LDP).

SR-MPLS est "transparent" pour le plan de contrôle VPN car il ne remplace que le mécanisme de distribution des labels de **transport** — tout ce qui est au-dessus (VRF, RD, RT, MP-BGP) fonctionne identiquement.

---

*Dernière mise à jour : juin 2026 · Prérequis : Tutoriel 1 (IGP) + Tutoriel 2 (MPLS)*