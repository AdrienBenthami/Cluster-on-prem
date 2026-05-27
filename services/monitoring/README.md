# Monitoring

## État

Déployé — Helm release `kube-prometheus-stack-1762720643` revision 1, chart kube-prometheus-stack-79.4.1

Stack installée via Lens.

## Accès

| Composant | Endpoint |
|---|---|
| Grafana | — (pas d'IngressRoute configuré) |
| Prometheus | — (accès interne uniquement) |
| Alertmanager | — (accès interne uniquement) |

## Composants actifs

| Pod | Namespace | Statut |
|---|---|---|
| prometheus-0 | monitoring | Running |
| grafana | monitoring | Running |
| alertmanager-0 | monitoring | Running |
| kube-state-metrics | monitoring | Running |
| node-exporter (×2) | monitoring | Running |
| kube-state-metrics | lens-metrics | Running |
| node-exporter (×2) | lens-metrics | Running |
| prometheus-0 | lens-metrics | Running |

> Doublon : deux stacks Prometheus coexistent (`monitoring` et `lens-metrics`). Voir [dette.md](../../dette.md).

## Dépendances

- PVC `data-prometheus-0` — 20 Gi (lens-metrics)
