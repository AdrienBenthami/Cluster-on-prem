# cert-manager

## État

Déployé — Helm release `cert-manager` revision 1, chart cert-manager-v1.19.2

**Migration en cours (mai 2026)** : bascule du solver DNS-01 d'acme-dns (zone gérée historiquement chez OVH) vers le solver natif Cloudflare. ClusterIssuers acme-dns désormais supprimés du cluster — démantèlement d'acme-dns lui-même en cours.

## Accès

Composant interne — pas d'URL publique.

## Dépendances

- API Cloudflare (solver DNS-01 natif, zone `benthami.eu`)
- Secret `cert-manager/cloudflare-api-token` — token Cloudflare scope `Zone:DNS:Edit` + `Zone:Zone:Read`, filtré sur IP publique WAN

## ClusterIssuers actifs

| Nom | Solver | Serveur ACME | Usage |
|---|---|---|---|
| `letsencrypt-staging-cloudflare` | Cloudflare DNS-01 | staging | Tests / validation chaîne |
| `letsencrypt-prod-cloudflare` | Cloudflare DNS-01 | prod | Certificats applicatifs |

## Configuration

- [clusterissuer-staging-cloudflare.yaml](clusterissuer-staging-cloudflare.yaml)
- [clusterissuer-prod-cloudflare.yaml](clusterissuer-prod-cloudflare.yaml)
- [values.yaml](values.yaml)
- [tests/](tests/README.md) — manifests sandbox (non déployés en permanence)
- [archive/](archive/README.md) — manifests historiques (webhook OVH, ClusterIssuers acme-dns)
