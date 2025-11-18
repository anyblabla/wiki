---
title: Rsync - Le guide de la synchronisation intelligente
description: Le guide complet de rsync, l'outil de synchronisation intelligent. Apprenez les commandes de base, l'exclusion de fichiers, les transferts SSH et la création de sauvegardes historiques incrémentales.
published: true
date: 2025-11-18T21:37:22.686Z
tags: rsync, sync, transfert
editor: markdown
dateCreated: 2025-11-18T21:37:22.686Z
---

## 💻 Qu'est-ce que rsync ?

**rsync** (pour **remote synchronization**) est un utilitaire de ligne de commande très puissant pour **synchroniser des fichiers et des répertoires** d'un endroit à un autre, que ce soit :

  * **Localement** (entre deux dossiers sur la même machine).
  * **À distance** (entre une machine locale et un serveur distant, ou vice-versa).

-----

## ✨ Fonctionnalités clés

La principale force de rsync réside dans son **algorithme de transfert différentiel** :

1.  **Transfert efficace :** Contrairement à une simple copie, rsync ne transfère que les **blocs de données modifiés** entre les fichiers source et destination. Si un fichier de 1 Go a été modifié, il ne renvoie que la petite partie qui a changé, ce qui le rend extrêmement **rapide** pour les mises à jour et réduit la bande passante utilisée.
2.  **Miroir/sauvegarde :** Il est parfait pour créer des **sauvegardes incrémentielles** ou des **miroirs exacts** (copie parfaite) de répertoires, car il gère les permissions, les propriétaires, les horodatages, et les liens symboliques.
3.  **Flexibilité :** Il utilise généralement **SSH** pour le transport à distance, assurant un transfert **sécurisé**.

En résumé, rsync est l'outil de choix pour des **copies de fichiers rapides, sécurisées et intelligentes** sur des réseaux.

-----

## 4\. Sauvegarde et synchronisation

### 4.1. Commande de sauvegarde de base

Voici la commande type pour synchroniser de façon incrémentale le contenu du répertoire source vers le répertoire de destination :

```bash
rsync -av --delete-after /home/utilisateur/ /media/sauvegarde/home_backup/
```

| Option | Signification | Détails |
| :--- | :--- | :--- |
| **`-a`** | Archive mode | Copie de façon récursive tout le contenu du répertoire source en conservant les **permissions**, les **horodatages**, etc. (équivalent à `-rlptgoD`). |
| **`-v`** | Verbose | Affiche les fichiers transférés au fur et à mesure (utile pour le débogage ou l'exécution manuelle). |
| **`--delete-after`** | Suppression | Supprime dans la destination les fichiers qui ont été supprimés dans la source. La suppression se fait **après** le transfert. |

> **⚠️ Note sur le slash final (`/`) :** L'utilisation de `/home/utilisateur/` (avec un slash final) copie le **contenu** du dossier. Sans le slash (`/home/utilisateur`), c'est le **dossier lui-même** qui serait copié.

### 4.2. Exclure des fichiers ou dossiers spécifiques

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

### 4.3. Sauvegarde vers un serveur distant (via SSH)

`rsync` utilise le protocole **SSH** par défaut pour garantir la sécurité et le chiffrement du transfert.

  * **Sauvegarde locale vers un serveur distant :**
    ```bash
    rsync -av --delete-after \
        /home/utilisateur/ utilisateur_ssh@mon_serveur:/sauvegarde/mon_pc/
    ```

-----

## 5\. Stratégies avancées et contrôle

### 5.1. Créer des sauvegardes historiques (Liens durs via `--link-dest`)

Utilisez l'option `--link-dest` pour créer des sauvegardes horodatées complètes, mais qui n'utilisent de l'espace disque supplémentaire que pour les fichiers *modifiés* grâce aux liens durs.

  * **Commande :**
    ```bash
    rsync -av --delete \
        --link-dest=/media/sauvegarde/2025-11-17/ \
        /home/utilisateur/ /media/sauvegarde/2025-11-18/
    ```
    > **Note :** La destination (`2025-11-18/`) sera un miroir exact de la source, mais les fichiers inchangés pointeront vers ceux du répertoire précédent (`2025-11-17/`).

### 5.2. Limiter la bande passante

Contrôlez la vitesse de transfert sur les réseaux lents ou partagés.

  * **Exemple :** Limiter le transfert à 500 Kilo-Octets par seconde (KBPS).

<!-- end list -->

```bash
rsync -av --bwlimit=500 /source/ /destination/
```

| Option | Rôle |
| :--- | :--- |
| **`--bwlimit=KBPS`** | Limite la vitesse du transfert à la valeur spécifiée en KBPS. |

### 5.3. Tester sans transférer (Dry run)

Utilisez ce mode pour vérifier l'effet de votre commande (surtout avec `--delete`) avant de l'exécuter réellement.

  * **Exemple :**

<!-- end list -->

```bash
rsync -av --delete --dry-run /source/ /destination/
```

| Option | Rôle |
| :--- | :--- |
| **`--dry-run` ou `-n`** | **Simule** l'opération. Ne touche pas aux fichiers. |