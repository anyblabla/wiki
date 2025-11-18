---
title: Rsync - Le guide de la synchronisation intelligente
description: Le guide complet de rsync, l'outil de synchronisation intelligent. Apprenez les commandes de base, l'exclusion de fichiers, les transferts SSH et la création de sauvegardes historiques incrémentales.
published: true
date: 2025-11-18T21:42:05.251Z
tags: rsync, sync, transfert
editor: markdown
dateCreated: 2025-11-18T21:37:22.686Z
---

**Rsync** est un pilier incontournable de l'administration système sous **Linux**. Cet outil, essentiel pour l'automatisation des tâches de sauvegarde, est reconnu pour son efficacité grâce à son algorithme unique qui minimise les transferts de données. Que vous cherchiez à synchroniser deux dossiers locaux ou à sécuriser des données sur un serveur distant via SSH, ce guide vous fournira toutes les commandes clés.

-----

## 1\. Qu'est-ce que rsync ?

**rsync** (pour **remote synchronization**) est un utilitaire de ligne de commande très puissant pour **synchroniser des fichiers et des répertoires** d'un endroit à un autre, que ce soit :

  * **Localement** (entre deux dossiers sur la même machine).
  * **À distance** (entre une machine locale et un serveur distant, ou vice-versa).

-----

## 2\. Fonctionnalités clés

La principale force de rsync réside dans son **algorithme de transfert différentiel** :

1.  **Transfert efficace :** Contrairement à une simple copie, rsync ne transfère que les **blocs de données modifiés** entre les fichiers source et destination. Cela le rend extrêmement **rapide** pour les mises à jour et réduit la bande passante utilisée.
2.  **Miroir/sauvegarde :** Il est parfait pour créer des **sauvegardes incrémentielles** ou des **miroirs exacts** (copie parfaite) de répertoires, car il gère les permissions, les propriétaires, les horodatages, et les liens symboliques.
3.  **Flexibilité :** Il utilise généralement **SSH** pour le transport à distance, assurant un transfert **sécurisé**.

En résumé, rsync est l'outil de choix pour des **copies de fichiers rapides, sécurisées et intelligentes** sur des réseaux.

-----

## 3\. Sauvegarde et synchronisation de base

### 3.1. Commande de sauvegarde de base

Voici la commande type pour synchroniser de façon incrémentale le contenu du répertoire source vers le répertoire de destination :

```bash
rsync -av --delete-after /home/utilisateur/ /media/sauvegarde/home_backup/
```

| Option | Signification | Détails |
| :--- | :--- | :--- |
| **`-a`** | Archive mode | Copie de façon récursive tout le contenu en conservant les **permissions**, les **horodatages**, etc. |
| **`-v`** | Verbose | Affiche les fichiers transférés au fur et à mesure (utile pour le débogage). |
| **`--delete-after`** | Suppression | Supprime dans la destination les fichiers qui ont été supprimés dans la source. |

> **⚠️ Note sur le slash final (`/`) :** L'utilisation de `/home/utilisateur/` (avec un slash final) copie le **contenu** du dossier. Sans le slash, c'est le **dossier lui-même** qui serait copié.

> **Note sur l'utilisateur :** Pour que l'option `-a` puisse conserver correctement les propriétaires et les permissions des fichiers (y compris ceux appartenant à d'autres utilisateurs), la commande `rsync` doit généralement être exécutée avec les droits de superutilisateur (via **`sudo`**).

### 3.2. Exclure des fichiers ou dossiers spécifiques

Utilisez l'option `--exclude` pour ignorer les éléments temporaires, les caches ou les Corbeilles.

```bash
rsync -av --delete-after \
    --exclude '.cache' \
    --exclude '.local/share/Trash' \
    /home/utilisateur/ /media/sauvegarde/home_backup/
```

| Option | Rôle |
| :--- | :--- |
| **`--exclude 'NOM'`** | Exclut les fichiers ou dossiers correspondant au nom spécifié. |

### 3.3. Sauvegarde vers un serveur distant (via SSH)

`rsync` utilise le protocole **SSH** par défaut pour garantir la sécurité et le chiffrement du transfert.

  * **Sauvegarde locale vers un serveur distant :**
    ```bash
    rsync -av --delete-after \
        /home/utilisateur/ utilisateur_ssh@mon_serveur:/sauvegarde/mon_pc/
    ```

-----

## 4\. Stratégies avancées et contrôle

### 4.1. Créer des sauvegardes historiques (Liens durs via `--link-dest`)

Utilisez l'option `--link-dest` pour créer des sauvegardes horodatées complètes, mais qui n'utilisent de l'espace disque supplémentaire que pour les fichiers *modifiés* grâce aux liens durs.

  * **Commande :**
    ```bash
    rsync -av --delete \
        --link-dest=/media/sauvegarde/2025-11-17/ \
        /home/utilisateur/ /media/sauvegarde/2025-11-18/
    ```
    > **Note :** Les fichiers inchangés dans le répertoire `2025-11-18/` pointeront vers ceux du répertoire précédent (`2025-11-17/`).

### 4.2. Limiter la bande passante

Contrôlez la vitesse de transfert sur les réseaux lents ou partagés.

  * **Exemple :** Limiter le transfert à 500 Kilo-Octets par seconde (KBPS).

<!-- end list -->

```bash
rsync -av --bwlimit=500 /source/ /destination/
```

| Option | Rôle |
| :--- | :--- |
| **`--bwlimit=KBPS`** | Limite la vitesse du transfert à la valeur spécifiée en KBPS. |

### 4.3. Tester sans transférer (Dry run)

Utilisez ce mode pour vérifier l'effet de votre commande (surtout avec `--delete`) avant de l'exécuter réellement.

  * **Exemple :**

<!-- end list -->

```bash
rsync -av --delete --dry-run /source/ /destination/
```

| Option | Rôle |
| :--- | :--- |
| **`--dry-run` ou `-n`** | **Simule** l'opération. Ne touche pas aux fichiers. |

-----

## Conclusion

Maîtriser `rsync` permet de mettre en place des stratégies de sauvegarde **robustes**, rapides et économes en ressources. Les options avancées, notamment `--link-dest`, sont la clé pour passer d'une simple copie à un système de gestion d'historique de fichiers professionnel. N'hésitez pas à tester vos commandes avec l'option **`--dry-run`** avant de les intégrer à vos scripts de sauvegarde quotidiens \!