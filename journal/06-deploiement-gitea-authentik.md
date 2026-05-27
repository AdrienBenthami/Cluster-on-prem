# Compte rendu d’intervention — Déploiement Gitea (Kubernetes / Traefik / Authentik)

## 0. Contexte et objectif

**Contexte technique**

* Cluster Kubernetes : `k8s-master-1` (control-plane) + `k8s-worker-1` (worker), Kubernetes `v1.34.1`
* CNI : Cilium
* Ingress Controller : Traefik (namespace `ingress`)
* LoadBalancer Kubernetes : MetalLB (VIP Traefik = `192.168.100.200`)
* DNS interne : Unbound (OPNsense) résolvant `auth.benthami.eu` et `gitea.benthami.eu` vers `192.168.100.200`
* Authentik déjà déployé et fonctionnel (namespace `authentik`)

**Objectif**

* Déployer **Gitea** via le **chart Helm officiel**.
* Activer immédiatement le **SSO OIDC via Authentik**.
* Exposer Gitea via **Traefik** (Ingress) et **HTTPS**, dans une trajectoire compatible exposition Internet.

---

## 1. Vérification et validation de l’architecture existante

### 1.1 Santé cluster

Commande :

* `kubectl get nodes -o wide`
* `kubectl get pods -A`

Résultat :

* Nœuds `Ready`
* Composants critiques (Cilium, CoreDNS, API server) en état nominal
* Services existants (Authentik, Traefik, MetalLB) en `Running`

Raison :

* S’assurer que les incidents éventuels à venir relèvent du déploiement Gitea et non d’un cluster instable.

---

### 1.2 MetalLB et Traefik

Commandes :

* `kubectl -n metallb-system get ipaddresspools,l2advertisements`
* `kubectl -n ingress get svc`
* `curl -kI https://192.168.100.200`

Résultat :

* Pool IP MetalLB : `192.168.100.200-239`
* Service Traefik `LoadBalancer` avec VIP `192.168.100.200`
* Réponse Traefik sur la VIP (404 attendu tant qu’aucun Ingress ne correspond)

Raison :

* Valider le point d’entrée L4/L7 du cluster (VIP → Traefik → Ingress).

---

### 1.3 DNS interne

Commandes :

* `nslookup auth.benthami.eu 192.168.100.1`
* `nslookup gitea.benthami.eu 192.168.100.1`

Résultat :

* Résolution des deux FQDN vers `192.168.100.200`

Raison :

* Assurer la cohérence du routage HTTP(S) via un point d’entrée unique (Traefik VIP).

---

### 1.4 Vérification “API Kubernetes non exposée”

Côté Kubernetes :

* `ss -lntp | egrep ':(6443)\s'`
* `kubectl -n kube-system get svc`
* `kubectl get svc -A | egrep -i 'kubernetes|apiserver'`

Résultat :

* kube-apiserver écoute sur `*:6443` (comportement kubeadm standard)
* Service `default/kubernetes` est `ClusterIP` uniquement
* Aucun `LoadBalancer`/`NodePort` n’expose l’API

Côté OPNsense (captures + corrections effectuées) :

* Ajustements des règles WAN et WG0 pour supprimer redondances et corriger une règle mal typée
* Objectif : WAN autorise strictement WireGuard, WG0 autorise l’administration vers LAN

Raison :

* Confirmer que l’administration K8s reste interne/VPN.

---

## 2. Authentik — Mise en place OIDC pour Gitea

### 2.1 Création du Provider OAuth2/OpenID

Action :

* Création d’un provider `gitea-oidc` de type **Confidential**
* Ajout de l’URI de redirection (redirect URI) côté Authentik correspondant à Gitea
* Clé de signature : `authentik Self-signed Certificate` conservée à ce stade (clé dédiée créée mais non appliquée immédiatement)

Constat :

* Avertissement initial “provider non utilisé par une application” normal tant qu’aucune application n’est liée.

Raison :

* Préparer le flux OIDC complet avant déploiement applicatif.

---

### 2.2 Création de l’Application Authentik

Action :

* Création application `Gitea`, slug `gitea`
* Association de l’application au provider `gitea-oidc`
* L’interface provider a ensuite affiché : Issuer / JWKS / endpoints complétés automatiquement

Résultat :

* Provider et Application correctement liés
* Endpoint `.well-known/openid-configuration` disponible (fonctionnel par design dès que provider+app sont cohérents)

Raison :

* Permettre à Gitea d’utiliser l’auto-discovery OIDC.

---

## 3. Structuration “infra-as-code” sur le master

### 3.1 Constat initial

Fichiers présents à la racine du home :

* archives/outils (ex : binaire/archives Cilium)
* manifestes YAML MetalLB
* values Helm (Traefik, Authentik)

Problème :

* Dispersion des artefacts, risque opérationnel (perte de traçabilité, erreurs de chemin).

---

### 3.2 Organisation effectuée

Création d’une arborescence stable :

* `~/infra/helm/…`
* `~/infra/manifests/…`
* rangement des values/manifests existants

Impact :

* Aucun impact runtime : déplacement de fichiers locaux uniquement.

Raison :

* Préparer un mode opératoire reproductible et versionnable.

---

## 4. Gestion des secrets Kubernetes pour OAuth

### 4.1 Namespace Gitea

Action :

* Création du namespace `gitea` (Lens)

Raison :

* Isolation logique (RBAC, policies, ressources, exploitation).

---

### 4.2 Secret OAuth

Constat :

* Le chart Gitea attend un secret avec clés **`key`** et **`secret`** (conforme à la doc “OAuth2 Settings”).

Action :

* Création du secret `gitea-oauth-secret` dans `gitea` avec 2 data entries

Résultat :

* `kubectl -n gitea get secret gitea-oauth-secret` montre `DATA=2`

Raison :

* Interdire la présence de secrets en clair dans values (aligné GitOps).

---

## 5. Déploiement Helm — premières itérations et incidents rencontrés

### 5.1 Déploiement initial et symptomatique Traefik

Symptôme :

* `curl -kI https://gitea.benthami.eu` renvoie `503` lors d’un déploiement précédent

Diagnostic :

* Traefik renvoie `503` lorsque le backend (Service/Pods) n’a aucun endpoint prêt.
* `kubectl -n gitea get pods` a montré un pod Gitea bloqué en init.

Raison :

* L’Ingress routait vers un service sans endpoints Ready.

---

### 5.2 Erreur de type SecretKeyRef (incident confirmé)

Erreur relevée (events/describe) :

* `couldn't find key key in Secret gitea/gitea-oauth-secret`

Cause :

* Secret initial ne contenait pas la clé `key` (nom différent type `client_id`)

Correction :

* Suppression/recréation du secret en respectant strictement `key` et `secret`

Résultat :

* L’erreur “missing key” a été corrigée, et le processus init a progressé à l’étape suivante.

---

### 5.3 Échec init-container `configure-gitea`

Symptôme :

* `Back-off restarting failed container configure-gitea`

Approche retenue ensuite :

* Arrêt de la navigation par hypothèses
* Décision de repartir sur un environnement propre car accumulation de ReplicaSets et historiques de rollout.

---

## 6. Nettoyage complet et redéploiement

### 6.1 Désinstallation Helm

Commande :

* `helm -n gitea uninstall gitea`

Résultat :

* Désinstallation effectuée
* Message indiquant conservation de ressources via policy : PVC `gitea-shared-storage` conservé (resource-policy `keep`)

Raison :

* Le chart protège volontairement les volumes persistants.

---

### 6.2 Observation des résidus

Constat :

* Pods en `Terminating` durant la phase de suppression

Raison :

* Kubernetes finalise l’arrêt et libère les ressources.

---

### 6.3 PVC observés avant suppression de namespace

Liste PVC (avant suppression namespace) :

* `data-gitea-postgresql-0`
* `gitea-shared-storage`
* `valkey-data-gitea-valkey-primary-0`

Ces PVC étaient tous en `Bound` sur StorageClass `local-path`.

---

### 6.4 Suppression anticipée du namespace

Action :

* `kubectl delete namespace gitea` exécuté avant suppression manuelle des PVC

Effet :

* Les ressources namespacées (dont les PVC) ont été supprimées avec le namespace
* Vérification après recréation :

  * `kubectl -n gitea get pvc` : aucun PVC
  * `kubectl get pvc` : aucun PVC en default

Raison :

* Reset intégral et rapide sans état résiduel.

---

### 6.5 Recréation namespace + secret OAuth

Actions :

* `kubectl create namespace gitea`
* Recréation `gitea-oauth-secret` (clé/secret)
* Vérification `kubectl -n gitea get secret gitea-oauth-secret`

Résultat :

* Secret présent et prêt

---

## 7. Validation “à blanc” du chart (helm template) — diagnostic précis

### 7.1 Objectif

* Valider que le chart rend des manifestes cohérents **avant** installation (`helm template` + inspection).

Commande :

* `helm template … -f values-gitea.yaml > rendered.yaml`

---

### 7.2 Problème Ingress TLS : `secretName` vide

Constat rendu :

* Ingress `tls` rendu avec `secretName:` vide lorsque `tls.hosts` est défini sans `secretName`

Correction effectuée :

* Passage à `ingress.tls: []` afin de supprimer toute configuration TLS incohérente
* Vérification TLS côté Traefik (cert présenté) :

  * `subject CN=TRAEFIK DEFAULT CERT`
  * `issuer CN=TRAEFIK DEFAULT CERT`

Raison :

* Traefik termine bien le TLS, mais le certificat est par défaut (non production).

---

### 7.3 Problème majeur : Service `gitea-http` rendu avec `targetPort` vide

Constat rendu (preuve par extrait de `rendered.yaml`) :

* Service `gitea-http` :

  * `port: 3000`
  * `targetPort:` vide

Effets attendus :

* Manifest invalide / service non fonctionnel / endpoints non exposables correctement

Analyse :

* Le champ `targetPort` n’est pas configurable via les champs documentés (`service.http.*`) dans l’upstream values.
* Les tentatives d’ajout de `targetPort` dans values ont été rejetées par principe, car non documentées dans l’upstream.
* Le problème est donc traité comme :

  * soit un comportement non prévu du chart 12.4.0 avec la combinaison de values
  * soit une régression de version

---

## 8. Stratégie de résolution conforme doc : pinning de version de chart

### 8.1 Version actuelle

Commande :

* `helm show chart gitea-charts/gitea | egrep '^(name|version|appVersion):'`

Résultat :

* `name: gitea`
* `version: 12.4.0`
* `appVersion: 1.24.6`

---

### 8.2 Listing des versions disponibles

Commande :

* `helm search repo gitea-charts/gitea --versions | head -n 20`

Résultat :

* Versions disponibles : 12.4.0, 12.3.0, 12.2.0, 12.1.x, etc.

Décision :

* Tester le rendu `helm template` avec `--version 12.3.0`, puis `12.2.0` si nécessaire, afin d’identifier une version ne produisant pas `targetPort:` vide.

Raison :

* Approche rigoureuse et alignée doc :

  * pas de patch manuel,
  * pas de champs non documentés,
  * résolution par choix de version du chart.

---

## 9. Points de gouvernance et décisions de méthode

### 9.1 Rejet des “hypothèses”

Décision explicite :

* Les corrections doivent être fondées sur :

  * outputs `helm template`
  * inspection `rendered.yaml`
  * `kubectl describe` / events
  * logs init-containers

Raison :

* Stabiliser le dépannage, éviter les boucles d’essais.

---

### 9.2 Rejet des ressources “pansement”

Décision explicite :

* Pas de création manuelle de Service/Ingress hors chart (extraDeploy, manifests à la main) tant que la doc du chart ne l’impose pas.

Raison :

* Maintenabilité, conformité “chart as source of truth”.

---

## 10. État actuel (au moment de la pause)

**Environnement**

* Namespace `gitea` recréé et vide (hors secret)
* Secret `gitea-oauth-secret` présent et conforme

**Blocage**

* Chart `gitea 12.4.0` rend un Service `gitea-http` avec `targetPort:` vide dans `rendered.yaml`
* Tant que ce point n’est pas résolu, installation à blanc non fiable

**Prochaine action prévue**

* Tester `helm template` avec :

  * `--version 12.3.0`
  * puis `--version 12.2.0` si nécessaire
* Objectif : trouver une version de chart dont le rendu n’inclut pas le champ invalide.

---

## 11. Annexes — Commandes et vérifications déjà exécutées

* Vérification cluster : `kubectl get nodes`, `kubectl get pods -A`
* Vérification MetalLB : `kubectl -n metallb-system get ipaddresspools,l2advertisements`
* Vérification Traefik VIP : `kubectl -n ingress get svc`, `curl -kI https://192.168.100.200`
* DNS : `nslookup … 192.168.100.1`
* TLS Traefik : `curl -kIv https://auth.benthami.eu`, `curl -kIv https://gitea.benthami.eu`
* Helm baseline : `helm show values … > values-upstream.yaml`
* Template : `helm template … > rendered.yaml`
* Détection erreurs : `grep -nE 'targetPort:\s*$|secretName:\s*$' rendered.yaml`
* Nettoyage : `helm uninstall`, suppression namespace

---

Si tu veux, je peux produire une version “auditable” avec une chronologie horodatée (T0/T1/…) et une liste de fichiers modifiés (`values-gitea.yaml`, structure `~/infra/helm/gitea/`) telle qu’elle est à l’instant, mais ce compte rendu couvre déjà l’ensemble des actions, raisons et incidents rencontrés.
