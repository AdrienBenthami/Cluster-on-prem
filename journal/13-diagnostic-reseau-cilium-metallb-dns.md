# Compte rendu technique complet (version corrigée)

**Sujet : diagnostic réseau Kubernetes avancé (Cilium + kube-proxy + MetalLB), correction des flux DNS, refonte des NetworkPolicies, exposition DNS LAN/WAN et validation opérationnelle d’acme-dns pour DNS-01**

**Périmètre :**

* Cluster Kubernetes (Cilium CNI – datapath eBPF, kube-proxy iptables, MetalLB L2)
* Namespace `acme-dns`
* Réseau LAN : 192.168.100.0/24
* Infrastructure réseau : OPNsense + Freebox (double NAT)
* Poste de test : Debian (LAN + WAN)

**Objectif :**

* Rendre fonctionnel le DNS acme-dns (UDP/TCP 53)
* Comprendre précisément un blocage réseau multi-couches
* Refondre les politiques réseau de manière lisible et prédictible
* Exposer le DNS en LAN puis WAN
* Valider la chaîne DNS nécessaire au DNS-01 (Let’s Encrypt)

---

# 1. Situation initiale

## 1.1. Exposition Kubernetes

Service :

```bash
kubectl -n acme-dns get svc acme-dns-dns
```

* Type : LoadBalancer
* IP : `192.168.100.201` (MetalLB)
* Ports : 53 UDP/TCP
* Backend : pod `10.244.1.140`

---

## 1.2. Symptôme observé

Depuis le master ou une machine LAN :

```bash
dig @192.168.100.201 TXT <record>
```

Résultat :

* timeout
* absence totale de réponse

---

## 1.3. Premières observations réseau

### tcpdump côté worker

* requêtes DNS entrantes visibles
* aucune réponse
* aucun trafic vers le pod

👉 Le trafic n’atteint pas le backend applicatif

---

# 2. Analyse du datapath Kubernetes

## 2.1. kube-proxy (iptables)

Inspection :

```bash
iptables-save -t nat
```

Constat :

* chaînes `KUBE-EXT`, `KUBE-SVC`, `KUBE-SEP` présentes
* DNAT correctement configuré vers `10.244.1.140:53`

👉 kube-proxy valide

---

## 2.2. nftables

* règles cohérentes avec iptables
* redirection vers service effective

👉 pipeline NAT fonctionnel

---

## 2.3. Conclusion intermédiaire

* Service Kubernetes : valide
* MetalLB : valide
* kube-proxy / NAT : valides

👉 Le blocage est **post-DNAT**

---

# 3. Rôle déterminant de Cilium

## 3.1. Observation des drops

Commande :

```bash
cilium monitor --type drop
```

Résultat :

```
Policy denied
identity world → pod:53
```

---

## 3.2. Interprétation

* le trafic est bien redirigé vers le pod
* mais bloqué par Cilium (eBPF)
* classification source : `world`

---

## 3.3. Compréhension de `world`

Dans Cilium :

* `world` = trafic externe au cluster
* inclut :

  * WAN
  * LAN NATé
  * toute source non identifiée comme pod

👉 Le trafic venant du LAN est traité comme externe

---

## 3.4. Conclusion technique

> Le datapath réel est contrôlé par Cilium (eBPF), et non par iptables seul.

* iptables était correct
* mais insuffisant
* Cilium appliquait une politique bloquante

---

# 4. Limites du diagnostic tcpdump

## 4.1. Constat

* tcpdump ne montre pas le flux vers le pod
* incohérence avec les règles iptables

## 4.2. Explication réelle

* Cilium intercepte les paquets en eBPF
* filtrage effectué **avant la stack réseau classique**

👉 tcpdump ne voit pas :

* certains flux internes
* ni les drops eBPF

---

## 4.3. Conclusion

> tcpdump ne reflète pas fidèlement le datapath en environnement Cilium.

---

# 5. Cause racine

## 5.1. Nature du problème

Ce n’était pas une simple règle bloquante.

👉 C’était un **modèle de NetworkPolicies inadapté**

---

## 5.2. Problème structurel

* policies nombreuses
* règles dispersées
* ingress/egress fragmentés
* absence de vision globale
* non prise en compte de `world`

---

## 5.3. Conséquence

> blocage systémique des flux exposés (LoadBalancer)

---

# 6. Refonte des NetworkPolicies

## 6.1. Changement de paradigme

Passage de :

* policies orientées objets (pods)

à :

* policies orientées **flux métier**

---

## 6.2. Principes retenus

* 1 policy = 1 fonction
* ingress + egress dans un même manifest
* nommage structuré
* lisibilité prioritaire

---

## 6.3. Structure finale

```bash
kubectl get netpol -n acme-dns
```

* `00-default-deny`
* `10-acme-dns-control-plane`
* `20-acme-dns-dns53` (CiliumNetworkPolicy)
* `30-acme-dns-postgres-client`
* `30-acme-dns-postgres-server`

---

## 6.4. Point clé

La policy DNS autorise explicitement :

* source : `world`
* ports : UDP/TCP 53

---

# 7. Résolution

## 7.1. Test après refonte

```bash
dig @192.168.100.201 TXT ...
```

Résultat :

* réponse immédiate
* UDP validé
* TCP validé

---

## 7.2. Vérification Cilium

```bash
cilium monitor --type drop
```

* aucun drop

---

## 7.3. Conclusion

> Blocage levé via refonte complète des politiques réseau.

---

# 8. Validation LAN

Depuis Debian (LAN) :

```bash
dig @192.168.100.201 TXT ...
```

Résultat :

* réponse conforme
* latence faible

👉 DNS opérationnel en LAN

---

# 9. Exposition WAN

## 9.1. Architecture

```
Internet
→ Freebox (NAT)
→ OPNsense (WAN)
→ VIP MetalLB (192.168.100.201)
→ Node Kubernetes
→ Pod acme-dns
```

---

## 9.2. Configuration OPNsense

Deux règles NAT :

* UDP 53 → 192.168.100.201
* TCP 53 → 192.168.100.201

- règles firewall associées

---

## 9.3. Spécificité

* double NAT (Freebox + OPNsense)
* nécessité de tests réels externes

---

# 10. Validation WAN

## 10.1. Test externe réel

```bash
dig @<IP publique> TXT <fulldomain>
```

Résultat :

* réponse valide
* UDP et TCP fonctionnels

---

## 10.2. Conclusion

> Validation complète de la chaîne réseau :

* NAT
* MetalLB
* kube-proxy
* Cilium
* acme-dns

---

# 11. Validation DNS publique

## 11.1. CNAME OVH

```bash
dig @1.1.1.1 _acme-challenge.auth.benthami.eu
```

Résultat :

* CNAME correct vers acme-dns

---

## 11.2. Résolution TXT

```bash
dig @IP_publique TXT <subdomain>.acme-dns...
```

Résultat :

* enregistrement présent
* cohérent avec challenge

---

## 11.3. Conclusion

> Infrastructure DNS prête pour DNS-01.

---

# 12. État réel du projet

## 12.1. Terminé

* exposition DNS (LAN + WAN)
* résolution publique validée
* acme-dns opérationnel
* policies réseau maîtrisées
* compréhension datapath Cilium

---

## 12.2. Non terminé

* émission complète de certificat (Ready=True)
* intégration Gitea
* industrialisation multi-domaines

---

# 13. Enseignements clés

## 13.1. Cilium

> Une NetworkPolicy valide peut bloquer totalement un flux si la notion d’identité (`world`) n’est pas prise en compte.

---

## 13.2. Datapath

> iptables n’est pas la source de vérité en présence de Cilium.

---

## 13.3. Design réseau

> Les policies doivent être pensées par flux métier, pas par ressource.

---

## 13.4. Debug

> `cilium monitor` est l’outil critique pour diagnostiquer les drops.

---

# 14. Dette technique

* backup PostgreSQL absent
* pas de HA
* gouvernance des policies à documenter
* observabilité Cilium limitée

---

# 15. Architecture future

## 15.1. API Gateway

* envisageable
* non prioritaire
* découplé du DNS-01

---

# 16. Conclusion

Cette séquence a permis :

* d’identifier un blocage réseau complexe multi-couches
* de comprendre le rôle réel de Cilium dans le datapath
* de corriger une architecture de policies inadéquate
* de simplifier et stabiliser la segmentation réseau
* de valider une exposition DNS fonctionnelle en LAN et WAN

Le système est désormais :

* cohérent
* prédictible
* sécurisé
* prêt pour finalisation de la chaîne TLS via DNS-01.
