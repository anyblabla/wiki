---
title: DD/RSYNC - Distant/Local 
description: Récemment, j'ai du rapatrier un VPS hébergé chez OVH vers mon hyperviseur local Proxmox ! Voici la commande magique 😉
published: true
date: 2025-04-19T20:54:51.017Z
tags: rsync, clone
editor: markdown
dateCreated: 2024-12-01T15:36:34.400Z
---

*Juste pour l'histoire, l'ensemble de mes services tournent à la maison sur un cluster* [Proxmox VE](https://www.proxmox.com/en/proxmox-virtual-environment/overview)*.*

*L'ensemble ? Non ! Deux services tournent sur des* [VPS](https://w.wiki/CG6P) *chez* [OVH](https://www.ovhcloud.com/fr/). [Mastodon](https://mastodon-blablalinux.be/@blablalinux) *et* [PeerTube](https://peertube-blablalinux.be/a/anyblabla/video-channels)*.*

*Je me suis donc attelé à leur rapatriement vers la maison.*

*Voici la commande magique que j'ai utilisée, à partir du serveur local, afin de réaliser un clone complet du disque au travers de* [SSH](https://w.wiki/Acov)*.*

```plaintext
ssh root@xx.xx.xx.xxx -p xxxx "dd if=/dev/sda bs=100M" | dd of=/dev/sdb status=progress
```

-   **ssh root@xx.xx.xx.xxx** : on appelle SSH avec son nom utilisateur vers l'adresse IP du serveur distant.
-   **\-p xxxx** : on précise le port SSH, si ce dernier a été changé et n'est plus le port 22 par défaut.
-   **dd if=/dev/sda bs=100M** : on appelle la commande DD pour cloner, en précisant la source (IF) suivie de la taille des blocs. Ici “bs=100M” car c'est un disque SSD.
-   **dd of=/dev/sdb** : on continue avec la commande DD pour le clonage, mais cette fois sur la destination (of=/dev/sdb) en demandant l'affichage de la progression (status=progress).

Sur l'image ci-dessous, on peut voir que l'opération de clonage avec la commande DD au travers de SSH, d'un serveur distant vers un serveur local, c'est déroulé sans problème.

![](/dd-rsync-distant-local/dd.png)

Si comme moi, vous n'arrêtez pas directement le serveur distant pour mettre en route le serveur local à la place, il va falloir lancer une synchronisation, à partir du serveur local, au moment arrivé ! Voici la commende magique 😉

```plaintext
rsync -avz --inplace --delete -P --stats --human-readable --progress --exclude={"/dev/*","/proc/*","/sys/*","/tmp/*","/run/*","/media/*","/lost+found"} -e "ssh -p xxxx" root@xx.xx.xx.xxx:/ /media/mastodon/
```

-   **\-avz**
    -   **\-a (archive)** : cette option augmente la quantité d'informations qui vous sont données lors du transfert. Par défaut, rsync fonctionne en silence.
    -   **\-v (verbose)** : c'est une façon rapide de dire que vous voulez de la récursion et envie de préserver presque tout.
    -   **\-z (compress)** : avec cette option, rsync compresse les données du fichier lorsqu'elles sont envoyées à la machine de destination, ce qui réduit la quantité de données transmises - quelque chose qui est utile sur une connexion lente.
-   **\--inplace** : cette option modifie la manière dont rsync transfère un fichier lorsque ses données doivent être mises à jour : au lieu de la méthode par défaut consistant à créer une nouvelle copie du fichier et à la mettre en place quand il est terminé, rsync écrit plutôt les données mises à jour directement dans le fichier de destination.
-   **\--delete** : cela indique à rsync de supprimer les fichiers étrangers du côté récepteur (ceux qui ne sont pas du côté émetteur).
-   **\-P** : équivaut à --partiel --progrès. Son objectif est de faciliter grandement leur précision, deux options pour un long transfert qui peut être interrompu.
-   **\--stats** :  indique à rsync d'afficher un ensemble verbeux de statistiques sur le transfert de fichiers, vous permettant de voir l'efficacité de rsync.
-   **\--human-readable** : sortie dans un format plus lisible par l'homme.
-   **\--progress** : cette option indique à rsync d'afficher des informations montrant la progression du transfert.
-   **\--exclude** : cette option permet d'exclure sélectivement certains dossiers/fichiers de la liste à transférer.
-   **\-e** : cette option vous permet de choisir un programme shell distant alternatif à utiliser pour la communication entre le local et des copies distantes de rsync.

Pour le restant de la commande…

-   **"ssh -p xxxx" root@xx.xx.xx.xxx:/ /media/mastodon/**

…c'est ce que l'on a vu plus haut. Connexion SSH avec précision de la source et de la destination.

### Liens utiles

-   [https://linux.die.net/man/1/rsync](https://linux.die.net/man/1/rsync)
-   [https://ss64.com/bash/rsync\_options.html](https://ss64.com/bash/rsync_options.html)