# Stockage

## StorageClass

| Nom | Provisioner | Reclaim Policy | Binding Mode | Expansion | Défaut |
|---|---|---|---|---|---|
| local-path | rancher.io/local-path | Delete | WaitForFirstConsumer | Non | Oui |

Une seule StorageClass active. Provisionnement local sur le nœud qui schedule le pod.
Pas de réplication, pas d'expansion de volume, pas de backup.

## PVCs actifs

| Namespace | PVC | Statut | Taille | StorageClass |
|---|---|---|---|---|
| authentik | data-authentik-postgresql-0 | Bound | 8 Gi | local-path |
| dev | devbox-home | Bound | 20 Gi | local-path |
| gitea | data-gitea-postgresql-0 | Bound | 10 Gi | local-path |
| gitea | gitea-shared-storage | Bound | 10 Gi | local-path |
| gitea | valkey-data-gitea-valkey-primary-0 | Bound | 8 Gi | local-path |
| lens-metrics | data-prometheus-0 | Bound | 20 Gi | local-path |
| ops | opsbox-home | Bound | 20 Gi | local-path |

> PVCs `data-acme-dns-postgres-0` et `acme-dns-data` supprimés le 2026-05-21 avec le démantèlement d'acme-dns.

## Total stockage utilisé

~96 Gi répartis sur k8s-worker-1 (tous les workloads y tournent).

## Cible migration

Local-path est provisoire. Cibles envisagées : NFS, Longhorn ou Rook-Ceph.
Prérequis : définir une stratégie de backup avant toute migration.
