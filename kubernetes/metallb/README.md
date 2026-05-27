# MetalLB

## VIPs en production

| VIP | Service | Namespace | Protocole / Port |
|---|---|---|---|
| 192.168.100.200 | traefik | ingress | TCP 80, TCP 443 |
| 192.168.100.201 | opsbox-ssh | ops | TCP 22 |
| 192.168.100.202 | devbox-ssh | dev | TCP 22 |
| 192.168.100.203 | gitea-ssh | gitea | TCP 22 |

> VIP `192.168.100.201` avait été libérée le 2026-05-21 (démantèlement acme-dns), recyclée le 2026-05-26 pour `opsbox-ssh`.

Pool L2 configuré : `192.168.100.200 – 192.168.100.239`

## Fichiers de configuration

| Fichier | Contenu |
|---|---|
| [metallb-l2.yaml](metallb-l2.yaml) | L2Advertisement |
| [metallb-pool.yaml](metallb-pool.yaml) | IPAddressPool |
