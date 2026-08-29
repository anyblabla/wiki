---
title: Configurer un datastore S3 (MinIO) sur Proxmox Backup Server
description: Guide complet pour créer un datastore S3 MinIO sur PBS, avec gestion du cache dédié selon le type d'installation : disque virtuel (VM) ou dataset ZFS avec quota (serveur physique).
published: true
date: 2026-08-29T18:37:08.913Z
tags: proxmox, pbs, zfs, backup, minio, s3, homelab, selfhosting
editor: markdown
dateCreated: 2026-08-28T23:28:31.706Z
---

Depuis la version 4.x, Proxmox Backup Server (PBS) peut utiliser un stockage compatible S3 (MinIO, Ceph RGW, Backblaze B2, Wasabi, AWS S3...) comme backend de datastore, en complément ou en remplacement d'un stockage local classique.

Ce guide couvre la configuration complète avec **MinIO auto-hébergé**, en tenant compte de deux cas de figure fréquents en homelab :

- **PBS en machine virtuelle** avec de la marge disque disponible → cache sur un **disque virtuel dédié**
- **PBS physique déjà entièrement provisionné** (tous les disques dans un pool ZFS) → cache sur un **dataset ZFS avec quota**

## Pourquoi un datastore S3 ?

Un datastore S3 permet de sortir ses sauvegardes du serveur PBS lui-même, vers un stockage objet (local ou distant), tout en gardant les fonctionnalités habituelles de PBS : déduplication, vérification, purge, synchronisation entre datastores.

⚠️ **Limitations à connaître avant de se lancer** :
- La fonctionnalité est encore marquée *technology preview* selon les versions.
- PBS ne supporte pas le versioning ni l'Object Lock au niveau du bucket S3 — les activer peut corrompre la structure du datastore.
- Les restaurations et vérifications sont plus lentes qu'en local, car elles nécessitent de retélécharger les chunks non présents en cache.
- Pour un usage critique, privilégier un datastore local en primaire et le S3 en copie secondaire/offsite.

## Prérequis

- Une instance MinIO (ou autre S3 compatible) déjà déployée et accessible en **HTTPS** (le HTTP seul n'est pas supporté par le client S3 de PBS).
- Un certificat TLS valide (Let's Encrypt ou équivalent). Pour un certificat auto-signé, il faudra renseigner l'empreinte SHA-256 du certificat dans la configuration du point de terminaison.
- Une clé d'accès et une clé secrète MinIO.

## Étape 1 — Créer un bucket dédié

Dans l'interface MinIO (ou via `mc`), créer un bucket dédié à ce datastore PBS, avec les réglages suivants :

| Option | Valeur recommandée |
| :--- | :--- |
| Versioning | Désactivé |
| Object Locking | Désactivé (ne peut plus être activé après coup, mais incompatible avec PBS de toute façon) |
| Quota | Optionnel, selon l'espace que l'on souhaite réserver |
| Access Policy | Private |

Ne pas partager ce bucket avec d'autres usages : PBS gère lui-même le cycle de vie des objets qu'il y stocke.

## Étape 2 — Ajouter le point de terminaison S3

Dans PBS : **Configuration → Points de terminaison S3 → Ajouter**.

| Champ | Valeur |
| :--- | :--- |
| Identifiant | Un nom libre, ex. `minio-local` |
| Point de terminaison | Le nom d'hôte de l'instance MinIO (ex. `minio.exemple.tld`), sans `https://` |
| Port | 443 (par défaut) |
| Région | `us-east-1` fonctionne dans la plupart des cas avec MinIO |
| Clé d'accès / Clef secrète | Les identifiants créés côté MinIO |
| Empreinte | À laisser vide avec un certificat valide ; à renseigner uniquement pour un certificat auto-signé |
| Par chemin d'accès (path-style) | Voir note ci-dessous |

### 🔧 À propos du mode "Par chemin d'accès"

C'est le réglage qui pose le plus souvent problème. Deux modes existent pour adresser un bucket S3 :

- **Path-style** : `https://minio.exemple.tld/mon-bucket/...`
- **Virtual-hosted style** : `https://mon-bucket.minio.exemple.tld/...`

Le bon mode dépend de la configuration DNS et du reverse proxy devant MinIO :

- Si un seul sous-domaine est configuré pour MinIO (sans wildcard `*.minio.exemple.tld`), **le mode path-style est requis**. Sans lui, l'ajout du datastore échoue avec une erreur `400 Bad Request : bucket does not exist or no permission to access it`.
- Si MinIO est configuré avec un domaine wildcard (variable `MINIO_DOMAIN`), c'est parfois l'inverse qui est vrai.

**En cas d'erreur 400** (que ce soit au listing des buckets ou à l'accès à un bucket précis), basculer cette case (cocher/décocher) est le premier réflexe à avoir avant de chercher plus loin.

## Étape 3 — Préparer un espace de cache dédié

Même en mode S3, PBS a besoin d'un **cache local persistant**. Ce n'est pas une simple option de confort : sans lui, le datastore ne fonctionne pas. Proxmox recommande **64 à 128 Gio** minimum, sur un disque, une partition ou un dataset dédié — jamais partagé avec le disque système.

Le cache est **glouton par défaut** : il consomme tout l'espace disponible sur le système de fichiers qui l'héberge. Sans isolation, un cache qui grossit peut donc remplir le disque racine et planter l'OS de PBS, pas seulement le datastore.

Deux approches selon le type d'installation :

### Cas A — PBS en VM avec de la marge disque

1. Ajouter un second disque virtuel à la VM depuis l'interface Proxmox VE (Matériel → Ajouter → Disque dur), sur un pool de stockage séparé du disque système si possible.
2. **Décocher l'option "Sauvegarde"** pour ce disque dans la configuration Proxmox VE : c'est un cache reconstructible, l'inclure dans les sauvegardes de la VM n'apporte rien et gaspille du temps/espace.
3. Dans PBS, formater et monter ce disque via **Stockage et disques → Répertoire → Créer : Directory**, en sélectionnant le disque, un système de fichiers ext4, et en **décochant "Ajouter en tant qu'entrepôt de données"** (on veut seulement le montage).

![vm-proxmox-backup-server-datastore-s3-minio.png](/proxmox-backup-server-datastore-s3-minio/vm-proxmox-backup-server-datastore-s3-minio.png)

### Cas B — PBS physique sans disque libre (tout en pool ZFS)

Quand tous les disques physiques sont déjà utilisés dans un pool ZFS, pas besoin d'ajouter de matériel : un **dataset ZFS dédié avec quota** remplit exactement le même rôle qu'un disque séparé.

```bash
# Identifier le pool existant
zfs list

# Créer un dataset dédié avec un quota strict (ex. 100 Gio)
zfs create -o quota=100G rpool/pbs-s3-cache

# Optionnel : définir un point de montage explicite
zfs set mountpoint=/mnt/datastore/minio-cache rpool/pbs-s3-cache

# Vérifier
df -h /mnt/datastore/minio-cache
```

Le quota isole le cache du reste du pool (OS, autres datastores) exactement comme le ferait une partition séparée. C'est d'ailleurs l'option officiellement recommandée par la documentation Proxmox au même titre qu'un disque ou une partition dédiée.

⚠️ Sur un pool ZFS en RAID0 (striping simple, sans redondance), garder à l'esprit qu'une panne d'un seul disque fait perdre l'intégralité du pool — cache compris, mais surtout tout le reste de ce qui y est stocké.

![proxmox-backup-server-datastore-s3-minio.png](/proxmox-backup-server-datastore-s3-minio/proxmox-backup-server-datastore-s3-minio.png)

## Étape 4 — Créer le datastore S3

Dans PBS : **Entrepôt de données → Ajouter un entrepôt de données**.

| Champ | Valeur |
| :--- | :--- |
| Nom | Un nom libre pour le datastore |
| Type d'entrepôt de données | `S3` |
| Cache local | Le chemin préparé à l'étape 3 (ex. `/mnt/datastore/minio-cache`) |
| Identifiant du point de terminaison S3 | Celui créé à l'étape 2 |
| Compartiment (bucket) | Le bucket créé à l'étape 1 |

Si une erreur 400 apparaît à la sélection du bucket, retourner éditer le point de terminaison S3 et basculer l'option path-style (voir étape 2).

## Comprendre le rôle réel du cache

Contrainte importante à intégrer : le cache **n'est pas une étape intermédiaire** avant l'envoi vers S3. Les chunks d'une sauvegarde partent **simultanément** vers le bucket S3 et vers le cache local — ce n'est pas un système où les données transiteraient d'abord par le cache avant d'être poussées vers le bucket.

Le cache sert à deux choses :
- **Déduplication** : il retient quels chunks sont déjà connus côté S3, pour éviter de les ré-uploader inutilement.
- **Performance de lecture** : il garde les chunks les plus récemment utilisés (cache LRU) pour accélérer les restaurations et vérifications, en évitant de retélécharger depuis S3 à chaque fois.

Quand le cache est plein, les chunks les moins récemment utilisés sont évincés — sans perte de données, puisqu'ils restent stockés sur le bucket S3.

## Bonnes pratiques

- **Documenter la configuration** : consigner le point de terminaison, le bucket, le chemin du cache et les avertissements spécifiques (ex. "ne pas sauvegarder ce disque") dans les Notes de l'interface PBS ou un wiki, pour se faciliter la vie lors d'une future intervention.
- **Clé d'accès dédiée et scopée** : idéalement, éviter de réutiliser une clé root/admin MinIO partagée entre plusieurs usages. Créer une clé limitée en droits au bucket du datastore (`GetObject`, `PutObject`, `ListBucket`, `DeleteObject`), en gardant à l'esprit que le menu de sélection du bucket dans PBS nécessite aussi un droit de listing global des buckets du compte.
- **Surveiller la charge du garbage collector** : lors d'un pruning de datastore, la suppression en masse des chunks orphelins sur le backend S3 peut générer un pic de requêtes DELETE, à surveiller notamment sur un backend distant facturé à la requête.

## Sources

- [Documentation officielle Proxmox Backup Server — Storage](https://pbs.proxmox.com/docs/storage.html)
- Forums Proxmox — retours d'expérience communautaires sur le fonctionnement du cache S3 et les erreurs de configuration courantes
