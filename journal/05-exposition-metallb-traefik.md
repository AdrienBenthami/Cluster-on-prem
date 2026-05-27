# **Compte rendu 5 – Exposition des services (MetalLB + Traefik), DNS/TLS, tests et REX**

## 1. Objectif

Mettre en place et valider la chaîne d’exposition HTTP(S) du cluster Kubernetes LAB (**192.168.100.0/24**) :

* **MetalLB (Layer2)** pour fournir une **VIP** aux Services `LoadBalancer`.
* **Traefik** comme **Ingress Controller** (terminaison TLS, routage L7).
* Stratégie **DNS/TLS** interne (`infra.lan`) et pré-cadrage pour le public (`benthami.eu`).
* Jeux d’**essais** (avec *whoami*), **troubleshooting** et **sécurisation** (dashboard, headers, etc.).

---

## 2. Contexte & périmètre

### 2.1 Infrastructure et versions

| Élément       | Valeur / Rôle                                                         |
| ------------- | --------------------------------------------------------------------- |
| Hyperviseur   | Proxmox VE 8.x                                                        |
| Pare-feu/DNS  | OPNsense – GW/DNS interne **192.168.100.1** (Unbound)                 |
| Nœuds K8s     | `k8s-master-1` **192.168.100.31** ; `k8s-worker-1` **192.168.100.51** |
| Kubernetes    | v1.34.x (kubeadm) ; runtime **containerd** (SystemdCgroup)            |
| CNI           | **Cilium** ; **PodCIDR 10.244.0.0/16**                                |
| Services CIDR | **10.96.0.0/12**                                                      |
| MetalLB       | Mode **Layer2** ; pool contenant **192.168.100.200**                  |
| Traefik       | **v3.6.0** ; Service `LoadBalancer` → **192.168.100.200**             |

### 2.2 Exposition & ports

| Composant           | Type         | Adresse/Port        | Commentaire                   |
| ------------------- | ------------ | ------------------- | ----------------------------- |
| Traefik `web`       | Entrypoint   | :8000/TCP           | Redirection 308 → `websecure` |
| Traefik `websecure` | Entrypoint   | :8443/TCP           | HTTPS (terminaison TLS)       |
| Traefik `traefik`   | Entrypoint   | :8080/TCP           | API/Dashboard (à restreindre) |
| Traefik `metrics`   | Entrypoint   | :9100/TCP           | Prometheus scrape             |
| Service Traefik     | LoadBalancer | **192.168.100.200** | Annoncé par MetalLB (L2)      |

---

## 3. Architecture de publication (vue d’ensemble)

```
[Client LAN/VPN]
       │  DNS (infra.lan)            (ou Header Host en test)
       ▼
  whoami.infra.lan  ─────────────►  192.168.100.200  (VIP MetalLB)
                                 ┌───────────────────────────────────┐
                                 │  Service LoadBalancer "traefik"   │
                                 │   CLUSTER-IP 10.109.100.26        │
                                 └───────────────┬───────────────────┘
                                                 │
                                         Entrypoints 80/443
                                                 │
                                      ┌──────────▼───────────┐
                                      │   Traefik Ingress    │
                                      │  (règles/routers)    │
                                      └──────────┬───────────┘
                                                 │
                                   Ingress → Service (ClusterIP)
                                                 │
                                           Pod(s) applicatifs
                                           (CIDR 10.244.0.0/16)
```

---

## 4. Implémentation

### 4.1 MetalLB (Layer2)

* **Pool L2** incluant **192.168.100.200** (hors Pod/Service CIDR).
* Annonce **layer2** visible dans les Events (`announcing from node "k8s-worker-1"`).

**Vérifications utiles :**

```bash
kubectl -n metallb-system get pods
kubectl get svc -A | grep LoadBalancer
kubectl describe svc -n ingress traefik | grep -E "IP:|External.*"
```

### 4.2 Traefik (Ingress Controller)

* Déployé via Helm (chart récent) avec image **docker.io/traefik:v3.6.0**.
* **IngressClass** : `traefik` (controller `traefik.io/ingress-controller`).
* **Service type** `LoadBalancer` → **192.168.100.200**.
* **Comportements clés** : redirection **HTTP→HTTPS (308)**, **dashboard activé**, **metrics Prometheus** exposées.

**Contrôles standard :**

```bash
kubectl -n ingress get deploy,rs,pods,svc -o wide
kubectl -n ingress get ingressclass
kubectl -n ingress logs deploy/traefik -c traefik --tail=200
curl -I  http://192.168.100.200       # 308
curl -kI https://192.168.100.200      # 404 tant qu’aucune règle n’est publiée
```

---

## 5. Validation par un service de test (puis nettoyage)

### 5.1 Déploiement de test (*whoami*)

* **Ingress** avec `ingressClassName: traefik`.
* **Host** de test : `whoami.infra.lan` (recommandé), sinon `Host` forcé via curl.

**Exemple de test fonctionnel (curl)**

```bash
curl -kH 'Host: whoami.infra.lan' https://192.168.100.200/
# Retourne headers X-Forwarded-*, IP Pod 10.244.x.y, etc.
```

### 5.2 Nettoyage (réalisé)

* Suppression **Ingress/Service/Deployment** *whoami*.
* État attendu : `http://192.168.100.200` → **308**, `https://192.168.100.200` → **404**.

---

## 6. DNS interne & exposition publique

### 6.1 Interne (`infra.lan`)

Créer des **A records** dans Unbound pour les FQDN exposés via Traefik :

| FQDN                      | Type | Valeur          |
| ------------------------- | ---- | --------------- |
| `*.infra.lan` (option)    | A    | 192.168.100.200 |
| `whoami.infra.lan`        | A    | 192.168.100.200 |
| `grafana.infra.lan` (ex.) | A    | 192.168.100.200 |

> Variante labo rapide : `/etc/hosts` sur postes d’admin.

### 6.2 Public (`benthami.eu`)

Deux options selon l’objectif :

* **Exposition directe** : port-forward/NAT WAN → VIP ou nœud (non recommandé en direct).
* **Reverse proxy/relay** en DMZ ou chez l’hébergeur.
  Pour **TLS public**, prévoir **ACME Let’s Encrypt** (DNS-01 conseillé si pas d’HTTP-01 reachable).

---

## 7. TLS : modes et recommandations

| Mode                             | Où ?            | Avantages                           | Points d’attention                           |
| -------------------------------- | --------------- | ----------------------------------- | -------------------------------------------- |
| **Termination TLS** dans Traefik | Interne/public  | Simplicité, centralise certificats  | Gérer redirections/HSTS, renouvellement ACME |
| **TLS Passthrough** (mTLS/app)   | Cas spécifiques | L7 côté app, utile pour mTLS        | Moins de visibilité L7 dans Traefik          |
| **ACME (Let’s Encrypt)**         | Public          | Certificats valides auto-renouvelés | HTTP-01/DNS-01, droits API DNS si DNS-01     |
| **CA interne**                   | Interne         | Contrôle total                      | Déployer la racine sur postes/clients        |

> Sécuriser le **dashboard** Traefik : host dédié (ex. `traefik.infra.lan`), **auth**, IP allow-list.

---

## 8. Sécurité & durcissement

| Mesure                        | Détail                                                           |
| ----------------------------- | ---------------------------------------------------------------- |
| Restreindre `:8080`           | Exposer le dashboard uniquement via Ingress protégé + allow-list |
| `forwardedHeaders.trustedIPs` | Définir le range LAN/VPN pour prévenir le spoofing               |
| Redirections & HSTS           | Forcer HTTPS, HSTS si usage interne maîtrisé                     |
| RBAC K8s                      | Rôles/SA dédiés aux contrôleurs, éviter droits cluster-admin     |
| Mises à jour                  | Pinner chart/images ; procédure de rollback Helm                 |

---

## 9. Troubleshooting & REX

### 9.1 Matrice de symptômes

| Symptôme                                        | Causes probables                        | Actions                                         |
| ----------------------------------------------- | --------------------------------------- | ----------------------------------------------- |
| `curl https://VIP` → **404**                    | Aucune règle Ingress                    | Créer Ingress, vérifier `ingressClassName`      |
| Service `LoadBalancer` sans IP                  | Pool MetalLB non valide / conflit IP    | Vérifier `IPAddressPool`, éviter chevauchements |
| Redirection **HTTP→HTTPS** OK mais handshake KO | Cert/TLS, SNI, firewall                 | Vérifier certificats, ports, SNI & ciphers      |
| Pods Traefik en BackOff                         | Valeurs Helm, ressources, image         | Lire logs, corriger valeurs, vérifier image/tag |
| Helm `--kube-version` erreur                    | Flag non supporté sur `install/upgrade` | Ne l’utiliser que sur `helm template`           |

### 9.2 Retours d’expérience (faits marquants)

* Mauvaise utilisation du flag Helm `--kube-version` lors d’un `upgrade/install` → **corrigé** (flag retiré).
* Séquence d’**Events** cohérente : **MetalLB** attribue **192.168.100.200**, Traefik écoute `web/websecure`, **308** puis **404** tant qu’aucune route n’est publiée.
* Déploiement *whoami* : validation complète du chemin **VIP → Ingress → Service → Pod** puis **nettoyage**.

---

## 10. Opérations courantes (extraits)

```bash
# État Ingress/Traefik
kubectl -n ingress get all
kubectl -n ingress logs deploy/traefik -c traefik --tail=200

# Règles Ingress en cluster
kubectl get ingress -A
kubectl describe ingress -n <ns> <name>

# Sanity HTTP(S)
curl -I  http://192.168.100.200       # 308 attendu
curl -kI https://192.168.100.200      # 404 sans règles, 200/3xx/4xx selon app
```

---

## 11. État final

| Élément       | Statut                                                                 |
| ------------- | ---------------------------------------------------------------------- |
| MetalLB (L2)  | Opérationnel, VIP **192.168.100.200** attribuée à Traefik              |
| Traefik       | `Deployment` Ready, IngressClass `traefik` disponible                  |
| Routage       | Redirection 80→443 active, 404 par défaut (aucune app publiée)         |
| Test *whoami* | **Supprimé** (environnement propre)                                    |
| DNS           | Prêt côté `infra.lan` (à renseigner) ; cadrage `benthami.eu` établi    |
| TLS           | Modes et méthodes actés ; implémentation à planifier (ACME/CA interne) |

---

## 12. Prochaines étapes recommandées

1. **DNS interne** : créer les enregistrements `A` → **192.168.100.200** pour les services cibles (ex. `grafana.infra.lan`, `argocd.infra.lan`, etc.).
2. **TLS** : choisir **ACME** (si public) ou **CA interne** (interne) ; intégrer dans les `IngressRoute`/`Middleware`.
3. **Dashboard Traefik** : isoler sur un FQDN dédié + **auth** + **allow-list** IP.
4. **Observabilité** : exposer Grafana via Ingress et activer la **persistance** (StorageClass par défaut `local-path`, cf. CR correspondant).
5. **Industrialisation** : Helm charts applicatifs, modèles Ingress, CI/CD, tests de non-régression.

---

**Conclusion.**
La chaîne d’exposition **MetalLB (L2) → Traefik (Ingress)** est en place et validée. Le socle réseau L7 est sain (308/404 attendus), prêt à accueillir des applications publiées sous `infra.lan` (et, si besoin, `benthami.eu`) avec une politique TLS maîtrisée et des contrôles d’accès adaptés.
