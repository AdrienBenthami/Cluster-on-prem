# devbox

Devbox de **développement applicatif** : pod Ubuntu accessible en SSH depuis VS Code Remote-SSH.
Pas de privilèges Kubernetes (ServiceAccount `default`, aucun RoleBinding) — la devbox **infra/K8s** est traitée séparément dans [devbox-infra/](../devbox-infra/README.md).

## État

Déployée — Deployment `devbox` dans le namespace `dev` (49 jours au 2026-05-26).

| Composant | État |
|---|---|
| Pod (`ubuntu:24.04` + install au boot) | Running |
| Service SSH (`devbox-ssh`, LoadBalancer) | VIP `192.168.100.202:22` |
| PVC `devbox-home` (20 Gi local-path) | Bound → `/home/adrien` |
| Secret `devbox-ssh-key` (clé publique) | Présent |
| RBAC | Aucune (SA `default`) |
| NetworkPolicies | Non déployées |

## Accès

| Endpoint | Adresse |
|---|---|
| SSH | `ssh adrien@192.168.100.202` |
| Authentification | Clé publique uniquement (`PasswordAuthentication no`, `PermitRootLogin no`) |
| Utilisateur dans le pod | `adrien` (`sudo NOPASSWD` activé) |

VS Code Remote-SSH : ajouter dans `~/.ssh/config` côté poste :

```
Host devbox
  HostName 192.168.100.202
  User adrien
  IdentityFile ~/.ssh/id_ed25519
```

## Architecture

```
client → 192.168.100.202:22 (MetalLB) → svc devbox-ssh → pod devbox (dev/)
                                                          └─ /home/adrien ← PVC devbox-home (20 Gi)
```

### Image et démarrage

Le pod tourne sur `ubuntu:24.04` **brut** ; toute la stack est installée à chaque démarrage par le `command:` du conteneur :

- `openssh-server`, `sudo`, `curl`, `wget`, `git`, `vim`, `nano`, `less`, `procps`, `iputils-ping`, `net-tools`
- `python3`, `python3-pip`, `build-essential`
- Node.js 20 (dépôt NodeSource)
- Création de l'utilisateur `adrien` + sudoers `NOPASSWD`
- Copie de la clé publique depuis `/tmp/sshkey/authorized_key` (Secret `devbox-ssh-key`) vers `~adrien/.ssh/authorized_keys`
- Hardening sshd minimal (no password, no root, pubkey only)
- `exec /usr/sbin/sshd -D -e`

**Conséquences à connaître :**
- Chaque redémarrage du pod = ~1–2 min d'`apt-get` avant d'avoir SSH dispo.
- Dépendance aux dépôts publics Ubuntu et NodeSource au boot.
- Pas d'image versionnée : la stack peut dériver entre deux pulls.

### Persistance

| Chemin | Persisté | Notes |
|---|---|---|
| `/home/adrien` | oui (PVC `devbox-home`) | dotfiles, repos clonés, historique shell |
| `/workspace` | **non** | créé par le script de boot, **disparaît** au redémarrage |
| Reste (`/etc`, `/usr`, paquets installés) | non | reconstruit par le script de boot |

⚠️ Tout fichier hors `/home/adrien` est perdu au redémarrage.

### Stockage

| PVC | Taille | StorageClass | Mount |
|---|---|---|---|
| `devbox-home` | 20 Gi | local-path | `/home/adrien` |

`local-path` = volume sur le nœud où le pod a démarré. Si le nœud meurt, le PVC est perdu (pas de backup automatique). `reclaimPolicy: Delete` sur la StorageClass : supprimer le PVC supprime irréversiblement le contenu.

## Rotation de la clé SSH

```bash
kubectl -n dev create secret generic devbox-ssh-key \
  --from-file=authorized_key=$HOME/.ssh/id_ed25519.pub \
  --dry-run=client -o yaml | kubectl apply -f -
kubectl -n dev rollout restart deploy/devbox
```

## Reste à faire

1. Versionner le manifest dans ce dossier (actuellement appliqué via `last-applied-configuration`, pas de fichier source dans le repo).
2. Décider d'une stratégie de persistance pour `/workspace` ou documenter qu'on l'évite.
3. Backup du PVC `devbox-home` (Velero ou cron `tar` vers NAS/Gitea).
4. NetworkPolicies (default-deny + egress maîtrisé).
5. Passer à une image versionnée pour supprimer l'install au boot.
