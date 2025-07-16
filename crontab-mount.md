---
title: Crontab Mount
description: Montage automatique d'un périphérique de stockage. On oublie Fstab. On va utiliser Crontab.
published: true
date: 2025-07-16T22:15:19.694Z
tags: cron, crontab, mount
editor: markdown
dateCreated: 2024-06-24T00:12:35.609Z
---

# Situation

Le démarrage de votre machine s'effectue sur un périphérique de stockage externe [USB](https://fr.wikipedia.org/wiki/USB) où se trouve la partition /boot. Tout se passe bien. 

Vous connectez un deuxième périphérique de stockage externe USB, vous avez demandé un montage automatique via /etc/fstab.

Le démarrage ne s'effectue plus ! 

Pourquoi ?

Dans votre firmware [BIOS](https://fr.wikipedia.org/wiki/BIOS_(informatique))/[EFI](https://fr.wikipedia.org/wiki/UEFI), vous avez défini le démarrage sur périphérique amovible USB, mais vous ne pouvez pas spécifier sur quel périphérique précisément le démarrage doit s'effectuer ! Comme il y en a plusieurs, le démarrage bloque.

Vous pouvez désactiver l'entrée du deuxième périphérique dans le fichier /etc/fstab. Le démarrage s'effectuera correctement sur le premier périphérique contenant la partition /boot, mais le montage automatique du deuxième périphérique ne se fera pas !

L'astuce consiste alors à demander un montage automatique, non pas au démarrage via /etc/fstab, mais à la connexion de la session via [Crontab](https://fr.wikipedia.org/wiki/Cron).

# Connaître l'UUID

La première chose à faire est de connaître l'[UUID](https://fr.wikipedia.org/wiki/Universally_unique_identifier) (Identifiant Universel Unique) de votre périphérique.

-   Nous allons utiliser cette commande…

```plaintext
sudo blkid
```

Vous devriez obtenir ce genre de sortie…

![](/crontab-mount/blkid.png)

On peut voir en surbrillance le numéro UUID que je vais utiliser. Il concerne un disque externe USB dédié aux sauvegardes [RSYNC](https://fr.wikipedia.org/wiki/Rsync).

# Répertoire du point de montage

-   Nous allons maintenant créer un répertoire (pour moi backup-rsync) dans /media pour accueillir notre point de montage…

```plaintext
sudo mkdir /media/backup-rsync
```

# Ajout dans Crontab

-   Nous pouvons maintenant éditer Crontab…

```plaintext
sudo crontab -e
```

-   Nous pouvons ajouter la ligne suivante (à adapter selon vos besoins) à la fin du fichier…

```plaintext
@reboot sudo mount -U edd2b28e-b65e-474b-bd7d-6c1563144ab6 /media/backup-rsync
```

Vous pouvez redémarrer, le tout est joué 😎