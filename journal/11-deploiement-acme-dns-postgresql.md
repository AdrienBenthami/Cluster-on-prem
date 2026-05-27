# Compte rendu technique complet

**Sujet : déploiement d’acme-dns avec backend PostgreSQL dans Kubernetes (mode non-HA préparé pour migration Rook-Ceph)**
**Périmètre : cluster Kubernetes LAB (Traefik, MetalLB, cert-manager), namespace dédié `acme-dns`**
**Objectif final de la séquence : déployer acme-dns proprement, sans webhook tiers, avec backend PostgreSQL, architecture migrable vers une infra HA ultérieure**

---

## 1. Contexte et objectifs

Après suppression complète du webhook OVH et remise à plat de cert-manager, l’objectif était de :

* Déployer une solution DNS-01 **sans webhook tiers**
* Garder une architecture cloud-native
* Éviter toute dette technique bloquante pour un futur passage en HA
* Ne stocker aucun secret dans des ConfigMap
* Garder la migration vers **Rook-Ceph** simple

Contraintes décidées :

* Services accessibles uniquement en LAN pour l’instant
* Mode non-HA assumé temporairement
* Backend PostgreSQL dès maintenant (préparation HA future)

---

## 2. Décisions d’architecture structurantes

### 2.1. Backend PostgreSQL dans le cluster

Choix assumé :

* PostgreSQL déployé en **StatefulSet**
* 1 replica (non-HA)
* StorageClass : `local-path`
* Données persistées via `volumeClaimTemplates`

Justification :

* Préparer une migration future vers Rook-Ceph
* Éviter SQLite (limite de scalabilité future)
* Avoir un backend standard

Limite actuelle :

* Non-HA (lié au nœud)
* Dépendance à `local-path`
* Pas de réplication

---

### 2.2. Gestion des secrets

Choix final propre :

* Secret `acme-dns-postgres-auth` → credentials DB
* Secret `acme-dns-config` → fichier complet `config.cfg`
* Aucune donnée sensible dans une ConfigMap
* Aucun templating initContainer

Ce modèle :

* Respecte le principe Kubernetes (Secret pour données sensibles)
* Évite les substitutions shell fragiles
* Évite la duplication des secrets

---

## 3. Déploiement PostgreSQL

### 3.1. Création du Secret DB

Secret `acme-dns-postgres-auth` contenant :

* `POSTGRES_DB`
* `POSTGRES_USER`
* `POSTGRES_PASSWORD`

---

### 3.2. Service PostgreSQL

Service `ClusterIP` :

* Nom : `acme-dns-postgres`
* Port : 5432
* DNS interne stable :

  ```
  acme-dns-postgres.acme-dns.svc.cluster.local
  ```

---

### 3.3. StatefulSet PostgreSQL

* Image : `postgres:16`
* 1 replica
* PVC généré automatiquement
* Volume monté sur `/var/lib/postgresql/data`

Validation :

* Pod `acme-dns-postgres-0` en Running
* PVC en Bound
* Service avec Endpoints actifs

---

## 4. Déploiement acme-dns

### 4.1. Problème rencontré

acme-dns exige :

```ini
[database]
connection = "postgres://..."
```

Impossible d’utiliser uniquement des variables d’environnement.
La connection doit exister dans le fichier `config.cfg`.

---

### 4.2. Solution retenue

Création d’un Secret :

`acme-dns-config` contenant le fichier complet :

```ini
[database]
engine = "postgres"
connection = "postgres://acmedns:xxxxx@acme-dns-postgres.acme-dns.svc.cluster.local:5432/acmedns?sslmode=disable"
```

Montage via :

```yaml
volumeMounts:
  - name: config
    mountPath: /etc/acme-dns/config.cfg
    subPath: config.cfg
```

Aucune ConfigMap utilisée pour la partie sensible.

---

### 4.3. Validation applicative

Tests effectués :

1. `curl http://<pod-ip>:8080`

   * 404 page not found → API répond

2. `POST /register`

   * Retour JSON valide
   * Insertion correcte dans PostgreSQL

Validation DB :

* Table `records` correctement alimentée
* Table `txt` correctement alimentée
* Hash bcrypt des passwords

Conclusion :
✔ API fonctionnelle
✔ Connexion DB valide
✔ Écriture en base confirmée

---

## 5. État final du système

Namespace `acme-dns` :

* StatefulSet PostgreSQL : Running
* Deployment acme-dns : Running
* Service PostgreSQL : OK
* Service API interne : OK
* Secrets correctement isolés

Aucun secret en ConfigMap.
Aucun templating runtime.
Architecture claire et maintenable.

---

## 6. Dette technique / Points de sécurité à adresser

### 6.1. Sécurité API acme-dns

Actuellement :

```ini
disable_registration = false
```

Risque :

* Toute machine du LAN peut créer des registrations

À corriger avant production :

```ini
disable_registration = true
```

Et laisser uniquement cert-manager créer les entrées.

---

### 6.2. Absence de NetworkPolicy

Aucune restriction actuelle :

* Tout pod du cluster peut contacter PostgreSQL
* Tout pod peut contacter l’API acme-dns

À implémenter :

* NetworkPolicy limitant :

  * PostgreSQL → uniquement namespace `acme-dns`
  * API → uniquement cert-manager

---

### 6.3. Absence de backup PostgreSQL

Actuellement :

* Aucun backup automatisé
* Données critiques pour renouvellement ACME

À prévoir :

* CronJob dump SQL
* Ou snapshot PVC
* Ou futur backup Rook-Ceph

---

### 6.4. Non-HA

Limites actuelles :

* PostgreSQL mono-instance
* Storage local-path
* Perte du nœud = indisponibilité

Migration future prévue :

* Déploiement Rook-Ceph
* Nouvelle StorageClass `ceph-rbd`
* Migration volume
* Éventuellement opérateur PostgreSQL HA

---

### 6.5. Exposition DNS non configurée

Actuellement :

* DNS 53 non exposé
* MetalLB non configuré pour acme-dns

À venir :

* Service LoadBalancer UDP/TCP 53
* VIP dédiée (ex: 192.168.100.201)
* Override DNS interne
* Puis CNAME publics

---

## 7. Prochaines étapes logiques

### Phase 1 — Finaliser acme-dns

1. Créer Service LoadBalancer pour port 53 (UDP + TCP)
2. Assigner VIP MetalLB
3. Configurer DNS interne OPNsense
4. Tester résolution via `dig`

---

### Phase 2 — Intégration cert-manager

1. Créer ClusterIssuer staging
2. Configurer solver acme-dns
3. Test Certificate sur domaine de test
4. Valider challenge DNS-01

---

### Phase 3 — Durcissement

* Activer `disable_registration = true`
* Ajouter NetworkPolicies
* Ajouter backups
* Documenter migration Rook-Ceph

---

## 8. Conclusion

La séquence a permis :

* Un déploiement acme-dns propre
* Backend PostgreSQL fonctionnel
* Aucun secret exposé dans ConfigMap
* Architecture migrable vers HA
* Validation complète du flux applicatif

Le socle est désormais sain pour reconstruire la chaîne TLS complète via cert-manager et DNS-01, sans webhook tiers.

