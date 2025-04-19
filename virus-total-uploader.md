---
title: VirusTotal Uploader
description: Nous allons installer les dépendances nécessaire pour la compilation de VirusTotal Uploader à partir d'un dépôts GitHub cloné.
published: true
date: 2025-04-19T23:32:27.955Z
tags: virus, virustotal, uploader
editor: markdown
dateCreated: 2024-06-12T18:14:23.441Z
---

# VirusTotal Uploader, c'est quoi ?

*C'est un outil en ligne qui sert à analyser les fichiers, les domaines, les adresses IP et URL suspects afin de détecter les logiciels malveillants et autres violations, et de les partager automatiquement avec la communauté de la sécurité.*

-   [Site officiel](https://www.virustotal.com/gui/home/upload)
-   [Dépôt GitHub](https://github.com/VirusTotal/qt-virustotal-uploader)
-   [Wikipédia](https://fr.wikipedia.org/wiki/VirusTotal)

# Que va-t-on faire ?

*VirusTotal Uploader existe également en tant que logiciel, installable en clonant un dépôt* [GitHub](https://github.com/)*, suivi ensuite par une compilation. C'est ce que nous allons faire sur une distribution* [Linux Mint](https://www.linuxmint.com) *21.3 Cinnamon (base* [Ubuntu](https://ubuntu.com) *22.04.3).*

*Ce programme utilise en interne l'*[API](https://fr.wikipedia.org/wiki/Interface_de_programmation) *publique VirusTotal. Vous pouvez glisser et déposer un fichier ou un dossier dans le programme pour le mettre en file d'attente pour le téléchargement et l'analyse.*

![](/virus-total-uploader/virus-total-uploader.png)

Virustotal Uploader sur Linux Mint 21.3

# Step by Step

Instruction valable pour [Debian](https://www.debian.org/index.fr.html)/Ubuntu/Mint. Installation réalisée sur Linux Mint 21.3 (Cinnamon).

-   On rafraîchit les dépôts…

```plaintext
sudo apt update
```

-   Obtenir des dépendances (Linux Mint **Cinnamon**)…

```plaintext
sudo apt-get install build-essential qtchooser qt5-default libjansson-dev libcurl4-openssl-dev git zlib1g-dev
```

-   Obtenir des dépendances (Linux Mint XFCE)…

```plaintext
sudo apt install qtbase5-dev qtchooser qt5-qmake qtbase5-dev-tools build-essential libjansson-dev libcurl4-openssl-dev git zlib1g-dev
```

-   Cloner la bibliothèque “c-vtapi”…

```plaintext
git clone https://github.com/VirusTotal/c-vtapi.git
```

-   Passer au répertoire “c-vtapi”…

```plaintext
cd c-vtapi
```

-   Obtenir les dépendances "c-vtapi"…

```plaintext
sudo apt-get install automake autoconf libtool libjansson-dev libcurl4-openssl-dev
```

-   Configurer avec les options par défaut et créer…

```plaintext
autoreconf -fi && ./configure && make
```

-   Installer sur le système, par défaut cela va dans "/usr/local/lib"…

```plaintext
sudo make install
```

-   Configurer l'éditeur de liens dynamique pour ajouter “/usr/local/lib” au chemin…

```plaintext
sudo sh -c 'echo "/usr/local/lib" > /etc/ld.so.conf.d/usr-local-lib.conf'
```

```plaintext
sudo ldconfig
```

-   Revenir au répertoire de base…

```plaintext
cd ..
```

-   Cloner QT VirusTotal Uploader…

```plaintext
git clone https://github.com/VirusTotal/qt-virustotal-uploader.git 
```

```plaintext
cd qt-virustotal-uploader
```

-   Lancez “qmake” en spécifiant “qt5”…

```plaintext
qtchooser -run-tool=qmake -qt=5
```

-   Compiler avec 4 tâches parallèles…

```plaintext
make -j4
```

-   On peut maintenant lancer le logiciel avec…

```plaintext
sudo ./VirusTotalUploader
```

Ce sera à vous de créer un lanceur dans le menu principal des applications, ici le menu [Cinnamon](https://projects.linuxmint.com/cinnamon/) puisque nous sommes sur Linux Mint.

Ce qui suit n'est pas obligatoire, mais c'est mieux 😉

-   Rendez-vous sur le site [VirusTotal](https://www.virustotal.com/gui/home/upload) pour créer un compte et vous connecter.

![](/virus-total-uploader/virus-total-uploader-compte-amaury-libert.png)

-   Vous allez maintenant cliquer sur votre nom de profil en haut à droite et choisir “API Key” afin de copier votre clé personnelle API…

![](/virus-total-uploader/virus-total-uploader-api-key.png)

-   Dans le logiciel VirusTotal Uploader que l'on vient d'installer sur notre Linux Mint, vous allez vous rendre dans la barre d'outils, “Files” => “Preferences”, et coller votre clé API personnelle précédemment copiée…

![](/virus-total-uploader/virus-total-uploader-software-api-key.png)

-   Vous pouvez maintenant analyser 🫵…

![](/virus-total-uploader/virus-total-uploader-software-file-upload.gif)