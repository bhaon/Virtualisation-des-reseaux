### Cours Détaillé sur le Protocole IS-IS

Ce cours est conçu pour fournir une compréhension approfondie du protocole **IS-IS (Intermediate System to Intermediate System)**, un protocole de routage intérieur (IGP) de type link-state, défini par l'ISO/IEC 10589 et étendu par divers RFC (comme RFC 1195 pour l'intégration IP). IS-IS est particulièrement adapté aux grands réseaux, tels que ceux des fournisseurs de services, en raison de sa scalabilité, de sa flexibilité et de son indépendance vis-à-vis des protocoles de couche 3 (il supporte IPv4, IPv6 et d'autres comme CLNP). Contrairement à OSPF, IS-IS opère directement sur la couche 2, utilisant des adresses NSAP (Network Service Access Point) pour l'identification.

Le cours est structuré de manière progressive : introduction, mécanismes internes, configurations avancées, comparaison avec OSPF, et enfin plusieurs labs pratiques avec Containerlab et des images Arista cEOS. Les labs incluent des scénarios variés pour mettre en pratique les concepts. Ce cours suppose une connaissance de base des protocoles de routage et des concepts IP.

#### 1. Introduction à IS-IS
IS-IS est un protocole link-state similaire à OSPF, mais originaire du monde ISO/CLNS (Connectionless Network Service). Il a été adapté pour IP via Integrated IS-IS (ou Dual IS-IS). Ses principaux avantages :
- **Scalabilité :** Supporte des milliers de routeurs grâce à une hiérarchie à deux niveaux (Level 1 et Level 2).
- **Convergence Rapide :** Utilise l'algorithme SPF (Shortest Path First, Dijkstra) pour calculer les chemins optimaux.
- **Multi-Protocole :** Natif pour CLNP, étendu pour IP (IPv4/IPv6 via TLVs - Type-Length-Value).
- **Indépendance :** Pas de dépendance à IP pour les adjacencies ; utilise des adresses MAC ou SNPA (Subnetwork Point of Attachment).
- **Utilisation Typique :** Réseaux ISP, data centers, où OSPF est trop "IP-centrique".

Terminologie Clé :
- **Intermediate System (IS)** : Routeur.
- **End System (ES)** : Hôte (non routant).
- **Area** : Groupe d'IS Level 1.
- **Domain** : Ensemble des areas connectées via Level 2.
- **NET (Network Entity Title)** : Adresse NSAP unique pour chaque IS (similaire à Router ID en OSPF).
- **PDU (Protocol Data Unit)** : Paquets IS-IS (Hello, LSP, etc.).

#### 2. Mécanismes Internes de IS-IS
IS-IS repose sur l'échange d'informations d'état des liens (Link-State PDUs - LSPs) pour construire une base de données topologique (LSDB) identique sur tous les routeurs d'un niveau.

**Niveaux Hiérarchiques :**
- **Level 1 (L1)** : Routage intra-area. Les IS L1 ne connaissent que leur area et une route par défaut vers le L1/L2 le plus proche.
- **Level 2 (L2)** : Routage inter-area (backbone). Les IS L2 connectent les areas et propagent les résumés.
- **Level 1/2 (L1/L2)** : Routeurs border qui font les deux.

**Adresse NET :**
- Format : Area ID (variable, souvent 49.0001 pour privé) + System ID (6 octets, basé sur MAC ou loopback) + SEL (00 pour IP).
- Exemple : 49.0001.1921.6800.1001.00 (Area 49.0001, System ID 1921.6800.1001, SEL 00).
- Doit être unique dans le domaine.

**Adjacencies et Hello PDUs :**
- **IIH (IS-IS Hello)** : Envoyés pour découvrir les voisins (toutes les 10s par défaut, dead timer 30s).
- Types : Point-to-Point IIH (pour liens P2P), LAN IIH (pour broadcast, avec élection DIS - Designated IS).
- Adjacency se forme si : Même niveau, même area (pour L1), authentication match, etc.
- Pas de DR/BDR comme OSPF ; utilise DIS (Designated IS) élu par priorité + System ID le plus haut. Le DIS flood les LSPs pour optimiser.

**LSPs (Link-State PDUs) :**
- Générés par chaque IS pour décrire ses liens, voisins, préfixes attachés.
- **L1 LSP** : Intra-area.
- **L2 LSP** : Inter-area.
- Contenu via TLVs (extensibles) : TLV 22 (Extended IS Reachability), TLV 135 (Extended IP Reachability), TLV 236 (IPv6 Reachability), etc.
- Flooding : Fiable, avec acknowledgments (PSNP - Partial Sequence Number PDU, CSNP - Complete SNP).
- Âge : Max 1200s (rafraîchi toutes les 900s), Sequence Number pour versioning.

**Calcul de Routes :**
- Algorithme SPF sur la LSDB pour chaque niveau.
- Métrique : Par défaut 10 par lien (wide metrics jusqu'à 2^24-1 pour liens haut débit).
- Support ECMP (Equal Cost Multi-Path).
- Pour IP : Préfixes annoncés via TLVs ; support multi-topology (MT) pour IPv4/IPv6 séparés.

**Convergence et Stabilité :**
- Hello timers pour détection rapide.
- Overload Bit : Marque un IS comme surchargé (transit évité).
- Graceful Restart : Maintient adjacencies pendant restart.
- BFD (Bidirectional Forwarding Detection) : Intégrable pour sub-seconde failure detection.

**Authentification :**
- Simple (mot de passe en clair), HMAC-MD5, HMAC-SHA (via Key Chains).
- Par niveau (L1/L2) ou global.

**Extensions Avancées :**
- **Multi-Area :** Un IS peut appartenir à plusieurs areas (via multi-instance).
- **IPv6 :** Via TLVs dédiés ; single-topology (même topologie pour v4/v6) ou multi-topology.
- **Segment Routing (SR)** : Intégration via TLVs pour SR-TE.
- **Redistribution :** Routes externes via metric-type (internal/external).
- **Summarization :** Sur L1/L2 pour agréger préfixes.

**Pièges Courants :**
- Mismatch de NET/area/level causant adjacencies échouées.
- Flooding excessif sans tuning (ex: mesh groups).
- Métrics narrow vs wide : Assurer compatibilité.

#### 3. Comparaison avec OSPF
- **Similitudes :** Link-state, SPF, hiérarchie (areas), convergence rapide.
- **Différences :**
  - IS-IS : Couche 2, NSAP, niveaux L1/L2 (pas d'Area 0 obligatoire), TLVs extensibles, DIS sur tous liens broadcast.
  - OSPF : Couche 3 (IP), areas numérotées, DR/BDR seulement sur broadcast, LSAs rigides.
  - Scalabilité : IS-IS mieux pour très grands domaines (moins overhead).
  - Support : IS-IS natif multi-protocole ; OSPF séparé v2/v3.

#### 4. Configuration IS-IS sur Arista EOS
Arista supporte IS-IS avec des commandes CLI similaires à Cisco/Juniper.

**Configuration Basique :**
```
router isis <instance>  # Ex: default
 net 49.0001.0000.0000.0001.00
 is-type level-1-2       # Par défaut L1/L2
 metric-style wide       # Recommandé pour metrics larges
 log-adjacency-changes
interface Ethernet1
 no shutdown
 ip router isis <instance>
 isis enable <instance>
 isis metric 10
end
```

**Avancé :**
- Authentication : `authentication mode md5 level-1`, `authentication key-chain mykey`.
- IPv6 : `address-family ipv6`, `multi-topology`.
- Redistribution : `redistribute static level-2`.
- Summarization : `summary-address 10.0.0.0/8 level-1`.
- BFD : `isis bfd`.
- Vérification : `show isis neighbors`, `show isis database`, `show ip route isis`, `show isis topology`.

#### 5. Labs Pratiques avec Containerlab et cEOS Arista
Les labs utilisent une topologie de base avec 3-4 switches pour démontrer les concepts. Prérequis : Containerlab installé, image cEOS importée (`docker import cEOS-lab-<version>.tar ceos:<version>`). Déployez avec `containerlab deploy -t <yaml>`, accédez via `docker exec -it clab-<lab>-sw<N> Cli`.

**Lab 1 : Configuration Basique IS-IS (Single Area, Level 1)**
Topologie : Triangle SW1-SW2-SW3.
Fichier YAML (isis-basic.yaml) :
```
name: isis-basic
topology:
  kinds:
    arista_ceos:
      image: ceos:4.32.0F
  nodes:
    sw1: kind: arista_ceos
    sw2: kind: arista_ceos
    sw3: kind: arista_ceos
  links:
    - endpoints: ["sw1:eth1", "sw2:eth1"]
    - endpoints: ["sw2:eth2", "sw3:eth1"]
    - endpoints: ["sw3:eth2", "sw1:eth2"]
```

Configuration sur chaque SW (ex: SW1) :
```
configure terminal
hostname SW1
interface Loopback0
 ip address 1.1.1.1/32
interface Ethernet1
 no shutdown
 ip address 10.12.1.1/24
interface Ethernet2
 no shutdown
 ip address 10.13.1.1/24
router isis default
 net 49.0001.1921.6800.1001.00  # Adaptez System ID
 is-type level-1
 metric-style wide
interface Loopback0
 ip router isis default
interface Ethernet1
 ip router isis default
 isis metric 10 level-1
interface Ethernet2
 ip router isis default
 isis metric 10 level-1
end
write memory
```
Vérifiez : `show isis neighbors` (adjacencies), `show isis database` (LSPs), `show ip route isis` (routes). Simulez panne : Shutdown un lien, observez convergence.

**Lab 2 : Multi-Level (L1/L2 avec Summarization)**
Ajoutez SW4 en Area différente. YAML (isis-multi.yaml) : Ajoutez SW4 et lien SW1-eth3 à SW4-eth1.
Configuration : SW1 et SW4 comme L1/L2.
Sur SW1 (L1/L2) :
```
router isis default
 net 49.0001.1921.6800.1001.00
 is-type level-1-2
 summary-address 10.0.0.0/8 level-1  # Summarize dans L1
```
Sur SW4 (nouvelle area) : NET 49.0002...., is-type level-1.
Vérifiez : `show isis topology`, routes par défaut dans L1.

**Lab 3 : Authentication et BFD**
Même topologie que Lab 1. Ajoutez auth :
```
key chain mykey
 key 1
  key-string secret
router isis default
 authentication mode md5 level-1
 authentication key-chain mykey level-1
interface Ethernet1
 isis authentication mode md5 level-1
 isis authentication key-chain mykey level-1
 isis bfd
```
Vérifiez : Essayez mismatch auth (adjacency échoue), activez BFD et simulez panne rapide.

**Lab 4 : IPv6 et Redistribution**
Ajoutez IPv6 sur interfaces (ex: `ipv6 address 2001:db8:12::1/64`).
```
router isis default
 address-family ipv6
  multi-topology
  redistribute static level-2
interface Ethernet1
 ipv6 router isis default
```
Ajoutez route statique IPv6, vérifiez propagation.

**Lab 5 : Graceful Restart et Overload**
Sur SW1 :
```
router isis default
 graceful-restart
 set-overload-bit on-startup 60  # Overload pendant 60s au boot
```
Restart process : `restart isis`, vérifiez adjacencies maintenues.

Nettoyez chaque lab : `containerlab destroy -t <yaml>`.

Ces labs couvrent les essentials et avancés. Expérimentez pour maîtriser !
