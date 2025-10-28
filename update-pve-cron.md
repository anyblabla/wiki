---
title: Mise à jour automatique des hôtes Proxmox VE par Cron
description: Ce guide explique comment automatiser la mise à jour des paquets de vos hôtes Proxmox VE (PVE) en utilisant un script Bash planifié via une tâche Cron. Ces commandes sont destinées à être exécutées en tant qu'utilisateur root.
published: true
date: 2025-10-28T11:44:13.224Z
tags: proxmox, cron, crontab, pve, update, apt
editor: markdown
dateCreated: 2025-10-26T21:16:13.278Z
---

## ⚠️ Avertissement important

L'automatisation des mises à jour système (surtout celles du noyau) comporte toujours un risque. **Ce script n'inclut pas de redémarrage automatique.**

  * **Action requise :** Après l'exécution du script, **vérifiez toujours** le fichier journal (`/var/log/proxmox_update.log`). Si un nouveau noyau a été installé, vous devez **redémarrer manuellement** l'hôte pour appliquer les mises à jour du noyau et garantir la stabilité du système.
  * **Fréquence :** Il est recommandé de planifier cette tâche pendant les heures creuses (la nuit, le week-end).

-----

## I. Création du script Bash

Nous allons créer un script exécutable dans le répertoire standard pour les commandes locales de l'administrateur : `/usr/local/bin/`.

### Étape 1 : Créer le fichier de script `update_pve.sh`

1.  Connectez-vous à votre hôte Proxmox en SSH (en tant que `root`) et utilisez `nano` (ou `vi`) pour créer le fichier.

    ```bash
    nano /usr/local/bin/update_pve.sh
    ```

2.  Collez le contenu suivant (utilisant **`apt-get` et `dist-upgrade`** pour une meilleure stabilité dans les scripts) :

    ```bash
    #!/bin/bash

    # Fichier journal pour enregistrer le déroulement de la mise à jour
    LOGFILE="/var/log/proxmox_update.log"

    # Redirection de toute la sortie (standard et erreur) vers le fichier journal
    exec 1>>$LOGFILE 2>&1

    # --- Début du processus ---
    echo "======================================================"
    echo "Début de la mise à jour Proxmox VE : $(date)"
    echo "======================================================"

    # 1. Mise à jour de la liste des paquets (apt-get update)
    echo "--- Étape 1 : apt-get update (Mise à jour des listes de paquets) ---"
    apt-get update

    if [ $? -ne 0 ]; then
        echo "Échec de la mise à jour des listes de paquets. Arrêt du script."
        exit 1
    fi

    # 2. Mise à niveau des paquets installés (apt-get dist-upgrade)
    echo "--- Étape 2 : apt-get dist-upgrade (Mise à niveau des paquets) ---"
    # dist-upgrade est la méthode recommandée pour les mises à jour d'hôtes Debian/Proxmox
    apt-get dist-upgrade -y

    # 3. Suppression des dépendances inutiles (apt-get autoremove)
    echo "--- Étape 3 : apt-get autoremove (Nettoyage des dépendances et anciens noyaux) ---"
    apt-get autoremove -y

    # 4. Nettoyage du cache APT
    echo "--- Étape 4 : apt-get clean (Nettoyage du cache APT) ---"
    apt-get clean

    echo "======================================================"
    echo "Fin de la mise à jour Proxmox VE : $(date)"
    echo "======================================================"

    exit 0
    ```

### Étape 2 : rendre le script exécutable

```bash
chmod +x /usr/local/bin/update_pve.sh
```

-----

## II. Configuration de la tâche Cron

Nous allons ajouter une entrée au `crontab` de l'utilisateur `root` pour planifier l'exécution du script.

### Étape 1 : Ouvrir le crontab de l'utilisateur root

```bash
crontab -e
```

### Étape 2 : Ajouter la ligne de planification

Ajoutez la ligne suivante à la fin du fichier. Cet exemple planifie l'exécution du script tous les **dimanches à 3h30 du matin**.

```cron
# Mettre à jour Proxmox VE tous les dimanches à 3h30 du matin
30 3 * * 0 /usr/local/bin/update_pve.sh
```

| Champ | Valeur | Description |
| :--- | :--- | :--- |
| **Minute** | `30` | 30e minute |
| **Heure** | `3` | 3h du matin |
| **Jour de la semaine** | `0` ou `7` | Dimanche (0 et 7 sont des alias pour le dimanche) |

### Étape 3 : Enregistrer et quitter

-----

## III. Vérification (Post-exécution)

Après l'heure planifiée, vous devez vérifier que le script s'est bien exécuté et, surtout, s'il nécessite un redémarrage.

### 1\. Consulter le fichier journal

```bash
tail -f /var/log/proxmox_update.log
```

### 2\. Vérifier la nécessité d'un redémarrage

Si le journal mentionne l'installation de paquets **`pve-kernel-*`**, ou si la commande suivante retourne des informations, un redémarrage est nécessaire :

```bash
apt-get install -s | grep "reboot is required"
```

### 3\. Redémarrer l'hôte si nécessaire

Si un nouveau noyau a été installé, redémarrez l'hôte Proxmox via l'interface Web ou en SSH :

```bash
reboot
```