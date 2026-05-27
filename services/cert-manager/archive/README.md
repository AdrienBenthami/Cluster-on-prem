# cert-manager — archive

Manifests historiques de cert-manager, **non appliqués dans le cluster**. Conservés pour traçabilité (audit, retour en arrière documenté, lecture des CRs liés).

> Convention : ce dossier accueille les YAML **techniques** mis hors service. Les archives **documentaires** (markdown obsolètes) restent dans [archive/](../../../archive/) à la racine du repo.

## Contenu

| Fichier | Issuer | Origine | Raison de l'archivage |
|---|---|---|---|
| [letsencrypt-ovh-dns-staging.yaml](letsencrypt-ovh-dns-staging.yaml) | `letsencrypt-ovh-dns-staging` | Webhook OVH DNS-01 | Webhook abandonné en janvier 2026 — erreur `Invalid signature` non résolue. Voir [CR08](../../../journal/08-cert-manager-acme-dns01-ovh.md) et [CR09](../../../journal/09-diagnostic-acme-nettoyage-cert-manager.md). |
| [clusterissuer-prod-acme-dns.yaml](clusterissuer-prod-acme-dns.yaml) | `letsencrypt-prod-acme-dns` | Solver acme-dns (DNS-01) | Manifest jamais appliqué dans le cluster. Remplacé par `letsencrypt-prod-cloudflare` dans le cadre de la migration OVH → Cloudflare (mai 2026). |
| [clusterissuer-staging-acme-dns.yaml](clusterissuer-staging-acme-dns.yaml) | `letsencrypt-staging-acme-dns` | Solver acme-dns (DNS-01) | Issuer staging historiquement utilisé pour valider la chaîne acme-dns (CR12/CR13). Supprimé du cluster le 2026-05-21 dans le cadre du démantèlement acme-dns. Remplacé par `letsencrypt-staging-cloudflare`. |
