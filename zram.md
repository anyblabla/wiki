---
title: Zram - Compresser la RAM au lieu de swapper sur Linux
description: Zram est un module du noyau Linux qui crée un périphérique de stockage compressé en RAM.
published: true
date: 2025-12-05T00:32:01.559Z
tags: ram, zram, memory
editor: markdown
dateCreated: 2025-11-01T00:19:53.610Z
---

Au lieu d'écrire directement sur le disque dur (le *swap* traditionnel) lorsque la mémoire vive est pleine, **Zram** intercepte les pages mémoire et les compresse, réduisant ainsi la quantité de données échangées vers le disque. Cela améliore considérablement la réactivité du système, en particulier sur les machines avec une faible quantité de RAM ou des disques lents (comme les cartes SD ou les disques eMMC).

-----

### 📰 Pour aller plus loin : Le Contexte de l'Activation

> **Vous vous demandez pourquoi Zram n'est pas actif par défaut sous Linux ?**
>
> Avant de l'installer, comprenez pourquoi cette optimisation est cruciale, notamment dans le cadre du reconditionnement de matériel, en lisant notre article de blog détaillé :
>
> ➡️ **[Zram : Pourquoi ce "turbo" pour la RAM n'est pas activé par défaut sous Linux ?](https://blablalinux.be/2025/12/05/zram-pourquoi-pas-actif/)**

-----

## 1\. Fonctionnement de Zram

Le principe de Zram est de transformer une partie de la RAM en un périphérique de *swap* compressé.

1.  **Création du périphérique :** Zram crée un ou plusieurs périphériques virtuels (`/dev/zramX`).
2.  **Compression :** Lorsqu'une page mémoire est destinée au *swap*, elle est compressée par le processeur.
3.  **Stockage en RAM :** La page compressée est ensuite stockée dans une zone de la mémoire vive gérée par Zram.

**Avantages :**

  * **Vitesse :** L'accès à la RAM, même compressée, est beaucoup plus rapide que l'accès au disque.
  * **Usure réduite :** Diminue les écritures sur les périphériques de stockage, prolongeant la durée de vie des SSD/cartes Flash.
  * **Capacité effective :** Permet d'augmenter la quantité de mémoire utilisable par le système (en fonction du taux de compression).
  * **Note Reconditionnement :** C'est pour ces raisons que des distributions comme **Emmabuntüs** l'activent par défaut, rendant le matériel reconditionné plus réactif \!

**Inconvénient :**

  * **Charge CPU :** Le processus de compression/décompression utilise le processeur, mais l'impact est généralement négligeable par rapport au gain de performance.

-----

## 2\. Installation et activation

Le module Zram est inclus dans le noyau Linux. Pour l'activer de manière persistante et automatique, il est recommandé d'utiliser le paquet d'utilitaires **`zram-tools`** ou **`zramswap-init`** (selon la distribution).

### A. Installation sur Debian/Ubuntu

Installez le paquet qui gère l'activation automatique de Zram :

```bash
sudo apt update
sudo apt install zram-tools -y
```

### B. Configuration de la taille (optionnel)

Le paquet `zram-tools` configure par défaut la taille du *swap* Zram pour être une fraction de la RAM totale (souvent 50%).

Vous pouvez modifier cette configuration dans le fichier `/etc/default/zramswap` ou `/etc/default/zram-tools` (le nom exact dépend de la version de l'outil).

Par exemple, pour définir la taille à 100% de la RAM disponible (ou une valeur fixe) :

```bash
# Éditer le fichier de configuration (chemin à ajuster si besoin)
sudo nano /etc/default/zramswap

# Définir la taille (exemples) :
# - 100% de la RAM :
# PERCENT=100

# - Taille fixe de 2 Go :
# SIZE=2048
```

> Si vous définissez la taille en pourcentage (`PERCENT`), l'outil calculera la taille du périphérique `zram` à partir de la mémoire vive totale.

### C. Redémarrage du service

Après l'installation ou la modification de la configuration, activez ou redémarrez le service :

```bash
sudo systemctl enable zramswap
sudo systemctl start zramswap
```

-----

## 3\. Vérification de l'activation

Pour vérifier que Zram est actif et connaître sa taille, utilisez la commande `swapon` :

```bash
sudo swapon --show
```

Le résultat affichera le périphérique Zram (ex: `/dev/zram0`) avec son type `partition` et sa taille :

| Nom | Type | Taille | Utilisé | Priorité |
| :--- | :--- | :--- | :--- | :--- |
| `/dev/zram0` | partition | 1,8G | 0B | 100 |
| `/dev/sda3` | partition | 8G | 0B | -2 |

> **Remarque :** Zram doit avoir une **priorité** plus élevée (nombre positif, ex: `100`) que votre *swap* sur disque (nombre négatif, ex: `-2`) pour être utilisé en premier.

Vous pouvez également vérifier l'état détaillé de Zram pour le périphérique `/dev/zram0` :

```bash
cat /sys/block/zram0/mm_stat
```

Les valeurs indiquent notamment la quantité de données écrites (`compr_data_size`) et la taille de la mémoire occupée par les données compressées.

-----

## 4\. Désactivation (si nécessaire)

Si vous souhaitez désactiver Zram, exécutez les commandes suivantes pour arrêter le service et le désactiver au démarrage :

```bash
# Arrêter le service (cela désactive le périphérique swap zramX)
sudo systemctl stop zramswap

# Désactiver le service au démarrage
sudo systemctl disable zramswap

# Désinstaller les outils (optionnel)
sudo apt purge zram-tools
```