---
title: Coloriser les logs Nginx en temps réel dans le terminal (avec sauvegarde automatique)
description: Découvrez comment coloriser vos flux de logs Nginx en temps réel dans le terminal. Un guide simple et pas à pas pour mettre en évidence les adresses IP, les codes HTTP et vos mots-clés.
published: true
date: 2026-06-06T23:25:41.485Z
tags: nginx, bash, sysadmin, terminal, logs, colorisation
editor: markdown
dateCreated: 2026-06-03T12:44:10.907Z
---

Ce guide complet vous accompagne pas à pas pour ajouter de la couleur à vos flux de logs Nginx en direct. Grâce à cette méthode simple, vous pourrez repérer les adresses IP, les codes HTTP (200, 403...), les pays GeoIP, vos domaines ou vos propres mots-clés en un coup d'œil directement dans votre terminal.

🎁 **Le petit bonus :** en plus de l'affichage coloré à l'écran, cette méthode sauvegarde automatiquement tout l'historique de votre session en direct dans un fichier texte propre pour pouvoir le relire calmement plus tard !

## Étape 1 : préparation du système et des dossiers

Avant de commencer, nous allons nous assurer que votre système possède les outils nécessaires et que les dossiers requis pour l'historique sont bien créés.

1. Connectez-vous sur votre serveur en ligne de commande.
2. Créez le dossier qui servira à stocker les copies de vos affichages en direct avec cette commande :

```bash
mkdir -p /root/npm/sorties_live/
```

3. Par sécurité, assurez-vous que votre système est à jour et possède les outils de base en exécutant :

```bash
apt update && apt install -y coreutils grep sed
```

*(Ces outils sont généralement préinstallés sur la plupart des distributions comme Debian ou Ubuntu, mais cela évite toute mauvaise surprise !)*

---

## Étape 2 : ouvrir le fichier de configuration de votre terminal

Pour ajouter nos raccourcis, nous devons les enregistrer dans le fichier qui gère votre terminal.

Par simplicité, beaucoup d'utilisateurs ajoutent tout à la fin du fichier principal `~/.bashrc`. Ouvrez-le avec l'éditeur `nano` :

```bash
nano ~/.bashrc
```

💡 **Note sur les bonnes pratiques :** Si votre système est configuré pour séparer les fichiers (ce qui est plus propre), il est fortement recommandé de mettre vos alias et fonctions directement dans le fichier dédié aux raccourcis. Dans ce cas, ouvrez plutôt ce fichier :

```bash
nano ~/.bash_aliases
```

*Si vous choisissez d'utiliser `~/.bash_aliases`, n'oubliez pas d'adapter la commande de l'étape 6 en exécutant `source ~/.bash_aliases` pour recharger vos modifications.*

Déplacez-vous tout en bas du fichier ouvert à l'aide des flèches de votre clavier pour vous installer sur une nouvelle ligne vide.

---

## Étape 3 : définir la variable du fichier de log

Pour que nos futurs raccourcis fonctionnent, il faut indiquer au système où se trouve votre fichier de log Nginx actuel.

Copiez et collez cette ligne tout en bas de votre fichier `~/.bashrc` (ou `~/.bash_aliases`) :

```bash
# Chemin vers le fichier de log Nginx à surveiller
LOG="/var/log/nginx/access.log"
```

**Note importante :** Si vous utilisez **Nginx Proxy Manager** sous Docker, ce chemin doit pointer vers le fichier de log global de votre conteneur sur votre machine hôte (par exemple : `/root/npm/data/logs_geo/global_access_geo.log`). Modifiez le texte entre les guillemets si nécessaire pour l'adapter à votre installation.

---

## Étape 4 : créer la fonction de coloration (à personnaliser)

Toujours à la suite dans votre fichier, copiez et collez le bloc suivant. C'est cette fonction qui va intercepter le texte et lui appliquer de la couleur en direct :

```bash
# Fonction interne pour colorer l'affichage des logs
color_live() {
    sed -u \
        -e 's/\[Client [^]]*\]/\x1b[38;5;45m&\x1b[0m/g' \
        -e 's/\[yes [^]]*\]/\x1b[38;5;46m&\x1b[0m/g' \
        -e 's/\[no [^]]*\]/\x1b[38;5;196m&\x1b[0m/g' \
        -e 's/\[[a-zA-Z0-9._-]*\.votre-domaine\.com\]/\x1b[38;5;220m&\x1b[0m/g' \
        -e 's/"\([[:space:]][0-9]\{3\}[[:space:]]\)/"\x1b[38;5;201m\1\x1b[0m/g'
}
```

#### Comment adapter ce bloc à vos propres logs ?

Chaque ligne commençant par `-e` cherche un texte précis pour lui attribuer une couleur. Vous pouvez modifier ce qui se trouve entre les premiers slashs `/.../` pour y mettre vos propres mots-clés.

⚠️ **Important :** pensez à remplacer `votre-domaine.com` par votre propre nom de domaine dans la règle de colorisation du domaine cible.

Par exemple, si vous voulez afficher le mot-clé `POST` en orange, vous pouvez ajouter cette ligne dans le bloc :

`-e 's/POST/\x1b[38;5;208m&\x1b[0m/g'`

#### Légende des colorisations

| Couleur | Élément | Exemple |
|---|---|---|
| 🔵 Cyan électrique | IP client | `[Client 91.98.153.11]` |
| 🟢 Vert pur | Pays autorisé (GeoIP) | `[yes DE]` |
| 🔴 Rouge vif | Pays bloqué (GeoIP) | `[no CN]` |
| 🟡 Jaune doré | Domaine cible | `[mastodon.votre-domaine.com]` |
| 🟣 Magenta | Code HTTP | `200` `403` `202` |

---

## Étape 5 : créer vos raccourcis (alias)

Enfin, ajoutez ces deux lignes tout à la fin de votre fichier pour créer des commandes simples (alias) :

```bash
# Commande pour voir tous les logs en direct
alias live='tail -f "$LOG" | tee /root/npm/sorties_live/sortie_live.txt | color_live'

# Commande pour voir les logs en excluant vos propres réseaux ou IP (Exemple avec 192.168.1. et 172.)
alias live-ext='tail -f "$LOG" | grep --line-buffered -v -E "(192\.168\.1\.|172\.)" | tee /root/npm/sorties_live/sortie_live_ext.txt | color_live'
```

#### Sauvegarder et quitter :

1. Appuyez sur **`Ctrl + O`** puis sur **`Entrée`** pour sauvegarder le fichier.
2. Appuyez sur **`Ctrl + X`** pour quitter l'éditeur `nano`.

---

## Étape 6 : activer les modifications

Pour que votre terminal prenne en compte vos nouvelles commandes immédiatement sans avoir à vous déconnecter, exécutez la commande suivante (adaptez le nom du fichier si vous avez utilisé `~/.bash_aliases`) :

```bash
source ~/.bashrc
```

---

## 🚀 Utilisation

C'est prêt ! Il vous suffit maintenant de taper l'une de ces deux commandes dans votre terminal :

* **`live`** : Affiche la totalité de vos logs Nginx en direct et en couleur.
* **`live-ext`** : Affiche vos logs en masquant automatiquement les adresses IP que vous avez décidé de filtrer (comme votre propre connexion).

Pour arrêter l'affichage en direct à tout moment, appuyez simplement sur les touches **`Ctrl + C`**.

#### 📁 Accès aux fichiers de logs enregistrés

En plus de l'affichage en direct à l'écran, le système écrit et centralise automatiquement tout ce que vous voyez dans des fichiers textes grâce à la commande `tee`.

Même après avoir coupé le direct, vous pouvez retrouver l'historique complet de vos sessions à ces emplacements :

* Pour le flux total : `/root/npm/sorties_live/sortie_live.txt`
* Pour le flux externe filtré : `/root/npm/sorties_live/sortie_live_ext.txt`

Vous pouvez ainsi ouvrir, copier ou analyser ces fichiers texte calmement plus tard avec votre éditeur préféré.

---

## 🎨 Bonus : guide des codes couleur pour aller plus loin

Dans la fonction `color_live` de l'étape 4, toutes les couleurs utilisent le **mode 256 couleurs** avec la syntaxe `\x1b[38;5;NUMÉROm`. Remplacez `NUMÉRO` par un chiffre entre 0 et 255 pour obtenir la teinte souhaitée. Voici quelques valeurs utiles :

* **45** : Cyan électrique
* **46** : Vert pur
* **196** : Rouge vif
* **201** : Magenta
* **208** : Orange
* **220** : Jaune doré

Pour trouver votre couleur idéale, vous pouvez consulter un tableau complet des 256 couleurs ANSI disponible facilement en ligne.

#### Règle d'or pour bien choisir ses couleurs

Pour que la colorisation soit efficace, choisissez des teintes **spectralement distantes** les unes des autres : cyan, vert, rouge, jaune et magenta ne peuvent pas être confondus entre eux, même en vision périphérique. Évitez de placer deux couleurs chaudes ou deux couleurs froides côte à côte dans vos logs.

⚠️ **N'oubliez jamais** de laisser le code `\x1b[0m` à la fin de chaque motif. C'est lui qui indique au terminal d'arrêter la coloration, sinon tout le reste de la ligne prendra la même teinte !

---

> **Auteur** : ce guide est proposé par **Amaury aka BlablaLinux**. Retrouvez l'ensemble de mes services sur [blablalinux.be/mes-services-publics/](https://blablalinux.be/mes-services-publics/).