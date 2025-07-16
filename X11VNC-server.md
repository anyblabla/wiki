---
title: X11VNC server
description: Installation du serveur X11VNC pour une prise de contrôle via un client comme Remmina. Testée et fonctionnelle sur Linux Ubuntu/Mint.
published: true
date: 2025-07-16T23:33:29.370Z
tags: x11, vnc, x11vnc, remmina
editor: markdown
dateCreated: 2024-05-04T22:23:38.737Z
---

Tester et fonctionnel sur Linux Ubuntu 22.04/22.10/23.04/23.10 et Linux Mint 21.x/22.x/23.x 👍

-   Installation de “X11VNC server" :

```plaintext
sudo apt install x11vnc
```

-   On édite le fichier “x11vnc.service" :

```plaintext
sudo nano /lib/systemd/system/x11vnc.service
```

-   On copie et on colle les lignes suivantes en changeant "**password**" par son mot de passe personnel :

```plaintext
[Unit]
Description=x11vnc service
After=display-manager.service network.target syslog.target

[Service]
Type=simple
ExecStart=/usr/bin/x11vnc -forever -display :0 -auth guess -passwd password
ExecStop=/usr/bin/killall x11vnc
Restart=on-failure

[Install]
WantedBy=multi-user.target
```

-   On sauvegarde les modifications \[CTRL + X\], on confirme oui \[O\] et on valide par entrée \[ENTER\].
-   On redémarre le “deamon” \[daemon-reload\] parce qu'on a modifié un fichier “service” \[x11vnc.service\] :

```plaintext
sudo systemctl daemon-reload
```

-   On active automatiquement au démarrage du système le service “x11vnc” \[x11vnc.service\] :

```plaintext
sudo systemctl enable x11vnc.service
```

-   On démarre immédiatement le service “x11vnc” \[x11vnc.service\] :

```plaintext
sudo systemctl start x11vnc.service
```

-   On peut vérifier le status du service “x11vnc” :

```plaintext
sudo systemctl status x11vnc.service
```

-   Vous devriez obtenir un écran de ce genre :

![](/x11vnc-service/x11vnc-service-status-running.png)

-   Je conseille “Remmina”comme client pour procéder aux connexions :

```plaintext
sudo apt install remmina
```

-   Démonstration en vidéo : [https://peertube-blablalinux.be/w/fWBSLYLj3VBdzYNTjiHy1H](https://peertube-blablalinux.be/w/fWBSLYLj3VBdzYNTjiHy1H)