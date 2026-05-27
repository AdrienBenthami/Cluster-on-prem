# Traefik

## État

Déployé — Helm release `traefik` revision 3, chart traefik-38.0.2 (app v3.6.6)

## Accès

| Endpoint | VIP | Port |
|---|---|---|
| HTTP | 192.168.100.200 | 80 |
| HTTPS | 192.168.100.200 | 443 |

Domaines routés via IngressRoute :
- `auth.benthami.eu`
- `gitea.benthami.eu`

## Dépendances

- MetalLB (VIP 192.168.100.200)
- cert-manager (TLS)

## Configuration

- [values.yaml](values.yaml)
