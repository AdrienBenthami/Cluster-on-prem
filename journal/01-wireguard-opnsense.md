# Compte rendu 1 – Mise en place d’un VPN WireGuard sur OPNsense derrière Freebox

## 1. Objectif

Mettre en place un accès VPN sécurisé pour :

* joindre l’infrastructure interne (LAN 192.168.100.0/24),
* administrer OPNsense et les services internes,
* accéder à distance au datacenter hébergé sur Proxmox.

---

## 2. Démarche suivie

1. **Création de l’environnement réseau**

   * Installation d’une VM **OPNsense** sur **Proxmox**.
   * Attribution de deux interfaces :

     * WAN relié au réseau Freebox (192.168.1.0/24).
     * LAN interne isolé (192.168.100.0/24).

2. **Tests initiaux avec VM minimale**

   * Déploiement d’une VM **Debian légère** (500 Mo RAM, 1 CPU, 8 Go disque, sans GUI).
   * But : vérifier la connectivité LAN via OPNsense.

3. **Validation d’accès via VM plus complète**

   * Création d’une VM **Ubuntu Desktop**.
   * Utilisation de cette VM pour accéder au **WebGUI OPNsense (192.168.100.1)** et préparer la configuration VPN.

4. **Mise en place du VPN WireGuard**

   * Installation et activation du plugin WireGuard sur OPNsense.
   * Création de l’instance `wg0` (10.10.10.1/24).
   * Configuration d’un peer client (Adrien-PC : 10.10.10.2/32).
   * Mise en place des règles firewall sur WAN et WG0.

5. **Tests côté client**

   * Connexion via WireGuard Windows.
   * Vérification des handshakes, ping de 10.10.10.1 et 192.168.100.1.
   * Validation de l’accès aux ressources LAN depuis le VPN.

---

## 3. Architecture réseau

### Freebox (fournisseur d’accès)

* Adresse IPv4 publique : **<IP-PUBLIQUE>**.
* LAN Freebox : **192.168.1.0/24**.
* Adresse attribuée à OPNsense WAN : **192.168.1.181**.
* **NAT configuré :** WAN UDP 16000 → OPNsense UDP 51820.

### OPNsense (pare-feu/routeur virtuel)

* **WAN :** 192.168.1.181/24.
* **LAN :** 192.168.100.1/24.
* **WireGuard (wg0) :** 10.10.10.1/24.
* Service WireGuard écoute sur UDP 51820.

### Réseaux

* **LAN interne :** 192.168.100.0/24.
* **VPN WireGuard :** 10.10.10.0/24.

  * Adrien-PC : 10.10.10.2/32.

### Client Adrien-PC

* **En LAN Freebox :** 192.168.1.80.
* **En VPN :** 10.10.10.2.
* **Endpoint :** <IP-PUBLIQUE>:16000.
* **AllowedIPs :** 10.10.10.0/24, 192.168.100.0/24.

---

## 4. Problèmes rencontrés et résolutions

1. **Absence de handshake WireGuard**

   * Cause : option *Block private networks on WAN* bloquait le trafic 192.168.x.x en provenance de la Freebox.
   * Solution : décochage temporaire, puis création d’une règle WAN spécifique.

2. **Connexion locale depuis LAN Freebox bloquée**

   * OPNsense rejetait l’IP source privée (192.168.1.80).
   * Solution : autorisation du bloc 192.168.0.0/16 sur WAN port 51820.

3. **Redirection Freebox**

   * Vérifiée via `tcpdump` sur OPNsense.
   * Paquets UDP atteignent bien OPNsense sur 51820.

---

## 5. Choix techniques

* **WireGuard vs OpenVPN**

  * *WireGuard* choisi car :

    * Plus récent, conçu pour la simplicité et la performance.
    * Code source très léger (≈ 4000 lignes vs >100k pour OpenVPN).
    * Intégration native au noyau Linux (donc performance quasi au niveau du routage normal).

  * *OpenVPN* écarté car :

    * Moins performant (overhead TLS).
    * Plus ancien, mais toujours robuste (utilisé en entreprise).

---

## 6. État actuel

* VPN WireGuard fonctionnel.
* Adrien-PC (`10.10.10.2`) peut accéder :

  * au LAN OPNsense (`192.168.100.0/24`),
  * au WebGUI OPNsense (`192.168.100.1`).
* Redirection Freebox confirmée.
* Firewall ajusté pour gérer :

  * connexions distantes via IP publique,
  * connexions locales via IP privée (192.168.0.0/16).

---

## 7. Améliorations possibles

* **Règles firewall plus restrictives** :

  * Autoriser uniquement l’IP publique du client (via alias ou DDNS).
  * Limiter l’accès LAN aux seuls services nécessaires.
* **Sécurité WireGuard** :

  * Ajouter une clé pré-partagée (PSK) pour renforcer l’authentification.
* **Supervision** :

  * Activer les logs sur la règle WAN WireGuard.
  * Surveiller régulièrement l’état des peers.

---

## 8. Schéma réseau

```
[Adrien-PC]
   └─ IP locale : 192.168.1.80
   └─ IP VPN    : 10.10.10.2
        │
        ▼
[Freebox WAN:<IP-PUBLIQUE>]
   └─ NAT UDP 16000 → 51820
        │
        ▼
[OPNsense]
   └─ WAN 192.168.1.181
   └─ LAN 192.168.100.1
   └─ WG0 10.10.10.1
        │
        ▼
[LAN interne]
   └─ 192.168.100.0/24
```




