# **Compte rendu 4 – Installation complète de Kubernetes (master & worker) et choix/installation de Lens**

## 1. Objectif

Mettre en service un cluster **Kubernetes v1.34.x** minimal, composé d’un **nœud maître** (`k8s-master-1`) et d’un **nœud worker** (`k8s-worker-1`), prêt à accueillir un CNI **Cilium** ainsi que l’outillage d’observabilité côté poste d’admin via **Lens**.
Ce livrable couvre **uniquement** l’installation de Kubernetes sur les nœuds et la mise en place de Lens (connexion au cluster). Tout ce qui relève du stockage, du monitoring et du troubleshooting avancé est traité dans le **Compte rendu 5**.

---

## 2. Périmètre et prérequis

### 2.1. Contexte matériel et réseau

* Hyperviseur : **Proxmox VE 8.x**.
* Pare-feu / routage / DNS : **OPNsense** (`192.168.100.1`).
* Réseau LAB : **192.168.100.0/24**.
* Nœuds :

  * `k8s-master-1` – **192.168.100.31** – Ubuntu Server **24.04 LTS** – 2 vCPU / 16 Go RAM (RAM augmentée à 16 Go après l'installation) / 40 Go disque (cf. CR-3).
  * `k8s-worker-1` – **192.168.100.51** – Ubuntu Server **24.04 LTS** – 4 vCPU / 16 Go RAM (RAM augmentée à 16 Go après l'installation) / 60 Go disque (cf. CR-3).

### 2.2. Contraintes Kubernetes

* **Swap désactivé** (obligatoire).
* **Container runtime** : **containerd** (cgroups systemd).
* **Paramètres noyau** actifs : `overlay`, `br_netfilter`, `net.bridge.bridge-nf-call-iptables=1`, `net.ipv4.ip_forward=1`.
* **DNS fonctionnel** vers `192.168.100.1` (Unbound sur OPNsense).

### 2.3. Plages réseau du cluster

* **Pod CIDR** : `10.244.0.0/16` (valeur retenue pour cohérence avec kubeadm et compatibilité Cilium).
* **Service CIDR** : `10.96.0.0/12` (valeur par défaut kubeadm).

> Ces CIDR ne doivent **pas** chevaucher le LAN (`192.168.100.0/24`) ni les futures plages **MetalLB**.

---

## 3. Préparation des nœuds (master **et** worker)

> Les commandes suivantes sont appliquées **sur chaque nœud** (`root` ou `sudo`).

### 3.1. Désactivation du swap

```bash
swapoff -a
sed -ri 's|^([^#].*\s+swap\s+.*)$|# \1|' /etc/fstab
```

### 3.2. Modules noyau et sysctl

```bash
cat <<'EOF' >/etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF
modprobe overlay
modprobe br_netfilter

cat <<'EOF' >/etc/sysctl.d/99-kubernetes-cri.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF
sysctl --system
```

### 3.3. Installation et configuration **containerd**

```bash
apt-get update && apt-get install -y containerd
mkdir -p /etc/containerd
containerd config default >/etc/containerd/config.toml

# Basculer en cgroups "systemd" (recommandé par Kubernetes)
sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml

systemctl enable --now containerd
```

### 3.4. Dépôt Kubernetes et composants

```bash
# Dépôt officiel pkgs.k8s.io (Ubuntu 24.04)
apt-get install -y apt-transport-https ca-certificates curl gpg
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.34/deb/Release.key |
  gpg --dearmor -o /etc/apt/keyrings/kubernetes-1-34.gpg
echo "deb [signed-by=/etc/apt/keyrings/kubernetes-1-34.gpg] \
https://pkgs.k8s.io/core:/stable:/v1.34/deb/ /" \
  >/etc/apt/sources.list.d/kubernetes.list

apt-get update
apt-get install -y kubeadm kubelet kubectl
apt-mark hold kubeadm kubelet kubectl
systemctl enable kubelet
```

---

## 4. Initialisation du **control-plane** (master)

### 4.1. Bootstrap du cluster

```bash
kubeadm init \
  --apiserver-advertise-address=192.168.100.31 \
  --pod-network-cidr=10.244.0.0/16 \
  --service-cidr=10.96.0.0/12
```

> À l’issue, kubeadm affiche la **commande de join** à exécuter sur le worker.
> Conserver le **token** et le **hash CA** fournis.

### 4.2. Kubeconfig pour l’utilisateur admin

```bash
mkdir -p $HOME/.kube
cp /etc/kubernetes/admin.conf $HOME/.kube/config
chown $(id -u):$(id -g) $HOME/.kube/config
```

### 4.3. Validation rapide côté master

```bash
kubectl get nodes -o wide
kubectl -n kube-system get pods -o wide
```

---

## 5. Raccordement du **worker**

### 5.1. Préparation (identique §3)

* Swap off, modules noyau, sysctl, **containerd** (SystemdCgroup), packages **kubeadm/kubelet/kubectl**.

### 5.2. Jonction au cluster

Sur `k8s-worker-1`, exécuter la commande fournie par kubeadm (exemple) :

```bash
kubeadm join 192.168.100.31:6443 \
  --token <TOKEN> \
  --discovery-token-ca-cert-hash sha256:<HASH_CA>
```

### 5.3. Contrôle

Sur le master :

```bash
kubectl get nodes -o wide     # master et worker doivent passer en Ready
kubectl get pods -A
```

---

## 6. Mise en réseau des Pods – **Cilium** (CNI)

> Le CNI est **requis** pour que les Pods passent de `Pending` à `Running`.
> Dans cette première phase, Cilium a été **installé “en brut”** (CLI/manifests), le basculement Helm et l’industrialisation seront traités dans les CR suivants.

### 6.1. Installation minimale (exemple via cilium-cli)

```bash
# Sur le master (kubectl configuré)
curl -L --fail --remote-name-all https://github.com/cilium/cilium-cli/releases/latest/download/cilium-linux-amd64.tar.gz{,.sha256sum}
sha256sum --check cilium-linux-amd64.tar.gz.sha256sum
tar xzf cilium-linux-amd64.tar.gz
install -m 755 cilium /usr/local/bin/

# Déploiement
cilium install --set cluster.name=demo-k8s --set ipam.mode=kubernetes

# Vérifications
cilium status --wait
kubectl -n kube-system get pods -l k8s-app=cilium
```

> **Remarques :**
>
> * Le **Pod CIDR** `10.244.0.0/16` configuré via kubeadm est **compatible** avec Cilium.
> * À ce stade, aucune intégration Hubble ni remplacement kube-proxy n’a été activé (simplicité).

---

## 7. Choix et installation de **Lens** (poste d’admin)

### 7.1. Motifs du choix

* UI **légère** et **locale** pour administrer le cluster (workloads, événements, logs, exec, port-forward).
* Fonctionne **sans control-plane externe** (pas d’addon serveur).
* **Connexion multi-clusters** et bascule de contextes depuis un seul poste.

### 7.2. Installation (Windows)

1. Télécharger **Lens Desktop** (binaire Windows).
2. Installer l’application (droits locaux suffisants).

### 7.3. Connexion au cluster

1. Depuis le master, récupérer le kubeconfig :

   ```bash
   cat $HOME/.kube/config
   ```
2. Copier le contenu vers le poste Windows et **l’importer dans Lens** :

   * Lens → **Add Cluster** → **Kubeconfig** → coller le contenu.
   * Contexte utilisé : `kubernetes-admin@kubernetes` (par défaut kubeadm).
3. Vérifier l’état dans Lens :

   * **Navigator** → Cluster → **Nodes**/**Workloads** → les objets `kube-system` sont visibles.
   * À ce stade, l’onglet **metrics** peut rester vide si aucun stack de métriques n’est activé (voir CR-5).

> **Sécurité** : le kubeconfig contient des **certificats/jetons** admin. Le fichier doit être stocké sur le poste dans un emplacement **protégé** (profil utilisateur) et **non synchronisé** publiquement.

---

## 8. Résultat opérationnel

* **Plan de contrôle en ligne** (`k8s-master-1`), **API 6443** accessible sur le LAN.
* **Nœud de travail** (`k8s-worker-1`) joint et **Ready**.
* **Cilium** déployé, `cilium-agent` et `cilium-operator` **Running**, trafic Pod-to-Pod opérationnel.
* **Lens** installé côté admin, cluster importé et consultable.
* Aucun stockage persistant ni pile de métriques **à ce stade** (traités dans CR-5).

---

## 9. Bonnes pratiques/points d’attention

* **Maintenir le swap désactivé** (persistant dans `/etc/fstab`).
* **Containerd en SystemdCgroup** (alignement cgroups kubelet).
* **Épingler les versions** (`apt-mark hold`) pour maîtriser les upgrades Kubernetes.
* **Sauvegarder** le fichier `/etc/kubernetes/admin.conf` et le **join** kubeadm.
* **Séparer** les namespaces fonctionnels dès le départ (ex. `monitoring`, `ingress`, `apps`).
* **Ne pas exposer** l’API 6443 au WAN ; l’accès admin se fait depuis le **LAN/VPN** (WireGuard, cf. CR-1).

---

## 10. Annexes – commandes de vérification utiles

```bash
# Nœuds et versions
kubectl get nodes -o wide
kubectl get componentstatuses   # ou kubectl get --raw /healthz?verbose

# Plan de contrôle
kubectl -n kube-system get pods -o wide

# Réseau
kubectl -n kube-system get pods -l k8s-app=cilium
kubectl -n kube-system logs -l k8s-app=cilium --tail=100

# DNS
kubectl -n kube-system get svc kube-dns
kubectl -n kube-system logs -l k8s-app=kube-dns --tail=100
```

---

### Conclusion

L’installation **from-scratch** de Kubernetes a été menée sur le **master** et le **worker**, avec un runtime **containerd** correctement configuré, un **CNI Cilium** opérationnel et un poste d’administration **Lens** connecté. Le cluster est **fonctionnel** et prêt pour les couches supérieures (ingress, persistance, monitoring), qui ont été cadrées et documentées séparément dans le **Compte rendu 4ter**.
