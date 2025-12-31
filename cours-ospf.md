### Cours Détaillé sur les Protocoles OSPFv2 et OSPFv3

Bonjour ! Ce cours est conçu pour former des futurs experts en réseaux, en fournissant une analyse approfondie des protocoles OSPFv2 (Open Shortest Path First version 2) et OSPFv3 (version 3). Nous aborderons les fondements théoriques, les mécanismes internes, les algorithmes, les types de paquets, les configurations avancées, les considérations de sécurité, les optimisations pour la scalabilité, et les pièges courants. Le cours est structuré de manière progressive : introduction générale à OSPF, détails sur OSPFv2, détails sur OSPFv3, comparaison exhaustive, et enfin un lab pratique avec Containerlab et des images cEOS Arista. Nous nous baserons sur les standards IEEE/IETF, les meilleures pratiques industrielles, et des exemples concrets pour une compréhension experte.

Ce cours suppose une connaissance de base des protocoles de routage (comme RIP ou EIGRP), des concepts IP (IPv4/IPv6), et des topologies réseau. Pour une expertise, nous explorerons les RFC pertinents (RFC 2328 pour OSPFv2, RFC 5340 pour OSPFv3) à travers des descriptions détaillées, bien que les accès directs aux documents aient été limités ; les informations sont corroborées par des sources fiables.

#### 1. Introduction Générale à OSPF
OSPF est un protocole de routage intérieur (IGP - Interior Gateway Protocol) de type link-state, conçu pour les réseaux IP. Il utilise l'algorithme de Dijkstra (Shortest Path First - SPF) pour calculer les chemins optimaux vers les destinations, en se basant sur une base de données topologique partagée (Link-State Database - LSDB). Contrairement aux protocoles distance-vector comme RIP, OSPF propage des informations sur l'état des liens (Link-State Advertisements - LSAs) plutôt que des routes complètes, ce qui permet une convergence rapide et une scalabilité dans les grands réseaux.

**Principes Clés Communs à OSPFv2 et OSPFv3 :**
- **Hiérarchie des Zones (Areas) :** OSPF divise le réseau en zones pour réduire la charge de calcul et de flooding. La zone backbone (Area 0) est obligatoire et relie toutes les autres zones.
- **Élection de Routeurs Désignés :** Dans les segments multi-accès (comme Ethernet), un Designated Router (DR) et un Backup DR (BDR) sont élus pour optimiser le flooding des LSAs.
- **Métrique :** Basée sur le coût (cost), par défaut inversement proportionnel à la bande passante (référence 100 Mbps pour OSPFv2).
- **Convergence :** Rapide grâce au flooding fiable et aux recalculs SPF incrémentaux (Partial SPF pour les changements mineurs).
- **Support de Routage Égal-Coût (ECMP) :** Jusqu'à 16 chemins égaux par défaut.
- **Sécurité :** Authentification pour protéger contre les injections malveillantes.
- **Scalabilité :** Supporte des milliers de routeurs avec une bonne conception d'areas.

OSPF opère sur IP (protocole 89) et utilise des adresses multicast pour les échanges (224.0.0.5/6 pour OSPFv2, FF02::5/6 pour OSPFv3).

#### 2. OSPFv2 en Détail
OSPFv2, défini par le RFC 2328 (1998), est conçu pour IPv4. Il est largement déployé dans les réseaux d'entreprise et de fournisseurs de services pour sa robustesse et sa capacité à gérer des topologies complexes.

**Mécanismes Internes :**
- **Adjacences et Hello Protocol :** Les routeurs découvrent les voisins via des paquets Hello envoyés toutes les 10 secondes (par défaut) sur les interfaces actives. Un adjacency se forme si les paramètres correspondent (area ID, authentication, network type, etc.). Le processus inclut un échange bidirectionnel, puis une synchronisation de la LSDB.
- **Types de Paquets OSPF :**
  - Hello (Type 1) : Découverte et maintien des voisins ; inclut router ID, area ID, DR/BDR, timers (hello/dead), etc.
  - Database Description (DD, Type 2) : Échange de headers LSA pour synchroniser la LSDB lors de l'initialisation d'adjacency.
  - Link-State Request (LSR, Type 3) : Demande de LSAs spécifiques manquants.
  - Link-State Update (LSU, Type 4) : Flooding des LSAs complets.
  - Link-State Acknowledgment (LSAck, Type 5) : Accusé de réception fiable pour les LSAs.
- **Synchronisation de la LSDB :** Lors de la formation d'adjacency, les routeurs échangent des DD packets pour comparer leurs LSDB. Le master (plus haut router ID) initie ; les slaves répondent. Les LSAs manquants sont demandés via LSR et envoyés via LSU. Une fois synchronisée, la LSDB est identique sur tous les routeurs d'une area.
- **Flooding des LSAs :** Les LSAs sont flooded de manière fiable (avec acknowledgments) dans une area. Chaque LSA a un âge (age) max de 3600s et est rafraîchi toutes les 1800s. Le flooding est optimisé via DR/BDR sur les réseaux broadcast/non-broadcast.
- **Types d'Areas :**
  - Standard Area : Flooding complet des LSAs.
  - Stub Area : Pas de LSAs externes (Type 5) ; route par défaut injectée par ABR.
  - Totally Stubby Area (Cisco/Arista propriétaire) : Pas de LSAs inter-area (Type 3) ni externes ; route par défaut.
  - Not-So-Stubby Area (NSSA) : Permet des LSAs externes locales (Type 7), converties en Type 5 par ABR.
  - Totally NSSA : Combinaison avec totally stubby.
- **Types de Routeurs :**
  - Internal Router (IR) : Tous les interfaces dans une même area.
  - Area Border Router (ABR) : Connecte plusieurs areas ; résume les routes.
  - Backbone Router (BR) : Dans Area 0.
  - Autonomous System Boundary Router (ASBR) : Injecte routes externes (redistribution).
- **Calcul de Routes :** Utilise l'algorithme de Dijkstra pour construire un arbre SPF à partir de la LSDB. Chaque routeur est la racine ; les coûts cumulés déterminent les next-hops. Supporte les summaries pour réduire la taille de la table de routage.
- **Authentification :** Null (aucune), Simple (mot de passe en clair), Cryptographic (MD5 avec clé).
- **Fonctionnalités Avancées pour Experts :**
  - **Non-Stop Forwarding (NSF)/Graceful Restart :** Permet de maintenir le forwarding pendant un restart du process OSPF, en signalant aux voisins de ne pas dropper l'adjacency.
  - **Bidirectional Forwarding Detection (BFD) :** Détection rapide de pannes de liens (sub-seconde) intégrée à OSPF.
  - **Summarization :** Sur ABR (area range) ou ASBR (summary-address) pour agréger les préfixes et réduire la LSDB.
  - **Virtual Links :** Pour connecter des areas non contiguës à travers Area 0.
  - **Demand Circuits :** Pour liens on-demand (ex: dial-up), supprime les hellos périodiques.
  - **Pièges Courants :** Mismatch de MTU, authentication, ou area type causant des adjacencies échouées ; boucles dues à des summaries mal configurées ; surcharge CPU en cas de flapping links sans dampening.

OSPFv2 est optimisé pour IPv4, avec les adresses IP embarquées dans les LSAs, ce qui le rend moins flexible pour d'autres protocoles.

#### 3. OSPFv3 en Détail
OSPFv3, défini par le RFC 5340 (2008), étend OSPFv2 pour supporter IPv6 nativement, tout en conservant les principes link-state. Il introduit des changements pour l'indépendance vis-à-vis des adresses IP et le support multi-address-family (AF).

**Mécanismes Internes :**
- **Adjacences et Hello Protocol :** Similaire à v2, mais utilise des adresses link-local IPv6 (FE80::/10) pour les communications. Les hellos incluent l'Instance ID pour séparer les processus (ex: un pour IPv6, un pour IPv4 via extensions).
- **Types de Paquets :** Identiques à v2 (Hello, DD, LSR, LSU, LSAck), mais avec des headers modifiés pour IPv6 (pas d'authentification intégrée ; utilise IPsec).
- **Synchronisation de la LSDB :** Processus identique, mais les LSAs sont scoped (Link, Area, AS) pour un flooding plus précis.
- **Flooding des LSAs :** Plus granulaire avec trois scopes : Link-local (pour liens spécifiques), Area (intra-area), AS (inter-area/externes). Les LSAs ne contiennent plus d'adresses IP directement ; elles sont dans des champs séparés.
- **Types d'Areas :** Identiques à v2 (standard, stub, NSSA, etc.), mais appliqués par instance.
- **Types de Routeurs :** Similaires, mais avec support pour multiple instances par lien.
- **Calcul de Routes :** Dijkstra appliqué par instance/AF. Supporte IPv6 et potentiellement IPv4 via extensions (RFC 5838).
- **Authentification :** Pas intégrée ; repose sur IPsec AH/ESP pour une sécurité plus forte.
- **Fonctionnalités Avancées pour Experts :**
  - **Instance ID :** Permet plusieurs instances OSPF par interface (0-31 pour IPv6, 64-95 pour IPv4). Utile pour multi-topologies ou séparation AF.
  - **Address Families (AF) :** OSPFv3 peut transporter IPv4 et IPv6 dans le même process via extensions, réduisant la charge.
  - **LSA Types Modifiés :**
    - Router LSA (Type 1 → 0x2001) : Décrit les liens du routeur.
    - Network LSA (Type 2 → 0x2002) : Généré par DR.
    - Inter-Area Prefix (Type 3 → 0x2003) : Summaries intra-area.
    - Inter-Area Router (Type 4 → 0x2004) : Vers ASBR.
    - AS-External (Type 5 → 0x4005) : Routes externes.
    - NSSA External (Type 7 → 0x2007).
    - Link LSA (Nouveau, 0x0008) : Adresses locales sur un lien.
    - Intra-Area Prefix (Nouveau, 0x2009) : Préfixes attachés à un routeur/réseau.
  - **NSF/Graceful Restart :** Amélioré pour IPv6, avec timers spécifiques.
  - **BFD :** Intégré pour détection rapide.
  - **Multi-Topology Routing (MTR) :** Via instances, pour routage différencié (ex: voice vs data).
  - **Pièges Courants :** Oubli d'activer IPv6 sur interfaces ; mismatch d'Instance ID ; problèmes IPsec ; flooding excessif sans scoping proper.

OSPFv3 est plus modulaire, avec les adresses découplées des LSAs topologiques, facilitant l'extension à d'autres AF.

#### 4. Comparaison Exhaustive OSPFv2 vs OSPFv3
Pour les experts, comprendre les différences est crucial pour la migration IPv4 vers dual-stack ou IPv6-only. Voici une comparaison détaillée basée sur des analyses standard :

- **Protocole Supporté :** OSPFv2 pour IPv4 uniquement ; OSPFv3 pour IPv6 nativement, et IPv4 via extensions (RFC 5838).
- **Adresses dans LSAs :** v2 embarque IPv4 dans LSAs (ex: Router LSA inclut IPs) ; v3 sépare topologie et adresses (nouveaux LSAs comme Link/Intra-Area Prefix).
- **Authentification :** v2 supporte simple/MD5 ; v3 utilise IPsec (plus sécurisé mais complexe).
- **Instances Multiples :** v2 non supporté par interface ; v3 oui via Instance ID.
- **Flooding Scope :** v2 : Area/AS ; v3 ajoute Link-local pour efficacité.
- **Multicast :** v2 : 224.0.0.5/6 ; v3 : FF02::5/6.
- **LSA Types :** v3 renomme et ajoute (ex: pas de Group-Membership LSA ; nouveaux pour prefixes).
- **Élection DR/BDR :** Identique, mais v3 utilise router ID (32-bit, souvent basé sur IPv4 loopback).
- **Métrique :** Identique, mais v3 supporte coûts plus larges pour liens haut débit.
- **Scalabilité :** v3 mieux pour grands réseaux IPv6 grâce au découplage et instances.
- **Compatibilité :** v3 peut router IPv4, mais v2 ne gère pas IPv6 ; migration via ships-in-the-night (deux processus séparés).
- **Avantages v3 :** Plus flexible pour future-proofing ; support multi-AF réduit overhead.

En pratique, pour les réseaux dual-stack, exécuter OSPFv2 pour IPv4 et OSPFv3 pour IPv6 est courant, mais v3 unifié est préféré pour la simplicité.

#### 5. Configuration OSPF sur Arista EOS
Arista EOS supporte OSPFv2 et v3 avec des commandes similaires, mais adaptées aux AF. Voici des détails experts, basés sur la documentation Arista (configurations globales, par VRF si besoin).

**Configuration Basique OSPFv2 :**
```
router ospf 1  # Process ID
 network 10.0.0.0/8 area 0.0.0.0  # Active OSPF sur préfixes
 router-id 1.1.1.1
 max-lsa 12000  # Limite LSAs pour protection
 passive-interface Loopback0  # Pas d'hellos sur loopback
 area 0 authentication message-digest  # MD5 par area
interface Ethernet1
 ip ospf area 0.0.0.0
 ip ospf cost 10
 ip ospf authentication message-digest
 ip ospf message-digest-key 1 md5 mykey
```

**Configuration Avancée OSPFv2 :**
- Summarization : `area 1 range 10.1.0.0/16 cost 20`
- Redistribution : `redistribute static`
- NSF : `graceful-restart`
- BFD : `bfd interval 50 min_rx 50 multiplier 3` (global), puis `ip ospf bfd` sur interface.
- Stub Area : `area 1 stub no-summary` (totally stubby)

**Configuration Basique OSPFv3 :**
```
router ospfv3 1  # Process ID
 address-family ipv6 unicast
  router-id 1.1.1.1
  area 0.0.0.0 stub
interface Ethernet1
 ospfv3 ipv6 area 0.0.0.0
 ospfv3 ipv6 cost 10
```

**Configuration Avancée OSPFv3 :**
- Instance ID : `ospfv3 ipv6 instance-id 64` (pour IPv4)
- Authentification : Via IPsec (configure profiles IPsec séparément).
- Summarization : `area 1 ipv6 range 2001:db8::/32`
- BFD : Similaire, `ospfv3 bfd`
- Multi-AF : Dans le même process, active `address-family ipv4 unicast`

**Vérification (Commun) :**
- `show ip ospf neighbor` / `show ospfv3 neighbor` : Adjacencies.
- `show ip ospf database` / `show ospfv3 database` : LSDB.
- `show ip route ospf` / `show ipv6 route ospf` : Tables de routage.
- `show ospf/ospfv3 interface` : Détails interfaces.
- Debugging : `debug ospf lsa` pour tracing expert.

Meilleures Pratiques : Utilise loopback pour router ID stable ; configure authentication toujours ; limite LSAs ; intègre BFD pour haute disponibilité ; teste NSF avec restarts contrôlés.

#### 6. Lab Pratique : OSPF avec Containerlab et cEOS Arista
Pour ce lab expert, nous simulons une topologie multi-area avec OSPFv2 et v3. Utilise 4 switches cEOS : SW1 (backbone), SW2 (area 1, ABR), SW3 (area 1), SW4 (area 2, stub). Ajoute redistribution et BFD.

**Prérequis :**
- Docker, Containerlab installé.
- Image cEOS importée : `docker import cEOS64-lab-<version>.tar ceos:<version>` (ex: 4.32.0F).
- Crée un fichier YAML : ospf-lab.yaml

```yaml
name: ospf-lab
topology:
  kinds:
    arista_ceos:
      image: ceos:4.32.0F
  nodes:
    sw1:
      kind: arista_ceos
    sw2:
      kind: arista_ceos
    sw3:
      kind: arista_ceos
    sw4:
      kind: arista_ceos
  links:
    - endpoints: ["sw1:eth1", "sw2:eth1"]  # Backbone to Area 1
    - endpoints: ["sw2:eth2", "sw3:eth1"]  # Area 1 internal
    - endpoints: ["sw1:eth2", "sw4:eth1"]  # Backbone to Area 2
```

Déploie : `containerlab deploy -t ospf-lab.yaml`

**Configuration OSPFv2 (sur chaque node via docker exec -it clab-ospf-lab-sw<N> Cli) :**
- SW1 (Backbone, ABR) :
  ```
  enable
  configure terminal
  hostname SW1
  interface Loopback0
   ip address 1.1.1.1/32
  interface Ethernet1
   no shutdown
   ip address 10.12.1.1/24
  interface Ethernet2
   no shutdown
   ip address 10.14.1.1/24
  router ospf 1
   router-id 1.1.1.1
   network 1.1.1.1/32 area 0
   network 10.12.1.0/24 area 0
   network 10.14.1.0/24 area 0
   area 0 authentication message-digest
  end
  write memory
  ```
- SW2 (ABR, Area 1) :
  Similaire, avec `network 10.12.1.0/24 area 0`, `network 10.23.1.0/24 area 1`, area 1 stub.
- SW3 (Area 1) : Seulement area 1.
- SW4 (Area 2, Stub) : Area 2 comme stub.

**Ajoute OSPFv3 :** Sur les mêmes interfaces, ajoute IPv6 et configure `router ospfv3 1`, `ospfv3 ipv6 area ...`.

**Tests Experts :**
- Vérifie adjacencies et LSDB.
- Simule panne : Shutdown interface, observe convergence (<5s avec BFD).
- Redistribue une route statique sur SW4, vérifie propagation.
- Teste NSF : `restart ospf graceful` sur SW1, vérifie pas de drop.

Nettoie : `containerlab destroy -t ospf-lab.yaml`

Ce lab permet d'explorer la convergence, les summaries, et les différences v2/v3. Pour approfondir, ajoute VRF ou multi-instances.



### Détails Approfondis sur les Not-So-Stubby Areas (NSSA) dans OSPF

Les **Not-So-Stubby Areas (NSSA)** sont un type spécial d'area OSPF introduit par le RFC 3101 (qui remplace le RFC 1587 initial). Elles représentent une extension des areas stub classiques, conçues pour résoudre un problème spécifique : permettre la présence d'un **ASBR (Autonomous System Boundary Router)** dans une area qui reste "stub-like" (c'est-à-dire protégée contre le flooding massif de routes externes), tout en bloquant les LSAs externes provenant d'autres areas.

#### 1. Contexte et Motivation
Les areas stub bloquent les **LSA Type 5** (AS-External LSAs) et **Type 4** (ASBR Summary LSAs) pour réduire la taille de la LSDB et limiter les recalculs SPF. À la place, l'ABR injecte une route par défaut (Type 3 Summary avec 0.0.0.0/0).

Problème : Une area stub ne peut pas contenir d'ASBR, car la redistribution de routes externes générerait des Type 5 LSAs, ce qui est interdit.

Solution : NSSA permet un ASBR local dans l'area, mais utilise un nouveau type de LSA (**Type 7 - NSSA External LSA**) pour les routes redistribuées. Ces Type 7 sont confinés à l'area NSSA et traduits en Type 5 par l'ABR avant d'être propagés vers le reste du domaine OSPF.

#### 2. Comportement des LSAs dans une NSSA
- **LSAs Autorisés dans l'Area NSSA :**
  - Type 1 (Router) et Type 2 (Network) : Intra-area, normaux.
  - Type 3 (Summary) : Inter-area (résumés d'autres areas, injectés par l'ABR).
  - Type 7 (NSSA External) : Générés par l'ASBR local pour les routes redistribuées (ex: static, BGP, EIGRP).
- **LSAs Bloqués :**
  - Type 4 et Type 5 : Pas de routes externes provenant d'autres areas.
- **Route par Défaut :**
  - Pas générée automatiquement (contrairement aux stub classiques).
  - Peut être injectée manuellement via `area <id> nssa default-information-originate` (génère un Type 7 default).
- **Traduction Type 7 → Type 5 :**
  - Effectuée par l'**NSSA ABR** (Area Border Router).
  - Si plusieurs ABR, l'ABR avec le **plus haut Router ID** est élu translator (P-bit dans Type 7 pour indiquer la préférence).
  - Le forwarding address est copié (si non-zéro) pour éviter des boucles.
  - Aggregation possible via `summary-address` sur l'ABR.

#### 3. NSSA vs Autres Types d'Areas
| Type d'Area          | ASBR Autorisé ? | LSA Type 3 (Inter-Area) | LSA Type 4/5 (Externes) | LSA Type 7 | Route par Défaut | Usage Typique |
|----------------------|-----------------|--------------------------|--------------------------|------------|------------------|--------------|
| **Standard/Normal** | Oui            | Oui                     | Oui                     | Non       | Non             | Areas centrales |
| **Stub**            | Non            | Oui                     | Non                     | Non       | Oui (Type 3)    | Branches sans redistribution |
| **Totally Stubby**  | Non            | Non                     | Non                     | Non       | Oui (Type 3)    | Branches très petites (propriétaire Cisco/Arista) |
| **NSSA**            | Oui            | Oui                     | Non (sauf traduits)     | Oui       | Optionnelle (Type 7) | Branches avec redistribution locale |
| **Totally NSSA**    | Oui            | Non                     | Non (sauf traduits)     | Oui       | Oui (Type 3)    | Branches avec redistribution, minimisant LSDB (propriétaire) |

#### 4. Totally NSSA (NSSA Totally Stubby)
Extension propriétaire (Cisco/Arista) combinant NSSA et Totally Stubby :
- Bloque Type 3, 4 et 5.
- Autorise Type 7 pour redistribution locale.
- ABR injecte automatiquement une route par défaut (Type 3 Summary).
- Traduction Type 7 → Type 5 comme en NSSA.

Configuration : `area <id> nssa no-summary` (sur l'ABR uniquement).

#### 5. Configuration sur Arista EOS (OSPFv2)
Sur tous les routeurs de l'area :
```
router ospf <process-id>
 area <area-id> nssa
```

Pour Totally NSSA (sur ABR uniquement) :
```
router ospf <process-id>
 area <area-id> nssa no-summary
```

Pour injecter default en NSSA :
```
area <area-id> nssa default-information-originate
```

Pour contrôler la traduction (si plusieurs ABR) :
- `area <area-id> nssa translate always` (force ce routeur à traduire).
- `area <area-id> nssa no-summary` pour Totally NSSA.

Vérifications :
- `show ip ospf database` : Voir Type 7 (N2/N1 routes).
- `show ip ospf database nssa-external` : Détails Type 7.
- `show ip route ospf` : Routes N1/N2 pour externes NSSA.

#### 6. Pièges Courants et Meilleures Pratiques
- Tous les routeurs de l'area doivent être configurés en NSSA (sinon adjacency échoue, car N-bit dans Options des Hello).
- Éviter les boucles : Bien gérer le translator (un seul préféré).
- Utiliser NSSA pour des sites distants avec redistribution locale (ex: filiale connectée à un autre AS via BGP).
- Pour scalabilité : Préférer Totally NSSA si pas besoin de détails inter-area.

Les NSSA offrent un excellent compromis pour les réseaux hiérarchiques où certaines branches nécessitent de la redistribution sans inonder tout le domaine OSPF de routes externes.


### Lab Pratique Containerlab : Démonstration d'une Not-So-Stubby Area (NSSA) avec OSPFv2 sur Arista cEOS

Ce lab simule une topologie OSPF classique pour observer le comportement d'une **NSSA (Area 1)** :
- **SW1** : Routeur dans l'Area 0 (backbone), agit comme ABR (Area Border Router).
- **SW2** : Routeur dans l'Area 1 (NSSA), agit comme ABR et translator Type 7 → Type 5.
- **SW3** : Routeur dans l'Area 1 (NSSA), agit comme ASBR (redistribue une route statique externe, générant un LSA Type 7).

Objectifs :
- Configurer une NSSA.
- Observer les LSA Type 7 dans l'area 1.
- Voir la traduction en Type 5 dans l'area 0.
- Vérifier la route par défaut (optionnelle) et la propagation des routes externes.

#### Prérequis
- Containerlab installé.
- Image cEOS importée (ex: `docker import cEOS64-lab-4.32.0F.tar.xz ceos:4.32.0F`).
- Au moins 4 Go RAM disponibles.

#### Fichier de Topologie (nssa-lab.yaml)
```yaml
name: nssa-lab
topology:
  kinds:
    arista_ceos:
      image: ceos:4.32.0F  # Adapte à ta version
  nodes:
    sw1:
      kind: arista_ceos
    sw2:
      kind: arista_ceos
    sw3:
      kind: arista_ceos
  links:
    - endpoints: ["sw1:eth1", "sw2:eth1"]  # Lien Area 0 ↔ Area 1
    - endpoints: ["sw2:eth2", "sw3:eth1"]  # Lien interne Area 1
```

Déploiement du lab :
```
containerlab deploy -t nssa-lab.yaml
```

#### Configuration Détaillée

Acces à chaque switch :
- `docker exec -it clab-nssa-lab-sw1 Cli`
- Idem pour sw2 et sw3.

**Configuration Commune (Loopback et Interfaces)**
Sur **SW1** :
```
enable
configure terminal
hostname SW1
interface Loopback0
   ip address 1.1.1.1/32
interface Ethernet1
   no shutdown
   ip address 10.0.12.1/24
end
write memory
```

Sur **SW2** :
```
enable
configure terminal
hostname SW2
interface Loopback0
   ip address 2.2.2.2/32
interface Ethernet1
   no shutdown
   ip address 10.0.12.2/24
interface Ethernet2
   no shutdown
   ip address 10.0.23.2/24
end
write memory
```

Sur **SW3** (ASBR) :
```
enable
configure terminal
hostname SW3
interface Loopback0
   ip address 3.3.3.3/32
interface Ethernet1
   no shutdown
   ip address 10.0.23.3/24
! Route statique à redistribuer (externe)
ip route 192.168.100.0/24 Null0
end
write memory
```

**Configuration OSPF**

Sur **SW1** (Area 0 uniquement) :
```
router ospf 1
   router-id 1.1.1.1
   network 1.1.1.1/32 area 0
   network 10.0.12.0/24 area 0
   max-lsa 12000
end
write memory
```

Sur **SW2** (ABR - NSSA Area 1) :
```
router ospf 1
   router-id 2.2.2.2
   network 2.2.2.2/32 area 1
   network 10.0.12.0/24 area 0
   network 10.0.23.0/24 area 1
   area 1 nssa                  ! Configure l'area 1 comme NSSA
   ! Optionnel : Injecter une route par défaut dans la NSSA
   ! area 1 nssa default-information-originate
   ! Pour Totally NSSA (bloque Type 3) : area 1 nssa no-summary (sur ABR uniquement)
end
write memory
```

Sur **SW3** (ASBR dans NSSA Area 1) :
```
router ospf 1
   router-id 3.3.3.3
   network 3.3.3.3/32 area 1
   network 10.0.23.0/24 area 1
   area 1 nssa                  ! Obligatoire sur tous les routeurs de l'area
   redistribute static          ! Génère LSA Type 7 pour 192.168.100.0/24
end
write memory
```

#### Vérifications et Observations

Attends 30-60 secondes pour la convergence.

**Sur SW3 (ASBR)** :
```
show ip ospf database

# Tu verras un LSA Type 7 (NSSA External) pour 192.168.100.0/24
show ip ospf database nssa-external
```

**Sur SW2 (ABR)** :
```
show ip ospf database
# Dans Area 1 : Type 7 visible
# Dans Area 0 : Type 5 (traduit) pour la route externe
show ip ospf database external
show ip ospf database nssa-external
```

**Sur SW1 (Area 0)** :
```
show ip ospf database
# Seulement Type 5 (pas de Type 7)
show ip route ospf
# Route externe 192.168.100.0/24 visible via O N2 (ou N1 selon metric-type)
```

**Tests Supplémentaires** :
- Activation de la default route sur SW2 : `area 1 nssa default-information-originate` → Vérification d'une route par défaut Type 7 dans Area 1.
- Pour Totally NSSA : Ajout de `area 1 nssa no-summary` sur SW2 uniquement → Pas de Type 3 dans Area 1, mais default Type 3 injectée.
- Simulation d'une panne du lien SW2-SW3 → Observation de la reconvergence et nouveau translator si plusieurs ABR.

#### Nettoyage
```
containerlab destroy -t nssa-lab.yaml
```

Ce lab permet de visualiser concrètement la traduction Type 7 → Type 5, l'isolation des routes externes, et l'utilité des NSSA dans des scénarios réels (ex: sites distants avec redistribution BGP locale).
