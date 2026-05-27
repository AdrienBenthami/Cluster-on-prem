# Compte rendu technique complet

**Sujet : diagnostic ACME DNS-01 OVH, nettoyage cert-manager et suppression du webhook OVH**
**Périmètre : cluster Kubernetes (Traefik, cert-manager), applications Authentik et Gitea, DNS OVH**
**Objectif final de la séquence : supprimer proprement le webhook OVH et remettre l’état TLS à plat avant migration vers acme-dns**

---

## 1. Contexte et situation initiale

Le cluster Kubernetes du LAB expose des services via Traefik (IngressClass `traefik`) et MetalLB (VIP `192.168.100.200`). La stratégie TLS initiale reposait sur cert-manager avec Let’s Encrypt en environnement staging, et une résolution DNS-01 OVH via un webhook tiers (`cert-manager-webhook-ovh`).

Un incident bloquant a été observé sur l’émission de certificats, avec notamment :

* Des `Challenge` ACME bloqués en état `pending`
* Une erreur OVH répétée : `Client::BadRequest: "Invalid signature"`
* Un bruit important dans les événements cert-manager et les logs du contrôleur (boucles de cleanup)
* Une incapacité à supprimer certaines ressources via Lens du fait de finalizersd

La contrainte de méthode était de conserver une approche cloud-native (cert-manager, ressources déclaratives), avec un objectif de maintenabilité et de réversibilité.

---

## 2. Diagnostic initial : invalid signature OVH

### 2.1. Vérifications Kubernetes de premier niveau

Les contrôles suivants ont montré un cluster stable :

* Pods cert-manager, cainjector, webhook cert-manager en état `Running`
* Webhook OVH `cert-manager-webhook-ovh` en état `Running`
* Cycle cert-manager observable et cohérent : `Certificate → CertificateRequest → Order → Challenge`

L’échec se produisait au niveau du solver DNS-01 OVH :

* Appel OVH observé dans les ressources :

  * `GET /domain/zone/benthami.eu/record?fieldType=TXT&subDomain=_acme-challenge.gitea`
* Erreur systématique :

  * `Invalid signature`

### 2.2. Hypothèses évaluées et éliminées

Plusieurs causes fréquentes ont été testées et éliminées de manière déterministe :

1. Erreur de RBAC empêchant le webhook de lire le secret
   Ce point avait été traité auparavant (Role/RoleBinding de lecture du secret). Les objets webhook internes étaient Ready. L’absence d’erreur RBAC dans les logs récents confirmait que ce n’était plus le problème.

2. Mauvais secret / mauvaise valeur
   Les valeurs dans Kubernetes ont été comparées à des valeurs fonctionnelles via empreintes SHA256 (applicationKey, applicationSecret, consumerKey). Les hashes étaient strictement identiques.

3. Problème de time skew (timestamp OVH)
   La synchronisation NTP du master, du worker et l’heure à l’intérieur du pod webhook ont été comparées ; elles étaient alignées. Le worker indiquait `System clock synchronized: yes`.

4. Problème d’egress ou de DNS depuis le cluster
   Un pod de test déployé dans Kubernetes (avec les credentials injectés via `envFrom`) a exécuté un script Python utilisant le client officiel OVH. L’appel exact `GET /domain/zone/benthami.eu/record` retournait `[]` (succès), démontrant que l’accès OVH fonctionne depuis le cluster.

### 2.3. Conclusion de diagnostic

À ce stade, la conclusion est devenue robuste :

* Les credentials OVH sont valides.
* Le réseau et l’horloge du cluster sont valides.
* L’appel OVH fonctionne depuis un pod Kubernetes avec le client OVH.
* L’erreur `Invalid signature` est donc spécifique au webhook OVH (implémentation, gestion de signature, encodage de la requête, ou défaut de compatibilité).

Le webhook en place était :

* Image : `ghcr.io/aureq/cert-manager-webhook-ovh:0.8.0`
* Chart : `cert-manager-webhook-ovh-0.8.0`
* `configVersion: 0.0.2`

Un rollback Helm vers 0.7.6 a été tenté, mais a échoué sur une incompatibilité volontaire du chart (`configVersion` attendu 0.0.1). Ce point a confirmé que la voie “downgrade rapide” impliquait une migration de values et n’était pas l’action la plus efficace dans l’immédiat, compte tenu de l’objectif de stabilisation.

---

## 3. Choix architectural : suppression du webhook OVH avant migration acme-dns

### 3.1. Motivation

Le webhook OVH introduisait une dépendance à un composant non officiel OVH et dont la fiabilité opérationnelle n’était pas suffisante pour servir de socle durable.

Le choix a été fait de :

* Retirer cette dépendance
* Remettre le cluster dans un état propre (pas d’objets ACME orphelins, pas de finalizers bloquants)
* Préparer la migration vers une stratégie DNS-01 sans webhook (acme-dns), mais en différant son implémentation

Cette décision répondait à quatre objectifs :

* Maintenabilité : réduire le nombre de composants spécialisés non essentiels
* Débogabilité : éviter les boucles de ressources cert-manager difficiles à purger
* Réversibilité : restaurer un état “baseline” cert-manager stable
* Sécurité : supprimer les credentials OVH du cluster une fois inutiles

### 3.2. Point de vigilance

Supprimer le webhook sans stopper la création automatique de `Certificate/Order/Challenge` aurait aggravé la situation :

* cert-manager continuerait à créer des ressources
* les finalizers empêcheraient leur suppression
* le cluster accumulerait des objets bloqués

La séquence correcte nécessitait de :

1. Stopper la source de création automatique des certificats
2. Purger les ressources existantes (y compris via suppression de finalizers)
3. Désinstaller le webhook
4. Retirer secrets et RBAC

---

## 4. Problème majeur rencontré : recréation automatique des certificats (Ingress Shim)

### 4.1. Symptomatologie

Après suppression manuelle de certificats et certificaterequests, des ressources réapparaissaient immédiatement :

* `Certificate authentik-tls` recréé après suppression
* `CertificateRequest authentik-tls-1` recréé après suppression

Ce comportement pouvait donner l’impression que “des pods génèrent des challenges”, alors qu’en réalité cert-manager réagit à une configuration déclarative persistante.

### 4.2. Cause identifiée

La cause a été objectivée par inspection YAML :

* `Certificate authentik-tls` contenait :

  * `managed-by: Helm`
  * `ownerReferences → Ingress authentik-server`
  * `issuerRef.name: letsencrypt-staging`
  * `secretName: authentik-tls`

* `Ingress authentik-server` contenait :

  * annotation `cert-manager.io/cluster-issuer: letsencrypt-staging`
  * `spec.tls` pointant sur `secretName: authentik-tls`

Conclusion : le certificat Authentik était généré par cert-manager ingress-shim, déclenché par l’Ingress Helm, et non créé manuellement.

### 4.3. Décision et justification

Le TLS automatique a été mis en pause côté Ingress Authentik, pour stabiliser le cluster, en assumant que :

* la continuité de service applicatif (HTTP) prime sur une automatisation TLS cassée
* la stratégie TLS sera réintroduite proprement lors de la migration acme-dns

Ce choix évite la dette technique consistant à laisser des ressources cert-manager se recréer en boucle sans issuers valides.

---

## 5. Nettoyage contrôlé : Authentik

### 5.1. Actions effectuées

1. Suppression de l’annotation cert-manager sur l’Ingress Authentik :

* retrait de `cert-manager.io/cluster-issuer`

2. Retrait temporaire de `spec.tls` sur l’Ingress Authentik :

* objectif : éviter toute dépendance à un secret TLS absent
* effet : maintien d’un service accessible en HTTP, sans tentative de gestion TLS

3. Suppression des ressources cert-manager Authentik :

* suppression des `CertificateRequest` (namespace authentik)
* suppression du `Certificate authentik-tls`

### 5.2. Résultat

* Namespace authentik : plus aucun `Certificate` ni `CertificateRequest`
* Aucune recréation automatique (source supprimée)
* Stabilisation des événements cert-manager côté Authentik

---

## 6. Nettoyage contrôlé : Gitea et suppression des ressources ACME orphelines

### 6.1. Problème rencontré

Un `Challenge` Gitea persistait encore en `pending` et générait des erreurs de cleanup dans les logs cert-manager. Le cas était plus délicat car :

* l’Ingress Gitea ne portait plus d’annotation cert-manager
* aucun `Certificate` n’existait dans le namespace `gitea`
* le `Challenge` était donc un orphelin, maintenu par finalizer

Le contrôleur cert-manager tentait de “cleanup” via OVH et échouait, empêchant la suppression classique.

### 6.2. Actions effectuées

1. Vérification de non-recréation par l’Ingress Gitea :

* absence totale d’annotation cert-manager
* absence de `spec.tls`
* absence de `Certificate` dans `gitea`

2. Suppression forcée du Challenge via suppression de finalizers :

* patch du `Challenge` pour vider `metadata.finalizers`
* suppression asynchrone (`--wait=false`)

### 6.3. Résultat

* Namespace gitea : aucun `Challenge` restant
* Fin des logs cert-manager de type “error cleaning up challenge”

---

## 7. Suppression du webhook OVH et nettoyage sécurité

### 7.1. Désinstallation Helm

* Désinstallation de la release Helm `cert-manager-webhook-ovh`
* Vérification : plus aucun deployment/pod/service OVH dans le namespace `cert-manager`

### 7.2. Nettoyage secrets et RBAC

* Suppression du secret `ovh-dns-credentials`
* Suppression du Role et RoleBinding de lecture du secret (ajoutés précédemment)

### 7.3. Vérification finale cluster-wide

* Aucun `Issuer` / `ClusterIssuer`
* Aucun `Certificate`, `CertificateRequest`, `Order`, `Challenge` résiduel
* cert-manager opérationnel, sans bruit

---

## 8. Changements d’architecture et impacts

### 8.1. TLS applicatif mis en pause

* Authentik : Ingress passé en HTTP-only temporaire (suppression `spec.tls`).
* Gitea : Ingress sans TLS et sans cert-manager.
* Le cluster est remis en état stable en attendant une stratégie TLS cohérente.

Impact :

* Perte temporaire de TLS interne automatisé.
* Gain immédiat : stabilité, absence de ressources cert-manager en boucle.

### 8.2. Suppression des credentials OVH du cluster

Impact sécurité :

* Réduction de l’exposition : aucune clé API OVH stockée dans Kubernetes.
* Disparition des RBAC supplémentaires de lecture de secret.

### 8.3. Préparation acme-dns

Le nettoyage réalisé permet une migration vers acme-dns dans de bonnes conditions :

* plus de ressources héritées ou orphelines
* plus de dépendance webhook
* environnement cert-manager neutre (issuers à recréer proprement)

---

## 9. Problèmes rencontrés et résolutions

### 9.1. Rollback Helm impossible

* Échec downgrade 0.8.0 → 0.7.6 à cause de `configVersion` incompatible.
* Résolution : abandon du rollback et suppression du webhook, car la cible n’était plus de “réparer OVH DNS-01” mais de stabiliser.

### 9.2. Suppression impossible de ressources cert-manager via Lens

* Cause : finalizers ACME.
* Résolution : suppression des finalizers via `kubectl patch` avant suppression.

### 9.3. Recréation automatique “invisible”

* Cause : ingress-shim cert-manager déclenché par Ingress Helm.
* Résolution : suppression d’annotation cert-manager et de `spec.tls` là où nécessaire.

### 9.4. Confusion potentielle “pods générateurs de challenges”

* Clarification : ce ne sont pas des pods applicatifs qui créent des challenges, mais cert-manager qui réagit à l’état désiré (Ingress/annotations).

---

## 10. État final

* Cluster cert-manager propre, aucun objet ACME résiduel.
* Webhook OVH supprimé.
* Secrets OVH supprimés.
* RBAC OVH supprimé.
* Authentik et Gitea opérationnels mais TLS automatisé mis en pause.
* Base saine pour déployer acme-dns et reconstruire une stratégie TLS stable, cloud-native et maintenable.

---

## 11. Prochaine étape recommandée (cadre, sans implémentation)

Pour passer à acme-dns, la séquence logique sera :

1. Déployer acme-dns (namespace dédié, persistance, exposition contrôlée)
2. Recréer des `ClusterIssuer` staging/prod adaptés
3. Réactiver TLS sur Authentik et Gitea en déclaratif (Ingress ou Certificate explicites), sans webhook, avec un modèle stable

Le point clé est de reconstruire la chaîne TLS sur une base propre, ce qui est désormais le cas.
