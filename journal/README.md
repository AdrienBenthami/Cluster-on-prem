# journal

Comptes rendus de sessions de travail. Trace du travail accompli et de l'évolution du lab.

> CR6 de l'ancienne numérotation supprimé — document de contexte hors sujet.

| N° | Titre | Date | Composants |
|---|---|---|---|
| [CR01](01-wireguard-opnsense.md) | Mise en place VPN WireGuard sur OPNsense | 2025-10-29 | OPNsense, WireGuard |
| [CR02](02-dhcp-dns-opnsense.md) | DHCP et DNS local sur OPNsense | 2025-10-29 | OPNsense, DHCP, DNS |
| [CR03](03-vms-kubernetes-proxmox.md) | Création VMs Kubernetes sous Proxmox | 2025-10-30 | Proxmox, VMs |
| [CR04](04-installation-kubernetes-lens.md) | Installation Kubernetes master & worker + Lens | 2025-11-10 | Kubernetes, Cilium, Lens |
| [CR05](05-exposition-metallb-traefik.md) | Exposition services — MetalLB + Traefik | 2025-11-18 | MetalLB, Traefik, DNS, TLS |
| [CR06](06-deploiement-gitea-authentik.md) | Déploiement Gitea + Authentik | 2026-01-12 | Gitea, Authentik, Traefik |
| [CR07](07-integration-gitea-authentik-traefik.md) | Intégration Gitea, Authentik, Traefik, MetalLB | 2026-01-12 | Gitea, Authentik, Traefik, MetalLB |
| [CR08](08-cert-manager-acme-dns01-ovh.md) | cert-manager ACME DNS-01 via webhook OVH | 2026-01-16 | cert-manager, OVH, ACME |
| [CR09](09-diagnostic-acme-nettoyage-cert-manager.md) | Diagnostic ACME DNS-01, nettoyage cert-manager | 2026-01-20 | cert-manager, OVH, Traefik |
| [CR10](10-migration-stockage-proxmox-zfs.md) | Migration stockage Proxmox vers ZFS mirror | 2026-02-12 | Proxmox, ZFS, stockage |
| [CR11](11-deploiement-acme-dns-postgresql.md) | Déploiement acme-dns + PostgreSQL dans K8s | 2026-03-04 | acme-dns, PostgreSQL, K8s |
| [CR12](12-integration-acme-dns-cert-manager-networkpolicies.md) | Intégration acme-dns ↔ cert-manager + NetworkPolicies | 2026-03-04 | acme-dns, cert-manager, Cilium |
| [CR13](13-diagnostic-reseau-cilium-metallb-dns.md) | Diagnostic réseau avancé — Cilium, MetalLB, DNS | 2026-04-07 | Cilium, kube-proxy, MetalLB, acme-dns |
| [CR14](14-migration-cloudflare-tls.md) | Migration TLS DNS-01 acme-dns → Cloudflare + démantèlement acme-dns | 2026-05-21 | cert-manager, Cloudflare, acme-dns, OPNsense, Authentik |
| [CR15](15-refonte-gitea-tls-oidc-vips.md) | Refonte Gitea — TLS Let's Encrypt dédié, OIDC Authentik fonctionnel, VIP SSH `.203` | 2026-05-26 | Gitea, Authentik, cert-manager, MetalLB |
| [CR16](16-opsbox-devbox-infra.md) | Modélisation RBAC + création de l'**opsbox** (devbox infra/K8s, ns `ops`, VIP `.201`) | 2026-05-26 | RBAC, ServiceAccount, MetalLB, SSH, Lens |
