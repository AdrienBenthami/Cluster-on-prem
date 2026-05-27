# Authentik

## État

Déployé — Helm release `authentik` revision 3, chart authentik-2025.10.3

## Accès

| Endpoint | URL |
|---|---|
| Interface web | https://auth.benthami.eu |

## Dépendances

- PostgreSQL (StatefulSet interne, PVC `data-authentik-postgresql-0` — 8 Gi)
- Traefik (ingress)
- cert-manager (TLS)

## Configuration

- [values.yaml](values.yaml)
