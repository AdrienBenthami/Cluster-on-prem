# Compte rendu – Migration stockage Proxmox vers ZFS mirror

## 1. Contexte initial

Mon hyperviseur Proxmox était configuré comme suit :

* `rpool` en ZFS sur un seul disque (`/dev/sda`)
* Un second disque (`/dev/sdb`) formaté en ext4
* Ce second disque exposé comme datastore `ssd-data`
* Certaines VM (notamment le worker Kubernetes) stockées sur `ssd-data`

Problème identifié :

* Le système reposait sur un seul disque ZFS pour l’OS.
* Le stockage était hybride (ZFS + ext4).
* Il existait un SPOF disque critique.

Objectif :

* Migrer l’ensemble des VM vers ZFS.
* Supprimer le datastore ext4.
* Transformer le second disque en miroir ZFS du disque système.
* Obtenir une tolérance à la panne disque complète.

---

## 2. Audit du disque secondaire

J’ai vérifié le contenu réel de `/dev/sdb` :

```bash
lsblk -f
du -sh /mnt/pve/ssd-data/*
grep -R "ssd-data" /etc/pve/qemu-server/
```

Constat :

* 33 Go utilisés dans `images/`
* VM concernées :

  * 251 (k8s-worker-1)
  * 302 (test-ubuntu)

Conclusion :

Le disque était actif. Impossible de le supprimer sans migration préalable.

---

## 3. Migration des disques VM vers ZFS

### 3.1 Drain Kubernetes

Avant migration du worker :

```bash
kubectl drain k8s-worker-1 --ignore-daemonsets --delete-emptydir-data
```

### 3.2 Migration via interface Proxmox

Pour chaque disque de VM :

* Hardware → Move Storage
* Destination : `local-zfs`
* Suppression de la source

J’ai migré :

* scsi0
* efidisk
* tpmstate

### 3.3 Suppression des disques “Unused”

Après migration :

* Suppression des “Unused Disk”
* Vérification :

```bash
grep -R "ssd-data" /etc/pve/qemu-server/
du -sh /mnt/pve/ssd-data/images
```

Résultat :

* Plus aucune VM sur `ssd-data`
* Dossier `images` vide

---

## 4. Réintégration du worker Kubernetes

Après redémarrage VM :

```bash
kubectl uncordon k8s-worker-1
kubectl get nodes
kubectl get pods -A
```

Cluster stable.
Pods redistribués correctement.

---

## 5. Suppression du datastore ext4

Actions :

```bash
umount /mnt/pve/ssd-data
sgdisk --zap-all /dev/sdb
wipefs -a /dev/sdb
```

Vérification :

```bash
lsblk
```

Disque totalement vierge.

---

## 6. Création du miroir ZFS

### 6.1 Vérification du pool

```bash
zpool status
```

Le pool utilisait un identifiant `by-id`, pas `/dev/sda3`.

Important : toujours utiliser les identifiants persistants.

---

### 6.2 Clonage de la table GPT

```bash
sgdisk -R=/dev/sdb /dev/sda
sgdisk -G /dev/sdb
```

Vérification :

```bash
ls -l /dev/disk/by-id/ | grep sdb
```

Les partitions `-part1`, `-part2`, `-part3` étaient présentes.

---

### 6.3 Attachement du miroir

Commande exécutée avec identifiants persistants :

```bash
zpool attach rpool \
scsi-...sda-part3 \
scsi-...sdb-part3
```

Suivi :

```bash
watch -n 2 zpool status
```

Resilver terminé sans erreur.

Résultat :

```
mirror-0
  sda3
  sdb3
```

---

## 7. Vérification du boot (BIOS)

### 7.1 Détermination du mode boot

```bash
ls /sys/firmware/efi
```

Résultat : absent → mode BIOS.

### 7.2 Vérification MBR

```bash
dd if=/dev/sda bs=512 count=1 | hexdump -C | head
dd if=/dev/sdb bs=512 count=1 | hexdump -C | head
```

Les deux MBR étaient identiques.

Conclusion :

* Le bootloader GRUB est présent sur les deux disques.
* Aucune réinstallation GRUB nécessaire.
* proxmox-boot-tool refresh suffisant.

---

## 8. État final

Stockage :

* rpool en miroir ZFS
* Deux disques ONLINE
* Resilver complet
* 0 erreur

Boot :

* BIOS
* MBR présent sur les deux disques
* Redondance boot assurée

VM :

* Toutes sur ZFS
* Plus de stockage hybride
* Architecture cohérente

Cluster Kubernetes :

* Stable
* Worker réintégré
* Aucun impact persistant

---

## 9. Enseignements clés

1. Toujours auditer les VM avant suppression d’un datastore.
2. Toujours utiliser les identifiants `by-id` avec ZFS.
3. Ne jamais manipuler GRUB sans vérifier le mode de boot.
4. Ne pas mélanger UEFI et BIOS sans validation.
5. Tester le boot réel via BIOS pour validation finale.

---

## 10. Résultat stratégique

Avant :

* SPOF disque
* Stockage hybride
* Risque élevé

Après :

* Tolérance panne disque
* Architecture ZFS homogène
* Base saine pour évolution vers cluster Proxmox
* Préparation à Ceph future
