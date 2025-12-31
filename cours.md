# Cours : Virtualisation des Réseaux Informatiques

Ce cours aborde les concepts fondamentaux de la virtualisation des réseaux dans les data centers modernes, en se concentrant sur les architectures **Underlay/Overlay** et les **Control Planes**. Nous explorerons chaque aspect en détail, avec des explications théoriques, des exemples pratiques et des configurations basées sur des commutateurs virtuels **Arista cEOS** déployés via **Containerlab**. Cela permettra une compréhension approfondie des problèmes résolus par ces technologies et de leur mise en œuvre.

> **Prérequis** : Connaissances de base en réseaux (modèle OSI, switching L2, routage IP L3), familiarité avec la ligne de commande (CLI) des équipements réseaux, et notions de virtualisation (VMs, containers).  
> **Objectifs** : À la fin de ce cours, vous serez capable de configurer un réseau virtualisé simple, de comprendre les mécanismes underlay/overlay, et d'analyser un control plane comme EVPN.

## Configuration du Laboratoire Containerlab

Pour reproduire les exemples pratiques, nous utilisons **Containerlab**, un outil open-source pour simuler des topologies réseaux avec des containers. Cela permet de tester sans hardware physique, en rendant l'apprentissage accessible et reproductible.

Voici un fichier `topology.yaml` de base pour une topologie **leaf-spine** (2 spines, 2 leaves) typique des data centers modernes. Cette topologie assure une redondance et une bande passante élevée, avec les spines comme backbone et les leaves connectés aux hôtes/VMs.

```yaml
name: vxlan-evpn-lab
topology:
  nodes:
    spine1:
      kind: ceos
      image: ceos:4.35.0F  # Téléchargez l'image depuis le site Arista (compte gratuit requis)
      startup-config: spine1.cfg  # Fichier de config initial optionnel
    spine2:
      kind: ceos
      image: ceos:4.35.0F
      startup-config: spine2.cfg
    leaf1:
      kind: ceos
      image: ceos:4.35.0F
      startup-config: leaf1.cfg
    leaf2:
      kind: ceos
      image: ceos:4.35.0F
      startup-config: leaf2.cfg
  links:
    - endpoints: ["spine1:eth1", "leaf1:eth1"]  # Lien spine1-leaf1
    - endpoints: ["spine1:eth2", "leaf2:eth1"]  # Lien spine1-leaf2
    - endpoints: ["spine2:eth1", "leaf1:eth2"]  # Lien spine2-leaf1
    - endpoints: ["spine2:eth2", "leaf2:eth2"]  # Lien spine2-leaf2
```

**Étapes de déploiement** :
1. Installez Containerlab : `pip install containerlab` (ou via Docker).
2. Démarrez le lab : `containerlab deploy -t topology.yaml`. Cela crée les containers et les interconnecte via des bridges virtuels.
3. Accédez à un switch : `docker exec -it clab-vxlan-evpn-lab-spine1 Cli` pour entrer en mode CLI Arista EOS.
4. Arrêtez le lab : `containerlab destroy -t topology.yaml` pour nettoyer.

**Astuces** : Si vous rencontrez des problèmes de licence Arista, utilisez des versions gratuites ou testez avec d'autres images comme Cumulus VX. Pour une visualisation, intégrez Containerlab avec des outils comme Netlab ou EVE-NG.

## Introduction : Problèmes à Résoudre

Cette section introduit les limitations des réseaux traditionnels et motive l'adoption de la virtualisation. Dans les data centers actuels, avec la prolifération des VMs et des containers, les réseaux legacy ne scalent plus efficacement.

### Problèmes des réseaux de niveau 2
Les réseaux L2 (basés sur Ethernet et MAC addresses) sont simples mais deviennent problématiques à grande échelle :
- **Boucles** : Dans une topologie redondante, des boucles peuvent se former, entraînant des "broadcast storms" où les paquets circulent indéfiniment, saturant la bande passante et causant des pannes.
- **ARP (Address Resolution Protocol)** : ARP résout les IPs en MAC via des broadcasts, qui sont envoyés à tous les hôtes du domaine L2. Cela augmente le trafic inutile, pose des risques de sécurité (ARP spoofing) et limite l'échelle (ex. : dans un data center avec 10k+ VMs).
- **Broadcast** : Tout trafic broadcast/multicast (ARP, DHCP, etc.) inonde l'ensemble du segment L2, consommant des ressources et réduisant les performances.
- **Pas de Control Plane** : Absence d'un plan de contrôle dédié pour gérer dynamiquement les adresses et les flux, rendant les réseaux statiques, difficiles à automatiser et sensibles aux défaillances.

Exemple concret : Dans un data center cloud, étendre un domaine L2 sur plusieurs racks sans contrôle mène à une explosion de trafic broadcast, rendant le réseau instable.

### Solutions legacy niveau 2
Ces solutions historiques atténuent les problèmes mais restent limitées par leur dépendance à la topologie physique :
- **Spanning Tree Protocol (STP/RSTP/MSTP)** : Calcule un arbre couvrant pour bloquer les liens redondants et éviter les boucles. Avantages : simple. Inconvénients : convergence lente (jusqu'à 50s pour STP), sous-utilisation des liens (seulement ~50% actifs), et pas adapté aux topologies mesh comme leaf-spine.
- **LACP (Link Aggregation Control Protocol)** : Agrège plusieurs liens physiques en un bundle logique (ex. : EtherChannel), augmentant la bande passante et offrant une redondance. Utilise des PDUs pour négocier l'agrégation.
- **MC-LAG (Multi-Chassis Link Aggregation)** : Étend LACP sur deux chassis (ex. : deux switches agissant comme un seul), améliorant la HA sans STP. Exemples : vPC (Cisco) ou MLAG (Arista). Inconvénients : Complexe à configurer, limité à deux chassis, et ne résout pas les broadcasts à grande échelle.

Ces approches fonctionnent pour les petits réseaux mais échouent dans les environnements cloud où la mobilité des VMs nécessite une extension L2 flexible sans floods.

### Problèmes des réseaux de niveau 3
Les réseaux L3 (basés sur IP et routage) ajoutent de la scalabilité mais introduisent d'autres défis :
- **Table de routage unique** : Une seule table FIB (Forwarding Information Base) globale empêche l'isolation multi-tenant, où différents clients partagent l'infrastructure sans voir les trafics des autres.
- **Sécurité** : Sans segmentation fine, les flux IP peuvent traverser des domaines non autorisés, exposant à des attaques (ex. : lateral movement en cas de breach). De plus, le routage statique ou basique manque de flexibilité pour les politiques dynamiques.

Exemple : Dans un cloud public, sans isolation L3, un tenant pourrait accéder aux routes d'un autre, violant la confidentialité.

### Solutions legacy niveau 3
- **VRF (Virtual Routing and Forwarding)** : Crée des instances de routage virtuelles isolées, chacune avec sa propre table de routage, interfaces et protocoles. Souvent combiné avec MPLS pour le transport. Avantages : Isolation forte. Inconvénients : Nécessite MPLS, complexe à scaler.
- **VRF Lite** : Version simplifiée sans MPLS, utilisant des VLANs ou interfaces physiques pour séparer les trafics. Plus léger mais limité aux topologies locales, sans extension sur WAN.

Ces solutions sont efficaces pour les entreprises traditionnelles mais trop rigides pour les data centers hyperscale, où la virtualisation overlay (comme VXLAN) offre plus de flexibilité.

## Un Réseau Underlay : Infrastructure de Support à la Virtualisation

L'underlay est le réseau physique/IP sous-jacent, généralement un fabric L3 routé, qui sert de base aux overlays. Il doit être stable, scalable et à faible latence.

### Interface Loopback
Une interface loopback est une interface logicielle virtuelle, non liée à un port physique, toujours active (up/up) tant que le device est allumé. Elle sert d'identifiant stable pour le routage (router-ID), les tunnels (source VTEP) ou les services (BGP peering).
- Avantages : Immunisée aux pannes de liens physiques ; idéale pour les annonces de routes stables.
- Configuration sur Arista EOS :

```eos
interface Loopback0
   description Router ID et source pour tunnels
   ip address 10.0.0.1/32  # /32 pour une adresse host unique
   no shutdown  # Optionnel, toujours up par défaut
```

Dans notre lab, assignez des IPs uniques : spine1 (10.0.0.1), spine2 (10.0.0.2), leaf1 (10.0.0.3), leaf2 (10.0.0.4).

### Configuration des interfaces d’interconnexion
Les interfaces physiques (ou virtuelles dans Containerlab) interconnectent les devices. Utilisez un adressage point-to-point (/31 ou /30) pour minimiser les pertes d'IPs et simplifier le routage.
- Exemple pour spine1-eth1 (vers leaf1) :

```eos
interface Ethernet1
   description Lien vers leaf1-eth1
   no switchport  # Passe en mode routed (L3)
   ip address 192.168.1.0/31  # /31 : 192.168.1.0 pour spine1, .1 pour leaf1
   no shutdown
```

- Répétez pour tous les liens : Utilisez des subnets distincts par lien pour éviter les conflits (ex. : 192.168.2.0/31 pour spine1-leaf2).
- Vérification : `show interfaces Ethernet1` pour confirmer l'état up/up et l'IP.

Cela crée un fabric L3 sans L2 entre devices, évitant les problèmes de boucles dès le départ.

### Protocoles de routage IGP : OSPF vs IS-IS
Les IGP (Interior Gateway Protocols) assurent la connectivité dans l'AS (Autonomous System).
- **OSPF (Open Shortest Path First)** : Protocole link-state, utilise l'algorithme Dijkstra pour calculer les chemins les plus courts basés sur le coût (métrique). Supporte les areas pour hiérarchiser (area 0 backbone).
- **IS-IS (Intermediate System to Intermediate System)** : Similaire à OSPF, mais utilise des niveaux (Level 1/2) au lieu d'areas. Plus extensible pour les tags (ex. : MPLS Traffic Engineering).

**Pourquoi OSPF/IS-IS plutôt que RIP ou EIGRP ?**
- **Scalabilité et convergence** : OSPF/IS-IS convergent en sub-seconde avec des milliers de routes ; RIP (distance-vector) met 30s par update et est limité à 15 hops. EIGRP (hybride) est scalable mais propriétaire Cisco, limitant l'interopérabilité.
- **Standards et simplicité** : OSPF/IS-IS sont IETF standards, ouverts, et optimisés pour les fabrics leaf-spine (ECMP - Equal Cost Multi-Path). RIP manque de hiérarchie et de support ECMP natif ; EIGRP nécessite des licenses Cisco et est moins courant en multi-vendor.
- Exemple d'usage : Dans un data center Google ou AWS, IS-IS est préféré pour sa flexibilité ; OSPF pour sa familiarité.

Nous utiliserons OSPF pour sa simplicité dans ce cours.

### Mise en place d’OSPF
Configurez OSPF pour annoncer les loopbacks et interfaces, permettant à tous les devices de se reacher.
- Sur chaque device :

```eos
router ospf 1  # Process ID local
   router-id 10.0.0.1  # Utilise la loopback pour stabilité
   passive-interface Loopback0  # Pas d'hello sur loopback (évite les floods inutiles)
   network 10.0.0.0/8 area 0.0.0.0  # Annonce les loopbacks
   network 192.168.0.0/16 area 0.0.0.0  # Annonce les interfaces physiques
   max-lsa 12000  # Limite pour éviter les overloads dans de grands réseaux
```

- Sur chaque interface L3 :

```eos
interface Ethernet1
   ip ospf network point-to-point  # Optimisé pour liens P2P, pas de DR/BDR
   ip ospf area 0.0.0.0
```

Cela établit des adjacences OSPF et propage les routes.

### Vérification du routage Underlay
Assurez-vous que l'underlay est fully meshed : chaque device reach toutes les loopbacks via OSPF.
- Commandes Arista :
  - `show ip route ospf` : Liste les routes OSPF (ex. : O 10.0.0.2/32 via 192.168.1.1).
  - `show ip ospf neighbor` : Vérifie les adjacences (état FULL).
  - `show ip ospf database` : Affiche la LSDB (Link State Database) pour valider la synchronisation.
  - `ping 10.0.0.4 source Loopback0` : Teste la reachability depuis la loopback source.
  - `traceroute 10.0.0.4` : Vérifie le chemin (devrait montrer ECMP si multiple paths).

Si un ping échoue, vérifiez les configs IP, OSPF areas, et l'état des interfaces. L'underlay doit être parfait avant d'ajouter l'overlay.

## Les Réseaux Overlay

Les overlays superposent des réseaux virtuels sur l'underlay physique, via tunnelisation, pour étendre L2/L3 sans modifier l'infrastructure sous-jacente.

### Mécanisme de tunnelisation (encapsulation générique)
La tunnelisation encapsule un paquet original (payload) dans un nouveau paquet externe (header IP/UDP), qui traverse l'underlay. À destination, le paquet est décapsulé.
- Exemples généraux : GRE (Generic Routing Encapsulation) ajoute un header IP simple ; IP-in-IP pour des cas basiques.
- Avantages : Indépendance de l'underlay ; support multi-tenant via IDs virtuels.
- Inconvénients : Overhead (20-50 bytes ajoutés), MTU réduit (nécessite jumbo frames).

Processus : Source encapsule (ajoute header), underlay route, destination décapsule.

### Cas MPLS
MPLS est une technologie legacy mais puissante pour les WAN et data centers.
- **Protocole LDP (Label Distribution Protocol)** : Distribue dynamiquement les labels via des sessions TCP (port 646). Chaque router assigne un label par préfixe FEC (Forwarding Equivalence Class).
- **Entêtes MPLS** : Header de 32 bits (label 20 bits, EXP 3 bits pour QoS, S 1 bit pour stack bottom, TTL 8 bits) inséré entre L2 (Ethernet) et L3 (IP).
- Usage : MPLS VPN pour isolation ; rapide car label switching (pas lookup IP full).

Exemple : Dans un provider network, MPLS transporte des VRFs clients.

### Cas VXLAN (dominant aujourd’hui)
VXLAN est le standard pour les data centers cloud (RFC 7348), étendant les VLANs legacy.
- **VXLAN (Virtual Extensible LAN)** : Utilise 24 bits pour le VNI (VXLAN Network Identifier), permettant 16 millions de segments vs 4096 VLANs.
- **VTEP (VXLAN Tunnel Endpoint)** : Device ou software (ex. : vSwitch) qui encapsule/décapsule. Typiquement une loopback IP.
- **Entêtes VXLAN** : Header UDP (port 4789, source port hashé pour ECMP) + 8 bytes VXLAN (flags, VNI 24 bits, reserved).
- Avantages : Traverse L3 underlay ; support multicast pour BUM (Broadcast/Unknown/Multicast).

Overhead total : 50 bytes (IP 20 + UDP 8 + VXLAN 8 + Ethernet inner 14).

### Configuration VXLAN basique (Flood & Learn)
- Mise en place d'une loopback VTEP :

```eos
interface Loopback1
   description VTEP Source IP
   ip address 10.1.1.1/32  # Annoncée via OSPF
```

- Configuration VXLAN interface :

```eos
interface Vxlan1
   vxlan source-interface Loopback1  # Source pour encapsulation
   vxlan udp-port 4789  # Port standard
   vxlan vlan 10 vni 10010  # Mappe VLAN local 10 à VNI global 10010
   vxlan vlan 20 vni 10020
   vxlan flood vtep 10.1.1.2 10.1.1.3  # Liste statique des VTEPs distants pour flood (mode legacy)
```

Cela permet une encapsulation simple : Un frame L2 entrant sur VLAN 10 est encapsulé en VXLAN avec VNI 10010 et envoyé via underlay.

## Le Control Plane

Le control plane gère dynamiquement les mappings (MAC/IP à VTEP), évitant les mécanismes legacy inefficaces.

### Mode legacy : Flood and Learn
- Mécanisme : Pour un MAC inconnu, le VTEP flood le trafic BUM via multicast (IGMP) ou head-end replication (unicast copies à tous VTEPs connus).
- Apprentissage : Le VTEP source apprend les MAC distants via les réponses décapsulées.
- Problèmes : Floods excessifs dans de grands fabrics (ex. : 100 VTEPs = 99 copies par flood) ; pas scalable pour des data centers avec mobilité VM ; gaspillage de bande passante.

Utilisé dans les configs basiques VXLAN sans control plane.

### Rappel BGP / MP-BGP
- **BGP (Border Gateway Protocol)** : Protocole de routage externe (EGP), scalable pour Internet, basé sur TCP (port 179). Utilise des attributes (AS-Path, etc.) pour éviter les loops.
- **MP-BGP (Multiprotocol BGP)** : Extension (RFC 4760) pour supporter multiples address-families (AFI/SAFI), comme IPv4 unicast, IPv6, VPNv4, EVPN. Permet d'échanger non seulement des routes IP mais aussi des MACs, VNIs, etc.

Dans les data centers, BGP est utilisé en interne (iBGP) avec route-reflectors pour scaler.

### LISP (Locator/ID Separation Protocol)
LISP (RFC 6830) sépare l'identité d'un endpoint (EID - Endpoint Identifier, ex. : IP VM) de sa localisation (RLOC - Routing Locator, ex. : IP VTEP).
- Mécanisme : ITR (Ingress Tunnel Router) query un Mapping Server (MS/MR) pour résoudre EID → RLOC, puis encapsule le paquet vers le RLOC. ETR (Egress) décapsule.
- Avantages : Mobilité (VM migre sans changer EID) ; scalable pour Internet.
- Inconvénients : Nécessite infrastructure LISP globale ; moins courant que EVPN dans les data centers.

Exemple : Utilisé par Cisco pour SD-Access.

### EVPN (Ethernet VPN) – Standard actuel
EVPN (RFC 7432) est le control plane de choix pour VXLAN, utilisant MP-BGP pour un apprentissage proactif.
- Mécanisme : Les VTEPs annoncent via BGP des routes EVPN :
  - Type 2 : MAC/IP Advertisement (MAC + IP + VNI + VTEP).
  - Type 3 : Inclusive Multicast Ethernet Tag (pour BUM handling).
  - Type 5 : IP Prefix (pour L3 routing).
- Route Targets (RT) et Route Distinguishers (RD) assurent l'isolation multi-tenant.
- Avantages : Évite les floods (apprentissage via BGP) ; supporte L2 stretching, L3 gateway, mobilité VM ; intégré avec VXLAN/MPLS.

EVPN transforme VXLAN en un réseau intelligent, scalable pour des milliers de tenants.

### Configuration EVPN + VXLAN (Arista)
EVPN nécessite BGP comme underlay control plane. Les spines agissent comme route-reflectors pour réduire les peerings full-mesh.

- **Sur les spines (Route Reflectors)** :

```eos
router bgp 65000  # AS commun pour iBGP
   router-id 10.0.0.1
   bgp cluster-id 1.1.1.1  # Pour RR
   neighbor LEAF peer-group
   neighbor LEAF remote-as 65000
   neighbor LEAF update-source Loopback0
   neighbor LEAF next-hop-self
   bgp listen range 10.0.0.0/8 peer-group LEAF  # Peerings dynamiques
   !
   address-family evpn
      neighbor LEAF activate
   !
   address-family ipv4
      neighbor LEAF activate
      redistribute connected  # Annonce loopbacks/VTEPs
```

- **Sur les leaves** :

```eos
router bgp 65000
   router-id 10.0.0.3
   neighbor 10.0.0.1 remote-as 65000  # Peering avec spine1
   neighbor 10.0.0.2 remote-as 65000  # Peering avec spine2
   neighbor 10.0.0.1 update-source Loopback0
   neighbor 10.0.0.2 update-source Loopback0
   !
   address-family evpn
      neighbor 10.0.0.1 activate
      neighbor 10.0.0.2 activate
   !
   address-family ipv4
      neighbor 10.0.0.1 activate
      neighbor 10.0.0.2 activate
      redistribute connected
   !
   vlan 10
      rd 10.0.0.3:10010  # RD unique par leaf/VNI
      route-target import 1:10010
      route-target export 1:10010
      redistribute learned  # Annonce MACs learned
   !
   interface Vxlan1
      vxlan source-interface Loopback1
      vxlan udp-port 4789
      vxlan vlan 10 vni 10010
      vxlan vrf TenantA vni 50000  # Pour L3 overlay optionnel
      vxlan learn bgp  # Active EVPN pour learning
```

### Vérification et analyse
- Commandes Arista :
  - `show bgp evpn summary` : Vérifie les peerings EVPN.
  - `show bgp evpn route-type 2` : Liste les MAC/IP annoncés.
  - `show vxlan address-table` : Affiche la table MAC-VTEP (apprise via EVPN).
  - `show vxlan vtep` : Liste les VTEPs distants.
  - `ping 192.168.10.10 vrf TenantA` : Teste la connectivité overlay L3.
  - `traceroute 192.168.10.10` : Vérifie le chemin underlay (devrait montrer VTEPs).

- **Avec Wireshark** (capturez via tcpdump dans Containerlab) :
  - Filtre `udp.port == 4789` : Inspecte les headers VXLAN (VNI, inner Ethernet).
  - Filtre `tcp.port == 179` : Analyse les updates BGP EVPN (NLRI Type 2, etc.).
  - Exemple : Capturez sur une interface leaf pour voir l'encapsulation d'un ARP en VXLAN.

Si des routes manquent, vérifiez les RT/RD et les activations address-family.

## Conclusion

La virtualisation des réseaux, via un underlay robuste (OSPF/IS-IS), des overlays flexibles (VXLAN/MPLS) et un control plane intelligent (EVPN/BGP), résout les limitations des réseaux legacy en offrant scalabilité, mobilité et isolation. Dans les data centers modernes (ex. : AWS VPC, Azure VNet), ces technologies enables des services cloud dynamiques.

**Avantages globaux** :
- Scalabilité : Supporte des millions de tenants sans floods.
- Flexibilité : Mobilité VM sans reconfiguration.
- Sécurité : Isolation via VNIs et VRFs.
- Opérationnel : Automatisation via API (ex. : Ansible pour Arista).

**Prochaines étapes** : Expérimentez dans votre lab Containerlab. Ajoutez des features avancées comme Symmetric IRB pour L3 gateway, ou intégrez avec Kubernetes pour overlay SDN. Pour approfondir, consultez les RFCs (7348 pour VXLAN, 7432 pour EVPN) ou des livres comme "Building Data Centers with VXLAN BGP EVPN".

Bon apprentissage et experimentation ! 🚀 Si vous avez des questions spécifiques, n'hésitez pas.
