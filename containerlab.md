# Cours : Utilisation de Containerlab

Ce cours complet vous guide dans l'utilisation de **Containerlab**, un outil open-source puissant pour déployer et gérer des laboratoires réseaux basés sur des containers Docker. Containerlab permet de créer rapidement des topologies réseaux multi-vendor (Arista cEOS, Nokia SR Linux, Cumulus, Cisco XRd, etc.) pour tester configurations, protocoles, automation ou validation de designs.

**Prérequis** :  
- Un système Linux (Ubuntu recommandé).  
- Docker installé et en marche.  
- Connaissances basiques de YAML et de la ligne de commande.

**Date de mise à jour** : Décembre 2025 (basé sur la version la plus récente disponible, autour de 0.72.x).

## 1. Introduction à Containerlab

Containerlab est un orchestrateur de labs réseaux containerisés. Il permet de :
- Déployer des topologies déclaratives via un fichier YAML (.clab.yml).
- Interconnecter automatiquement les containers (virtual wiring).
- Gérer le cycle de vie des labs : deploy, destroy, inspect, graph, etc.
- Supporter de nombreux Network Operating Systems (NOS) containerisés.
- Intégrer des hôtes Linux, des outils de test ou des VMs via vrnetlab.

Avantages :
- Gratuit et open-source (développé par Nokia/SRLabs).
- Très rapide et léger (pas besoin de GNS3/EVE-NG lourds).
- Idéal pour NetDevOps, CI/CD, tests automatisés.

Site officiel : https://containerlab.dev

## 2. Installation

La méthode la plus simple :

```bash
# Télécharge et installe la dernière version
bash -c "$(curl -sL https://get.containerlab.dev)"
```

Vérifiez l'installation :

```bash
containerlab version
```

Si Docker n'est pas installé, utilisez le script d'installation complet :

```bash
curl -sL https://containerlab.dev/setup | sudo -E bash -s "all"
```

Pour une version spécifique :

```bash
bash -c "$(curl -sL https://get.containerlab.dev)" -- -v 0.72.0
```

Sur macOS/Windows : Utilisez un VM Linux ou un devcontainer VS Code (voir docs officielles).

Mise à jour :

```bash
sudo containerlab version upgrade
```

## 3. Premier Lab : Quickstart

Containerlab inclut des exemples prêts à l'emploi.

Exemple simple (Nokia SR Linux + Arista cEOS) :

```bash
# Créez un répertoire
mkdir ~/clab-demo && cd ~/clab-demo

# Téléchargez un exemple
curl -LO https://raw.githubusercontent.com/srl-labs/containerlab/main/lab-examples/srlceos01/srlceos01.clab.yml
```

Contenu du fichier `srlceos01.clab.yml` :

```yaml
name: srlceos01
topology:
  nodes:
    srl:
      kind: nokia_srlinux
      image: ghcr.io/nokia/srlinux:24.10
    ceos:
      kind: arista_ceos
      image: ceos:4.32.0F
  links:
    - endpoints: ["srl:ethernet-1/1", "ceos:eth1"]
```

Déployez le lab :

```bash
containerlab deploy -t srlceos01.clab.yml
```

Containerlab télécharge les images si nécessaire et interconnecte les nodes.

Accédez aux nodes :
- SR Linux : `docker exec -it clab-srlceos01-srl sr_cli`
- cEOS : `docker exec -it clab-srlceos01-ceos Cli`

Détruisez le lab :

```bash
containerlab destroy -t srlceos01.clab.yml
```

## 4. Structure d'un Fichier de Topologie (.clab.yml)

Un fichier YAML avec sections principales :

```yaml
name: mon-lab                  # Nom du lab
mgmt:                          # Optionnel : réseau management
  network: clab-mgmt
  ipv4-subnet: 172.20.20.0/24

topology:
  kinds:                       # Configurations par kind (optionnel)
    arista_ceos:
      image: ceos:4.32.0F
      startup-config: ceos.cfg

  nodes:                       # Liste des nodes
    spine1:
      kind: arista_ceos        # Type de node
      image: ceos:4.32.0F      # Image Docker
      startup-config: spine1.cfg  # Config initiale (bind)
    leaf1:
      kind: cumulus
      image: cumulus:5.0.0

  links:                       # Connexions
    - endpoints: ["spine1:eth1", "leaf1:swp1"]
    - endpoints: ["spine1:eth2", "leaf2:swp1"]
```

- **kinds** : Définit des paramètres communs par type de node.
- **nodes** : Chaque node a un nom unique, kind, image, etc.
- **links** : Connexions point-to-point ou via bridges.

## 5. Kinds Supportés (Exemples Populaires)

- `arista_ceos` : Arista cEOS (image à télécharger manuellement depuis Arista).
- `nokia_srlinux` : Nokia SR Linux.
- `cumulus` : NVIDIA Cumulus Linux.
- `linux` : Hôte Linux générique (Alpine/Ubuntu).
- `vr-*` : Via vrnetlab (ex. : vr-veos, vr-csr, vr-vmx pour VMs).

Pour Arista cEOS :
- Téléchargez l'image depuis le site Arista.
- Importez : `docker import ceos-image.tar ceos:4.32.0F`

## 6. Commandes Principales

| Commande                          | Description                                      |
|-----------------------------------|--------------------------------------------------|
| `containerlab deploy -t topo.clab.yml` | Déploie un lab                                   |
| `containerlab destroy --all`      | Détruit tous les labs                            |
| `containerlab inspect -a`         | Liste tous les labs et nodes                     |
| `containerlab graph -t topo.clab.yml` | Génère un diagramme (ouvre dans navigateur)      |
| `containerlab save -t topo.clab.yml` | Sauvegarde les configs running                   |
| `containerlab tools ssh ...`      | Outils avancés (ex. : exec sur nodes)            |

## 7. Exemple Avancé : Topologie Leaf-Spine avec Arista cEOS

Fichier `leaf-spine.clab.yml` :

```yaml
name: leaf-spine-evpn
topology:
  nodes:
    spine1:
      kind: arista_ceos
      image: ceos:4.32.0F
    spine2:
      kind: arista_ceos
      image: ceos:4.32.0F
    leaf1:
      kind: arista_ceos
      image: ceos:4.32.0F
    leaf2:
      kind: arista_ceos
      image: ceos:4.32.0F
  links:
    - endpoints: ["spine1:eth1", "leaf1:eth1"]
    - endpoints: ["spine1:eth2", "leaf2:eth1"]
    - endpoints: ["spine2:eth1", "leaf1:eth2"]
    - endpoints: ["spine2:eth2", "leaf2:eth2"]
```

Déployez et configurez EVPN/VXLAN comme dans les cours précédents !

## 8. Astuces et Bonnes Pratiques

- **Startup-config** : Placez des fichiers .cfg et utilisez `binds` ou `startup-config`.
- **Graph** : Visualisez avec `containerlab graph`.
- **Intégration VS Code** : Extension officielle pour édition visuelle.
- **Automation** : Combinez avec Ansible/Netmiko pour config auto.
- **Performances** : Utilisez des MTU jumbo si VXLAN.
- **Dépannage** : `docker logs <container>` ou `containerlab inspect`.

## 9. Conclusion

Containerlab révolutionne les labs réseaux en rendant les topologies reproductibles, rapides et intégrables dans des workflows DevOps. Commencez par les exemples officiels, puis créez vos propres labs multi-vendor.

Ressources :
- Documentation : https://containerlab.dev
- Lab examples : https://containerlab.dev/lab-examples/
- GitHub : https://github.com/srl-labs/containerlab

Amusez-vous bien à builder vos labs ! 🚀 Si vous avez des questions, explorez la communauté Discord ou GitHub.
