# Cours : Mise en Place de Routage BGP via Cilium sur un Cluster Kubernetes KIND pour Annoncer les Routes vers un Commutateur Arista cEOS dans Containerlab

Ce cours vous guide pas à pas pour configurer un routage BGP avec Cilium sur un cluster Kubernetes KIND (Kubernetes IN Docker). Nous utiliserons Containerlab pour orchestrer le laboratoire, en intégrant un cluster KIND et un commutateur virtuel Arista cEOS. L'objectif est d'annoncer les routes du cluster (ex. : pod CIDR) vers le commutateur Arista via BGP, permettant une connectivité externe au cluster.

Ce setup est idéal pour tester des scénarios de data center où Kubernetes s'intègre avec un réseau legacy via BGP.

**Prérequis** :
- Un hôte Linux avec Docker installé.
- Containerlab installé (voir cours précédent sur Containerlab).
- Helm installé (pour Cilium).
- kubectl installé.
- Image Arista cEOS téléchargée et importée dans Docker (ex. : `docker import ceosimage.tar ceos:4.32.0F`).
- Connaissances basiques en Kubernetes, BGP et YAML.

**Topologie** :
- Un cluster KIND avec 1 control-plane et 1 worker.
- Un commutateur Arista cEOS connecté au worker KIND via un lien point-to-point (pour le peering BGP).
- Adressage : Lien entre cEOS (eth1 : 192.168.1.1/31) et KIND worker (eth1 : 192.168.1.0/31).
- AS BGP : 65000 pour Arista, 65001 pour le cluster Cilium.
- Routes annoncées : Pod CIDR du cluster (par défaut 10.244.0.0/16 dans KIND).

## 1. Configuration du Laboratoire dans Containerlab

Utilisez Containerlab pour déployer le cluster KIND et le cEOS. Le kind `k8s-kind` de Containerlab déploie le cluster KIND, et `ext-container` expose les nœuds KIND pour l'intégration réseau.

Créez le fichier `bgp-cilium-kind.clab.yml` :

```yaml
name: bgp-cilium-kind
topology:
  nodes:
    ceos:
      kind: arista_ceos
      image: ceos:4.32.0F  # Remplacez par votre version
    k8s-cluster:
      kind: k8s-kind
      startup-config: kind-config.yaml  # Fichier de config KIND ci-dessous
    k8s-control-plane:
      kind: ext-container
      exec: "ip addr add dev eth1 192.168.1.0/31"  # IP pour peering BGP
    k8s-worker:
      kind: ext-container
      exec: "ip addr add dev eth1 192.168.1.1/31"  # IP pour peering BGP
  links:
    - endpoints: ["ceos:eth1", "k8s-worker:eth1"]  # Lien pour BGP
```

Créez `kind-config.yaml` pour le cluster KIND (1 control-plane, 1 worker) :

```yaml
apiVersion: kind.x-k8s.io/v1alpha4
kind: Cluster
nodes:
  - role: control-plane
  - role: worker
networking:
  podSubnet: "10.244.0.0/16"  # CIDR à annoncer via BGP
  serviceSubnet: "10.96.0.0/12"
```

Déployez le lab :

```bash
containerlab deploy -t bgp-cilium-kind.clab.yml
```

Vérifiez :
- `containerlab inspect -a` : Liste les nœuds.
- Accédez au cEOS : `docker exec -it clab-bgp-cilium-kind-ceos Cli`
- Vérifiez l'IP sur le worker KIND : `docker exec -it clab-bgp-cilium-kind-k8s-worker ip a show eth1`

Le cluster KIND est accessible via kubectl sur l'hôte (Containerlab configure automatiquement le kubeconfig).

## 2. Installation de Cilium sur le Cluster KIND

Cilium est installé comme CNI avec BGP activé. Utilisez Helm pour une installation flexible.

Installez Helm si nécessaire : `curl https://raw.githubusercontent.com/helm/helm/master/scripts/get-helm-3 | bash`

Ajoutez le repo Helm de Cilium :

```bash
helm repo add cilium https://helm.cilium.io/
helm repo update
```

Installez Cilium avec BGP Control Plane activé :

```bash
helm install cilium cilium/cilium --version 1.15.0 \
  --namespace kube-system \
  --set tunnel=disabled \  # Mode natif pour BGP
  --set bgpControlPlane.enabled=true \  # Active BGP
  --set ipam.mode=kubernetes  # IPAM Kubernetes
```

Vérifiez l'installation :

```bash
kubectl get pods -n kube-system | grep cilium
```

Tous les pods Cilium doivent être en Running.

## 3. Configuration BGP dans Cilium

Créez une ressource CiliumBGPConfig pour définir le peering BGP. Cela configure un router BGP virtuel sur les nœuds Kubernetes, avec peering vers l'Arista cEOS.

Créez `cilium-bgp-config.yaml` :

```yaml
apiVersion: cilium.io/v2alpha1
kind: CiliumBGPConfig
metadata:
  name: bgp-config
spec:
  bgpInstances:
  - name: "bgp-instance"
    localASN: 65001  # AS du cluster
    peers:
    - name: "arista-peer"
      peerAddress: "192.168.1.0/31"  # IP du worker KIND (peer avec cEOS)
      peerASN: 65000  # AS de l'Arista
      eBGPMultihopTTL: 2  # Optionnel pour multi-hop si besoin
  exportPodCIDR: true  # Annonce le pod CIDR (10.244.0.0/16)
```

Appliquez :

```bash
kubectl apply -f cilium-bgp-config.yaml
```

Vérifiez le statut BGP dans Cilium :

```bash
cilium status --wait  # Vérifie l'installation globale
cilium bgp peers  # Affiche les peers BGP
```

Le peering doit être en état "Established".

## 4. Configuration BGP sur le Commutateur Arista cEOS

Configurez le cEOS pour peer avec le nœud KIND worker.

Entrez en mode CLI sur cEOS :

```bash
docker exec -it clab-bgp-cilium-kind-ceos Cli
enable
configure terminal
```

Configuration BGP :

```eos
interface Ethernet1
   no switchport
   ip address 192.168.1.1/31  # IP locale
   no shutdown

router bgp 65000  # AS Arista
   neighbor 192.168.1.0 remote-as 65001  # Peer avec KIND worker
   neighbor 192.168.1.0 maximum-routes 12000  # Limite optionnelle
   maximum-paths 32  # ECMP si multiple paths
```

Sauvegardez : `write memory`

Vérifiez :

```eos
show ip bgp summary  # Doit montrer Established
show ip route bgp  # Doit montrer la route 10.244.0.0/16 annoncée
```

## 5. Vérification et Analyse

- **Vérifiez les routes sur Arista** : `show ip route` devrait inclure 10.244.0.0/16 via BGP.
- **Test de connectivité** : Déployez un pod dans le cluster KIND (ex. : `kubectl run nginx --image=nginx`) et pinguez son IP pod depuis Arista (ex. : `ping <pod-ip>`). Si besoin, ajoutez des routes statiques ou testez avec des outils.
- **Capture de trafic** : Utilisez la capture dans Containerlab (voir cours précédent) sur l'interface eth1 du worker : 
  ```bash
  sudo ip netns exec clab-bgp-cilium-kind-k8s-worker tcpdump -nni eth1 port 179 -w bgp.pcap
  ```
  Analysez avec Wireshark pour voir les updates BGP (annonces de routes).

- **Dépannage courant** :
  - Peering non established : Vérifiez IPs, AS, et firewall (ex. : allow TCP 179).
  - Routes non annoncées : Assurez-vous que `exportPodCIDR: true` et que Cilium voit les pods.
  - Utilisez `cilium bgp routes` pour voir les routes exportées.

## 6. Conclusion

Ce laboratoire démontre comment Cilium BGP Control Plane permet d'intégrer un cluster Kubernetes avec un réseau externe via BGP, sans MetalLB ou d'autres outils. Dans un environnement de production, étendez à plusieurs peers, annonces de services LoadBalancer, ou multi-cluster avec ClusterMesh.

Pour approfondir :
- Docs Cilium BGP : https://docs.cilium.io/en/stable/network/bgp-control-plane/
- Exemples Containerlab : https://containerlab.dev/lab-examples/

Détruisez le lab : `containerlab destroy -t bgp-cilium-kind.clab.yml`

Bon apprentissage ! 🚀 Si des erreurs, vérifiez les versions et logs (ex. : `kubectl logs -n kube-system -l app.kubernetes.io/name=cilium-operator`).
