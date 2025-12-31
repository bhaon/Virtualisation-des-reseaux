# Cours : Virtualisation des Réseaux Informatiques

Ce cours aborde les concepts fondamentaux de la virtualisation des réseaux dans les data centers modernes, en se concentrant sur les architectures **Underlay/Overlay** et les **Control Planes**.  

Les démonstrations pratiques utilisent des commutateurs virtuels **Arista cEOS** en mode container, déployés via **Containerlab**.

> **Prérequis** : Connaissances de base en réseaux (OSI, switching, routing IP) et familiarité avec la ligne de commande.

## Configuration du Laboratoire Containerlab

Pour reproduire les exemples, utilisez un fichier `topology.yaml` Containerlab avec une topologie leaf-spine simple (2 spines, 2 leaves) :

```yaml
name: vxlan-evpn-lab
topology:
  nodes:
    spine1:
      kind: ceos
      image: ceos:4.35.0F
    spine2:
      kind: ceos
      image: ceos:4.35.0F
    leaf1:
      kind: ceos
      image: ceos:4.35.0F
    leaf2:
      kind: ceos
      image: ceos:4.35.0F
  links:
    - endpoints: ["spine1:eth1", "leaf1:eth1"]
    - endpoints: ["spine1:eth2", "leaf2:eth1"]
    - endpoints: ["spine2:eth1", "leaf1:eth2"]
    - endpoints: ["spine2:eth2", "leaf2:eth2"]
```

Démarrez le lab avec :

```bash
containerlab deploy -t topology.yaml
```

Accédez à un switch avec :

```bash
docker exec -it clab-vxlan-evpn-lab-spine1 Cli
```

## Introduction : Problèmes à Résoudre

### Problèmes des réseaux de niveau 2
- **Boucles** : Risque de broadcast storm si redondance non contrôlée.
- **ARP et Broadcast** : Requêtes ARP et broadcasts inondent tout le domaine L2.
- **Pas de Control Plane** : Aucune découverte dynamique avancée des hôtes.
- **Limites d’échelle** : Domaine L2 trop large = instabilité.

### Solutions legacy niveau 2
- **Spanning Tree Protocol (STP/RSTP/MSTP)** : Bloque les liens redondants → convergence lente, sous-utilisation.
- **LACP** : Agrégation de liens pour bande passante et redondance.
- **MC-LAG** : LACP multi-châssis pour haute disponibilité sans STP.

### Problèmes des réseaux de niveau 3
- **Table de routage unique** : Impossible d’isoler les tenants (multi-tenancy).
- **Sécurité** : Manque de segmentation native entre domaines.

### Solutions legacy niveau 3
- **VRF / VRF Lite** : Instances de routage virtuelles isolées.
- Limites : Complexité de gestion et difficulté à scaler dans les clouds.

## Un Réseau Underlay : Infrastructure de Support à la Virtualisation

### Interface Loopback
Interface virtuelle toujours up, utilisée comme identifiant stable du routeur.

```eos
interface Loopback0
   description Router ID / VTEP source
   ip address 10.0.0.1/32
```

### Configuration des interfaces d’interconnexion
Utilisation d’adressage point-to-point (/31) pour économiser les IPs :

```eos
interface Ethernet1
   no switchport
   ip address 192.168.1.0/31
```

### Protocoles de routage IGP : OSPF vs IS-IS
- **OSPF** et **IS-IS** sont des protocoles link-state à convergence rapide.
- Choix privilégié en data center car :
  - Scalables (milliers de routes)
  - Convergence sub-seconde
  - Standards ouverts (multi-vendor)
  - Simples à configurer en underlay

**Pourquoi pas RIP ou EIGRP ?**
- RIP : trop lent, limité à 15 hops, pas hiérarchique.
- EIGRP : propriétaire Cisco, moins adapté aux topologies modernes multi-vendor.

### Mise en place d’OSPF

```eos
router ospf 1
   router-id 10.0.0.1
   passive-interface Loopback0
   network 10.0.0.1/32 area 0
   network 192.168.0.0/16 area 0
   max-lsa 12000
```

Sur chaque interface physique :

```eos
interface Ethernet1
   ip ospf network point-to-point
   ip ospf area 0
```

### Vérification du routage Underlay
```eos
show ip route ospf
show ip ospf neighbor
show ip ospf database
ping 10.0.0.4 source Loopback0   # Toutes les loopbacks doivent être atteignables
traceroute 10.0.0.4
```

## Les Réseaux Overlay

### Mécanisme de tunnelisation (encapsulation générique)
Un paquet original (L2 ou L3) est encapsulé dans un nouveau paquet IP traversant l’underlay.

### Cas MPLS
- Utilise des **labels** pour le forwarding.
- **LDP** distribue les labels dynamiquement.
- Header MPLS : 32 bits inséré entre L2 et L3.

### Cas VXLAN (dominant aujourd’hui)
- **VXLAN** : extension VLAN avec 24 bits de VNI → 16 millions de segments.
- **VTEP** (VXLAN Tunnel Endpoint) : point d’encapsulation/décapsulation.
- Header VXLAN : UDP (port 4789) + 8 octets VXLAN (dont VNI 24 bits).

### Configuration VXLAN basique (Flood & Learn)

```eos
interface Loopback1
   description VTEP
   ip address 10.1.1.1/32

interface Vxlan1
   vxlan source-interface Loopback1
   vxlan udp-port 4789
   vxlan vlan 10 vni 10010
   vxlan vlan 20 vni 10020
```

## Le Control Plane

### Mode legacy : Flood and Learn
- Inondation des BUM (Broadcast, Unknown unicast, Multicast) via multicast ou head-end replication.
- Apprentissage MAC par observation du trafic retour.
- Problèmes : floods excessifs, pas scalable.

### Rappel BGP / MP-BGP
- **BGP** : protocole de routage scalable.
- **MP-BGP** : extensions multi-protocoles (address-family EVPN, IPv6, etc.).

### LISP (Locator/ID Separation Protocol)
Sépare l’identité (EID) de la localisation (RLOC).  
Utilise un Mapping Server pour résoudre les destinations.

### EVPN (Ethernet VPN) – Standard actuel
- Utilise **MP-BGP** pour échanger :
  - Routes MAC (Type 2)
  - Routes MAC+IP (Type 2)
  - Préfixes IP (Type 5)
- Évite les floods grâce à l’apprentissage proactif via BGP.
- Supporte mobilité des VMs, multi-tenancy, L2 et L3 overlay.

### Configuration EVPN + VXLAN (Arista)

**Sur les spines (Route Reflectors)** :

```eos
router bgp 65000
   router-id 10.0.0.1
   bgp cluster-id 1.1.1.1
   neighbor LEAF peer group
   neighbor LEAF remote-as 65000
   neighbor LEAF next-hop-self
   neighbor 10.0.0.0/8 peer group LEAF   # iBGP dynamique si besoin
   !
   address-family evpn
      neighbor LEAF activate
```

**Sur les leaves** :

```eos
router bgp 65000
   router-id 10.0.0.3
   neighbor 10.0.0.1 remote-as 65000   # spine1
   neighbor 10.0.0.2 remote-as 65000   # spine2
   !
   address-family evpn
      neighbor 10.0.0.1 activate
      neighbor 10.0.0.2 activate
   !
   vlan 10
      rd 10.0.0.3:10010
      route-target both 1:10010
      redistribute learned
   !
   interface Vxlan1
      vxlan source-interface Loopback1
      vxlan vlan 10 vni 10010
      vxlan learn bgp
```

### Vérification et analyse

```eos
show bgp evpn summary
show bgp evpn route-type 2
show vxlan address-table
show vxlan vtep
ping 192.168.10.10 vrf TenantA
traceroute 192.168.10.10
```

**Avec Wireshark** :
- Filtrez `udp.port == 4789` pour voir les headers VXLAN.
- Filtrez `tcp.port == 179` pour les updates BGP EVPN.

## Conclusion

La virtualisation des réseaux avec une architecture **Underlay IP (OSPF/IS-IS)**, **Overlay VXLAN**, et **Control Plane EVPN/BGP** représente l’état de l’art des data centers modernes. Elle permet :

- Une scalabilité massive
- Une mobilité des workloads
- Une isolation multi-tenant
- Une résilience et une simplicité opérationnelle

En pratiquant sur un lab Containerlab avec Arista cEOS, vous pouvez reproduire ces concepts et les adapter à des topologies plus complexes (route servers, symmetric IRB, etc.).

Bon apprentissage ! 🚀