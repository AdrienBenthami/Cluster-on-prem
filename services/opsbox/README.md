# opsbox

Devbox **infra/K8s** (pendant ops de la [devbox](../devbox/README.md) de dev applicatif).
Permissions K8s explicites via `ServiceAccount` + `ClusterRoleBinding`, à raffiner en personas nominatifs quand OIDC Authentik sera branché sur `kube-apiserver`.

L'opsbox est un **confort** : un shell Linux persistant *dans* le cluster (tmux, scripts, dotfiles, helm/kubectl/k9s/stern à portée), pas un point d'accès indispensable — Lens depuis le poste couvre l'inspection et l'édition de la plupart des objets.

## État

Déployée — Deployment `opsbox` dans le namespace `ops`.

| Composant | État |
|---|---|
| Namespace `ops` | Créé (manuellement) |
| Secret `opsbox-ssh-key` | Créé (manuellement, clé `authorized_key`) |
| Workload `Deployment` `opsbox` (1 réplique, strategy `Recreate`) | Running |
| Image `ubuntu:24.04` + install au boot | OK |
| Service SSH (`opsbox-ssh`, LoadBalancer) | VIP `192.168.100.201:22` |
| PVC `opsbox-home` (20 Gi local-path) | Bound → `/home/adrien` |
| ServiceAccount + ClusterRoleBinding `opsbox` → `cluster-admin` | Actif (temporaire) |
| NetworkPolicies | Non |

## Accès

Deux voies, selon le besoin :

### Lens (inspection rapide, exec shell)

Onglet Pods → `ops/opsbox-…` → Shell. Pas de config à gérer, mais pas de tmux long-running ni de VS Code Remote.

### SSH (shell quotidien, VS Code Remote-SSH)

| Endpoint | Adresse |
|---|---|
| SSH | `ssh adrien@192.168.100.201` |
| Auth | clé publique uniquement |
| User dans le pod | `adrien` (sudo NOPASSWD) |

Côté poste, dans `~/.ssh/config` :

```
Host opsbox
  HostName 192.168.100.201
  User adrien
  IdentityFile ~/.ssh/<ta-clé-privée>
```

⚠️ **Les host keys SSH du pod sont régénérées à chaque redémarrage** (non persistées dans le PVC). Symptôme : `REMOTE HOST IDENTIFICATION HAS CHANGED!` au prochain SSH après un `rollout restart`. Correctif côté client :

```powershell
ssh-keygen -R 192.168.100.201
```

À traiter durablement (cf. *Reste à faire*).

## Architecture

```
client → 192.168.100.201:22 (MetalLB) → svc opsbox-ssh → pod opsbox (ops/)
                                                          ├─ /home/adrien ← PVC opsbox-home (20 Gi)
                                                          └─ SA opsbox → cluster-admin (kubectl in-pod sans kubeconfig)
```

Le pod hérite automatiquement du token du `ServiceAccount` (monté en `/var/run/secrets/kubernetes.io/serviceaccount/`). `kubectl` détecte ce token et appelle l'apiserver sans `~/.kube/config` à gérer.

### Image et démarrage

Boot identique à la [devbox](../devbox/README.md) (apt + outils), avec **en plus** :

- `kubectl` (stable.txt)
- `helm` (script officiel)
- `k9s` (dernier release GitHub)
- `stern` (dernier release GitHub)

Et **sans** : `python3`, `nodejs`, `build-essential` (pas utile pour de l'admin K8s).

⚠️ Mêmes limites que devbox : install au boot ⇒ ~1–2 min après redémarrage avant SSH dispo, dépendance dépôts publics, pas de version pinnée.

### Persistance

| Chemin | Persisté |
|---|---|
| `/home/adrien` | oui (PVC `opsbox-home`, 20 Gi, `local-path`) |
| reste du conteneur | non |

Tout ce qui doit survivre à un redémarrage doit vivre sous `/home/adrien`. `local-path` lie le volume au nœud où le pod a démarré — pas de migration automatique, pas de backup intégré.

## Déploiement

Pré-requis : namespace `ops` et Secret `opsbox-ssh-key` contenant la clé publique sous la clé `authorized_key`.

```bash
kubectl apply -f services/opsbox/opsbox.yaml
kubectl -n ops logs -f deploy/opsbox          # suivre l'install au boot
ssh adrien@192.168.100.201
```

## Rotation de la clé SSH

```bash
kubectl -n ops create secret generic opsbox-ssh-key \
  --from-file=authorized_key=$HOME/.ssh/id_ed25519.pub \
  --dry-run=client -o yaml | kubectl apply -f -
kubectl -n ops rollout restart deploy/opsbox
```

Le script de boot recopie `authorized_key` (Secret monté en `/tmp/sshkey/`) vers `/home/adrien/.ssh/authorized_keys` **une fois au démarrage**. Sans `rollout restart`, la mise à jour du Secret n'a pas d'effet sur sshd. En urgence sans restart :

```bash
# depuis Lens shell ou kubectl exec
cp /tmp/sshkey/authorized_key /home/adrien/.ssh/authorized_keys
chown adrien:adrien /home/adrien/.ssh/authorized_keys
chmod 600 /home/adrien/.ssh/authorized_keys
```

## Pièges rencontrés (mémo)

- **Home en `0777`** : le mountpoint du PVC arrive en `drwxrwxrwx`, sshd avec `StrictModes yes` (défaut) refuse alors `Permission denied (publickey)` même avec la bonne clé. Le boot fait désormais `chmod 750 /home/adrien` après le `chown`.
- **Host keys volatiles** : régénérées à chaque démarrage (cf. avertissement plus haut). Sur le client : `ssh-keygen -R 192.168.100.201` après chaque restart.
- **Variables d'env perdues en SSH** : sshd ouvre un nouveau shell qui n'hérite **pas** des variables du conteneur (`KUBERNETES_SERVICE_HOST` / `_PORT`). Sans elles, `kubectl` fallback sur `localhost:8080`. Le boot écrit désormais `/etc/profile.d/opsbox-env.sh` pour exposer ces variables aux shells de login.
- **Secret mis à jour ≠ `authorized_keys` mis à jour** : le script de boot fait un `cp` une seule fois. Toute modification du Secret nécessite `kubectl -n ops rollout restart deploy/opsbox` (ou `cp` manuel dans le pod en urgence).
- **`rollout restart` ≠ `kubectl apply`** : restart relance le pod avec la spec **présente dans le cluster**. Pour qu'une modification de [opsbox.yaml](opsbox.yaml) prenne effet, il faut `kubectl apply -f` d'abord — `apply` déclenche le rollout automatiquement si la spec a changé.
- **Diagnostic publickey** : les logs `sshd` sortent en stdout du pod (`-D -e`), donc visibles dans l'onglet Logs de Lens (ou `kubectl logs deploy/opsbox`). La ligne `Authentication refused: bad ownership or modes for directory /home/adrien` est explicite.

## Reste à faire

1. **Persister les host keys SSH** dans le PVC (`/home/adrien/.ssh/host_keys/`) et adapter `sshd_config` — supprime l'avertissement client à chaque restart.
2. NetworkPolicies (default-deny + egress maîtrisé : DNS, API K8s, dépôts publics au boot).
3. Backup du PVC `opsbox-home` (Velero ou `tar` cron vers NAS/Gitea).
4. Image versionnée (supprimer l'install au boot) une fois Gitea registry opérationnel.
5. Brancher Authentik OIDC sur `kube-apiserver` et basculer du `cluster-admin` via SA vers des personas par groupe.
6. Activer l'`--audit-policy-file` côté apiserver avant d'ouvrir l'opsbox à des tiers.
7. Formaliser le break-glass (kubeconfig admin sur le master, accès restreint).
