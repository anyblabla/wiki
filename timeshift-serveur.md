---
title: Timeshift sur Serveur (CLI) - Configuration et Automatisation
description: Timeshift Serveur (CLI) : Guide pour installer et utiliser Timeshift sur Ubuntu Server. Concentré sur la configuration des instantanés automatiques via ligne de commande et le fichier JSON, puis l'activation par Cron.
published: true
date: 2025-11-21T22:10:27.010Z
tags: timeshift, sauvegarde, serveur, cron
editor: markdown
dateCreated: 2024-06-05T18:52:33.986Z
---

> **Problématique :** La plupart des tutoriels sont incomplets et n'abordent pas l'édition et l'initialisation du fichier **JSON** essentiel pour l'automatisation.

-----

## 1\. Installation et Premier Instantané Manuelle

Les commandes ci-dessous sont à exécuter avec `sudo`.

1.  **Mettre à jour et installer Timeshift :**

    ```bash
    sudo apt update
    sudo apt install timeshift -y
    ```

2.  **Identifier le périphérique cible :**
    Avant de créer le premier instantané, identifiez le système de fichiers ou le périphérique de destination (où les sauvegardes seront stockées).

    ```bash
    df -h
    ```
Exemple de sortie…

![](/timeshift-serveur/df-h.png)
    

3.  **Créer le premier instantané :**
    L'exécution de la première commande `timeshift --create` est nécessaire pour que le fichier de configuration `timeshift.json` soit généré.

    ```bash
    sudo timeshift --create --comments "First Snapshot" --snapshot-device /dev/sda1
    ```

    *(Remplacez `/dev/sda1` par votre périphérique de destination)*
    
Exemple de sortie…

![](/timeshift-serveur/timeshift-create.png)

-----

## 2\. Configuration de l'Automatisation (Fichier JSON)

Après la création du premier instantané, le fichier de configuration est généré. Vous devez maintenant l'éditer pour définir la planification des tâches Cron.

1.  **Éditer le fichier de configuration :**

    ```bash
    sudo nano /etc/timeshift/timeshift.json
    ```
    
Mon fichier…

![](/timeshift-serveur/timeshift.json.png)


2.  **Personnalisation des lignes clés :**
    Modifiez les valeurs (`"true"`/`"false"` pour activer/désactiver) et les nombres (pour la rétention) selon vos besoins :

    | Paramètre | Exemple / Valeur | Description |
    | :--- | :--- | :--- |
    | **`backup_device_uuid`** | `"d00a20aa-cdd1-486d-9e6f-e3f87b3b6aff"` | **UUID** du disque qui accueille les sauvegardes. Utilisez `lsblk -fs` pour l'obtenir si nécessaire. |
    | **`stop_cron_emails`** | `"true"` | Arrête l'envoi d'emails par Cron pour les tâches terminées. |
    | **`schedule_monthly`** | `"true"` | Active la planification mensuelle. |
    | **`count_monthly`** | `"1"` | Nombre de sauvegardes mensuelles à conserver. |
    | **`schedule_daily`** | `"true"` | Active la planification quotidienne. |
    | **`count_daily`** | `"3"` | Nombre de sauvegardes quotidiennes à conserver. |
    | **`schedule_boot`** | `"true"` | Active la planification au démarrage. |

3.  **Sauvegarder** le fichier JSON (`CTRL+X`, `O`, `ENTER`).

-----

## 3\. Activation des Tâches Cron

C'est l'étape la plus importante : vous forcez Timeshift à vérifier le fichier JSON et à créer les liens symboliques vers les fichiers Cron (`cron.d`, `cron.daily`, etc.) pour activer la planification.

1.  **Vérifier et initialiser les tâches automatiques :**

    ```bash
    sudo timeshift --check
    ```

    Cette commande vérifie la validité du fichier `timeshift.json` et configure les scripts Cron nécessaires.
    
Exemple de sortie…

![](/timeshift-serveur/timeshift-check.png)

2.  **Lister les instantanés existants :**

    ```bash
    sudo timeshift --list
    ```
    
Exemple de sortie…

![](/timeshift-serveur/timeshift-list.png)

-----

## 4\. Commandes Supplémentaires (CLI)

| Action | Commande (à adapter) |
| :--- | :--- |
| **Lister sur un périphérique** | `sudo timeshift --list --snapshot-device /dev/sda` |
| **Créer manuellement avec tag** | `sudo timeshift --create --comments "after update" --tags D` |
| **Restaurer (interactive)** | `sudo timeshift --restore` |
| **Restaurer un instantané spécifique** | `sudo timeshift --restore --snapshot '2014-10-12_16-29-08' --target /dev/sda1` |
| **Supprimer un instantané spécifique** | `sudo timeshift --delete --snapshot '2014-10-12_16-29-08'` |
| **Supprimer TOUS les instantanés** | `sudo timeshift --delete-all` |

-----

## ⚠️ Avertissement sur l'Utilisation

**Timeshift n'est pas un outil de sauvegarde de fichiers personnels \!**

Timeshift est conçu pour les **fichiers système** (`/`). Les répertoires utilisateurs ne sont pas pris en compte par défaut.

Si vous décidez d'inclure des répertoires personnels et de restaurer un instantané, les fichiers personnels **reviendront à un état antérieur, voire seront supprimés \!** Utilisez un autre logiciel pour vos sauvegardes de données (fichiers personnels).