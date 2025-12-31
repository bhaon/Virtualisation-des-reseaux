### Cours Détailé sur le Spanning Tree Protocol (STP)

Voici un cours détaillé sur le Spanning Tree Protocol (STP), en commençant par les bases théoriques et en terminant par un lab pratique utilisant des containers Arista cEOS avec Containerlab. Le cours est structuré pour être progressif, avec des explications claires et des exemples. STP est un protocole essentiel en réseau Ethernet pour éviter les boucles et assurer une topologie sans redondance active qui pourrait causer des tempêtes de broadcast.

#### 1. Introduction au Spanning Tree Protocol
Le Spanning Tree Protocol (STP) est un protocole de niveau 2 (couche OSI 2) défini par la norme IEEE 802.1D. Son objectif principal est de créer une topologie logique sans boucle dans un réseau Ethernet commuté, tout en permettant la redondance physique pour la tolérance aux pannes.

- **Pourquoi des boucles en réseau ?** Dans un réseau avec plusieurs commutateurs interconnectés (par exemple, via des liens redondants pour la haute disponibilité), les trames Ethernet peuvent circuler en boucle infinie si rien ne les arrête. Cela provoque des "broadcast storms" (tempêtes de diffusion), où les trames broadcast/multicast sont dupliquées à l'infini, saturant le réseau et rendant les communications impossibles.
- **Solution STP :** STP calcule un arbre couvrant (spanning tree) qui connecte tous les commutateurs sans former de boucle. Il bloque certains liens redondants, mais les active automatiquement en cas de panne sur un lien principal.
- **Évolution :** L'original STP (802.1D-1998) est lent (convergence en 30-50 secondes). Des variantes plus rapides existent : Rapid STP (RSTP, 802.1w), Multiple STP (MSTP, 802.1s), et Per-VLAN STP (PVST/PVST+ propriétaire Cisco, ou RPVST sur Arista).

#### 2. Pourquoi STP est Nécessaire ?
Imaginez trois commutateurs (SW1, SW2, SW3) connectés en triangle :
- SW1 relié à SW2, SW2 à SW3, SW3 à SW1.
- Sans STP, une trame broadcast envoyée par un hôte sur SW1 sera dupliquée via les deux chemins (direct et via SW3), créant une boucle.

Conséquences :
- **Duplication des trames :** Les tables MAC apprennent des adresses erronées (flip-flopping).
- **Saturation :** Le réseau s'effondre en quelques secondes.
- **Redondance sans STP :** Pas de failover automatique.

STP résout cela en élisant un "root bridge" et en bloquant un lien, transformant le triangle en une ligne logique (arbre).

#### 3. Fonctionnement Détaillé de STP
STP repose sur l'échange de paquets appelés **BPDU (Bridge Protocol Data Units)**, envoyés toutes les 2 secondes (hello time par défaut).

##### a. Élection du Root Bridge
- Chaque commutateur a un **Bridge ID (BID)** : Priorité (multiples de 4096, défaut 32768) + Adresse MAC du commutateur.
- Le commutateur avec le BID le plus bas (priorité basse + MAC basse) devient le **Root Bridge**.
- Processus : Tous les commutateurs s'annoncent comme root initialement via BPDU. Ils comparent et propagent le meilleur BID.
- Exemple : Si SW1 a priorité 32768 et MAC 00:01, SW2 32768 et MAC 00:02, SW1 gagne (MAC plus basse).

##### b. Rôles des Ports
Après élection du root :
- **Root Port (RP) :** Le port le plus proche du root (coût le plus bas). Un par commutateur non-root.
- **Designated Port (DP) :** Le port qui forwarde le trafic vers un segment (un par segment réseau).
- **Blocking Port (BP) / Alternate Port :** Bloqué pour éviter les boucles ; écoute les BPDU mais ne forwarde pas de données.
- **Disabled Port :** Non connecté ou administrativement down.

##### c. Calcul des Coûts de Chemin (Path Cost)
- Chaque lien a un coût basé sur la vitesse : 100 Mbps = 19, 1 Gbps = 4, 10 Gbps = 2 (norme courte ; il y a aussi longue pour liens lents).
- Le chemin cumulatif le plus bas détermine les rôles.
- Exemple : Dans un triangle, supposons tous les liens 1 Gbps (coût 4). Le root (SW1) aura ses deux ports DP. SW2 et SW3 éliront leur RP vers SW1, et le lien SW2-SW3 sera bloqué.

##### d. États des Ports (Convergence)
- **Blocking :** Écoute BPDU, pas de forwarding (20s max-age).
- **Listening :** Écoute et envoie BPDU, pas de forwarding (15s forward delay).
- **Learning :** Apprend adresses MAC, pas de forwarding (15s forward delay).
- **Forwarding :** Forwarde données.
- Temps total : Jusqu'à 50s pour STP classique.

##### e. Topologie Changes
- Si un lien tombe, STP reconverge : Envoi de TCN (Topology Change Notification) pour flush les tables MAC.

#### 4. Variantes de STP
- **RSTP (Rapid STP) :** Améliore la convergence (<1s). Ajoute Proposal/Agreement pour négociation rapide. Ports : Root, Designated, Alternate (backup RP), Backup (backup DP). Compatible avec STP.
- **MSTP (Multiple STP) :** Groupe plusieurs VLAN en instances (MST Instances). Économise ressources : Un arbre par instance au lieu d'un par VLAN. Région MST : Commutateurs avec même config (nom, révision, mapping VLAN-instance).
- **PVST+/RPVST (Per-VLAN/Rapid Per-VLAN) :** Un arbre par VLAN, permet load-balancing (root différent par VLAN). Propriétaire mais interopérable.
- **Fonctionnalités Avancées :**
  - **PortFast :** Pour ports edge (vers hôtes), passe directement en forwarding.
  - **BPDU Guard :** Désactive port si BPDU reçu (protège contre commutateurs rogue).
  - **Root Guard :** Empêche un port de devenir root.
  - **Loop Guard :** Détecte liens unidirectionnels.

#### 5. Configuration de STP sur Arista EOS
Arista supporte RSTP, MSTP (défaut) et Rapid-PVST (RPVST). Voici un résumé basé sur la documentation Arista.

- **Activer un Mode :**
  ```
  configure terminal
  spanning-tree mode mstp  ! Défaut
  spanning-tree mode rstp
  spanning-tree mode rapid-pvst
  ```

- **Configurer Priorité (pour Root) :**
  ```
  spanning-tree priority 4096  ! Pour RSTP/MST instance 0
  spanning-tree root primary   ! Set priorité à 8192
  spanning-tree mst 1 priority 8192  ! Pour MST instance 1
  spanning-tree vlan-id 10 priority 16384  ! Pour PVST VLAN 10
  ```

- **Configurer Ports :**
  Sur une interface (ex: Ethernet1) :
  ```
  interface Ethernet1
  spanning-tree port-priority 128  ! Défaut
  spanning-tree cost 20000         ! Coût personnalisé
  spanning-tree portfast edge      ! Pour ports vers hôtes
  spanning-tree bpduguard enable   ! Active BPDU Guard
  spanning-tree guard root         ! Root Guard
  spanning-tree guard loop         ! Loop Guard
  ```

- **Configurer MSTP Région :**
  ```
  spanning-tree mst configuration
  name ma-region
  revision 1
  instance 1 vlans 10-20
  exit
  ```

- **Timers (Globaux) :**
  ```
  spanning-tree hello-time 2     ! Défaut 2s
  spanning-tree forward-time 15  ! Défaut 15s
  spanning-tree max-age 20       ! Défaut 20s
  ```

- **Vérification :**
  ```
  show spanning-tree               ! Status global
  show spanning-tree root          ! Root info
  show spanning-tree interface Ethernet1  ! Détails port
  show spanning-tree mst configuration    ! Région MST
  ```

Pour désactiver STP : `spanning-tree mode none` (attention, risque de boucles !).

#### 6. Lab Pratique : Simulation de STP avec Containerlab et Arista cEOS
Pour rendre cela concret, configurons un lab avec 3 commutateurs Arista cEOS en triangle (boucle physique). Nous observerons STP bloquer un lien.

##### Prérequis
- **Système Hôte :** Linux (Ubuntu recommandé) avec Docker installé.
- **Containerlab :** Installez via `bash -c "$(curl -sL https://get.containerlab.dev)"`.
- **Image cEOS :** Téléchargez depuis Arista (inscription requise) : [Arista Software Download](https://www.arista.com/en/support/software-download). Ex: `cEOS64-lab-4.32.0F.tar.xz` (ou version récente).
- Ressources : Au moins 4 Go RAM, 2 CPU pour le VM/hôte.

##### Étape 1 : Importer l'Image cEOS
```
docker import cEOS64-lab-4.32.0F.tar.xz ceos:4.32.0F
```

##### Étape 2 : Créer le Fichier de Topologie (triangle.yaml)
Créez un fichier YAML :
```yaml
name: stp-lab
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
  links:
    - endpoints: ["sw1:eth1", "sw2:eth1"]
    - endpoints: ["sw2:eth2", "sw3:eth1"]
    - endpoints: ["sw3:eth2", "sw1:eth2"]
```

Cela crée une boucle : SW1-eth1 <-> SW2-eth1, SW2-eth2 <-> SW3-eth1, SW3-eth2 <-> SW1-eth2.

##### Étape 3 : Déployer le Lab
```
containerlab deploy -t triangle.yaml
```
Attendez que les containers démarrent (vérifiez avec `docker ps`).

##### Étape 4 : Accéder aux CLI et Configurer
Accédez à chaque switch :
- SW1 : `docker exec -it clab-stp-lab-sw1 Cli`
- Même pour SW2 et SW3.

Sur chaque switch, configurez (mode enable/configure) :
```
enable
configure terminal
hostname SW1  ! Changez pour SW2/SW3
interface Ethernet1
  no shutdown
  switchport mode trunk  ! Pour permettre VLANs si besoin
interface Ethernet2
  no shutdown
  switchport mode trunk
spanning-tree mode rstp  ! Utilisons RSTP pour convergence rapide
end
write memory
```

Pour faire de SW1 le root probable : Sur SW1 :
```
configure terminal
spanning-tree priority 4096
end
write
```

##### Étape 5 : Vérifier STP
Sur chaque switch :
```
show spanning-tree
```
- Cherchez le Root Bridge (SW1 si priorité basse).
- Vérifiez les ports : Un devrait être Blocking (ex: sur SW3, un port vers SW2 pourrait être bloqué).
- Exemple sortie : Root ID, Bridge ID, Port roles (Root, Designated, Blocking).

Pour simuler une panne : Shutdown un port sur le root (ex: sur SW1, `interface eth1 shutdown`), puis revérifiez `show spanning-tree` pour voir la reconvergence (le port bloqué passe en forwarding).

##### Étape 6 : Nettoyer
```
containerlab destroy -t triangle.yaml
```

Ce lab démontre comment STP/RSTP évite les boucles. Expérimentez avec priorités ou VLANs pour MSTP/PVST. Si vous avez des erreurs (ex: image non trouvée), vérifiez les logs Docker.

### Détail des Mécanismes STP, RSTP et MSTP

Voici le détail des mécanismes de **STP (Spanning Tree Protocol, IEEE 802.1D)**, **RSTP (Rapid Spanning Tree Protocol, IEEE 802.1w)** et **MSTP (Multiple Spanning Tree Protocol, IEEE 802.1s)** pour bien comprendre les différences. Chaque protocole sera expliqué avec son fonctionnement et ses avantages/inconvénients. Ensuite, un lab pratique pour chaque mode est proposé en utilisant **Containerlab** et des containers **Arista cEOS**. Les labs utilisent une topologie simple en triangle (3 switches interconnectés) pour démontrer les boucles et la convergence.

Les labs supposent que Containerlab soit installé et que l'image cEOS soit importée (comme expliqué précédemment : `docker import cEOS-lab-<version>.tar ceos:<version>`). Le même fichier YAML de topologie est utilisé pour tous les labs, mais la configuration STP doit être changée sur les switches.

#### Fichier YAML de Topologie Commun (triangle.yaml)
```yaml
name: stp-lab
topology:
  kinds:
    arista_ceos:
      image: ceos:4.32.0F  # Remplace par ta version
  nodes:
    sw1:
      kind: arista_ceos
    sw2:
      kind: arista_ceos
    sw3:
      kind: arista_ceos
  links:
    - endpoints: ["sw1:eth1", "sw2:eth1"]
    - endpoints: ["sw2:eth2", "sw3:eth1"]
    - endpoints: ["sw3:eth2", "sw1:eth2"]
```

Déploiement avec `containerlab deploy -t triangle.yaml`. Accès aux CLI : `docker exec -it clab-stp-lab-sw<N> Cli`.

#### 1. STP (Spanning Tree Protocol) - Mécanisme Détailé
STP est le protocole original pour éviter les boucles en Ethernet. Il crée un arbre unique couvrant tous les switches sans cycle.

**Mécanisme Étape par Étape :**
- **Élection du Root Bridge :** Chaque switch envoie des BPDU (Bridge Protocol Data Units) avec son Bridge ID (BID = Priorité + MAC). Le switch avec le BID le plus bas devient root. Priorité par défaut : 32768 (multiples de 4096). Si égalité, MAC la plus basse gagne.
- **Calcul des Chemins :** Chaque lien a un coût basé sur la vitesse (ex: 1Gbps = 19 dans l'ancienne norme, 4 dans la nouvelle). Le chemin cumulatif le plus bas vers le root détermine les rôles.
- **Rôles des Ports :**
  - Root Port (RP) : Port le plus proche du root (coût min).
  - Designated Port (DP) : Port qui forwarde vers un segment (un par segment).
  - Non-Designated/Blocking Port : Bloqué pour éviter boucles.
- **États des Ports (Convergence Lente) :**
  - Disabled : Port down.
  - Blocking (20s max-age) : Écoute BPDU, pas de data.
  - Listening (15s forward delay) : Envoie/reçoit BPDU, pas de data/MAC learning.
  - Learning (15s forward delay) : Apprend MAC, pas de forwarding.
  - Forwarding : Full opération.
- Temps de convergence : 30-50s (2x forward delay + max-age). En cas de changement (ex: lien down), envoi de TCN (Topology Change Notification) pour flush MAC tables.
- **Avantages :** Simple, évite boucles.
- **Inconvénients :** Lent, un seul arbre pour tout le réseau (pas de load-balancing VLAN).

**Lab pour STP :**
Arista ne supporte pas nativement STP legacy (802.1D pur), mais RSTP est backward compatible. Pour simuler STP, il faut configurer le mode RSTP avec "spanning-tree compatibility stp" sur les interfaces pour forcer le comportement legacy (convergence lente).

1. Déploiement du lab.
2. Sur chaque switch (SW1, SW2, SW3) :
   ```
   enable
   configure terminal
   hostname SW<N>  # Remplace par 1,2,3
   interface Ethernet1
     no shutdown
     switchport mode trunk
     spanning-tree compatibility stp  # Force legacy STP
   interface Ethernet2
     no shutdown
     switchport mode trunk
     spanning-tree compatibility stp
   spanning-tree mode rstp  # Base RSTP, mais compatibility STP
   spanning-tree priority 4096  # Sur SW1 pour le rendre root
   end
   write memory
   ```
3. Vérifications :
   - `show spanning-tree` : Permet de voir le root (SW1), ports RP/DP/Blocking.
   - Simulation d'une panne : `interface eth1 shutdown` sur SW1, attendre 30-50s, revérifier la convergence.
   - On peut noter la lenteur comparée à RSTP.
4. Nettoyage : `containerlab destroy -t triangle.yaml`.

#### 2. RSTP (Rapid Spanning Tree Protocol) - Mécanisme Détailé
RSTP améliore STP pour une convergence rapide (<1-3s). Compatible avec STP (switches RSTP détectent STP et s'adaptent).

**Mécanisme Étape par Étape :**
- **Élection du Root :** Identique à STP, via BPDU.
- **Rôles des Ports (Améliorés) :**
  - Root Port : Comme STP.
  - Designated Port : Comme STP.
  - Alternate Port : Backup pour RP (bloqué, mais prêt à switcher vite).
  - Backup Port : Backup pour DP (sur le même segment).
- **États des Ports (Simplifiés) :**
  - Discarding (équiv. Blocking/Listening) : Pas de forwarding/learning.
  - Learning : Apprend MAC.
  - Forwarding : Full.
- **Convergence Rapide :** Via "Proposal/Agreement" handshake. Quand un port détecte un meilleur chemin, il propose, et l'autre agree pour transition immédiate (sans timers). Les Edge ports (vers hôtes) passent direct en forwarding.
- **Autres Améliorations :** BPDU envoyés même sur ports bloqués ; détection rapide de changements via Sync mechanism.
- **Avantages :** Convergence rapide, compatible STP.
- **Inconvénients :** Toujours un seul arbre (pas de multi-instances comme MSTP).

**Lab pour RSTP :**
1. Déploiement du lab.
2. Sur chaque switch :
   ```
   enable
   configure terminal
   hostname SW<N>
   interface Ethernet1
     no shutdown
     switchport mode trunk
   interface Ethernet2
     no shutdown
     switchport mode trunk
   spanning-tree mode rstp
   spanning-tree priority 4096  # Sur SW1
   end
   write memory
   ```
3. Vérifications :
   - `show spanning-tree` : Ports en Discarding/Learning/Forwarding, convergence rapide.
   - Simulation de panne : Shutdown un port, observation de la reconvergence en secondes.
   - `show spanning-tree detail` pour voir proposals/agreements.
4. Nettoyage.

#### 3. MSTP (Multiple Spanning Tree Protocol) - Mécanisme Détailé
MSTP étend RSTP pour plusieurs arbres (instances) mappés à des VLANs, économisant les ressources et permettant le load-balancing.

**Mécanisme Étape par Étape :**
- **Régions MST :** Switches avec même config (nom région, révision, mapping VLAN-instance) forment une région. Hors région = traités comme STP unique.
- **Instances :** 
  - IST (Internal Spanning Tree, instance 0) : Gère VLANs non mappés + CIST (Common & Internal ST) pour inter-régions.
  - MSTI (Multiple ST Instances, 1-4094) : Arbres séparés pour groupes de VLANs.
- **Élection :** Root par instance (BID par instance). CIST Root pour tout le réseau.
- **Rôles/États :** Comme RSTP, mais par instance. VLANs mappés à une instance suivent son arbre.
- **Convergence :** Rapide comme RSTP.
- **Avantages :** Économise BPDU (un par instance vs un par VLAN en PVST), load-balancing (roots différents par instance).
- **Inconvénients :** Config complexe, nécessite régions cohérentes.

**Lab pour MSTP :**
Ajout des VLANs pour démontrer les instances.

1. Déploiement du lab.
2. Sur chaque switch (même région) :
   ```
   enable
   configure terminal
   hostname SW<N>
   vlan 10,20
   interface Ethernet1
     no shutdown
     switchport mode trunk
   interface Ethernet2
     no shutdown
     switchport mode trunk
   spanning-tree mode mstp
   spanning-tree mst configuration
     name ma-region
     revision 1
     instance 1 vlan 10
     instance 2 vlan 20
   exit
   spanning-tree mst 0 priority 4096  # IST root sur SW1
   spanning-tree mst 1 priority 8192  # Instance 1 root sur SW2
   end
   write memory
   ```
3. Vérifie :
   - `show spanning-tree mst configuration` : Région et mapping.
   - `show spanning-tree mst 0` : Arbre IST.
   - `show spanning-tree mst 1` : Arbre instance 1 (différent root possible).
   - Simulation de panne et observation du load-balancing (trafic VLAN 10 via un chemin, VLAN 20 via autre).
4. Nettoyage.
   
Ces labs permettent de comparer les protocoles en pratique.
