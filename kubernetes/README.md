# kubernetes

## État du cluster

| Composant | Version | Notes |
|---|---|---|
| Kubernetes | v1.34.1 | |
| containerd | 1.7.28 | |
| Cilium (CNI) | 1.18.2 | |
| kube-proxy | v1.34.1 | mode iptables |
| CoreDNS | — | 2 replicas sur master |

## Nœuds

| Nom | IP | Rôle | OS | Kernel | Statut |
|---|---|---|---|---|---|
| k8s-master-1 | 192.168.100.31 | control-plane | Ubuntu 24.04.3 LTS | 6.8.0-100-generic | Ready |
| k8s-worker-1 | 192.168.100.51 | worker | Ubuntu 24.04.3 LTS | 6.8.0-107-generic | Ready |

## CIDR

| Usage | Plage |
|---|---|
| Pods | 10.244.0.0/16 |
| Services | 10.96.0.0/12 |
| DNS cluster | cluster.local |

## Namespaces

| Namespace | Usage |
|---|---|
| kube-system | Composants système K8s, Cilium, CoreDNS |
| metallb-system | MetalLB LoadBalancer |
| cert-manager | Gestion des certificats TLS |
| ingress | Traefik ingress controller |
| authentik | SSO / IdP |
| gitea | Git self-hosted |
| monitoring | kube-prometheus-stack, Grafana |
| lens-metrics | Prometheus + exporters installés par Lens |
| local-path-storage | Provisioner de stockage local |
| cilium-secrets | Secrets internes Cilium |
| dev | Environnement de développement (devbox) |
| test | Namespace de test |
| default | Namespace par défaut |

## Contenu

| Fichier | Contenu |
|---|---|
| [cilium.md](cilium.md) | CNI Cilium, eBPF, commandes de debug |
| [metallb/](metallb/README.md) | LoadBalancer L2, pools, VIPs |
| [rbac.md](rbac.md) | Roles, ServiceAccounts, kubeconfig |
| [stockage.md](stockage.md) | StorageClass, PVCs actifs, cible migration |
