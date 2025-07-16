---
title: Wine
description: Procédure d'installation de Wine. Testée et fonctionnelle sous Linux Mint.
published: true
date: 2025-07-16T22:14:56.293Z
tags: wine, émulateur, émulation, qemu
editor: markdown
dateCreated: 2024-05-05T20:15:53.231Z
---

Testé et fonctionnel sur Linux Mint 22.x 👍

La procédure de cette page installe la dernière version de Wine “**Stable**”, Wine “**Development**” ou Wine “**Staging**” (expérimentale).

-   Vérifiez si vous êtes sous architecture 64 bits, la commande suivante doit répondre "amd64" :

```plaintext
dpkg --print-architecture
```

-   Vérifiez si l'architecture 32 bits est prise en charge, la commande suivante devrait répondre "i386" :

```plaintext
dpkg --print-foreign-architectures
```

-   Si "i386" ne s'affiche pas, exécutez ce qui suit pour prendre en charge l'architecture 32 bits :

```plaintext
sudo dpkg --add-architecture i386
```

-   Vérifiez maintenant que les deux architectures sont bien prises en charge, la commande suivante doit répondre “amd64” ET “i386” :

```plaintext
dpkg --print-foreign-architectures
```

-   Créez le répertoire “keyrings” avec les bons droits :

```plaintext
sudo mkdir -pm755 /etc/apt/keyrings
```

-   Téléchargez et ajoutez la clé du référentiel “WineHQ” :

```plaintext
sudo wget -O /etc/apt/keyrings/winehq-archive.key https://dl.winehq.org/wine-builds/winehq.key
```

-   Téléchargez le fichier des sources de “WineHQ” :

```plaintext
sudo wget -NP /etc/apt/sources.list.d/ https://dl.winehq.org/wine-builds/ubuntu/dists/jammy/winehq-jammy.sources
```

-   Mettez à jour la base de données des paquets :

```plaintext
sudo apt update
```

-   Installez “Wine” **Stable** :

```plaintext
sudo apt install --install-recommends winehq-stable
```

OU

-   Pour “Wine” **Development** :

```plaintext
sudo apt install --install-recommends winehq-devel
```

OU

-   Pour “Wine” **Staging** (éxpérimentale) :

```plaintext
sudo apt install --install-recommends winehq-staging
```

-   Vérifiez que l'installation a réussi :

```plaintext
wine --version
```

-   Configurez “Wine” :

```plaintext
wine winecfg
```

-   Testez pour s'amuser :

```plaintext
wine clock
```

-   Créer le menu “Wine” dans le menu principal des applications (mint-menu) :

```plaintext
sudo apt install wine-installer
```

![](/wine/wine-mint-menu.png)

-   Démonstration en vidéo : [https://peertube.blablalinux.be/w/13JYaBK9TFASrHPwYfwBAT](https://peertube-blablalinux.be/w/x96eafxHoB2z6svWksSLRy)