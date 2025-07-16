---
title: Docker Compose PsiTransfert
description: Solution simple de partage de fichiers auto-hébergée open source. C'est une alternative aux services payants comme Dropbox, WeTransfer.
published: true
date: 2025-07-16T23:34:17.579Z
tags: docker, portainer, psitransfer, share, file
editor: markdown
dateCreated: 2024-06-30T14:00:46.077Z
---

Pratique pour déployer rapidement [PsiTransfer](https://psi.cx/2017/psitransfer/) dans [Portainer](https://www.portainer.io/) en créant une pile ([stack](https://docs.portainer.io/user/docker/stacks)) à partir d'un fichier [compose YAML](https://docs.docker.com/compose/compose-application-model/).

# Liens utiles

-   [Site officiel](https://psi.cx/2017/psitransfer/)
-   [Documentation (site officiel) installation](https://psi.cx/2017/psitransfer-installation/)
-   [Page de projet GitHub](https://github.com/psi-4ward/psitransfer)
-   [Documentation (page de projet GitHub)](https://github.com/psi-4ward/psitransfer/tree/master/docs)
-   [Page de projet DockerHub](https://hub.docker.com/r/psitrax/psitransfer)

# Compose

```plaintext
name: psitransfer
services:
    psitransfer:
        ports:
            - 0.0.0.0:3000:3000
        environment:
            #- PSITRANSFER_ADMIN_PASS=blablalinux
        volumes:
            - /data:/data
        image: psitrax/psitransfer:latest
        restart: always
```

Fichier compose également disponible sur [ByteStash Blabla Linux](https://bytestash.blablalinux.be/public/snippets).

# Personnalisation Compose

-   Vous pouvez modifier les ports d'accès de l'hôte (premier) et du container (deuxième)…

`- 0.0.0.0:3000:3000`

-   Vous pouvez dé-commenter (supprimer la dièse) la ligne suivante et préciser un mot de passe pour activer et sécuriser la page “admin” qui vous permettra d'accéder aux données hébergées sur votre PsiTransfer…

`#- PSITRANSFER_ADMIN_PASS=blablalinux`

![](/docker-compose-psitransfer/psitransfer-password-admin.png)

Page “admin” de PsiTransfer - Demande de mot de passe

![](/docker-compose-psitransfer/psitransfer-admin.png)

Page “admin” de PsiTransfer

-   Nous avons monté un volume pour conserver les données et y avoir accès. Vous pouvez bien sûr spécifier un autre emplacement. Pour ma part, il est monté à la racine de l'hôte. Les données envoyées sont chiffrées. Je peux voir que des données sont hébergées sur mon PsiTransfert, mais je ne peux pas y avoir accès. Les utilisateurs peuvent dormir tranquilles 😉…

`- /data:/data`

![](/docker-compose-psitransfer/psitransfer-data.png)

PsiTranfer - Données /data

![](/docker-compose-psitransfer/psitransfer-data-json.png)

PsiTranfer - Données /data - Fichier .json

![](/docker-compose-psitransfer/psitransfer-data-2.png)

PsiTranfer - Données /data chiffrées

![](/docker-compose-psitransfer/psitransfer-data-3.png)

PsiTranfer - Données /data chiffrées

-   J'utilise ici l'image docker “latest” qui est la dernière image stable en date. Vous pouvez bien sûr spécifier un [tag spécifique](https://hub.docker.com/r/psitrax/psitransfer/tags) et ainsi utiliser une autre image…

`image: psitrax/psitransfer:latest`

# Démonstration

Vous avez le choix…

-   [Facebook](https://www.facebook.com/blablalinux/videos/453133560991741)
-   [Twitter](https://x.com/i/status/1806849212237173113)
-   [Mastodon](https://mastodon-blablalinux.be/@blablalinux/112697108025204971)