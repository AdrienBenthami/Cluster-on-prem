# Compte rendu technique complet

**Sujet : intégration complète d’acme-dns avec cert-manager (DNS-01), sécurisation NetworkPolicy strict-minimum et validation de flux ACME staging**
**Périmètre : cluster Kubernetes LAB (Traefik, MetalLB, cert-manager), namespaces `acme-dns`, `cert-manager`, application Authentik (`authentik`)**
**Objectif final de la séquence : stabiliser une chaîne TLS DNS-01 sans webhook tiers, sécuriser l’API acme-dns, figer la version applicative, créer un ClusterIssuer staging et valider un Challenge jusqu’à l’état `Presented: true`**

---

## 1. Contexte et situation initiale

Le cluster Kubernetes du LAB est structuré autour de :

* Traefik comme Ingress Controller (IngressClass `traefik`)
* MetalLB pour l’attribution de VIP (traefik actuellement sur `192.168.100.200`, VIP `192.168.100.201` réservée pour DNS)
* cert-manager installé et fonctionnel, mais volontairement “neutre” (aucun Issuer/ClusterIssuer actif, aucun objet ACME résiduel)
* Une stratégie TLS en pause (Ingress applicatifs sans déclenchement cert-manager)
* acme-dns déjà déployé dans `acme-dns` avec backend PostgreSQL, non-HA, stockage `local-path`

Le besoin couvert par cette phase n’était plus “déployer acme-dns”, mais de :

* Comprendre le rôle exact de l’API HTTP acme-dns et du DNS 53
* Sécuriser l’API (contrôle) pour qu’elle ne soit utilisable que par cert-manager
* Mettre en place le solver acme-dns côté cert-manager (ACME staging)
* Valider le cycle `Certificate → Order → Challenge` en DNS-01 jusqu’à la présentation effective du TXT
* Documenter et corriger les formats de Secrets attendus par cert-manager

Contraintes explicites :

* Ne rien exposer sur Internet pour l’instant
* Procéder par étapes courtes, testées
* Maintenir une posture de sécurité “strict minimum” intra-cluster

---

## 2. Clarification fonctionnelle : rôles des ports 8080 et 53

### 2.1. Port 8080 (API HTTP acme-dns)

Rôle :

* Interface de contrôle (provisionnement et mise à jour des enregistrements TXT)
* Utilisée par cert-manager pour présenter les challenges via DNS-01

Caractéristiques :

* Sensible, car elle permet l’écriture d’enregistrements TXT et la création de comptes (si registration ouverte)
* Doit rester interne et cloisonnée

Positionnement retenu :

* Service Kubernetes `ClusterIP` uniquement
* Accès limité via NetworkPolicy

---

### 2.2. Port 53 (DNS UDP/TCP)

Rôle :

* Interface de données : réponse aux requêtes DNS sur les enregistrements `_acme-challenge`
* Utilisée par Let’s Encrypt pour valider le challenge DNS-01

Positionnement retenu :

* Non exposé WAN pour le moment
* Prévu à terme via MetalLB (VIP dédiée) puis délégation publique (OVH)

Conclusion structurante :

* Les deux ports sont nécessaires dans la chaîne complète, mais :

  * 8080 = contrôle, doit rester strictement interne
  * 53 = données DNS, devra être exposé selon la cible (LAN puis WAN)

---

## 3. Contrôle d’exposition initial : état réel des Services

Les contrôles Kubernetes ont confirmé l’absence d’exposition externe :

* Service `acme-dns-api` en `ClusterIP` uniquement, port 8080
* Service `acme-dns-postgres` en `ClusterIP` uniquement, port 5432
* Aucun `LoadBalancer`, aucun `NodePort`, aucun `Ingress` exposant l’API

Conclusion :

* Pas d’exposition LAN/WAN persistante
* Surface d’attaque principale = intra-cluster (tout pod pouvant joindre l’API en absence de NetworkPolicy)

---

## 4. Choix architectural : sécurisation “strict minimum” via NetworkPolicies

### 4.1. Motivation

L’API acme-dns est considérée sensible, en particulier tant que :

* la registration n’est pas fermée (`disable_registration`),
* et qu’aucune segmentation réseau intra-cluster n’est en place.

Le choix a été de converger vers une politique de type :

* deny-all namespace + allowlist des flux nécessaires

Cette décision répond à :

* Réduction de surface d’attaque intra-cluster
* Prévisibilité (flux explicitement documentés)
* Préparation future à l’exposition du DNS 53 sans compromettre l’API

---

### 4.2. Pré-requis vérifié : CNI appliquant les NetworkPolicies

Contrôles effectués :

* Présence de CoreDNS avec labels attendus :

  * pods `k8s-app=kube-dns`
  * service `kube-dns` sur `10.96.0.10`

Ce point a permis de créer une policy DNS egress stable (TCP/UDP 53) sans hypothèse fragile.

---

### 4.3. Policies appliquées (set strict minimum)

1. Default deny (ingress + egress) sur tout le namespace `acme-dns` :

* `default-deny-ingress-egress`

2. Autoriser la résolution DNS (egress vers CoreDNS) :

* `allow-egress-dns` vers `kube-system`/`k8s-app=kube-dns` en TCP/UDP 53

3. Autoriser cert-manager à accéder à l’API acme-dns :

* `allow-ingress-acme-dns-api-from-cert-manager` sur TCP/8080, `namespaceSelector` cert-manager

4. Autoriser acme-dns à joindre PostgreSQL :

* `allow-egress-acme-dns-to-postgres` sur TCP/5432

5. Autoriser PostgreSQL à recevoir uniquement depuis acme-dns :

* `allow-ingress-postgres-from-acme-dns` sur TCP/5432

---

### 4.4. Validation de la segmentation

Tests effectués :

* Depuis namespace `default` :

  * appel `curl` vers `acme-dns-api` → timeout (bloqué)
* Depuis namespace `cert-manager` :

  * pod de test `curlimages/curl` → HTTP 404 (accès OK)

Point important :

* `kubectl port-forward` n’est pas un contournement de la policy : il passe par l’API Kubernetes et sert uniquement au débogage depuis la machine d’administration.

Conclusion :

* cloisonnement effectif
* flux minimal nécessaire maintenu

---

## 5. Gestion de version : correction majeure sur l’image acme-dns

### 5.1. Problème rencontré

acme-dns tournait initialement avec :

* `joohoi/acme-dns:latest`

Symptôme :

* incertitude sur les endpoints et comportement API (routes non évidentes, tests incohérents)
* risque de divergence de documentation

---

### 5.2. Correction appliquée

La version effective a été identifiée :

* `v2.0.2`

Décision :

* fixation explicite sur `joohoi/acme-dns:v2.0.2`

Justification :

* alignement sur la documentation
* reproductibilité
* réduction du risque de regression future

---

## 6. Registration acme-dns : provisioning et fermeture

### 6.1. Registration initiale

Procédure :

* port-forward temporaire sur `svc/acme-dns-api` (8080)
* `POST /register` pour générer les credentials d’account

Bonne pratique appliquée :

* le fichier JSON local de registration a été supprimé immédiatement après stockage en Secret Kubernetes
* suppression sécurisée (`shred` si disponible)

---

### 6.2. Création du Secret Kubernetes

Choix retenu :

* Secret placé dans le namespace `cert-manager`
* nom : `acme-dns-auth`
* clé : `acme-dns-auth.json`

Justification :

* cert-manager est le seul consommateur fonctionnel des credentials
* principe de moindre privilège (accès restreint par NetworkPolicy)

---

### 6.3. Fermeture de la registration

Action :

* passage de `disable_registration = true` dans `config.cfg` (Secret `acme-dns-config`)
* rollout restart du deployment acme-dns
* validation via endpoints :

  * `GET /health` → 200 OK
  * `POST /register` → 404

Conclusion :

* le provisioning initial est figé
* l’enrôlement libre est fermé

---

## 7. Mise en place cert-manager : ClusterIssuer staging

### 7.1. Création du ClusterIssuer

Création de :

* `ClusterIssuer letsencrypt-staging-acme-dns`
* serveur : `https://acme-staging-v02.api.letsencrypt.org/directory`
* solver : `dns01.acmeDNS`
* host API : `http://acme-dns-api.acme-dns.svc.cluster.local:8080`
* secret : `cert-manager/acme-dns-auth` clé `acme-dns-auth.json`

---

### 7.2. Validation

Résultat observé :

* `READY: True`
* compte ACME enregistré
* secret de clé de compte généré :

  * `letsencrypt-staging-acme-dns-account-key`

Conclusion :

* cert-manager ↔ Let’s Encrypt staging est opérationnel
* la couche “compte ACME” est stable

---

## 8. Déploiement Certificate et diagnostic des erreurs de solver

### 8.1. Création d’un Certificate de test

Ressource créée :

* `Certificate auth-benthami-eu` dans namespace `authentik`
* `dnsNames: auth.benthami.eu`
* `issuerRef: letsencrypt-staging-acme-dns`

Résultat :

* `Order` créé en `pending`
* `Challenge` créé en `pending`
* `Certificate READY=False` (attendu)

---

### 8.2. Problème 1 : format Secret non conforme

Erreur :

```text
Error unmarshalling accountJSON: json: cannot unmarshal string into Go value of type goacmedns.Account
```

Analyse :

* le solver acmeDNS attend un format structuré, pas un JSON “stringifié” ou interprété comme tel.

---

### 8.3. Problème 2 : tentative tableau

Après conversion en tableau :

```text
json: cannot unmarshal array into Go value of type map[string]goacmedns.Account
```

Analyse :

* cert-manager attend une **map**, pas un tableau.

---

### 8.4. Problème 3 : credentials non trouvés

Après conversion en map indexée par `fulldomain` :

```text
account credentials not found for domain auth.benthami.eu
```

Analyse :

* la clé de la map doit correspondre au domaine du Challenge (`dnsName`), pas au `fulldomain` acme-dns.

---

### 8.5. Complexité additionnelle rencontrée : encapsulations successives

Le secret a subi une double encapsulation involontaire (`array` de `array`) lors de transformations successives. Ce point a été objectivé via :

* `jq` montrant `array` puis `array` sur le premier niveau

Une normalisation robuste a été réalisée pour extraire un account unique avant reconstruction propre.

---

## 9. Format final correct du Secret (acmeDNS solver)

Format qui a permis de résoudre l’ensemble des erreurs :

* Secret contenant une map indexée par le domaine du challenge :

```json
{
  "auth.benthami.eu": {
    "username": "...",
    "password": "...",
    "subdomain": "...",
    "fulldomain": "...",
    "allowfrom": []
  }
}
```

Points clés :

* type attendu : `map[string]goacmedns.Account`
* clé attendue : `dnsName` du Challenge (ex. `auth.benthami.eu`)
* valeur : account acme-dns complet

---

## 10. Validation finale : Challenge présenté

Après correction du Secret et relance du Challenge :

* `Presented: true`
* `Reason: Waiting for DNS-01 challenge propagation: DNS record for "auth.benthami.eu" not yet propagated`
* Events :

  * `Presented challenge using DNS-01 challenge mechanism`

Conclusion :

✔ cert-manager lit correctement le Secret
✔ cert-manager contacte l’API acme-dns (8080) en posture réseau cloisonnée
✔ acme-dns enregistre le TXT (update OK)
✔ chaîne fonctionnelle jusqu’à la propagation DNS

Blocage restant (attendu) :

* La propagation DNS observable par Let’s Encrypt n’est pas possible tant que :

  * le DNS 53 n’est pas exposé publiquement
  * la délégation/CNAME OVH n’est pas en place

---

## 11. État final

### Namespace `acme-dns`

* API HTTP fonctionnelle (health OK)
* registration désactivée
* segmentation NetworkPolicy strict minimum
* PostgreSQL accessible uniquement depuis acme-dns
* DNS 53 actif au niveau pod (non exposé)

### Namespace `cert-manager`

* ClusterIssuer staging Ready
* Secret `acme-dns-auth` au format correct
* tests de connectivité validés (cert-manager → acme-dns)

### Namespace `authentik`

* Certificate présent, READY=False (attendu)
* Order pending
* Challenge pending mais `Presented: true`

---

## 12. Dette technique / Points de sécurité à adresser

### 12.1. Exposition DNS 53 non configurée

Actuellement :

* pas de Service `LoadBalancer` 53 UDP/TCP
* pas de VIP MetalLB affectée à acme-dns DNS
* pas de délégation DNS (OVH) ni override interne (OPNsense)

Conséquence :

* Let’s Encrypt ne peut pas valider un challenge DNS-01

---

### 12.2. Stockage PostgreSQL

* mono-instance
* local-path
* pas de backup automatisé

Risque :

* perte de nœud = perte de données de challenges/accounts

---

### 12.3. Gouvernance des accounts multi-domaines

Actuellement :

* un seul mapping pour `auth.benthami.eu`

À industrialiser :

* stratégie de registration (par FQDN)
* secret map multi-domaines
* procédure de rotation / révocation

---

### 12.4. Hygiène de version

Correction effectuée :

* abandon de `latest`
* fixation `v2.0.2`

À maintenir :

* documentation interne de la version
* contrôle des upgrades (changelog, tests)

---

## 13. Prochaines étapes recommandées

### Phase 1 — Validation DNS en LAN (sans Internet)

1. Exposer DNS 53 acme-dns en LAN via MetalLB (VIP `192.168.100.201`)
2. Ajouter une configuration DNS interne (OPNsense / Unbound) permettant la résolution vers acme-dns
3. Tester la résolution TXT via `dig` en ciblant la VIP LAN
4. Confirmer que le TXT présenté par cert-manager est résolu via le LAN

Objectif : valider la couche DNS avant toute ouverture WAN.

---

### Phase 2 — Passage à la validation publique

1. Exposer DNS 53 vers Internet (politique pare-feu maîtrisée)
2. Mettre en place la délégation publique OVH (CNAME `_acme-challenge.*` vers acme-dns)
3. Tester Let’s Encrypt staging jusqu’à émission du certificat
4. Basculer ensuite en production Let’s Encrypt

---

### Phase 3 — Durcissement / production

* sauvegarde PostgreSQL (CronJob dump + rotation)
* migration stockage vers Rook-Ceph
* PostgreSQL HA (opérateur dédié) si besoin
* monitoring acme-dns / PostgreSQL (si requis)
* documentation d’exploitation (procédure registration, format secret, renouvellement)

---

## 14. Conclusion

La séquence a permis :

* Une compréhension complète et opérationnelle du solver acmeDNS cert-manager
* Une sécurisation stricte de l’API acme-dns via NetworkPolicies
* Une fermeture de la registration (`disable_registration=true`)
* Une fixation de version acme-dns (`v2.0.2`)
* Un ClusterIssuer staging Ready, compte ACME enregistré
* Un Certificate déclenchant un Challenge DNS-01 présenté avec succès (`Presented: true`)
* Une identification claire du seul point manquant : la chaîne DNS (port 53 + délégation) pour permettre la validation côté ACME

Le socle est désormais maîtrisé et stable pour activer progressivement la résolution DNS (LAN puis WAN) et réintroduire une automatisation TLS complète sans webhook tiers.
