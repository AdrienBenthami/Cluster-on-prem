# Compte rendu technique

**Sujet : Mise en place de cert-manager avec ACME DNS-01 via OVH (webhook) pour Gitea / Authentik**
**Environnement : Kubernetes, cert-manager, Lens, OVH DNS**

---

## 1. Contexte et objectifs

L’objectif initial était de :

* Déployer des certificats TLS automatisés via **cert-manager**.
* Utiliser **Let’s Encrypt (staging)** avec le **challenge DNS-01**.
* Intégrer **OVH DNS** comme fournisseur DNS via un **webhook cert-manager**.
* Appliquer ce mécanisme à plusieurs applications (Gitea, Authentik).
* Versionner proprement l’infrastructure (manifests Helm / YAML).
* Travailler prioritairement via **interface graphique (Lens)** tout en gardant une traçabilité YAML.

---

## 2. Organisation de l’infrastructure

### 2.1 Structure de dépôt

Une arborescence claire a été mise en place et respectée :

* `infra/helm/`

  * `cert-manager/`

    * `clusterissuer-prod.yaml`
    * `clusterissuer-staging.yaml`
    * `letsencrypt-ovh-dns-staging.yaml`
  * `gitea/`

    * `values-gitea.yaml`
* `infra/manifests/`

  * `cert-manager/certificates/`

    * `gitea-tls-staging.yaml`
* `infra/manifests/metallb/`
* `infra/manifests/cilium/`

**Choix architectural :**

* Séparation claire entre :

  * Helm charts (outils, opérateurs)
  * Manifests applicatifs (Certificate, Secret, RBAC)
* Tout élément critique est versionné (hors secrets sensibles).

---

## 3. Mise en place de cert-manager et du webhook OVH

### 3.1 cert-manager

* cert-manager est déjà installé et fonctionnel.
* Les certificats internes du webhook (`cert-manager-webhook-ovh-ca`, `cert-manager-webhook-ovh-webhook-tls`) sont **émis correctement**.
* Les `CertificateRequest` internes au webhook sont **Approved = True / Ready = True**.

👉 Conclusion : **cert-manager fonctionne correctement en tant que contrôleur ACME.**

---

### 3.2 Webhook OVH (DNS-01)

* Le webhook OVH est déployé dans le namespace `cert-manager`.
* Il expose un solver `dns01.webhook`.
* Le solver est référencé via :

  * `groupName: acme.benthami.eu`
  * `solverName: ovh`
  * `endpoint: ovh-eu`

**Choix architectural :**

* DNS-01 via webhook permet :

  * Pas d’exposition HTTP
  * Indépendance vis-à-vis de l’ingress
  * Compatibilité multi-cluster

---

## 4. Gestion des secrets OVH

### 4.1 Création de la clé API OVH

Une clé API OVH a été créée avec les droits suivants :

* GET `/domain/zone/benthami.eu/*`
* POST `/domain/zone/benthami.eu/record`
* PUT `/domain/zone/benthami.eu/record/*`
* DELETE `/domain/zone/benthami.eu/record/*`
* POST `/domain/zone/benthami.eu/refresh`

**Description recommandée :**

> Accès DNS minimal pour cert-manager (ACME DNS-01)

### 4.2 Secret Kubernetes

Un secret `Opaque` a été créé :

* **Nom** : `ovh-dns-credentials`
* **Namespace** : `cert-manager`
* **Clés** :

  * `applicationKey`
  * `applicationSecret`
  * `consumerKey`

Les valeurs ont été vérifiées :

* Décodage base64 OK
* Correspondance avec l’interface OVH confirmée

👉 Le secret est **correctement formé et lisible**.

---

## 5. RBAC – permissions du webhook

### 5.1 Problème initial

Erreur observée :

```
User "system:serviceaccount:cert-manager:cert-manager-webhook-ovh"
cannot get resource "secrets" in namespace "cert-manager"
```

### 5.2 Résolution

Actions réalisées via Lens (GUI) :

1. Création du **Role** :

   * Nom : `cert-manager-webhook-ovh-secret-reader`
   * Namespace : `cert-manager`
   * Règle :

     * `apiGroups: [""]`
     * `resources: ["secrets"]`
     * `verbs: ["get"]`

2. Création du **RoleBinding** :

   * Binding du Role au ServiceAccount :

     * `cert-manager-webhook-ovh`
     * Namespace `cert-manager`

**Validation :**

* Le Role contient bien les règles
* Le RoleBinding cible le bon ServiceAccount
* Le webhook peut lire le secret

👉 Problème RBAC **corrigé**.

---

## 6. Création du certificat Gitea

### 6.1 Certificate

* Nom : `gitea-tls-staging`
* Namespace : `gitea`
* Issuer : `letsencrypt-ovh-dns-staging`
* Secret cible : `gitea-tls-staging`
* Domaine : `gitea.benthami.eu`

Le cycle ACME se déclenche correctement :

* `CertificateRequest` créé
* `Order` créé
* `Challenge` DNS-01 créé

---

## 7. Problème bloquant : échec OVH – “Invalid signature”

### 7.1 Symptômes

Le `Challenge` reste en état `pending`.

Erreur systématique :

```
OVH API call failed:
GET /domain/zone/benthami.eu/record?fieldType=TXT&subDomain=_acme-challenge.gitea
Client::BadRequest: "Invalid signature"
```

Effets observés :

* Challenge non présenté (`Presented: false`)
* Order bloqué en `pending`
* Recréation du Certificate n’aide pas
* Suppression des CertificateRequest / Order / Challenge sans effet durable
* cert-manager relance automatiquement le flux

---

### 7.2 Vérifications effectuées

| Élément              | Résultat      |
| -------------------- | ------------- |
| RBAC                 | OK            |
| Secret OVH           | OK            |
| Droits API OVH       | OK            |
| Endpoint OVH         | ovh-eu        |
| Auth method          | application   |
| Time skew (node/pod) | OK (NTP sync) |
| Webhook cert-manager | Fonctionnel   |
| cert-manager core    | Fonctionnel   |

👉 Le problème **n’est pas Kubernetes**, **n’est pas cert-manager**, **n’est pas le RBAC**.

---

## 8. Enseignements clés

### 8.1 Sur cert-manager

* cert-manager ne “cache” rien : chaque étape ACME est observable (Certificate → Request → Order → Challenge).
* Supprimer un Certificate ne supprime pas toujours immédiatement Order / Challenge (finalizers ACME).
* Les erreurs DNS-01 sont **exclusivement** liées au solver.

### 8.2 Sur OVH

* Une clé OVH peut être **active mais invalide cryptographiquement**.
* Le message “Invalid signature” indique :

  * ConsumerKey incohérente
  * Clé générée dans un état partiellement cassé
  * Ou clé non alignée avec l’application

C’est un comportement OVH connu.

### 8.3 Sur l’architecture TLS

* Un certificat **par application** est la bonne approche ici :

  * Rotation indépendante
  * Moins de blast radius
  * Meilleure lisibilité
* Un certificat unique central (Authentik only) n’est pas adapté à Gitea.

---

## 9. État actuel

* cert-manager : opérationnel
* Webhook OVH : opérationnel
* RBAC : correct
* Secrets : présents
* Certificat Gitea : **bloqué**
* Cause restante : **clé OVH invalide côté signature**

---

## 10. Prochaines étapes recommandées

### Étape 1 – Recréer proprement la clé OVH

* Supprimer la clé actuelle
* Recréer une nouvelle clé API OVH
* Sauvegarder immédiatement :

  * applicationKey
  * applicationSecret
  * consumerKey

### Étape 2 – Mettre à jour le secret Kubernetes

* Mettre à jour `ovh-dns-credentials`
* Vérifier le décodage

### Étape 3 – Forcer un nouveau cycle ACME

* Supprimer :

  * Certificate
  * CertificateRequest
  * Order
  * Challenge
* Recréer le Certificate

### Étape 4 – Tester en staging

* Attendre un `Challenge` en `presented=true`
* Vérifier création du TXT `_acme-challenge.gitea`

### Étape 5 – Passer en production

* Basculer vers `letsencrypt-ovh-dns-prod`
* Mettre à jour les Certificates
* Vérifier renouvellement automatique

---

## Conclusion

L’architecture mise en place est **saine, robuste et correcte**.
Le blocage rencontré est **externe à Kubernetes** et se situe au niveau de la signature OVH, ce qui est cohérent avec l’erreur observée et les vérifications réalisées.

La reprise se fera proprement en repartant d’une clé OVH neuve, sans remise en cause du travail déjà effectué.
