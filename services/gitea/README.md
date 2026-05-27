# Gitea

## État

Déployé — Helm release `gitea` revision 9, chart `gitea-12.6.0` (app v1.26.1).

Refonte d'architecture en cours (mai 2026) : durcissement TLS, séparation des VIPs MetalLB, authentification déléguée à Authentik, NetworkPolicies.

| Composant | État |
|---|---|
| Pod Gitea | Running |
| Postgres / Valkey embarqués | Running |
| HTTPS `gitea.benthami.eu` | Cert Let's Encrypt via `letsencrypt-prod-cloudflare` (secret `gitea-tls`) |
| OIDC Authentik | Provider `gitea-oidc` configuré, redirect URI déclarée |
| SSH | VIP dédiée `192.168.100.203` |
| NetworkPolicies | Non déployées |
| `DISABLE_REGISTRATION` + admin break-glass | Non configurés |

## Accès

| Endpoint | Adresse |
|---|---|
| HTTPS | `https://gitea.benthami.eu` |
| SSH | `192.168.100.203:22` |

## Architecture cible

### Flux d'accès (HTTPS via Traefik)

```
HTTPS : client → 192.168.100.200:443 (Traefik) → Ingress gitea.benthami.eu → svc gitea-http:3000
SSH   : client → 192.168.100.203:22  (MetalLB)  → svc gitea-ssh:22
```

Traefik termine TLS pour tous les services HTTP du cluster (cert centralisé via cert-manager). Gitea reçoit du HTTP en interne. SSH dispose d'une VIP dédiée — Traefik ne traite pas le SSH, et le partage `allow-shared-ip` avec `.200` est supprimé pour clarifier le firewall et les NetworkPolicies.

### VIPs MetalLB

| VIP | Service | Port | Statut |
|---|---|---|---|
| 192.168.100.200 | Traefik | 80, 443 | Déployé |
| 192.168.100.203 | Gitea SSH (dédié) | 22 | À provisionner |

### Authentification

- Provider OIDC dédié dans Authentik (application `Gitea`, provider `gitea-oidc`)
- Secret K8s `gitea-oauth-secret` (clés `key`, `secret`)
- Autodiscover : `https://auth.benthami.eu/application/o/gitea/.well-known/openid-configuration`
- `ENABLE_OPENID_SIGNIN=true`, `DISABLE_REGISTRATION=true` — création de comptes uniquement via SSO
- Compte admin local "break-glass" conservé pour reprise en cas d'Authentik indisponible
- Pas de ForwardAuth Traefik (incompatible avec `git clone https://`)

### NetworkPolicies (Cilium, à créer)

Namespace `gitea` :

- Default-deny ingress + egress
- Ingress depuis namespace `ingress` (Traefik) vers svc HTTP 3000
- Ingress depuis `world` vers svc SSH 22 (entrée publique MetalLB)
- Egress vers CoreDNS, Postgres et Valkey du namespace, et `auth.benthami.eu`

## Dépendances

- Traefik (ingress HTTP/HTTPS)
- cert-manager + ClusterIssuer `letsencrypt-prod-cloudflare`
- MetalLB (VIP SSH)
- Authentik (OIDC)
- Postgres et Valkey embarqués dans la release Gitea

## Stockage

| PVC | Taille | StorageClass |
|---|---|---|
| `data-gitea-postgresql-0` | 10 Gi | local-path |
| `valkey-data-gitea-valkey-primary-0` | 8 Gi | local-path |
| `gitea-shared-storage` | 10 Gi | local-path |

## Configuration

- [values.yaml](values.yaml) — overrides Helm (utilisés avec `--reuse-values`)
- [certificate.yaml](certificate.yaml) — Certificate cert-manager `gitea-tls`

## Déploiement

Version cible : dernière version disponible dans `gitea-charts`. Vérifier avant tout upgrade :

```bash
helm repo update gitea-charts
helm search repo gitea-charts/gitea --versions | head -3
helm history gitea -n gitea
```

Upgrade (en pinnant explicitement la version observée à l'étape précédente) :

```bash
helm upgrade gitea gitea-charts/gitea --version <X.Y.Z> \
  -n gitea -f services/gitea/values.yaml --reuse-values
```

`--reuse-values` réutilise les valeurs de la release précédente et applique les overrides de `values.yaml` par-dessus. Si un nouveau template du chart référence une clé sans default, `helm upgrade` lèvera une erreur `nil pointer` — il faut alors ajouter la clé manquante dans `values.yaml`.

Certificate TLS (idempotent) :

```bash
kubectl apply -f services/gitea/certificate.yaml
```

## Reste à faire

1. Créer le compte admin local break-glass
2. Activer `DISABLE_REGISTRATION=true`
3. Écrire les NetworkPolicies Cilium (`services/gitea/network-policies/`)
4. Valider clone HTTPS, push SSH, login SSO, fallback break-glass
