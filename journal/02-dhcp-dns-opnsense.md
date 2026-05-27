# Compte rendu 2 – Mise en place du service DHCP et DNS local sur OPNsense

## 1. Objectif

Configurer et valider un service d’adressage et de résolution de noms interne à l’infrastructure, permettant :

* l’attribution automatique des adresses IP aux hôtes du LAN via **DHCP**,
* la résolution locale et externe des noms via **Unbound DNS**,
* la création d’entrées DNS internes cohérentes avec le plan d’adressage du laboratoire,
* la compréhension et la maîtrise du comportement du pare-feu sur les flux réseaux générés.

---

## 2. Démarche suivie

1. **Création de l’environnement réseau**

   * Utilisation d’une VM **OPNsense** déployée sur **Proxmox**.
   * Affectation de deux interfaces réseau :

     * **WAN** reliée au réseau de la **Freebox** (192.168.1.0/24).
     * **LAN** interne isolé (192.168.100.0/24).

2. **Préparation du client de test**

   * Déploiement d’une VM **Debian minimale** connectée au réseau LAN d’OPNsense.
   * Objectif : tester la distribution DHCP et la résolution DNS locale.

3. **Configuration du service DHCP**

   * Accès : `Services → ISC DHCPv4 → [LAN]`.

   * Activation du service et définition des paramètres suivants :

     | Élément                | Valeur                            |
     | ---------------------- | --------------------------------- |
     | Sous-réseau            | 192.168.100.0                     |
     | Masque de sous-réseau  | 255.255.255.0                     |
     | Plage d’adresses       | 192.168.100.100 – 192.168.100.199 |
     | Passerelle             | 192.168.100.1                     |
     | Serveurs DNS           | 192.168.100.1, 1.1.1.1            |
     | Nom de domaine         | infra.lan                         |
     | Durée maximale du bail | 86400 s (24 h)                    |
     | Serveur NTP            | 192.168.100.1                     |

   * Résultat : le client Debian obtient automatiquement une adresse (`192.168.100.100`) ainsi que la passerelle et le DNS.

4. **Configuration du service DNS (Unbound DNS)**

   * Accès : `Services → Unbound DNS → Général`.
   * Activation du résolveur DNS et des options suivantes :

     * Interfaces d’écoute : LAN, WAN, WireGuard (WG0).
     * DNSSEC activé.
     * Enregistrement automatique des baux DHCP statiques.
     * Zone locale : *transparent* (résolution interne directe).

5. **Création d’un enregistrement DNS local**

   * Accès : `Services → Unbound DNS → Remappage d’hôte`.

   * Objectif : permettre la résolution du nom local `opnsense.lan`.

   * Paramètres :

     | Champ       | Valeur           |
     | ----------- | ---------------- |
     | Hôte        | opnsense         |
     | Domaine     | lan              |
     | Type        | A (IPv4)         |
     | TTL         | 3600             |
     | Adresse IP  | 192.168.100.1    |
     | Description | Routeur OPNsense |

   * Vérification via le client Debian :

     ```
     dig @192.168.100.1 opnsense.lan
     ```

     → Réponse correcte :

     ```
     opnsense.lan. 3600 IN A 192.168.100.1
     ```

6. **Validation de la connectivité externe**

   * Test de ping vers une adresse publique :

     ```
     ping -c 4 google.com
     ```

     → Résultat : 0 % de perte, temps de réponse stable.
   * La connectivité Internet et la résolution DNS externe fonctionnent correctement.

---

## 3. Architecture réseau

### Freebox (fournisseur d’accès)

* **Adresse IPv4 publique :** <IP-PUBLIQUE>
* **LAN Freebox :** 192.168.1.0/24
* **Adresse attribuée à OPNsense WAN :** 192.168.1.181

### OPNsense (pare-feu / routeur virtuel)

* **WAN :** 192.168.1.181/24
* **LAN :** 192.168.100.1/24
* **Serveur DHCP :** actif sur LAN (plage 192.168.100.100–199)
* **Serveur DNS (Unbound) :** 192.168.100.1

### Réseaux

* **LAN interne :** 192.168.100.0/24
* **Plage DHCP :** 192.168.100.100–192.168.100.199

### Client Debian-test

* **Adresse IP attribuée :** 192.168.100.100
* **Passerelle :** 192.168.100.1
* **DNS :** 192.168.100.1, 1.1.1.1
* **Nom de domaine :** infra.lan

---

## 4. Analyse des journaux du pare-feu

### Observations

De nombreux paquets bloqués apparaissent dans les journaux avec le message :

```
Default deny / state violation rule
```

### Analyse

Les flux rejetés correspondent à des paquets UDP à destination d’adresses :

* **224.0.0.251:5353** → protocole *mDNS* (Multicast DNS / Bonjour),
* **239.255.255.250:1900** → protocole *SSDP* (UPnP),
* **192.168.1.255** → diffusion *NetBIOS / SMB*.

Ces requêtes sont émises par les équipements domestiques connectés au LAN (PC, téléviseurs, smartphones, imprimantes, etc.) pour découvrir automatiquement d’autres périphériques sur le réseau local.

### Comportement d’OPNsense

Le pare-feu bloque ces paquets car :

1. Ils n’ont aucune utilité sur le réseau WAN.
2. Ils ne correspondent à aucune session TCP/UDP existante (“state violation”).
3. Leur transit vers l’extérieur représenterait une fuite d’informations internes.

### Conclusion

Ces blocages sont normaux et contribuent à la sécurité du réseau.
Aucune anomalie n’a été détectée.

---

## 5. Mise en place de règles de filtrage spécifiques (mDNS / SSDP)

### Contexte

À la suite de l’analyse des journaux du pare-feu, un volume important de paquets **UDP multicast** non sollicités a été constaté sur l’interface **WAN**.
Ces flux provenaient de protocoles de découverte automatique utilisés dans les environnements domestiques :

| Protocole                                    | Adresse de destination | Port     | Description                                                       |
| -------------------------------------------- | ---------------------- | -------- | ----------------------------------------------------------------- |
| **mDNS (Multicast DNS)**                     | 224.0.0.251            | UDP/5353 | Utilisé par Bonjour/Avahi pour découvrir les services sur le LAN. |
| **SSDP (Simple Service Discovery Protocol)** | 239.255.255.250        | UDP/1900 | Utilisé par UPnP pour l’annonce de périphériques connectés.       |

Bien que légitimes sur un réseau local, ces protocoles n’ont **aucune utilité sur l’interface WAN** et généraient un bruit important dans les journaux système sous la mention “Default deny / state violation rule”.

---

### Objectif

Mettre en place des règles de pare-feu dédiées pour :

* bloquer explicitement les flux multicast **mDNS** et **SSDP** en entrée sur le WAN,
* éviter leur journalisation afin d’alléger les logs,
* maintenir une visibilité claire sur les événements de sécurité réellement pertinents.

---

### Configuration appliquée

Les règles suivantes ont été ajoutées dans **Pare-feu → Règles → WAN** :

| Action  | Interface | Direction | Protocole | Source | Destination     | Port | Journalisation | Description                  |
| ------- | --------- | --------- | --------- | ------ | --------------- | ---- | -------------- | ---------------------------- |
| Bloquer | WAN       | In        | UDP       | any    | 224.0.0.251     | 5353 | Non            | Blocage mDNS (Bonjour/Avahi) |
| Bloquer | WAN       | In        | UDP       | any    | 239.255.255.250 | 1900 | Non            | Blocage SSDP/UPnP            |

Les deux règles ont été placées **au-dessus** de la règle par défaut “Default deny” afin d’être priorisées et de neutraliser ces flux avant leur enregistrement dans les journaux.

---

### Résultat observé

* Disparition des entrées répétitives liées à `UDP 5353` et `UDP 1900` dans les journaux du pare-feu.
* Réduction significative du bruit de log et amélioration de la lisibilité des événements réels de sécurité.
* Aucun impact sur la connectivité réseau ou les services DNS/DHCP internes.
* Validation de la stabilité de la configuration après redémarrage des services.

---

### Justification technique

Ces règles répondent à un double objectif :

1. **Hygiène réseau :**
   Bloquer la diffusion de protocoles de découverte inutiles sur les interfaces externes réduit les risques de fuite d’informations (noms d’hôtes, services exposés, etc.).

2. **Lisibilité opérationnelle :**
   La désactivation de la journalisation sur ces flux empêche la saturation des journaux système, facilitant l’analyse des véritables alertes.

---

## 6. Recommandations complémentaires

1. **Amélioration du service DNS**

   * Activer la mise à jour dynamique des baux DHCP dans Unbound.
   * Chaque machine obtient ainsi un enregistrement automatique (`debian.infra.lan`, `ubuntu.infra.lan`, etc.).

2. **Documentation et plan d’adressage**

   * Maintenir une table d’adresses réservées pour les serveurs internes.
   * Mettre à jour le plan d’adressage pour intégration future de VLANs.

---

## 7. État actuel

* **DHCP :** opérationnel, délivre correctement adresses, passerelle et DNS.
* **DNS (Unbound) :** fonctionnel, résolution locale et externe confirmée.
* **Pare-feu :** comportement normal, aucun blocage critique.
* **Client Debian :** pleinement intégré au domaine `infra.lan` avec connectivité Internet.

---

## 8. Schéma réseau

```
[LAN interne 192.168.100.0/24]
      │
      │ DHCP / DNS : 192.168.100.1
      ▼
 [OPNsense]
   ├─ LAN : 192.168.100.1
   ├─ WAN : 192.168.1.181
   └─ DNS local : Unbound
      │
      ▼
 [Freebox]
   └─ Réseau WAN : 192.168.1.0/24
      │
      ▼
 [Internet]
```

---

## 9. Conclusion

La mise en place du **service DHCP** et du **serveur DNS interne Unbound** sur OPNsense est pleinement fonctionnelle.
Les clients du réseau LAN obtiennent automatiquement une configuration réseau complète, avec une résolution de noms cohérente au domaine local `infra.lan`.
Les anomalies observées dans les journaux du pare-feu sont liées à des communications de découverte interne (mDNS, SSDP, NetBIOS) et ne constituent pas une menace.
Des règles de filtrage dédiées sur l’interface WAN bloquent explicitement mDNS (UDP/5353) et SSDP (UDP/1900) sans journalisation, réduisant significativement le bruit de logs et renforçant l’hygiène réseau, sans impact sur les services internes (DHCP/DNS).

L’environnement réseau est stable, correctement isolé, et prêt pour l’intégration d’autres services tels que les VLANs, Authentik, ou Kubernetes.

