# Compte rendu technique

**Sujet : migration de la chaîne TLS DNS-01 d'acme-dns vers le solver natif Cloudflare, démantèlement complet d'acme-dns, fermeture du port 53 WAN**

**Date : 2026-05-21**

**Périmètre :**
- cert-manager (namespace `cert-manager`)
- Authentik (namespace `authentik`)
- acme-dns (namespace `acme-dns` — démantelé)
- OPNsense (NAT, alias, firewall)
- Freebox (gestion des ports)
- Repo de documentation

**Déclencheur :** bascule du registrar de zone `benthami.eu` d'OVH vers Cloudflare. Décision de saisir l'opportunité pour migrer le solver DNS-01 vers la méthode officiellement supportée par cert-manager (Cloudflare natif), supprimer la dépendance acme-dns, et fermer le port 53 WAN.

---

## 1. Situation initiale

### 1.1. Chaîne TLS existante

- cert-manager v1.19.2 déployé avec solver `dns01.acmeDNS`
- ClusterIssuer `letsencrypt-staging-acme-dns` (Ready=True)
- ClusterIssuer `letsencrypt-prod-acme-dns` (jamais appliqué — manifest seul)
- acme-dns v2.0.2 déployé manuellement dans le namespace `acme-dns`
  - Deployment + StatefulSet PostgreSQL 16
  - VIP MetalLB `192.168.100.201` (UDP/TCP 53)
  - 4 NetworkPolicies standard + 1 CiliumNetworkPolicy `world`-aware
  - Secret cert-manager/`acme-dns-auth` au format `map[string]goacmedns.Account`
- DNAT OPNsense 53 UDP/TCP → `K8S_ACME_VIP` (alias 192.168.100.201)
- Port forward Freebox 53 UDP/TCP → OPNsense
- Délégations CNAME `_acme-challenge.*` historiquement chez OVH (non reprises chez Cloudflare)

### 1.2. État applicatif

- `Certificate auth-benthami-eu` (namespace `authentik`) : Ready=False depuis 78 jours
  - Référence `secretName: auth-benthami-eu-tls` (secret inexistant)
  - Issuer pointé : `letsencrypt-staging-acme-dns`
- Ingress `authentik-server` : annotations + bloc `spec.tls` historiquement strippés (CR09) — TLS non opérationnel
- Pas de certificat valide servi sur `auth.benthami.eu`

### 1.3. Résidus identifiés

- Manifest `services/cert-manager/letsencrypt-ovh-dns-staging.yaml` — webhook OVH abandonné en janvier 2026 (CR08, CR09)
- Manifest `services/cert-manager/clusterissuer-prod.yaml` — variante acme-dns jamais appliquée
- Dossier `services/cert-manager/rbac/` contenant un fichier YAML vide (Role du webhook OVH, créé via Lens, jamais exporté)

---

## 2. Décision d'architecture

Trois options envisagées :

1. **Garder acme-dns** + recréer les CNAME `_acme-challenge.*` chez Cloudflare. Préserve la dette acme-dns (PVC Pending, PG mono-instance, port 53 WAN).
2. **Solver `dns01.cloudflare` natif cert-manager**. Élimine acme-dns, supprime 1 port WAN, élimine 3 lignes de dette en bloc.
3. **Cloudflare Tunnel + Origin Certificates**. Refonte architecturale majeure, hors scope.

**Choix retenu : option 2.**

Justifications :
- Cloudflare est officiellement supporté par cert-manager (pas un webhook tiers) — ce qui avait coulé l'option OVH (CR08) ne s'applique plus
- Réduction de la dette technique : suppression acme-dns = 3 items de `dette.md` traités d'un coup
- Réduction de la surface d'attaque WAN : passage de 2 ports (53 + 16000) à 1 port (16000 WireGuard uniquement)
- Simplification architecturale : un namespace, un StatefulSet, un Deployment, des NetworkPolicies, un secret au format JSON spécifique en moins

---

## 3. Étape 1 — Token API Cloudflare

### 3.1. Création du token

Création via dashboard Cloudflare → My Profile → API Tokens → template *"Edit zone DNS"*.

| Paramètre | Valeur |
|---|---|
| Nom | `cert-manager-dns01-benthami-eu` |
| Permissions | `Zone:DNS:Edit` + `Zone:Zone:Read` |
| Zone Resources | `Specific zone: benthami.eu` |
| Client IP Filtering | `Is in <IP-PUBLIQUE>` (IP publique WAN Freebox) |
| TTL | Aucune expiration |

Le filtre IP est une défense en profondeur. Impose une procédure de mise à jour du token si l'IP publique change (Freebox dynamique).

### 3.2. Documentation

Ajout d'une section *IP publique* dans `reseau/README.md` avec la valeur courante (`<IP-PUBLIQUE>`), la commande de vérification (`curl -4 ifconfig.me`), et la note sur l'impact côté Cloudflare.

---

## 4. Étape 2 — Secret K8s + ClusterIssuer staging

### 4.1. Secret

Création dans `cert-manager` :

```yaml
kind: Secret
type: Opaque
metadata:
  name: cloudflare-api-token
  namespace: cert-manager
data:
  api-token: <token base64>
```

### 4.2. ClusterIssuer staging

Manifest `services/cert-manager/clusterissuer-staging-cloudflare.yaml` :

```yaml
spec:
  acme:
    email: <your-email@example.com>
    server: https://acme-staging-v02.api.letsencrypt.org/directory
    privateKeySecretRef:
      name: letsencrypt-staging-cloudflare-account-key
    solvers:
      - dns01:
          cloudflare:
            apiTokenSecretRef:
              name: cloudflare-api-token
              key: api-token
        selector:
          dnsZones:
            - benthami.eu
```

Choix structurants :
- `apiTokenSecretRef` (pas `apiKey + email`) — méthode recommandée, token scopé
- `selector.dnsZones` — bonne hygiène pour les multi-zones futures
- Email réutilisé de l'existant

Résultat : `Ready=True` en quelques secondes, compte ACME enregistré côté Let's Encrypt staging.

---

## 5. Étape 3 — Validation de chaîne (certificat de test)

### 5.1. Manifest jetable

Création de `services/cert-manager/tests/` avec :
- `test-certificate-staging-cloudflare.yaml` — Certificate sur `test-cf.benthami.eu` dans namespace `test`
- `README.md` — convention sandbox, procédure apply/verify/cleanup

### 5.2. Émission

- `Certificate test-cloudflare-staging` créé
- ACME order → challenge DNS-01
- TXT `_acme-challenge.test-cf` créé puis nettoyé par cert-manager via API Cloudflare
- `Ready=True` en ~1 min
- Issuer du cert émis : `(STAGING) Pretend Pear X1` — chaîne validée

### 5.3. Nettoyage

Suppression du Certificate de test et du secret. Manifest conservé en repo pour réutilisation future.

---

## 6. Étape 4 — ClusterIssuer production

Manifest `services/cert-manager/clusterissuer-prod-cloudflare.yaml` — identique au staging hormis :
- `server: https://acme-v02.api.letsencrypt.org/directory`
- `privateKeySecretRef.name: letsencrypt-prod-cloudflare-account-key`

Résultat : `Ready=True`, compte ACME prod enregistré.

État côté cluster à ce stade : 4 ClusterIssuers actifs côte-à-côte (2 acme-dns legacy + 2 cloudflare neufs). Cohabitation volontaire — démantèlement des anciens uniquement après bascule des Certificates applicatifs.

---

## 7. Étape 5 — Migration `auth.benthami.eu`

### 7.1. Diagnostic préalable

État réel observé (différent de l'état committé Helm) :
- `Ingress authentik-server` : aucune annotation cert-manager, pas de bloc `spec.tls`, ports 80 uniquement
- `Certificate auth-benthami-eu` : Ready=False, secret inexistant, issuer `letsencrypt-staging-acme-dns`
- Secrets `auth-benthami-eu-tls` et `authentik-tls` : aucun

Conclusion : Helm et cluster divergaient depuis CR09 (strip manuel de l'Ingress lors du démantèlement webhook OVH).

### 7.2. Action

1. Modification de `services/authentik/values.yaml` :
   - `cert-manager.io/cluster-issuer: letsencrypt-staging` → `letsencrypt-prod-cloudflare`
2. Suppression du Certificate cassé : `kubectl -n authentik delete certificate auth-benthami-eu`
3. `helm upgrade authentik authentik/authentik -n authentik -f services/authentik/values.yaml`
4. L'Ingress reconstruit récupère annotation + bloc TLS du chart
5. IngressShim de cert-manager crée automatiquement `Certificate authentik-tls`
6. Émission ACME Cloudflare → Let's Encrypt prod → `Ready=True` en ~2 min

### 7.3. Validation

- `Certificate authentik-tls` Ready=True
- Secret `authentik-tls` contient un cert signé par Let's Encrypt **prod** (`E5`/`E6`)
- Ingress en ports 80, 443 — TLS opérationnel sur `https://auth.benthami.eu`

---

## 8. Démantèlement acme-dns

### 8.1. Sujet 1 — Balayage des références (kubectl)

Vérification qu'aucun objet vivant ne référence un ClusterIssuer acme-dns :
- `kubectl get certificate -A` filtré : 0 résultat
- `kubectl get ingress -A` filtré sur annotation : 0 résultat
- `kubectl get order,challenge -A` : seul l'Order valid du nouveau cert Cloudflare reste (sera nettoyé par cert-manager)

### 8.2. Sujet 2 — Suppression du ClusterIssuer staging acme-dns

```bash
kubectl delete clusterissuer letsencrypt-staging-acme-dns
kubectl -n cert-manager delete secret letsencrypt-staging-acme-dns-account-key
```

Archivage du manifest `services/cert-manager/clusterissuer-staging.yaml` vers `services/cert-manager/archive/clusterissuer-staging-acme-dns.yaml` (`git mv`).

### 8.3. Nettoyage résidus webhook OVH

Vérification cluster : aucun Role/RoleBinding/Secret/SA/Deploy OVH résiduel (déjà nettoyé en janvier 2026).
Suppression du dossier `services/cert-manager/rbac/` (fichier YAML vide) via `git rm`.

### 8.4. Sujet 3 — Désinstallation acme-dns

Séquence exécutée :

```bash
kubectl -n cert-manager delete secret acme-dns-auth
kubectl -n acme-dns delete deployment acme-dns
kubectl -n acme-dns delete statefulset acme-dns-postgres
kubectl -n acme-dns delete svc acme-dns-api acme-dns-dns acme-dns-postgres
kubectl -n acme-dns delete secret acme-dns-config acme-dns-postgres-auth
kubectl -n acme-dns delete configmap acme-dns-config
kubectl -n acme-dns delete networkpolicy 00-default-deny 10-acme-dns-control-plane 30-acme-dns-postgres-client 30-acme-dns-postgres-server
kubectl -n acme-dns delete ciliumnetworkpolicy 20-acme-dns-dns53
kubectl -n acme-dns delete pvc acme-dns-data data-acme-dns-postgres-0
kubectl delete namespace acme-dns
```

Vérifications post-suppression :
- `kubectl get namespace acme-dns` → NotFound
- `kubectl get pv | grep acme-dns-postgres` → vide (reclaim policy `Delete` local-path appliquée)
- `kubectl get svc -A | grep 192.168.100.201` → vide (VIP MetalLB libérée)

Données PostgreSQL (5 Gi) supprimées sciemment — éphémères, comptes ACME-DNS obsolètes.

### 8.5. Sujet 4 — Archivage manifests + Cloudflare

Archivage `services/acme-dns/` → `archive/manifests/acme-dns/` (`git mv`).
Création de la convention `archive/manifests/` : sous-dossier dédié aux composants techniques démantelés, séparé de l'archive documentaire à la racine.

Vérification Cloudflare DNS : aucun enregistrement `_acme-challenge.*` à nettoyer (non reportés depuis OVH lors du transfert de registrar).

### 8.6. Sujet 5 — Fermeture port 53 WAN

**OPNsense (Pare-feu → NAT → Redirection de port)** :
- Suppression `WAN DNS UDP -> acme-dns` (DNAT UDP 53 → K8S_ACME_VIP)
- Suppression `WAN DNS TCP -> acme-dns` (DNAT TCP 53 → K8S_ACME_VIP)
- Règles firewall WAN auto-générées supprimées en cascade (règles liées)

**OPNsense (Pare-feu → Alias)** :
- Suppression alias `K8S_ACME_VIP` (192.168.100.201)

**Freebox (Gestion des ports)** :
- Suppression redirection `udp WAN:53 → LAN:53` vers OPNsense
- Suppression redirection `tcp WAN:53 → LAN:53` vers OPNsense
- Conservation `udp WAN:16000 → LAN:51820` (WireGuard)

Validation externe : scan de `<IP-PUBLIQUE>` port 53 → `closed`.

---

## 9. Découverte annexe : port WireGuard

Au cours du sujet 5, observation que le port WAN WireGuard est **16000** (pas 51820 comme dans la doc). Raison : le port 51820 n'est pas dans la plage allouable par la Freebox, port-mapping WAN ≠ LAN forcé.

Correction de 3 fichiers :
- `acces.md` (endpoint client)
- `reseau/wireguard.md` (table serveur)
- `securite.md` (tableau d'exposition WAN)

---

## 10. État final

### 10.1. ClusterIssuers actifs

| Nom | Solver | Serveur ACME |
|---|---|---|
| `letsencrypt-staging-cloudflare` | Cloudflare DNS-01 | staging |
| `letsencrypt-prod-cloudflare` | Cloudflare DNS-01 | prod |

### 10.2. Certificates actifs

| Namespace | Nom | Issuer | État |
|---|---|---|---|
| `authentik` | `authentik-tls` | `letsencrypt-prod-cloudflare` | Ready=True |

### 10.3. Exposition WAN

| Port WAN | Protocole | Service |
|---|---|---|
| 16000 | UDP | WireGuard VPN (mappé sur 51820 LAN) |

Surface ramenée à **1 port unique**.

### 10.4. Repo

Structure cible respectée :
- `services/` : services vivants uniquement
- `services/*/archive/` : variantes obsolètes locales (cert-manager ClusterIssuers anciens)
- `archive/manifests/` : composants entièrement démantelés (acme-dns)
- `archive/` (racine) : documentation obsolète (markdown)

---

## 11. Bénéfices obtenus

- Surface d'attaque WAN divisée par 2 (1 port au lieu de 2)
- 3 lignes de `dette.md` traitées (PVC `acme-dns-data` Pending, PostgreSQL acme-dns mono-instance, registration acme-dns)
- Suppression d'une dépendance externe (l'image `joohoi/acme-dns` n'est plus à maintenir)
- Suppression du namespace `acme-dns`, du Deployment, du StatefulSet, de 4 NetworkPolicies + 1 CiliumNetworkPolicy, de 2 PVCs (+1 PV), de 2 Secrets, de 2 ConfigMaps, de 3 Services
- Suppression de 2 ClusterIssuers (staging et prod acme-dns)
- Solver TLS désormais officiellement supporté par cert-manager (pas un webhook tiers)
- Doc réseau enrichie : double NAT Freebox→OPNsense documenté, alias OPNsense, port-mapping WireGuard

---

## 12. Finitions documentaires

Mise à jour de l'ensemble de la documentation transversale pour refléter l'état post-migration :

### 12.1. Fichiers transversaux

- `dette.md` — retrait des 3 lignes acme-dns (PVC `acme-dns-data` Pending, PostgreSQL mono-instance, registration). La ligne "TLS non finalisé" est reformulée pour ne concerner que Gitea, avec renvoi à ce CR pour Authentik.
- `inventaire.md` — date passée à 2026-05-22. Lignes acme-dns retirées des tableaux *Services LoadBalancer (VIPs MetalLB)* et *PVCs*. Note ajoutée pour la VIP `192.168.100.201` désormais libre.
- `securite.md` — *Surface d'exposition WAN* ramenée à un seul port. Tableau *État des certificats TLS* : `auth.benthami.eu` passé en `READY=True` (issuer `letsencrypt-prod-cloudflare`). Section *NetworkPolicies Cilium* réécrite (set acme-dns archivé, état actuel = aucune NetworkPolicy explicite).
- `changelog.md` — ligne 2026-05-22 ajoutée pointant vers ce CR.

### 12.2. Réseau

- `reseau/dns.md` — section *DNS public OVH* remplacée par *DNS public Cloudflare*. Rôle du DNS dans la chaîne TLS clarifié (TXT éphémères créés/nettoyés par cert-manager, plus de CNAME de délégation). Tableau des enregistrements actifs (MX, SPF, CNAME portfolio, etc.). Référence au token API et au filtre IP.
- `reseau/README.md` — section *IP publique* ajoutée (déjà fait en étape 1).
- `reseau/opnsense/README.md` — réécrit : double NAT Freebox→OPNsense documenté, tableau des alias K8S_*, état actuel des redirections (aucune DNAT WAN active), redirections Freebox listées (WireGuard uniquement).
- `reseau/wireguard.md` + `acces.md` — port WAN passé à 16000 (port-mapping WAN ≠ LAN forcé par limitation Freebox).

### 12.3. Services

- `services/README.md` — ligne acme-dns retirée du tableau des services actifs.
- `services/cert-manager/README.md` — réécrit : ClusterIssuers actifs, dépendances actuelles (Cloudflare API + Secret token), liens vers `tests/` et `archive/`.
- `services/cert-manager/archive/README.md` — tableau des manifests archivés (webhook OVH, ClusterIssuers acme-dns prod et staging).
- `services/cert-manager/tests/README.md` — convention sandbox documentée.

### 12.4. Archives

- `archive/README.md` — restructuré en deux sections (Documentation / Manifests). Convention d'archivage explicitée : un composant n'arrive ici qu'une fois supprimé du cluster.
- `archive/manifests/acme-dns/README.md` — réécrit en mode rétrospective : contexte historique, état au moment du démantèlement, liens vers les 5 CRs concernés (CR08, CR09, CR11, CR12, CR13, et ce CR14).

### 12.5. Journal

- `journal/README.md` — entrée CR14 ajoutée au tableau des comptes rendus.

---

## 13. Reste à traiter (hors scope de cette session)

- Migration TLS de Gitea (`gitea.benthami.eu`) — bloquée par le CrashLoopBackOff du service, sujet séparé. Quand Gitea sera réparé, la bascule sera identique à celle d'Authentik (Helm values → annotation `cert-manager.io/cluster-issuer: letsencrypt-prod-cloudflare`, bloc `spec.tls`, IngressShim s'occupe du reste).
- Incohérence WireGuard 51820/16000 dans la doc client (`acces.md`) initialement repérée puis corrigée pendant la session — vérifier si d'autres docs ou snippets ailleurs (Slack, gestionnaire de mdp, etc.) mentionnent encore `:51820` côté endpoint client.

---

## 14. Conclusion

La migration s'est faite sans interruption de service (rien d'utile ne tournait sur acme-dns, et Authentik était déjà en TLS cassé). La chaîne complète Cloudflare → cert-manager → Let's Encrypt prod est validée d'abord en bac à sable (`test-cf.benthami.eu`) puis en production (`auth.benthami.eu`).

Le démantèlement d'acme-dns a été méthodique (balayage références → ClusterIssuer → workloads → réseau) et n'a laissé aucun orphelin. Le repo a été restructuré pour distinguer `services/` (vivant) de `archive/manifests/` (techniquement démantelé), avec une convention `*/archive/` locale pour les variantes obsolètes d'un composant encore vivant.

La documentation transversale (`dette.md`, `inventaire.md`, `securite.md`, `reseau/dns.md`, `changelog.md`) a été intégralement réalignée sur l'état final dans la foulée — pas de dérive entre code et doc à la sortie de session.
