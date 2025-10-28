---
title: Mise à jour automatique des serveurs Ubuntu par Cron
description: Ce guide explique comment automatiser la mise à jour des paquets de votre serveur Ubuntu (APT et Snap) en utilisant un script Bash planifié via une tâche Cron.
published: true
date: 2025-10-28T11:46:10.536Z
tags: serveur, cron, crontab, ubuntu, script, update, apt
editor: markdown
dateCreated: 2025-10-28T01:52:12.614Z
---

## ⚠️ Avertissement important

L'automatisation des mises à jour système (surtout celles du noyau) comporte toujours un risque. **Ce script n'inclut pas de redémarrage automatique.**

  * **Action requise :** Après l'exécution du script, **vérifiez toujours** le fichier journal (`/var/log/ubuntu_update.log`). Si un nouveau noyau (kernel) a été installé, vous devez **redémarrer manuellement** le serveur pour que les mises à jour prennent effet.
  * **Privilèges :** Les commandes dans le script utilisent `sudo`. La tâche Cron doit donc être configurée dans la **crontab de l'utilisateur `root`** pour éviter les problèmes d'authentification par mot de passe.

-----

## I. Création du script Bash

Nous allons créer un script exécutable dans le répertoire standard pour les commandes locales de l'administrateur, soit `/usr/local/bin/`.

### Étape 1 : Créer le fichier de script `update_ubuntu.sh`

1.  Connectez-vous à votre serveur Ubuntu en SSH et utilisez `nano` (ou `vi`) pour créer le fichier.

    ```bash
    sudo nano /usr/local/bin/update_ubuntu.sh
    ```

2.  Collez le contenu suivant (utilisant **`apt-get`** pour une meilleure stabilité dans les scripts) :

    ```bash
    #!/bin/bash

    # Fichier journal pour enregistrer le déroulement de la mise à jour
    LOGFILE="/var/log/ubuntu_update.log"

    # Redirection de toute la sortie (standard et erreur) vers le fichier journal
    exec 1>>$LOGFILE 2>&1

    # --- Début du processus ---
    echo "======================================================"
    echo "Début de la mise à jour Ubuntu Server : $(date)"
    echo "======================================================"

    # 1. Mise à jour de la liste des paquets APT (apt-get update)
    echo "--- Étape 1 : apt-get update (Mise à jour des listes de paquets) ---"
    sudo apt-get update

    if [ $? -ne 0 ]; then
        echo "Échec de la mise à jour des listes de paquets. Arrêt du script."
        exit 1
    fi

    # 2. Mise à niveau des paquets installés (apt-get upgrade)
    echo "--- Étape 2 : apt-get upgrade (Mise à niveau des paquets) ---"
    # Utilisation de -y pour la réponse automatique "oui"
    sudo apt-get upgrade -y

    # 3. Suppression des dépendances inutiles (apt-get autoremove)
    echo "--- Étape 3 : apt-get autoremove (Nettoyage des dépendances et anciens noyaux) ---"
    sudo apt-get autoremove -y

    # 4. Mise à jour des paquets Snap (spécifique à Ubuntu)
    echo "--- Étape 4 : snap refresh (Mise à jour des packages Snap) ---"
    sudo snap refresh

    echo "======================================================"
    echo "Fin de la mise à jour Ubuntu Server : $(date)"
    echo "======================================================"

    exit 0
    ```

### Étape 2 : rendre le script exécutable

```bash
sudo chmod +x /usr/local/bin/update_ubuntu.sh
```

-----

## II. Configuration de la tâche Cron (Recommandé : utilisateur root)

Pour exécuter des commandes nécessitant `sudo` sans interaction, nous utilisons le `crontab` de l'utilisateur `root`.

### Étape 1 : Ouvrir le crontab de l'utilisateur root

```bash
sudo crontab -e
```

### Étape 2 : Ajouter la ligne de planification

Ajoutez la ligne suivante à la fin du fichier. Cet exemple planifie l'exécution du script tous les **dimanches à 3h30 du matin**.

```cron
# Mettre à jour Ubuntu Server tous les dimanches à 3h30 du matin
30 3 * * 0 /usr/local/bin/update_ubuntu.sh
```

| Champ | Valeur | Description |
| :--- | :--- | :--- |
| **Minute** | `30` | 30e minute |
| **Heure** | `3` | 3h du matin |
| **Jour de la semaine** | `0` ou `7` | Dimanche (0 et 7 sont des alias pour le dimanche) |

### Étape 3 : Enregistrer et quitter

-----

## III. Vérification (Post-exécution)

Après l'heure planifiée, vous devez vérifier que le script s'est bien exécuté et s'il nécessite un redémarrage.

### 1\. Consulter le fichier journal

```bash
tail -f /var/log/ubuntu_update.log
```

### 2\. Vérifier la nécessité d'un redémarrage

Si un nouveau noyau a été installé, le système l'indiquera. Utilisez cette commande pour vérifier si un redémarrage est nécessaire :

```bash
apt-get install -s | grep "reboot is required"
```

*(Note : Si vous voyez une sortie, un redémarrage est nécessaire.)*

### 3\. Redémarrer le serveur si nécessaire

```bash
sudo reboot
```