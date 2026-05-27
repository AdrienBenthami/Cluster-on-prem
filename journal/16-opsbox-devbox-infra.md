# Compte rendu technique

**Sujet : modélisation RBAC et création de l'opsbox (devbox infra/K8s dédiée)**

**Date : 2026-05-26**

**Périmètre :**
- Discussion architecture RBAC Kubernetes (humains, workloads, LLM, IdP externe)
- Position d'Authentik vis-à-vis de `kube-apiserver` (séparation authn / authz)
- Création d'une devbox dédiée à l'administration K8s (namespace `ops`, séparée de la devbox de dev applicatif)
- Documentation rétroactive de la devbox `dev` existante
- Documentation transversale (services, MetalLB, adressage, stockage, inventaire, changelog)

**Déclencheur :** intention de mettre en place une RBAC complète sur le cluster pour préparer l'arrivée éventuelle de tiers (auditeurs, devs, LLM). Constat fait en cours de session : la session a commencé en SSH sur le master avec le kubeconfig admin, ce qui contredit toute logique RBAC. Décision de monter une devbox infra avant de toucher à la RBAC.

---

## 1. Cadrage architectural

### 1.1. Le modèle mental retenu — 3 couches

```
[Authentik / OIDC]  ← QUI (identité humaine)
        ↓
[Kubernetes RBAC]   ← QUOI (permissions sur l'API)
        ↓
[ServiceAccounts]   ← QUOI (permissions des workloads in-cluster)
```

Point initial à clarifier côté utilisateur : « Authentik au-dessus de K8s, c'est bizarre ? ». Non — `kube-apiserver` délègue **volontairement** l'authentification à un IdP externe (OIDC, certs client, webhooks). Authentik joue exactement ce rôle. K8s ne connaît à la sortie de l'authn que des strings (`user=...`, `groups=[...]`) extraits du JWT, sur lesquelles les `RoleBinding` matchent.

### 1.2. Humains vs workloads — règle retenue

| Acteur | Mécanisme d'identité | Binding |
|---|---|---|
| Humains (admins, devs, auditeurs) | OIDC via Authentik (groupes) | `Group` → `ClusterRole`/`Role` |
| LLM interactif | OIDC si possible, sinon SA + token court | idem ou `ServiceAccount` → Role |
| Workloads (pods) | `ServiceAccount` (auto) | `ServiceAccount` → Role |

**Décision principe :** ne jamais créer de `ServiceAccount` pour un humain. Un SA n'a pas de MFA, pas de révocation centralisée, pas d'audit nominatif — exactement ce que résout l'IdP.

### 1.3. Pipeline `kube-apiserver` — rappel mécanique

Trois étages successifs :

1. **Authentication** — extrait `user` + `groups` (cert client, SA token, OIDC, webhook). À la sortie, l'apiserver ne sait pas si c'est un humain ou un robot, juste deux strings.
2. **Authorization** — RBAC évalue `(user, groups, verb, resource, namespace)` contre les `RoleBinding`/`ClusterRoleBinding`. Authentik n'intervient **plus** à cette étape.
3. **Admission** — `PodSecurity`, `ResourceQuota`, webhooks Kyverno/Gatekeeper. Hors sujet RBAC.

### 1.4. RBAC — les 4 objets

| Objet | Portée | Contient | Contient pas |
|---|---|---|---|
| `Role` | un namespace | règles (verbs + resources) | qui |
| `ClusterRole` | tout le cluster | règles | qui |
| `RoleBinding` | un namespace | qui ↔ Role/ClusterRole | les règles |
| `ClusterRoleBinding` | tout le cluster | qui ↔ ClusterRole | les règles |

`User` et `Group` n'existent **pas** comme objets K8s : ce sont juste les strings produits par l'authn. Donc on peut binder à `kind: Group, name: k8s-admins` sans rien créer côté K8s — il suffit que l'IdP mette ce string dans le claim `groups`.

---

## 2. Décision : devbox infra séparée

### 2.1. Faiblesse du modèle actuel

Tant que les opérations s'exécutent en SSH sur le master avec le cert admin de k3s (`CN=system:admin, O=system:masters`), RBAC ne protège de rien — le cert bypass tout. La devbox infra est une **précondition pratique** pour que la RBAC ait du sens.

### 2.2. Forme retenue — pod ops dans le cluster

Quatre formes envisagées :
1. **VM dédiée sur Proxmox** (hors cluster)
2. **Conteneur/devcontainer sur le poste**
3. **User dédié sur le poste**
4. **Pod ops dans le cluster**

**Choix retenu : pod ops in-cluster**, malgré le constat factuel listé en séance :

| Critère | Pod ops | VM Proxmox |
|---|---|---|
| Coût infra (Proxmox déjà là) | nul | nul |
| Survit à un cluster down | non | oui |
| Survit à une migration HA | non | oui |
| Bootstrap externe nécessaire | oui | non |
| Confort `kubectl exec` rapide | oui | non |
| Image reproductible | oui (manifest) | oui (cloud-init) |

Trade-off accepté : la devbox est inaccessible si l'apiserver est down. Break-glass = kubeconfig admin sur le master, conservé jusqu'à l'OIDC.

### 2.3. Transport VS Code — Remote-SSH

Trois transports possibles : Remote-SSH (sshd dans le pod), `kubectl exec` attach (extension Kubernetes), Remote-Tunnels (Microsoft). Choix retenu : **Remote-SSH** (extension la plus mature, robuste). Conséquence assumée : crée un 2e système d'authentification (clés SSH) en parallèle de l'IdP, à reconsidérer une fois OIDC sur l'apiserver.

---

## 3. Découverte d'une devbox existante — recentrage

### 3.1. Constat

Listing du cluster :

```bash
kubectl get pods,sts,deploy,svc,pvc -A | grep -iE "devbox|ops"
# dev   pod/devbox-fb9c77bd8-244sh                     Running   49d
# dev   deployment.apps/devbox                                       49d
# dev   service/devbox-ssh                  LoadBalancer 192.168.100.202   22:31859/TCP   49d
# dev   persistentvolumeclaim/devbox-home  Bound  20Gi  local-path  49d
```

Une devbox `dev` existait déjà depuis 49 jours, non documentée dans le repo.

### 3.2. Analyse comparative

| Aspect | Devbox `dev` (existante) | Nouveau besoin (admin K8s) |
|---|---|---|
| Workload | `Deployment` + PVC sur `/home/adrien` | idem |
| Image | `ubuntu:24.04` brute + install au boot | idem |
| User | `adrien` (sudo NOPASSWD) | idem |
| Outils | python3, nodejs, build-essential | kubectl, helm, k9s, stern |
| ServiceAccount | `default` (aucun droit K8s) | `cluster-admin` (temporaire) |
| Cible | dev applicatif | admin infra/K8s |

Constat : la devbox `dev` ne sait **rien** faire sur l'apiserver. L'idée d'une devbox infra séparée tient, ce n'est pas un dédoublement.

### 3.3. Mauvais départ — devbox/devbox-infra créés trop vite côté assistant

Premier jet (côté assistant) : génération de fichiers `services/devbox/` + `services/devbox-infra/` avant d'avoir constaté l'existence de `dev`. Utilisateur a supprimé l'ensemble : « tu ajoutais de la confusion ». Repris à zéro :
- Documentation **rétroactive** de la devbox `dev` existante → `services/devbox/README.md`
- Nouveau dossier pour l'infra → `services/devbox-infra/` puis **renommé `opsbox`** sur proposition utilisateur (« devbox pour de l'infra c'est nul »).

---

## 4. Décisions opérationnelles opsbox

| Sujet | Décision | Rationale |
|---|---|---|
| Namespace | `ops` | plus court qu'`opsbox`, peut accueillir d'autres outils ops plus tard |
| VIP MetalLB | `192.168.100.201` | prochain libre après Traefik `.200`. Recycle la VIP de l'ex-`acme-dns` |
| User dans le pod | `adrien` (sudo NOPASSWD) | identique à devbox `dev`, simplifie `~/.ssh/config` |
| Outils initiaux | `kubectl`, `helm`, `k9s`, `stern` | minimum + confort. Pas de kustomize/flux/sops/terraform pour le moment |
| RBAC initiale | SA `opsbox` → `ClusterRoleBinding` `cluster-admin` | temporaire, à raffiner en personas OIDC |
| Pattern image | `ubuntu:24.04` + install au boot | aligné sur devbox `dev`, pas de registry requis (Gitea registry indisponible) |
| Stockage | PVC `opsbox-home` 20 Gi `local-path` → `/home/adrien` | aligné sur devbox `dev` |
| Stratégie d'écriture (GitOps vs kubectl direct) | non tranchée | reportée à la phase RBAC nominative |

---

## 5. Manifests et bootstrap

### 5.1. Fichiers créés

- [`services/opsbox/opsbox.yaml`](../services/opsbox/opsbox.yaml) — `ServiceAccount`, `ClusterRoleBinding` `cluster-admin`, `PersistentVolumeClaim` `opsbox-home`, `Deployment` `opsbox` (1 réplique, strategy `Recreate`, image `ubuntu:24.04`, `command:` shell d'install au boot), `Service` `opsbox-ssh` LoadBalancer `192.168.100.201:22`.
- [`services/opsbox/README.md`](../services/opsbox/README.md) — état, accès (Lens + SSH), architecture, persistance, pièges rencontrés, reste à faire.

### 5.2. Pré-requis créés à la main (hors manifest)

- Namespace `ops` créé via Lens
- Secret `opsbox-ssh-key` (clé `authorized_key`) créé via Lens

**Décision** : le Secret est **sorti du manifest** pour éviter qu'un `kubectl apply` ne l'écrase par mégarde avec un placeholder. Bloc Secret remplacé par un commentaire dans le YAML expliquant la création/rotation manuelle.

### 5.3. Apply

```bash
kubectl apply -f services/opsbox/opsbox.yaml
kubectl -n ops rollout status deploy/opsbox
```

`apply` a créé les 4 objets restants (SA, CRB, PVC, Deployment, Service). Pod neuf, install au boot (~1–2 min apt-get + curl kubectl/helm/k9s/stern), sshd démarré.

---

## 6. Pièges rencontrés et corrections (chronologique)

### 6.1. Double `---` autour d'un bloc de commentaires

Après suppression du bloc Secret du YAML, la séquence devenue :

```yaml
...
---
# Secret géré hors manifest
# ...
---
apiVersion: v1
kind: PersistentVolumeClaim
```

Le parseur YAML interprétait cela comme **deux documents** dont un vide (que des commentaires). Conséquence : le schéma `PersistentVolumeClaim` était appliqué au document vide, et celui du `Deployment` au document `PVC` — diagnostics IDE en cascade. Correction : fusionner en un seul `---`. Mémo à retenir : les commentaires sans contenu YAML ne suffisent pas à constituer un document, et un `---` redondant crée un document `null`.

### 6.2. `Permission denied (publickey)` — clé tronquée dans le Secret

Première tentative SSH depuis Windows (`ssh -i vm_key adrien@192.168.100.201`) → refus. Diagnostic :
- Clé publique présente dans `authorized_keys` du pod, mode `600`, owner correct
- `diff` Secret monté ↔ `authorized_keys` : identiques
- Mais ce n'était pas la clé attendue côté client : caractères manquants en début (saisie tronquée dans Lens lors de la création du Secret)

Correction : Secret mis à jour avec la clé complète.

### 6.3. Mise à jour du Secret ≠ mise à jour de `authorized_keys`

Le script de boot fait `cp /tmp/sshkey/authorized_key → ~/adrien/.ssh/authorized_keys` **une seule fois**. Le volume Secret monté en `/tmp/sshkey/` se rafraîchit automatiquement (~60s), mais `authorized_keys` reste figé. Deux solutions :
- `kubectl -n ops rollout restart deploy/opsbox` (le nouveau pod re-cp au boot)
- `cp` manuel dans le pod en urgence

Mémo ajouté au README opsbox.

### 6.4. `/home/adrien` en `0777` — sshd `StrictModes` refuse

Après restart et clé corrigée, toujours `Permission denied (publickey)`. `kubectl logs deploy/opsbox` (logs sshd `-D -e` en stdout) :

```
Authentication refused: bad ownership or modes for directory /home/adrien
```

Cause : le mountpoint du PVC `opsbox-home` arrive en `drwxrwxrwx`. sshd avec `StrictModes yes` (défaut) refuse tout home group ou world-writable, même si `.ssh/` et `authorized_keys` sont stricts.

Correction dans le `command:` du Deployment :

```bash
chown -R adrien:adrien /home/adrien
chmod 750 /home/adrien        # ← ajouté
chmod 700 /home/adrien/.ssh
chmod 600 /home/adrien/.ssh/authorized_keys
```

Après `kubectl apply -f opsbox.yaml`, SSH passe.

### 6.5. Host keys SSH régénérées à chaque restart

Au prochain `rollout restart`, client Windows :

```
@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@
@    WARNING: REMOTE HOST IDENTIFICATION HAS CHANGED!     @
@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@
```

Cause : les host keys sshd sont générées à chaque démarrage par `dpkg-reconfigure openssh-server` (déclenché par `apt-get install`), pas persistées dans le PVC. Workaround client : `ssh-keygen -R 192.168.100.201` après chaque restart. **Reste à faire** : persister les host keys dans le PVC (sous `/home/adrien/.ssh/host_keys/`) et adapter `sshd_config`.

### 6.6. `kubectl` dans la session SSH → `localhost:8080`

Première exécution `kubectl auth whoami` en SSH :

```
The connection to the server localhost:8080 was refused
```

Pourtant le token SA est bien monté (`/var/run/secrets/kubernetes.io/serviceaccount/`). Cause : sshd ouvre un nouveau shell de login qui **n'hérite pas** des variables d'environnement du conteneur (`KUBERNETES_SERVICE_HOST`, `KUBERNETES_SERVICE_PORT`, injectées par kubelet et présentes dans l'entrypoint). Sans elles, le client `kubectl` fallback sur `localhost:8080`.

Correction dans le `command:` du Deployment :

```bash
cat >/etc/profile.d/opsbox-env.sh <<EOF
export KUBERNETES_SERVICE_HOST="${KUBERNETES_SERVICE_HOST}"
export KUBERNETES_SERVICE_PORT="${KUBERNETES_SERVICE_PORT}"
EOF
```

Le `<<EOF` non quoté expanse les variables au moment du run de bash (qui a accès à l'env du conteneur). `/etc/profile.d/*.sh` est sourcé par tous les shells de login.

### 6.7. `rollout restart` ≠ `kubectl apply`

Confusion méthode côté assistant : après avoir édité le `command:` du Deployment dans le YAML local, suggestion de `kubectl rollout restart deploy/opsbox`. Mais `rollout restart` relance le pod avec la spec **présente dans le cluster**, pas celle du fichier local. La spec n'avait pas été propagée — donc le `cat /etc/profile.d/opsbox-env.sh` au boot n'existait pas.

Mémo : pour qu'un changement de manifest prenne effet, il faut **`kubectl apply -f`** d'abord. Si la spec a changé, `apply` déclenche automatiquement un rollout. Documenté dans le README opsbox.

---

## 7. Tests de validation

Une fois tous les pièges levés, batterie de tests passée intégralement sur le nouveau pod (suffixe `5f9449fb98-t7jsw` puis `8476b879f9-btzgc`) :

| # | Test | Résultat |
|---|---|---|
| 1 | `whoami` / `id` / `sudo -n true` | OK — `adrien (1001) groups=…,27(sudo)` |
| 2 | Outils installés (kubectl, helm, k9s, stern, jq, git…) | OK — tous présents |
| 3 | SA token monté + `kubectl auth whoami` | OK — `system:serviceaccount:ops:opsbox` |
| 3 | `kubectl get nodes` / `kubectl get ns` | OK |
| 3 | `kubectl auth can-i '*' '*' --all-namespaces` | `yes` (cluster-admin confirmé) |
| 4 | `helm list -A` / `helm history gitea -n gitea` | OK |
| 5 | DNS interne, API K8s (`curl https://kubernetes.default.svc/healthz`), Internet sortant | OK |
| 6 | **Persistance** : `~/PERSIST_TEST` survit à `rollout restart`, `/tmp` et `/opt` éphémères, hostname changé | OK |
| 7 | `kubectl` re-téléchargé au boot (date `mtime` cohérente avec le dernier restart) | OK |

**Critère bloquant** : test 6 passe. La devbox est fonctionnelle de bout en bout.

---

## 8. État final

### 8.1. Cluster

| Objet | Valeur |
|---|---|
| Namespace | `ops` |
| Deployment | `opsbox` (1 réplique, strategy `Recreate`) |
| Pod | `opsbox-…` Running |
| Image | `ubuntu:24.04` |
| ServiceAccount | `opsbox` |
| ClusterRoleBinding | `opsbox-cluster-admin` → `cluster-admin` |
| Service | `opsbox-ssh` LoadBalancer `192.168.100.201:22` |
| PVC | `opsbox-home` Bound 20 Gi local-path → `/home/adrien` |
| Secret | `opsbox-ssh-key` (1 clé : `authorized_key`) |

### 8.2. Accès

| Voie | Usage |
|---|---|
| Lens (Pod → Shell) | inspection rapide, exec |
| SSH `adrien@192.168.100.201` | shell quotidien, VS Code Remote-SSH |

### 8.3. VIPs MetalLB

| VIP | Service | Port(s) |
|---|---|---|
| 192.168.100.200 | Traefik | 80, 443 |
| **192.168.100.201** | **`ops/opsbox-ssh`** | **22** (nouveau) |
| 192.168.100.202 | `dev/devbox-ssh` | 22 |
| 192.168.100.203 | `gitea/gitea-ssh` | 22 |

Le pool MetalLB est désormais utilisé sur 4 VIPs sur 40.

### 8.4. Documentation

- [`services/devbox/README.md`](../services/devbox/README.md) — créé (doc rétroactive de la devbox `dev` existante)
- [`services/opsbox/README.md`](../services/opsbox/README.md) — créé
- [`services/opsbox/opsbox.yaml`](../services/opsbox/opsbox.yaml) — créé (manifest versionné dans le repo cette fois)
- [`services/README.md`](../services/README.md) — entrées `devbox` et `opsbox` ajoutées
- [`reseau/adressage.md`](../reseau/adressage.md) — VIP `.201` réattribuée à `opsbox-ssh`
- [`kubernetes/metallb/README.md`](../kubernetes/metallb/README.md) — entrée `.201 / opsbox-ssh / ops`
- [`kubernetes/stockage.md`](../kubernetes/stockage.md) — PVC `ops/opsbox-home`, total 76 → 96 Gi
- [`inventaire.md`](../inventaire.md) — service `opsbox-ssh` + PVC `opsbox-home`
- [`changelog.md`](../changelog.md) — ligne 2026-05-26 pointant vers ce CR
- [`journal/README.md`](README.md) — entrée CR16

---

## 9. Bénéfices

- **Devbox infra dédiée**, séparée du dev applicatif, avec accès K8s explicite via SA
- **VIP `.201` recyclée** (anciennement acme-dns, libérée le 2026-05-21)
- **Modèle mental RBAC clarifié** côté utilisateur : 3 couches (IdP → RBAC → SA), Authentik comme authn pure, K8s comme authz pur
- **Manifest versionné** (contrairement à la devbox `dev` qui n'existait que dans le cluster)
- **Doc devbox `dev` créée a posteriori** — alignement repo/cluster sur les deux devboxes
- **Sept pièges techniques tracés** dans la doc opsbox pour ne pas avoir à les rediagnostiquer

---

## 10. Reste à faire

### 10.1. Hygiène opsbox

1. **Persister les host keys SSH** dans le PVC (`/home/adrien/.ssh/host_keys/`) et adapter `sshd_config` — supprime l'avertissement client à chaque restart
2. **NetworkPolicies Cilium** : default-deny + egress maîtrisé (DNS, API K8s, dépôts publics au boot)
3. **Backup du PVC `opsbox-home`** : Velero ou `tar` cron vers NAS/Gitea (une fois le registry up)
4. **Image versionnée** : supprimer l'install au boot une fois Gitea container registry opérationnel

### 10.2. Vers une RBAC nominative

Discussion ouverte mais non implémentée :

- Brancher Authentik OIDC sur `kube-apiserver` (`--oidc-issuer-url`, `--oidc-groups-claim`, etc.)
- Définir les personas : `admin`, `dev`, `auditor`, `llm-readonly`, `llm-ops-<scope>`
- Basculer le `ClusterRoleBinding` `opsbox-cluster-admin` (SA) vers des bindings sur `Group` Authentik
- Activer `--audit-policy-file` sur l'apiserver **avant** d'ouvrir l'opsbox à des tiers
- Formaliser le break-glass (kubeconfig admin sur le master, accès restreint)

### 10.3. Devbox `dev`

Points soulevés en documentant l'existant :

- `/workspace` n'est pas persisté dans la devbox `dev` (créé par le script de boot mais pas monté sur le PVC). Soit le persister, soit documenter qu'il faut éviter d'y mettre quoi que ce soit.
- Backup du PVC `dev/devbox-home`.

---

## 11. Conclusion

Session démarrée sur une question conceptuelle (« comment gérer les permissions si je veux faire venir des tiers ? ») et terminée avec une devbox infra opérationnelle, un manifest versionné, un modèle mental RBAC posé, et une doc transverse réalignée.

Le seul gros écart de méthode tracé : la production prématurée de manifests `services/devbox/` puis `services/devbox-infra/` côté assistant, avant d'avoir constaté que la devbox `dev` existait. Tout a été supprimé et repris à partir de la comparaison avec l'existant — ce qui a abouti au nom `opsbox` (sur proposition utilisateur) et à une doc rétroactive de la devbox `dev`.

Sept pièges techniques rencontrés pendant le bringup (clé tronquée, cp au boot non rejoué, home en `0777`, host keys volatiles, env vars perdues en SSH, `rollout restart` vs `apply`, document YAML vide) ont tous été tracés dans le README opsbox pour ne pas avoir à les rediagnostiquer la prochaine fois — qu'il s'agisse du même utilisateur ou d'un autre.

Le travail RBAC nominatif reste devant : ce CR pose les fondations (séparation devbox dev / opsbox, modèle 3 couches, audit logs à activer, OIDC à brancher) sans encore livrer les `ClusterRole` finaux par persona. C'est le sujet logique du CR17.
