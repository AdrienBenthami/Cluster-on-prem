# homelab-k8s

Homelab Kubernetes sur bare-metal — documentation complète de la construction, de la configuration réseau à la mise en production des services.

## Vue d'ensemble

Cluster Kubernetes auto-hébergé, monté de zéro sur Proxmox, avec une stack réseau OPNsense + WireGuard et des services orientés DevOps : Git self-hosted, SSO, TLS automatique, monitoring.

```
Freebox (FAI)
  └── OPNsense (pare-feu / DHCP / DNS / WireGuard VPN)
        └── LAN 192.168.100.0/24
              ├── Proxmox (hyperviseur)
              │     ├── k8s-master-1
              │     └── k8s-worker-1
              └── MetalLB L2 (VIPs .200–.239)
```

## Infrastructure

| Couche | Technologie | Détail |
|---|---|---|
| Hyperviseur | Proxmox VE | 2 VMs Ubuntu 24.04 |
| Réseau | OPNsense + Freebox | Double NAT, WireGuard |
| Kubernetes | v1.34.1 | 1 control-plane + 1 worker |
| CNI | Cilium 1.18.2 | eBPF, NetworkPolicies |
| LoadBalancer | MetalLB L2 | Pool `192.168.100.200–239` |
| Ingress | Traefik v3.6.6 | VIP `.200`, HTTP→HTTPS |
| TLS | cert-manager v1.19.2 | Let's Encrypt DNS-01 (Cloudflare) |
| Stockage | local-path + ZFS mirror | PVCs Proxmox |

## Services déployés

| Service | Chart | Namespace | Rôle |
|---|---|---|---|
| [Traefik](services/traefik/) | traefik-38.0.2 | ingress | Ingress controller |
| [cert-manager](services/cert-manager/) | cert-manager-v1.19.2 | cert-manager | Certificats TLS automatiques |
| [Authentik](services/authentik/) | authentik-2025.10.3 | authentik | SSO / IdP OIDC |
| [Gitea](services/gitea/) | gitea-12.6.0 | gitea | Git self-hosted |
| [Monitoring](services/monitoring/) | kube-prometheus-stack-79.4.1 | monitoring | Prometheus + Grafana |
| [opsbox](services/opsbox/) | manifest custom | ops | Shell ops + accès kubectl via SSH |
| [devbox](services/devbox/) | manifest custom | dev | Environnement de développement |

## Structure du dépôt

```
.
├── journal/          # Comptes rendus de sessions (CR01–CR16)
├── kubernetes/       # Config cluster : Cilium, MetalLB, RBAC
└── services/         # Valeurs Helm et manifests par service
    ├── authentik/
    ├── cert-manager/
    ├── devbox/
    ├── gitea/
    ├── monitoring/
    ├── opsbox/
    └── traefik/
```

## Journal de bord

16 comptes rendus documentant chaque étape de la construction, du réseau jusqu'aux services :

| CR | Sujet | Date |
|---|---|---|
| [CR01](journal/01-wireguard-opnsense.md) | VPN WireGuard sur OPNsense | 2025-10-29 |
| [CR02](journal/02-dhcp-dns-opnsense.md) | DHCP et DNS local sur OPNsense | 2025-10-29 |
| [CR03](journal/03-vms-kubernetes-proxmox.md) | Création des VMs Kubernetes sur Proxmox | 2025-10-30 |
| [CR04](journal/04-installation-kubernetes-lens.md) | Installation Kubernetes + Cilium + Lens | 2025-11-10 |
| [CR05](journal/05-exposition-metallb-traefik.md) | Exposition services — MetalLB + Traefik | 2025-11-18 |
| [CR06](journal/06-deploiement-gitea-authentik.md) | Déploiement Gitea + Authentik | 2026-01-12 |
| [CR07](journal/07-integration-gitea-authentik-traefik.md) | Intégration Gitea, Authentik, Traefik, MetalLB | 2026-01-12 |
| [CR08](journal/08-cert-manager-acme-dns01-ovh.md) | cert-manager ACME DNS-01 via webhook OVH | 2026-01-16 |
| [CR09](journal/09-diagnostic-acme-nettoyage-cert-manager.md) | Diagnostic ACME DNS-01, nettoyage cert-manager | 2026-01-20 |
| [CR10](journal/10-migration-stockage-proxmox-zfs.md) | Migration stockage Proxmox vers ZFS mirror | 2026-02-12 |
| [CR11](journal/11-deploiement-acme-dns-postgresql.md) | Déploiement acme-dns + PostgreSQL dans K8s | 2026-03-04 |
| [CR12](journal/12-integration-acme-dns-cert-manager-networkpolicies.md) | Intégration acme-dns ↔ cert-manager + NetworkPolicies | 2026-03-04 |
| [CR13](journal/13-diagnostic-reseau-cilium-metallb-dns.md) | Diagnostic réseau avancé — Cilium, MetalLB, DNS | 2026-04-07 |
| [CR14](journal/14-migration-cloudflare-tls.md) | Migration TLS DNS-01 → Cloudflare, démantèlement acme-dns | 2026-05-21 |
| [CR15](journal/15-refonte-gitea-tls-oidc-vips.md) | Refonte Gitea — TLS dédié, OIDC Authentik, VIP SSH | 2026-05-26 |
| [CR16](journal/16-opsbox-devbox-infra.md) | RBAC + opsbox (shell K8s via SSH) | 2026-05-26 |

## Utilisation

Les fichiers `values.yaml` et manifests sont pensés comme **référence documentaire**. Pour les réutiliser :

1. Adapter les noms de domaine (`benthami.eu` → votre domaine)
2. Remplacer `<your-email@example.com>` dans les ClusterIssuers cert-manager
3. Créer les secrets K8s manquants (clé API Cloudflare, clés SSH, secrets OIDC) — ils ne sont jamais commités ici
4. Ajuster les plages IP MetalLB selon votre réseau LAN

## Prérequis

- Proxmox VE (ou tout autre hyperviseur)
- OPNsense (ou équivalent) pour le routage / DNS local
- Un domaine géré sur Cloudflare (pour DNS-01 Let's Encrypt)
- `kubectl`, `helm`, `k9s` ou Lens en local
