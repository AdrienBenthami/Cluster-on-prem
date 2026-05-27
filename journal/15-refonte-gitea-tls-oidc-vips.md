# Compte rendu technique

**Sujet : refonte architecture Gitea — séparation des VIPs MetalLB, TLS Let's Encrypt dédié, OIDC Authentik fonctionnel**

**Date : 2026-05-26**

**Périmètre :**
- Gitea (namespace `gitea`)
- Authentik (provider OIDC `gitea-oidc`)
- cert-manager (émission `gitea-tls`)
- MetalLB (réallocation VIP `192.168.100.203`)
- Documentation transversale (résidus acme-dns, README Gitea, inventaire, dette, changelog)

**Déclencheur :** la doc indiquait Gitea en `CrashLoopBackOff` depuis 98 jours et le SSO Authentik "à confirmer". Décision de redessiner l'archi Gitea (VIPs, TLS, auth, NetworkPolicies) avant tout diagnostic, puis appliquer. En cours de session, plusieurs constats ont infirmé la doc.

---

## 1. Situation initiale

### 1.1. État supposé (d'après la doc)

- Helm release `gitea` revision 4, chart `gitea-12.4.0`, app `1.24.6`
- Pods Gitea en `CrashLoopBackOff` depuis 98 jours
- `services/gitea/README.md` listait `init-directories` comme l'init container fautif
- Ingress `tls: []` (vide)
- VIP `192.168.100.200` partagée entre Traefik (80/443) et `gitea-ssh` (22) via `metallb.universe.tf/allow-shared-ip: shared-lb`
- Authentik SSO "à confirmer"
- Aucune NetworkPolicy

### 1.2. État réel observé

Vérification effective en début de session :

```bash
kubectl -n gitea get pods
# gitea-7bdc484f5-db2jq    1/1     Running    103d
# gitea-postgresql-0       1/1     Running    103d
# gitea-valkey-primary-0   1/1     Running    103d

curl -sk -o /dev/null -w "HTTPS %{http_code}\n" https://gitea.benthami.eu/
# HTTPS 200
```

Le service répond en HTTPS. La doc était périmée. Le secret `gitea-oauth-secret` est présent (2 clés). Le SSH est bien sur `.200` partagée.

### 1.3. Résidus acme-dns dans la doc

Bien que le démantèlement d'acme-dns ait été acté le 2026-05-21 (CR14), plusieurs fichiers mentionnaient encore acme-dns comme composant actif :

- `README.md` (liste services)
- `reseau/adressage.md` (VIP `.201` marquée "Déployé")
- `kubernetes/metallb/README.md` (entrée `acme-dns-dns`)
- `kubernetes/README.md` (namespace `acme-dns`)
- `kubernetes/stockage.md` (PVCs `data-acme-dns-postgres-0`, `acme-dns-data Pending`)
- `procedures/debug.md` (section "acme-dns — validation DNS-01" + lien NetworkPolicies)
- `procedures/installation.md` (étape acme-dns dans l'ordre de déploiement)

---

## 2. Décision d'architecture

### 2.1. VIPs MetalLB

Séparation pour lisibilité et future hygiène des NetworkPolicies.

| VIP | Service | Port | Décision |
|---|---|---|---|
| 192.168.100.200 | Traefik (HTTP/HTTPS) | 80, 443 | Conservé |
| 192.168.100.203 | `gitea-ssh` dédié | 22 | Nouvelle |
| 192.168.100.201 | — | — | Reste libre (anciennement acme-dns, non recyclée — résidus DNS publics potentiels) |

L'annotation `allow-shared-ip: shared-lb` sera retirée du service `gitea-ssh`.

### 2.2. Flux d'accès — choix d'option

Trois options envisagées :

1. **A — tout via Traefik** : HTTPS terminé par Traefik, Gitea reçoit du HTTP en interne. SSH sur VIP dédiée.
2. **B — VIP HTTPS dédiée à Gitea** : Gitea termine TLS lui-même, isolation forte vis-à-vis de Traefik.
3. **C — TCP passthrough SNI** : Traefik route SNI sans terminer TLS, Gitea termine TLS.

**Choix retenu : option A.**

Justifications :
- Co-tenance Traefik n'est pas un problème : si Gitea crash, Traefik renvoie 502 sur `gitea.benthami.eu` mais continue à servir les autres hosts. SPOF asymétrique (Traefik SPOF pour tous, Gitea SPOF que pour Gitea).
- Option B introduit la gestion TLS dans le pod Gitea (montage cert-manager Secret, configuration `[server]` Gitea) sans bénéfice pour un homelab.
- Option C ajoute de la complexité (CRD `TLSRoute`, troubleshooting plus délicat).
- Les seuls risques résiduels (saturation CPU/RAM/disque du worker mono-nœud) se traitent par `resources.limits` et quotas PVC, pas par séparation d'ingress.

### 2.3. Authentification

- Provider OIDC dédié Authentik (`gitea-oidc`) — déjà créé, secret `gitea-oauth-secret` déjà présent
- Pas de ForwardAuth Traefik (incompatible avec `git clone https://`)
- À terme : `DISABLE_REGISTRATION=true` (création comptes uniquement via SSO) + compte admin local "break-glass" pour reprise si Authentik HS

### 2.4. NetworkPolicies (cible)

Namespace `gitea`, à créer dans une étape ultérieure :
- default-deny ingress + egress
- ingress depuis ns `ingress` (Traefik) → svc HTTP 3000
- ingress depuis `world` → svc SSH 22 (entrée publique MetalLB)
- egress vers CoreDNS, Postgres in-ns, Valkey in-ns, `auth.benthami.eu`

---

## 3. Étape 1 — Nettoyage des résidus acme-dns dans la doc

Sept fichiers corrigés :

- [`README.md`](../README.md) — retiré acme-dns de la liste des services
- [`reseau/adressage.md`](../reseau/adressage.md) — VIP `.201` retirée du tableau MetalLB actif, note historique (libérée 2026-05-21) ajoutée
- [`kubernetes/metallb/README.md`](../kubernetes/metallb/README.md) — ligne `acme-dns-dns` retirée
- [`kubernetes/README.md`](../kubernetes/README.md) — namespace `acme-dns` retiré du tableau
- [`kubernetes/stockage.md`](../kubernetes/stockage.md) — PVCs `data-acme-dns-postgres-0` et `acme-dns-data` retirés, total recalculé 81 → 76 Gi
- [`procedures/debug.md`](../procedures/debug.md) — section "acme-dns — validation DNS-01" supprimée, lien NetworkPolicies retiré des références croisées
- [`procedures/installation.md`](../procedures/installation.md) — étape `acme-dns` retirée de l'ordre de déploiement

Mentions historiques conservées dans `incidents.md`, `changelog.md`, `cert-manager/archive/README.md`, `reseau/opnsense/README.md`, `reseau/dns.md` (documentent explicitement le démantèlement du 2026-05-21).

---

## 4. Étape 2 — Diagnostic et correction OIDC

### 4.1. Symptôme

Tentative de connexion via le bouton "Sign in with Authentik" sur `https://gitea.benthami.eu` → redirection vers Authentik qui affiche :

> Redirect URI Error — The request fails due to a missing, invalid, or mismatching redirection URI (redirect_uri).

### 4.2. Incident — tentative de dump `app.ini`

Pour "vérifier les flags `ENABLE_OPENID_SIGNIN` / `REGISTRATION` côté Gitea", commande `kubectl exec ... cat /data/gitea/conf/app.ini` proposée. `app.ini` contient `SECRET_KEY`, `INTERNAL_TOKEN`, `JWT_SECRET`, mot de passe Postgres en clair.

Refus utilisateur explicite. Règle retenue : pour lire un flag d'un fichier sensible, `grep` ciblé uniquement, jamais de dump complet. Mémoire `feedback_no_secret_dump.md` ajoutée côté assistant.

### 4.3. Analyse de l'erreur

URL de la requête authorize Authentik (extraite de la barre d'adresse) :

```
https://auth.benthami.eu/application/o/authorize/
  ?client_id=<your-oidc-client-id>
  &redirect_uri=https%3A%2F%2Fgitea.benthami.eu%2Fuser%2Foauth2%2Fauthentik%2Fcallback
  &response_type=code
  &scope=openid+profile+email+openid
  &state=...
```

`redirect_uri` décodé : `https://gitea.benthami.eu/user/oauth2/authentik/callback`. Forme correcte, HTTPS.

Capture du provider Authentik : champ **"URIs de redirection" vide**. Cause unique : aucune URI autorisée → tout `redirect_uri` est rejeté comme "missing/mismatching".

### 4.4. Anatomie de la callback URL (référence)

Forme générale Gitea pour tout provider OAuth/OIDC :

```
{ROOT_URL} / user/oauth2/ {name du provider} /callback
```

- `ROOT_URL` : [`services/gitea/values.yaml`](../services/gitea/values.yaml) → `gitea.config.server.ROOT_URL`
- `name du provider` : `gitea.oauth[].name` (ici : `authentik`)
- `/user/oauth2/` et `/callback` : segments fixes côté Gitea (hardcoded dans le code)

Donc pour un futur provider `name: github`, callback = `https://gitea.benthami.eu/user/oauth2/github/callback`.

### 4.5. Correction

Ajout dans le provider `gitea-oidc` Authentik (Type `Strict`) :

```
https://gitea.benthami.eu/user/oauth2/authentik/callback
```

Test SSO après sauvegarde : OK — login Authentik, retour sur Gitea connecté.

---

## 5. Étape 3 — Diagnostic et correction TLS

### 5.1. Symptôme

Utilisateur signale que "Gitea n'est pas en HTTPS" depuis le navigateur (cadenas barré/avertissement).

### 5.2. Vérifications

```bash
curl -sI http://gitea.benthami.eu/
# HTTP 308 → Location: https://gitea.benthami.eu/   (redirection Traefik OK)

curl -sI https://gitea.benthami.eu/
# exit 60 — TLS verify failure
```

Inspection du certificat servi :

```bash
echo | openssl s_client -connect gitea.benthami.eu:443 -servername gitea.benthami.eu 2>/dev/null \
  | openssl x509 -noout -issuer -subject -ext subjectAltName

# issuer=CN = TRAEFIK DEFAULT CERT
# subject=CN = TRAEFIK DEFAULT CERT
# SAN = DNS:3cb4c9912ee9aab47c5c26270a5dfb79.5202ee2f6e4c6042eaba7100535090f0.traefik.default
```

### 5.3. Cause

`services/gitea/values.yaml` ligne `tls: []` (vide). Sans déclaration `tls:` dans l'Ingress, Traefik n'a aucun cert associé au host `gitea.benthami.eu` et fallback sur son cert auto-signé par défaut. Le navigateur affiche un avertissement TLS, interprété comme "pas en HTTPS".

### 5.4. Création du Certificate cert-manager

Manifest [`services/gitea/certificate.yaml`](../services/gitea/certificate.yaml) :

```yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: gitea-tls
  namespace: gitea
spec:
  secretName: gitea-tls
  privateKey:
    rotationPolicy: Always
  issuerRef:
    name: letsencrypt-prod-cloudflare
    kind: ClusterIssuer
  dnsNames:
    - gitea.benthami.eu
```

`privateKey.rotationPolicy: Always` ajouté explicitement pour silencer le warning cert-manager v1.18+ (changement de défaut `Never` → `Always`). Sémantique : la clé privée est régénérée à chaque renouvellement, limite l'impact d'une éventuelle compromission.

```bash
kubectl apply -f services/gitea/certificate.yaml
kubectl -n gitea get certificate gitea-tls
# NAME        READY   SECRET      AGE
# gitea-tls   True    gitea-tls   2m
```

Émission OK en quelques secondes via le solver DNS-01 Cloudflare.

### 5.5. Référencement du secret dans l'Ingress

Modification [`services/gitea/values.yaml`](../services/gitea/values.yaml) :

```yaml
ingress:
  className: traefik
  enabled: true
  hosts:
  - host: gitea.benthami.eu
    paths:
    - path: /
      pathType: Prefix
  tls:
  - secretName: gitea-tls
    hosts:
    - gitea.benthami.eu
```

Correction collatérale : `pathType: Prefix` était mal indenté au niveau `host`, déplacé dans l'entrée `paths` (le chart l'ignorait silencieusement).

---

## 6. Étape 4 — Cascade d'upgrades Helm

### 6.1. Premier essai (rev 5)

```bash
helm upgrade gitea gitea-charts/gitea -n gitea -f values.yaml --reuse-values
```

Pas de `--version` → Helm tire la dernière version disponible = `gitea-12.6.0` (app `1.26.1`).

Erreur :

```
Error: UPGRADE FAILED: gitea/templates/tests/test-http-connection.yaml:12:20
  executing "gitea.openshift.enabled" at <.Values.openshift.enabled>:
    nil pointer evaluating interface {}.enabled
```

Le template référence `.Values.openshift.enabled` sans default. `--reuse-values` court-circuite les defaults du chart, et la clé n'était pas dans la release rev 4 (chart 12.4 ne l'utilisait pas). Ajout dans `values.yaml` :

```yaml
openshift:
  enabled: false
```

Second essai → même problème, autre clé :

```
Error: UPGRADE FAILED: gitea/templates/gitea/route.yaml:1:14
  executing at <.Values.route.enabled>:
    nil pointer evaluating interface {}.enabled
```

Ajout :

```yaml
route:
  enabled: false
```

Troisième essai → succès. **Rev 5 deployed, chart 12.6.0, app 1.26.1.** Migration DB schéma Gitea 321 → 331 effectuée par le pod neuf au démarrage.

### 6.2. Découverte annexe : header `USER-SUPPLIED VALUES`

[`services/gitea/values.yaml`](../services/gitea/values.yaml) commençait par une ligne `USER-SUPPLIED VALUES:` — résidu de la sortie `helm get values gitea` enregistrée verbatim comme fichier. Helm la parse comme clé top-level à valeur nulle (inoffensive mais salissante). Retirée.

### 6.3. Incident — downgrade non sollicité (rev 6 à 8)

Pour "stabiliser sur la version connue du README", proposition (assistant) de pinner `--version 12.4.0`. Erreur méthodologique : `helm history` n'a pas été consulté avant. La version 12.4.0 mentionnée dans le README local était **périmée** (état rev 4 antérieur à la session) ; la version *réelle* en cours était 12.6.0.

Effet observé :

```bash
helm history gitea -n gitea
# 5  gitea-12.6.0  1.26.1  superseded
# 6  gitea-12.6.0  1.26.1  superseded   (tentatives d'upgrade)
# 7  gitea-12.6.0  1.26.1  superseded
# 8  gitea-12.4.0  1.24.6  superseded   (downgrade tenté)

kubectl -n gitea get pods
# gitea-6f5fd75678-lqzhn   1/1     Running       (1.26.1)
# gitea-7bdc484f5-b7l6w    0/1     Init:Error    (1.24.6)

kubectl -n gitea logs gitea-7bdc484f5-b7l6w -c configure-gitea --previous
# [F] Migration Error: Your database (migration version: 331) is for a newer Gitea,
# you can not use the newer database for this old Gitea release (321).
# Gitea will exit to keep your database safe and unchanged.
```

Le pod neuf 1.24.6 refuse de démarrer sur une DB schéma 331 — comportement de protection de Gitea, conforme. Le rolling update est bloqué, le pod 1.26.1 (rev 5) reste Running, **service jamais interrompu**.

Le downgrade n'avait pas été demandé par l'utilisateur. Excuses formulées, mémoire `feedback_never_downgrade_without_consent.md` ajoutée : toujours consulter `helm history` / `kubectl get -o jsonpath` avant de proposer un `--version`, ne jamais se baser sur la doc locale, tout downgrade demande consentement explicite.

### 6.4. Retour en avant (rev 9)

```bash
helm repo update gitea-charts
helm search repo gitea-charts/gitea --versions | head -3
# gitea-charts/gitea  12.6.0  1.26.1
# gitea-charts/gitea  12.5.3  1.25.5
# gitea-charts/gitea  12.5.2  1.25.5

helm upgrade gitea gitea-charts/gitea -n gitea -f values.yaml --reuse-values
# Release "gitea" has been upgraded.
# REVISION: 9
# CHART: gitea-12.6.0  APP VERSION: 1.26.1
```

État sain rétabli. Pod unique 1.26.1 Running.

---

## 7. Étape 5 — Migration VIP SSH `192.168.100.200` → `192.168.100.203`

### 7.1. Modification `values.yaml`

La section `service.ssh` est passée de :

```yaml
service:
  ssh:
    annotations:
      metallb.universe.tf/allow-shared-ip: shared-lb
    loadBalancerIP: 192.168.100.200
    port: 22
    type: LoadBalancer
```

à :

```yaml
service:
  ssh:
    loadBalancerIP: 192.168.100.203
    port: 22
    type: LoadBalancer
```

Suppression de l'annotation `allow-shared-ip: shared-lb` (plus de partage avec Traefik) et bascule de l'adresse cible.

### 7.2. Application et vérification

La modification a été appliquée dans le même flux que les corrections TLS et le retour en avant (§6.4) — pas d'upgrade Helm dédié. MetalLB a libéré `.200` sur `gitea-ssh` et alloué `.203`. Après stabilisation rev 9 :

```bash
kubectl -n gitea get svc gitea-ssh
# NAME        TYPE           EXTERNAL-IP       PORT(S)
# gitea-ssh   LoadBalancer   192.168.100.203   22:31876/TCP

ssh -T -p 22 -o StrictHostKeyChecking=no git@192.168.100.203
# Warning: Permanently added '192.168.100.203' (RSA) to the list of known hosts.
# git@192.168.100.203: Permission denied (publickey).
```

`Permission denied (publickey)` = service répond, refus normal en l'absence de clé associée à un compte Gitea. Migration validée.

---

## 8. État final

### 8.1. Helm release

| Champ | Valeur |
|---|---|
| Release | `gitea` |
| Revision | 9 |
| Chart | `gitea-12.6.0` |
| App version | `1.26.1` |
| Schéma DB Gitea | 331 |
| Statut | deployed |

### 8.2. Pods et services

```bash
kubectl -n gitea get pods
# gitea-6f5fd75678-lqzhn   1/1   Running   (unique)
# gitea-postgresql-0       1/1   Running
# gitea-valkey-primary-0   1/1   Running
```

| Service | Type | VIP | Port(s) |
|---|---|---|---|
| `gitea-http` | ClusterIP | — | 3000 |
| `gitea-ssh` | LoadBalancer | 192.168.100.203 | 22 |
| `gitea-postgresql` | ClusterIP | — | 5432 |
| `gitea-valkey-primary` | ClusterIP | — | 6379 |

### 8.3. TLS

| Champ | Valeur |
|---|---|
| Cert | Let's Encrypt R13 |
| Subject | `CN=gitea.benthami.eu` |
| Issuer cert-manager | `letsencrypt-prod-cloudflare` |
| Secret K8s | `gitea-tls` (`kubernetes.io/tls`) |
| Validité | jusqu'au 2026-08-24 (renouvellement auto ~30j avant) |
| `privateKey.rotationPolicy` | `Always` |

### 8.4. Authentification OIDC

| Champ | Valeur |
|---|---|
| Provider Authentik | `gitea-oidc` (Type Confidentiel) |
| Application Authentik | `Gitea` (slug `gitea`) |
| Redirect URI autorisée | `https://gitea.benthami.eu/user/oauth2/authentik/callback` (Strict) |
| Scopes | `openid profile email` |
| Subject mode | identifiant haché |
| Login SSO testé | OK |

### 8.5. VIPs MetalLB

| VIP | Service | Port(s) |
|---|---|---|
| 192.168.100.200 | Traefik | 80, 443 |
| 192.168.100.202 | `dev/devbox-ssh` | 22 |
| 192.168.100.203 | `gitea/gitea-ssh` | 22 |

`192.168.100.201` toujours libre (historique acme-dns, non recyclée).

---

## 9. Bénéfices obtenus

- **TLS finalisé** sur `gitea.benthami.eu` (cert dédié émis automatiquement, plus de fallback Traefik default)
- **SSO Authentik fonctionnel** — bouton "Sign in with Authentik" opérationnel, login validé
- **Séparation VIPs** : `gitea-ssh` n'est plus en `shared-ip` avec Traefik, surface de firewall future plus lisible
- **Montée de version Gitea** 1.24.6 → 1.26.1 effectuée en cours de session (effet de bord initialement non planifié — l'upgrade sans `--version` a tiré la dernière)
- **Doc réalignée** : 7 résidus acme-dns supprimés, README Gitea refondu, inventaire/dette/changelog à jour
- **Deux mémoires d'assistant** ajoutées (no-secret-dump, never-downgrade-without-consent) pour éviter la reproduction des deux incidents de session

---

## 10. Finitions documentaires

### 10.1. Fichiers transversaux

- [`README.md`](../README.md) — `acme-dns` retiré de la liste services
- [`inventaire.md`](../inventaire.md) — Helm release `gitea-12.6.0`/`1.26.1` dernière MAJ 2026-05-26, VIP `gitea-ssh` passée à `192.168.100.203`, section *Pods en erreur* vidée (aucun pod en erreur)
- [`dette.md`](../dette.md) — lignes obsolètes retirées (Gitea CrashLoopBackOff, TLS non finalisé, SSO non connecté). Ajout de 3 lignes : NetworkPolicies absentes (priorité Moyenne), break-glass + `DISABLE_REGISTRATION` non configurés (Moyenne), pas de backup PostgreSQL pré-migration (Moyenne)
- [`changelog.md`](../changelog.md) — ligne 2026-05-26 ajoutée pointant vers ce CR

### 10.2. Réseau

- [`reseau/adressage.md`](../reseau/adressage.md) — VIP `.203` réservée "Gitea SSH (dédié)" ; ligne historique pour `.201`
- [`kubernetes/metallb/README.md`](../kubernetes/metallb/README.md) — entrée `.203 / gitea-ssh / gitea / TCP 22` ajoutée
- [`kubernetes/README.md`](../kubernetes/README.md) — namespace `acme-dns` retiré
- [`kubernetes/stockage.md`](../kubernetes/stockage.md) — PVCs `acme-dns/*` retirés, total recalculé

### 10.3. Procédures

- [`procedures/debug.md`](../procedures/debug.md) — section *acme-dns — validation DNS-01* supprimée, lien NetworkPolicies acme-dns retiré
- [`procedures/installation.md`](../procedures/installation.md) — étape `acme-dns` retirée de l'ordre de déploiement, étape `cert-manager` annotée "solver DNS-01 Cloudflare natif"

### 10.4. Services

- [`services/README.md`](../services/README.md) — Gitea passé de "CrashLoopBackOff" à "Déployé", versions chart/app actualisées
- [`services/gitea/README.md`](../services/gitea/README.md) — refondu intégralement : état (tableau par composant), Accès, Architecture cible (flux, VIPs, OIDC, NetworkPolicies à venir), Dépendances, Stockage, Configuration, Déploiement (vérifier `helm history` + `helm search` avant chaque upgrade), Reste à faire
- [`services/gitea/values.yaml`](../services/gitea/values.yaml) — header `USER-SUPPLIED VALUES:` retiré, `tls:` rempli, `pathType` réindenté, `openshift.enabled` et `route.enabled` ajoutés, `ssh.loadBalancerIP` passé à `.203`, annotation `allow-shared-ip` retirée
- [`services/gitea/certificate.yaml`](../services/gitea/certificate.yaml) — créé

### 10.5. Journal

- [`journal/README.md`](README.md) — entrée CR15 ajoutée au tableau des comptes rendus

---

## 11. Reste à traiter (hors scope de cette session)

### 11.1. Gitea — finitions sécurité

1. Créer le compte admin local "break-glass" (login direct, hors SSO) avant toute désactivation de la registration
2. Activer `DISABLE_REGISTRATION=true` via `values.yaml` une fois l'admin local en place
3. Écrire les NetworkPolicies Cilium dans `services/gitea/network-policies/` (default-deny + ingress Traefik/world SSH + egress DNS/Postgres/Valkey/Authentik)
4. Valider end-to-end : clone HTTPS authentifié, push SSH, login SSO, fallback admin break-glass

### 11.2. Backup PostgreSQL Gitea

L'upgrade Gitea 1.24.6 → 1.26.1 a migré le schéma DB de 321 à 331 sans backup pré-migration disponible. Aucun rollback possible si la migration avait corrompu les données. Avant la prochaine montée majeure, mettre en place une stratégie de backup (`pg_dump` planifié, snapshot PVC, ou les deux). Listé dans `dette.md`.

### 11.3. Modélisation RBAC K8s

Discussion ouverte en fin de session, trois rôles cibles :
- `admin` (existant — droits actuels de l'utilisateur)
- `audit` (read-only, pour présenter le cluster à un tiers ou recevoir des conseils) — implémentation proposée : `ClusterRole view` built-in, qui exclut nativement `secrets` et RBAC
- `ia` (read-only restreint, pour brider les agents IA dans le cluster) — implémentation proposée : `ClusterRole` custom basé sur `view` mais excluant `secrets`, `configmaps`, `pods/exec`, `pods/attach`, `pods/portforward`

Cinq questions de design ouvertes : namespace des SA, `pods/log` (autorisé ou bloqué côté IA), `events`, accès aux Custom Resources, métadonnées Secrets. Décisions en attente.

### 11.4. Modélisation Authentik + Gitea Teams

Mapping groupes Authentik → Teams Gitea via les claims OIDC (`USERNAME_CLAIM`, `GROUPS_CLAIM`, `ADMIN_GROUP`) — à traiter quand le périmètre des utilisateurs/rôles humains sera défini. Pas urgent en mode admin solo.

---

## 12. Conclusion

La session a démarré sur un faux diagnostic ("Gitea CrashLoopBackOff") issu d'une doc périmée, et s'est terminée par un service Gitea durci sur trois plans (TLS, SSO, séparation des VIPs SSH). L'écart entre l'état supposé et l'état réel a été acté tôt, ce qui a permis de réorienter la session vers les vrais sujets (redirect URI manquant côté Authentik, cert auto-signé servi par Traefik).

Deux incidents de méthode côté assistant ont été tracés et corrigés : (1) une tentative de dump `app.ini` rejetée par l'utilisateur pour cause de secrets en clair, et (2) un downgrade Gitea 1.26.1 → 1.24.6 proposé sans avoir vérifié l'état Helm courant, intercepté par le mécanisme de protection de Gitea (pod neuf refuse une DB schéma plus récent que ce qu'il connaît). Aucune perte de service dans les deux cas. Les leçons sont matérialisées dans deux mémoires (`feedback_no_secret_dump.md`, `feedback_never_downgrade_without_consent.md`).

La documentation transversale a été réalignée dans la foulée — pas de dérive entre cluster et doc à la sortie de session. Le reste à faire est listé séparément (finitions sécurité Gitea, backup PG, RBAC K8s, Authentik Teams) et reste à arbitrer en fonction des priorités.
