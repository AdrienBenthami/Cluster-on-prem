# **Compte rendu 3 – Création et configuration des machines virtuelles Kubernetes sous Proxmox**

## 1. Objectif

L’objectif de cette étape est la création et la configuration des premières machines virtuelles dédiées au futur **cluster Kubernetes** du réseau de laboratoire (**192.168.100.0/24**).
Deux rôles principaux sont mis en place :

* un **nœud maître** (*control-plane*), destiné à héberger les composants centraux du cluster (API Server, etcd, scheduler, controller-manager) ;
* un **nœud de travail** (*worker*), destiné à exécuter les pods applicatifs et à supporter la charge du cluster.

Ces machines virtuelles constitueront la base du futur déploiement de **Kubernetes via kubeadm**, avec intégration progressive de :

* **Cilium** comme CNI (Container Network Interface),
* **Traefik** en tant qu’Ingress Controller,
* **MetalLB** pour la gestion des adresses IP de type LoadBalancer dans le réseau local.

Les objectifs spécifiques sont :

1. Créer des VMs stables, homogènes et performantes au sein de Proxmox ;
2. Documenter et justifier chaque choix technique (BIOS, stockage, CPU, RAM, réseau, etc.) ;
3. Identifier et résoudre les incidents rencontrés lors des déploiements ;
4. Préparer un environnement cohérent et reproductible pour la phase de déploiement du cluster Kubernetes.

---

## 2. Environnement hôte et topologie réseau

### 2.1. Infrastructure Proxmox

* **Hyperviseur :** Proxmox Virtual Environment (srv1.infra.lan)
* **Version :** 8.x (architecture Debian 12)
* **Stockages disponibles :**

  * `local-zfs` : pool ZFS hébergé sur disques durs (HDD), destiné aux machines de service.
  * `ssd-data` : pool SSD NVMe dédié aux VMs Kubernetes pour des performances d’I/O supérieures.
* **Bridge réseau interne :** `vmbr1` – connecté au réseau **LAB (192.168.100.0/24)**.
* **Bridge externe :** `vmbr0` – connecté au réseau **WAN (192.168.1.0/24)** via la Freebox.
* **Pare-feu et routage :** assurés par **OPNsense** (192.168.100.1).
* **Nom de domaine interne :** `infra.lan` (résolution interne gérée par Unbound DNS sur OPNsense).
* **Sortie Internet :** NAT sortant configuré sur OPNsense via l’interface WAN (Freebox).

Cette organisation garantit une isolation complète du réseau de laboratoire tout en maintenant un accès Internet fonctionnel pour les mises à jour, téléchargements de paquets et opérations Kubernetes.

---

## 3. Création de la machine virtuelle du nœud maître Kubernetes (`k8s-master-1`)

### 3.1. Paramètres techniques

| Élément                    | Valeur                                       | Justification                                                                            |
| -------------------------- | -------------------------------------------- | ---------------------------------------------------------------------------------------- |
| **VM ID**                  | 201                                          | Référence conforme au plan d’attribution interne (plage 200–249 pour les nœuds maîtres). |
| **Nom**                    | `k8s-master-1`                               | Convention de nommage alignée sur la topologie Kubernetes.                               |
| **Système d’exploitation** | Ubuntu Server 24.04 LTS                      | Version LTS stable, support prolongé, compatibilité native avec kubeadm et containerd.   |
| **Type de machine**        | i440FX                                       | Modèle historique de Proxmox, stable et compatible avec la majorité des distributions.   |
| **Firmware (BIOS)**        | SeaBIOS                                      | Mode BIOS simple et fiable pour les environnements ZFS.                                  |
| **Stockage**               | `local-zfs`                                  | Volume ZFS performant, support des snapshots et de la compression.                       |
| **Disque principal**       | 40 Go, format `raw`, discard=on, iothread=on | Volume adapté à un nœud de contrôle (sans conteneurs applicatifs).                       |
| **Contrôleur disque**      | VirtIO SCSI (single)                         | Interface para-virtualisée optimisée pour Linux.                                         |
| **Cache disque**           | none                                         | Évite le double cache entre QEMU et ZFS, garantissant la cohérence des écritures.        |
| **CPU**                    | 1 socket, 2 cœurs, type `x86-64-v2-AES`      | Bon compromis entre performance, compatibilité et migration possible.                    |
| **RAM**                    | 8 Go fixes, ballooning désactivé             | Stabilité nécessaire pour les processus kubelet, API Server et etcd.                     |
| **Carte réseau**           | VirtIO, bridge `vmbr1`, multiqueue=4         | Haut débit, latence faible, optimisé pour trafic interne.                                |
| **Adresse IP**             | 192.168.100.31/24                            | Statique, conforme au plan d’adressage du LAB.                                           |
| **Qemu Agent**             | Activé                                       | Communication simplifiée entre l’hyperviseur et la VM.                                   |
| **Tags Proxmox**           | `k8s;master;lab`                             | Facilite la gestion et le filtrage des VMs dans l’interface Proxmox.                     |

### 3.2. Justification des choix

* Le **ballooning** est désactivé pour éviter toute fluctuation de mémoire pouvant perturber le kubelet.
* Le **cache disque désactivé** empêche les incohérences entre le cache ZFS et celui de QEMU.
* Le **BIOS SeaBIOS** et le chipset **i440FX** ont été privilégiés pour garantir une compatibilité maximale avec l’existant.
* Ce nœud ne nécessitant pas de forte charge applicative, une allocation de 2 vCPU et 8 Go de RAM est suffisante.

---

## 4. Création de la machine virtuelle du nœud de travail Kubernetes (`k8s-worker-1`)

### 4.1. Premier déploiement – échec d’installation

Une première tentative d’installation d’**Ubuntu Server 24.04 LTS** sur une VM configurée de manière identique au nœud maître (SeaBIOS + i440FX) a échoué au cours du processus `curtin install` :

**Message d’erreur :**

> “Sorry, there was a problem completing the installation.”
> Échec lors de l’installation du chargeur de démarrage GRUB sur le disque virtuel.

**Analyse technique :**

* Le partitionnement GPT généré par l’installeur Ubuntu n’était pas pris en charge par SeaBIOS.
* Le contrôleur de disque VirtIO nécessitait un environnement UEFI pour initialiser correctement le disque sur un support SSD.
* Le firmware SeaBIOS ne supportait pas la configuration EFI, provoquant l’échec de GRUB.

**Décision corrective :**
Recréation complète de la VM avec :

* **Chipset Q35** (plus moderne, basé sur PCIe),
* **Firmware UEFI (OVMF)**,
* **Support TPM 2.0** et **EFI Disk**,
* Stockage migré vers **ssd-data** pour de meilleures performances I/O.

---

### 4.2. Configuration corrigée et validée

| Élément                    | Valeur                                          | Justification                                                                   |
| -------------------------- | ----------------------------------------------- | ------------------------------------------------------------------------------- |
| **VM ID**                  | 251                                             | Conformité au plan d’attribution (plage 250–299 dédiée aux workers).            |
| **Nom**                    | `k8s-worker-1`                                  | Cohérence avec la convention interne.                                           |
| **Système d’exploitation** | Ubuntu Server 24.04 LTS                         | Uniformité entre nœuds du cluster.                                              |
| **Type de machine**        | Q35                                             | Support complet de PCIe, UEFI, Secure Boot, meilleure compatibilité matérielle. |
| **Firmware (BIOS)**        | OVMF (UEFI)                                     | Support de GRUB EFI, nécessaire au démarrage sur GPT.                           |
| **Disque EFI**             | `ssd-data:1`, format `raw`, pre-enrolled-keys=1 | Disque de boot UEFI sécurisé.                                                   |
| **TPM State**              | `v2.0`                                          | Conformité Secure Boot, meilleure sécurité.                                     |
| **Stockage**               | `ssd-data`                                      | Volume SSD dédié aux workloads Kubernetes.                                      |
| **Disque principal**       | 60 Go, format `raw`, discard=on, iothread=on    | Capacité accrue pour accueillir les images conteneurs.                          |
| **Contrôleur disque**      | VirtIO SCSI (single)                            | Interface performante et stable pour Linux.                                     |
| **CPU**                    | 1 socket, 4 cœurs, type `x86-64-v2-AES`         | Puissance suffisante pour les pods applicatifs.                                 |
| **RAM**                    | 8 Go fixes                                      | Charge stable, adaptée aux workloads Kubernetes.                                |
| **Carte réseau**           | VirtIO, bridge `vmbr1`                          | Accès direct au réseau LAB.                                                     |
| **Adresse IP**             | 192.168.100.51/24                               | Statique selon le plan d’adressage.                                             |
| **Qemu Agent**             | Activé                                          | Gestion centralisée depuis Proxmox.                                             |
| **Tags Proxmox**           | `k8s;worker;lab`                                | Identification et filtrage rapide.                                              |

### 4.3. Résultat

L’installation a été menée à son terme sans erreur.
Les tests de connectivité (`ping 1.1.1.1`, `ping google.com`) ont confirmé :

* une connectivité WAN fonctionnelle via OPNsense ;
* un routage correct ;
* et une résolution DNS stable grâce au service Unbound.

---

## 5. Problèmes rencontrés et résolutions

| Problème rencontré                                      | Analyse                                              | Résolution                                                    |
| ------------------------------------------------------- | ---------------------------------------------------- | ------------------------------------------------------------- |
| **Échec d’installation Ubuntu (SeaBIOS)**               | Incompatibilité BIOS/GPT lors de l’installation GRUB | Recréation de la VM sous UEFI (OVMF) + Q35                    |
| **Perte de résolution DNS sur le master**               | DNS local OPNsense non joint                         | Vérification Unbound, ajout `nameserver 192.168.100.1`        |
| **Fichier Netplan vide après installation**             | Génération automatique par Cloud-Init non utilisée   | Création manuelle du fichier `/etc/netplan/01-netcfg.yaml`    |
| **Ping externe fonctionnel mais pas de résolution DNS** | Requête DNS bloquée ou non transmise à OPNsense      | Correction via configuration Unbound (interfaces LAN/WAN/WG0) |

---

## 6. Architecture logique

```
                  ┌─────────────────────────────────────┐
                  │            Proxmox VE (srv1)        │
                  │-------------------------------------│
                  │ vmbr0 → WAN (Freebox 192.168.1.0/24)│
                  │ vmbr1 → LAB (192.168.100.0/24)      │
                  │-------------------------------------│
                  │               VMs                   │
                  │  [100] gw-opnsense → 192.168.100.1  │
                  │  [201] k8s-master-1 → 192.168.100.31│
                  │  [251] k8s-worker-1 → 192.168.100.51│
                  └─────────────────────────────────────┘
```

Les deux machines virtuelles sont isolées sur le réseau interne LAB (`vmbr1`) et accèdent à Internet par le NAT sortant d’OPNsense.
OPNsense fournit à la fois la passerelle par défaut et le DNS interne (Unbound), assurant une résolution cohérente sur le domaine `infra.lan`.

---

## 7. Synthèse des choix architecturaux

| Décision                             | Justification technique                                                                   |
| ------------------------------------ | ----------------------------------------------------------------------------------------- |
| **Utilisation de VMs et non de LXC** | Kubernetes nécessite un kernel complet, une isolation système et un contrôle des cgroups. |
| **Ubuntu Server 24.04 LTS**          | Système stable, récent, et compatible avec les versions actuelles de Kubernetes.          |
| **Q35 + OVMF (UEFI)**                | Support PCIe, Secure Boot et meilleure compatibilité moderne.                             |
| **VirtIO (réseau/disque)**           | Faible overhead, support natif Linux et Cloud-Init.                                       |
| **ZFS / SSD en mode raw**            | Meilleures performances d’accès disque et support du TRIM.                                |
| **Mémoire fixe (8 Go)**              | Garantit la stabilité des nœuds Kubernetes.                                               |
| **Bridge réseau vmbr1**              | Isolation du cluster et maîtrise du routage via OPNsense.                                 |
| **TPM 2.0 et Secure Boot**           | Sécurité renforcée et compatibilité Ubuntu moderne.                                       |
| **CPU x86-64-v2-AES**                | Optimisation inter-nœuds, support AES pour chiffrement réseau.                            |

---

## 8. Vérifications post-création

* **Agent Qemu :** actif et fonctionnel.
* **Bridge réseau :** `vmbr1` sur chaque VM.
* **Disques :** `discard=on`, `iothread=on`, format `raw`.
* **Tags :** cohérents et exploitables (`k8s;master`, `k8s;worker`).
* **Démarrage automatique :** activé (`onboot=1`).
* **Résolution DNS :** fonctionnelle.
* **Connectivité externe :** validée vers `1.1.1.1` et `google.com`.

---

## 9. État actuel

Les deux machines virtuelles sont installées, fonctionnelles et intégrées au réseau LAB :

| VM             | Adresse IP     | Rôle        | Stockage    | Statut      |
| -------------- | -------------- | ----------- | ----------- | ----------- |
| `k8s-master-1` | 192.168.100.31 | Nœud maître | `local-zfs` | Fonctionnel |
| `k8s-worker-1` | 192.168.100.51 | Nœud worker | `ssd-data`  | Fonctionnel |

L’environnement est désormais prêt pour l’installation du cluster Kubernetes via **kubeadm** :

1. Installation des dépendances (`containerd`, `kubeadm`, `kubelet`, `kubectl`) ;
2. Initialisation du cluster (`kubeadm init`) ;
3. Intégration du nœud worker (`kubeadm join`).

---

## 10. Conclusion

La mise en place des deux premières machines virtuelles Kubernetes sous Proxmox constitue une base solide et conforme aux bonnes pratiques d’infrastructure virtualisée.
Les choix techniques opérés – migration vers Q35 et UEFI, désactivation du ballooning, usage du stockage SSD et du type de CPU `x86-64-v2-AES` – garantissent un environnement performant, durable et homogène.

Les difficultés rencontrées (notamment l’échec d’installation du worker sous SeaBIOS) ont permis de valider des solutions modernes (UEFI, TPM 2.0, Secure Boot) plus adaptées à Ubuntu 24.04 et à l’architecture Kubernetes actuelle.

L’ensemble du dispositif est prêt à accueillir le déploiement du cluster Kubernetes, dans un cadre isolé, contrôlé et extensible, conforme aux standards d’une infrastructure de laboratoire professionnelle.

