# Compte rendu technique

**Déploiement et intégration de Gitea, Authentik, Traefik et MetalLB sur Kubernetes**
**Objectif : mise en place progressive, maîtrisée et auditable d’une plateforme Git auto-hébergée**

---

## 1. Objectif initial

L’objectif était de déployer **Gitea** sur un cluster Kubernetes existant, avec les contraintes suivantes :

* Déploiement **structuré via Helm**
* Stockage persistant
* Accès HTTP(s) via Ingress (Traefik)
* Accès SSH pour les opérations Git
* Authentification centralisée via **Authentik (OIDC)**
* Préparation à une exposition publique **sans précipitation**
* Compréhension complète de chaque mécanisme mis en place
* Conservation des configurations dans un dépôt d’infrastructure (pas de “kubectl à la volée”)

---

## 2. État initial du cluster (constaté)

* Cluster Kubernetes fonctionnel
* **Traefik** déjà déployé comme Ingress Controller
* **MetalLB** opérationnel en mode L2

  * Pool IP : `192.168.100.200-192.168.100.239`
* **Authentik** déjà déployé et accessible
* Pas de cert-manager initialement
* Pas de Gitea déployé

---

## 3. Déploiement de Gitea

### 3.1 Méthode

* Utilisation du chart officiel : `gitea-charts/gitea`
* Déploiement dans le namespace `gitea`
* Fichier `values-gitea.yaml` versionné

### 3.2 Composants déployés

* Deployment Gitea (1 replica)
* PostgreSQL (StatefulSet, non-HA)
* Valkey (StatefulSet, non-HA)
* PVC pour les données Gitea
* Services :

  * `gitea-http` (ClusterIP)
  * `gitea-ssh` (initialement ClusterIP)

### 3.3 Résultat

* Gitea démarre correctement
* Accès HTTP interne validé (`curl` → HTTP 200)
* Accès via Ingress validé (HTTP 301 depuis Traefik)

---

## 4. Déploiement et compréhension de l’authentification (Authentik)

### 4.1 Intention

* Ne pas gérer les comptes utilisateurs dans Gitea
* Centraliser l’identité via Authentik
* Utiliser OpenID Connect (OIDC)

### 4.2 Actions réalisées

* Définition d’un provider OIDC côté Authentik
* Déclaration d’Authentik comme OAuth provider dans Gitea
* Stockage des secrets OIDC via `Secret` Kubernetes
* Tentative d’initialisation automatique via init container Gitea

### 4.3 Problème rencontré

* Échec de l’initialisation OIDC dû à un **problème de certificat TLS**
* Certificat présenté par Traefik non valide pour `auth.benthami.eu`
* Erreur bloquante : Gitea refuse de démarrer tant que l’OIDC échoue

### 4.4 Décision

* **Gel volontaire de l’intégration OIDC**
* Priorité donnée à la stabilisation interne avant toute exposition publique
* Reconnaissance que :

  * Gitea **crée toujours un utilisateur interne**
  * Mais que l’identité peut (et doit) être contrôlée par Authentik
* Décision de reprendre ce point plus tard, proprement

---

## 5. Introduction de cert-manager (puis mise en pause)

### 5.1 Action

* Installation de `cert-manager` via Helm
* Création d’un `ClusterIssuer` Let’s Encrypt staging
* Tentative d’émission de certificats pour Authentik

### 5.2 Problème critique identifié

* Let’s Encrypt staging échoue avec `NXDOMAIN`
* Cause : le domaine n’est **pas réellement résolu publiquement**
* Risque identifié :

  * Début d’exposition involontaire du cluster
  * Manque d’audit sécurité préalable

### 5.3 Décision

* **Arrêt immédiat de la démarche TLS publique**
* Retour à un déploiement **strictement interne**
* Cert-manager laissé en place, mais non utilisé activement

---

## 6. Exposition SSH pour Gitea – discussion et choix d’architecture

### 6.1 Besoin fonctionnel

* Permettre les opérations Git standard :

  * `git clone`
  * `git push`
  * `git fetch`
* Utilisation de SSH avec clés (standard Git)

### 6.2 Clarification fonctionnelle

* SSH **ne sert pas à s’authentifier via Authentik**
* SSH est un **canal de transport**
* L’identité SSH est liée :

  * à un compte Gitea
  * aux clés SSH associées à ce compte

---

## 7. Discussion d’architecture : exposition réseau

### Option 1 — IP dédiée par service

* Un LoadBalancer par application
* Avantages :

  * Simplicité
* Inconvénients :

  * Consommation d’IP
  * Multiplication des points d’entrée
  * Peu cohérent avec une architecture HA

### Option 2 — IP partagée via MetalLB + Traefik + services dédiés

* Une IP publique unique (`192.168.100.200`)
* Traefik gère HTTP/HTTPS
* Services TCP (SSH) exposés via MetalLB avec IP partagée
* Avantages :

  * Architecture standard en production
  * Centralisation du trafic
  * Compatible HA
* Inconvénients :

  * Légèrement plus complexe

### Décision

👉 **Option 2 retenue**, car :

* Plus robuste
* Plus proche d’un modèle production
* Meilleure maîtrise des flux
* Compatible avec un futur audit sécurité

---

## 8. Mise en œuvre de l’option 2

### 8.1 Traefik

* Déplacement du fichier `values` dans `helm/traefik/`
* Activation :

  * `LoadBalancer`
  * IP fixe `192.168.100.200`
  * Annotation `allow-shared-ip`
* Validation par `helm upgrade --dry-run`

### 8.2 Gitea SSH

* Passage du service `gitea-ssh` en `LoadBalancer`
* Ajout de l’annotation MetalLB de partage d’IP
* Résultat :

```text
EXTERNAL-IP: 192.168.100.200
PORT: 22 → targetPort 2222
ENDPOINT: pod Gitea
```

* Validation des endpoints
* Architecture fonctionnelle et propre

---

## 9. État final à la fin de la session

### Fonctionnel

* Gitea :

  * HTTP interne OK
  * Ingress OK
  * SSH exposé et fonctionnel
* Traefik :

  * IP unique
  * Configuration propre et versionnée
* MetalLB :

  * Fonctionnement validé
  * IP partagée opérationnelle

### En attente (volontairement)

* Authentification OIDC finale
* Gestion des utilisateurs exclusivement via Authentik
* TLS public (Let’s Encrypt)
* Audit sécurité du cluster

---

## 10. Conclusion

La démarche suivie a été :

* **Progressive**
* **Réfléchie**
* **Architecturalement saine**
* **Sans exposition prématurée**
* **Avec un fort souci de compréhension et de traçabilité**

Les fondations sont désormais solides :

* réseau maîtrisé
* services propres
* exposition contrôlée
* dette technique limitée

La suite logique consistera à :

1. Auditer le cluster
2. Finaliser l’OIDC
3. Verrouiller l’authentification Gitea
4. Activer le TLS public en toute confiance
