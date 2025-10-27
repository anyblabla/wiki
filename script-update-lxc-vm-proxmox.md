---
title: Automatisation des mises à jour Proxmox (LXC et VM)
description: Ces scripts Bash permettent d'automatiser la mise à jour des conteneurs LXC et des machines virtuelles (VM) basées sur Debian/Ubuntu sur votre hôte Proxmox VE, en utilisant la planification Cron.
published: true
date: 2025-10-27T10:30:33.318Z
tags: lxc, proxmox, cron, crontab, script, vm
editor: markdown
dateCreated: 2025-10-26T16:38:37.191Z
---

## 🎯 Utilité et Public Cible

| Section | Explication | Pour qui ? |
| :--- | :--- | :--- |
| **Objectif** | Maintien automatique de la sécurité et de la stabilité de vos systèmes d'exploitation (OS) invités sans intervention manuelle régulière. | Tous les utilisateurs. |
| **Sécurité** | Assure l'application rapide des correctifs de sécurité critiques et des mises à jour du noyau. | Essentiel pour les environnements de production ou exposés à Internet. |
| **Efficacité** | Libère du temps en centralisant la mise à jour de dizaines de conteneurs/VMs en une seule tâche planifiée. | Les administrateurs avec un grand nombre d'invités (homelab ou professionnel). |
| **Redémarrage** | Gère le redémarrage des VMs lorsque nécessaire (après une mise à jour du noyau), assurant l'application complète des correctifs sans risque de conflit. | Les utilisateurs qui souhaitent automatiser la gestion des dépendances du noyau. |

-----

## I. Script pour les Conteneurs LXC (Légers)

Ce script utilise la commande Proxmox **`pct exec`** pour exécuter les mises à jour à l'intérieur de chaque conteneur en cours d'exécution.

### 1\. Création du Script `update_lxcs.sh`

Connectez-vous à votre hôte Proxmox en tant que `root` et créez le fichier :

```bash
mkdir -p /root/scripts
nano /root/scripts/update_lxcs.sh
```

Collez le code suivant :

```
#!/bin/bash
#
# Script de mise à jour automatique des conteneurs LXC en cours d'exécution
# A exécuter sur l'hôte Proxmox VE
#

LOGFILE="/var/log/update_lxcs_cron.log"
# Liste des CTID à exclure (séparés par des espaces, laissez vide si aucune)
EXCLUDED_CTIDS="" 

echo "==================================================" >> $LOGFILE 2>&1
echo "Démarrage de la mise à jour des LXC le $(date)" >> $LOGFILE 2>&1
echo "==================================================" >> $LOGFILE 2>&1

# Boucle sur les conteneurs démarrés (status 'running')
# CORRIGÉ : Utilisation de /usr/sbin/pct list
for CTID in $(/usr/sbin/pct list | grep running | awk '{print $1}')
do
    # Vérifie si le CTID est dans la liste d'exclusion
    if [[ " $EXCLUDED_CTIDS " =~ " $CTID " ]]; then
        echo "    [SKIP] Conteneur $CTID exclu de la mise à jour." >> $LOGFILE 2>&1
        continue
    fi
    
    echo "--> Traitement du conteneur CTID $CTID..." >> $LOGFILE 2>&1
    
    # Exécute les commandes à l'intérieur du conteneur (apt full-upgrade pour gérer les dépendances)
    # CORRIGÉ : Utilisation de /usr/sbin/pct exec
    /usr/sbin/pct exec $CTID -- bash -c "apt update && apt full-upgrade -y && apt clean" >> $LOGFILE 2>&1
    
    if [ $? -eq 0 ]; then
        echo "    [OK] Conteneur $CTID mis à jour avec succès." >> $LOGFILE 2>&1
    else
        echo "    [ERREUR] La mise à jour du conteneur $CTID a échoué (voir log pour détails)." >> $LOGFILE 2>&1
    fi
done

echo "==================================================" >> $LOGFILE 2>&1
echo "Fin de la mise à jour des LXC le $(date)" >> $LOGFILE 2>&1
echo "==================================================" >> $LOGFILE 2>&1
```

### 2\. Planification Cron

Rendez le script exécutable et ajoutez la tâche Cron :

```bash
chmod +x /root/scripts/update_lxcs.sh
crontab -e
```

Ajoutez la ligne suivante (exemple : tous les dimanches à 03h00) :

```cron
# M H J M A Commande
0 3 * * 0 /root/scripts/update_lxcs.sh
```

-----

## II. Script pour les Machines Virtuelles (VM)

Ce script utilise le **QEMU Guest Agent** (via `qm guest exec`) pour communiquer avec l'OS invité, ce qui est **obligatoire** pour cette méthode.

### 1\. Prérequis : QEMU Guest Agent

Installez et activez l'agent dans **chaque VM Debian/Ubuntu** :

```bash
# À exécuter à l'intérieur de la VM (sudo est souvent nécessaire)
sudo apt update
sudo apt install qemu-guest-agent -y
sudo systemctl enable --now qemu-guest-agent
```

Puis, vérifiez que l'agent est **Enabled** dans les **Options** de la VM dans l'interface web de Proxmox.

### 2\. Création du Script `update_vms.sh`

Connectez-vous à l'hôte Proxmox et créez le fichier :

```bash
nano /root/scripts/update_vms.sh
```

Collez le code suivant :

```
#!/bin/bash
#
# SCRIPT : update_vms.sh
# OBJECTIF : Mettre à jour toutes les VMs Debian/Ubuntu en cours d'exécution
#            et les redémarrer si une mise à jour du noyau est détectée.
# CORRECTION : Utilisation des chemins absolus pour 'qm' pour la compatibilité Cron.
#
# ==============================================================================

# Fichier de log : tous les résultats et erreurs seront enregistrés ici
LOGFILE="/var/log/update_vms_cron.log"

# Liste des VMS à exclure (séparées par des espaces). Laissez vide si aucune.
EXCLUDED_VMS="" 

# Redirection de toutes les sorties (stdout et stderr) vers le fichier de log
exec 1>>$LOGFILE 2>&1

# Début de l'exécution
echo "=================================================="
echo "Démarrage de la mise à jour des VMs le $(date)"
echo "=================================================="

# Commandes internes pour la VM
UPDATE_COMMAND="export DEBIAN_FRONTEND=noninteractive && apt update -y && apt full-upgrade -y && apt clean"
REBOOT_CHECK_COMMAND="if [ -f /var/run/reboot-required ]; then echo 'REBOOT_YES'; else echo 'REBOOT_NO'; fi"

# Boucle sur les IDs de toutes les VMs en cours d'exécution
# CORRIGÉ : Utilisation de /usr/sbin/qm list
for VMID in $(/usr/sbin/qm list | grep running | awk '{print $1}')
do
    # ----------------------------------------------------
    # 1. Vérification des exclusions
    # ----------------------------------------------------
    if [[ " $EXCLUDED_VMS " =~ " $VMID " ]]; then
        echo "    [SKIP] VM $VMID exclue de la mise à jour."
        continue 
    fi
    
    echo "--> Traitement de la VM VMID $VMID..."
    
    # ----------------------------------------------------
    # 2. Exécution des mises à jour (Syntaxe corrigée)
    # ----------------------------------------------------
    echo "    - Exécution des mises à jour..."
    # CORRIGÉ : Utilisation de /usr/sbin/qm guest exec
    /usr/sbin/qm guest exec $VMID --timeout 300 /bin/bash -- -c "$UPDATE_COMMAND"
    
    # ----------------------------------------------------
    # 3. Vérification du besoin de redémarrage (Syntaxe corrigée)
    # ----------------------------------------------------
    
    echo "    - Vérification du besoin de redémarrage..."
    # CORRIGÉ : Utilisation de /usr/sbin/qm guest exec
    REBOOT_CHECK_OUTPUT=$(/usr/sbin/qm guest exec $VMID --timeout 60 /bin/bash -- -c "$REBOOT_CHECK_COMMAND")
    
    # Analyse de la sortie
    if [[ $REBOOT_CHECK_OUTPUT == *"REBOOT_YES"* ]]; then
        
        echo "    [ALERTE] Redémarrage nécessaire pour la VM $VMID. Redémarrage en cours..."
        
        # ----------------------------------------------------
        # 4. Redémarrage sécurisé (COMMANDES D'ARRÊT CORRIGÉES)
        # ----------------------------------------------------
        
        # 4a. Tentative d'arrêt gracieux via 'qm shutdown'
        echo "    - Arrêt de la VM $VMID (shutdown)..."
        # CORRIGÉ : Utilisation de /usr/sbin/qm shutdown
        /usr/sbin/qm shutdown $VMID --timeout 120 
        
        # 4b. Vérification si la VM s'est bien arrêtée (si non, on force l'arrêt)
        # CORRIGÉ : Utilisation de /usr/sbin/qm status
        if /usr/sbin/qm status $VMID | grep -q running; then
            echo "    - Arrêt gracieux échoué. Forçage de l'arrêt (stop)..."
            # CORRIGÉ : Utilisation de /usr/sbin/qm stop
            /usr/sbin/qm stop $VMID
            sleep 5
        fi
        
        # 4c. Démarrage de la VM
        echo "    - Démarrage de la VM $VMID..."
        # CORRIGÉ : Utilisation de /usr/sbin/qm start
        /usr/sbin/qm start $VMID
        
        echo "    [OK] Redémarrage de la VM $VMID terminé."
        
    else
        echo "    [OK] Aucune mise à jour critique nécessitant un redémarrage pour la VM $VMID."
    fi
    
done

echo "=================================================="
echo "Fin de la mise à jour des VMs le $(date)"
echo "=================================================="

# Rétablir la sortie standard
exec 1>&- 2>&-
```

### 3\. Planification Cron

Rendez le script exécutable et ajoutez la tâche Cron :

```bash
chmod +x /root/scripts/update_vms.sh
crontab -e
```

Ajoutez la ligne suivante (exemple : tous les dimanches à 04h00, une heure après les LXC) :

```cron
# M H J M A Commande
0 4 * * 0 /root/scripts/update_vms.sh
```

-----

## III. Adaptations et Personnalisations

### A. Exclure des Conteneurs/VMs

Si vous avez des invités (LXC ou VM) très sensibles qui nécessitent une mise à jour manuelle (par exemple, un contrôleur de domaine ou un système de base de données critique), vous pouvez les exclure :

  * **Action :** Modifiez la variable **`EXCLUDED_CTIDS`** ou **`EXCLUDED_VMS`** au début du script correspondant.
  * **Exemple :**
    ```bash
    EXCLUDED_VMS="101 105 133" # VMIDs 101, 105 et 133 seront ignorées.
    ```

### B. Changer l'Heure d'Exécution

Il est recommandé d'exécuter ces scripts pendant les heures de faible activité (souvent la nuit ou tôt le matin le week-end).

  * **Action :** Modifiez les deux premiers chiffres de la ligne Cron (`M H`).
  * **Exemple (tous les jours à 02h30) :**
    ```cron
    30 2 * * * /root/scripts/update_lxcs.sh
    ```

### C. Ajout d'un Nettoyage Automatique (`autoremove`)

Vous pouvez ajouter la commande `apt autoremove -y` après `apt full-upgrade -y` pour supprimer les dépendances inutiles et nettoyer l'espace disque.

  * **Action :** Modifiez la variable **`UPDATE_COMMAND`** (dans les scripts LXC et VM) :
    ```bash
    # Nouvelle version pour les VMs
    UPDATE_COMMAND="export DEBIAN_FRONTEND=noninteractive && apt update -y && apt full-upgrade -y && apt autoremove -y && apt clean"
    ```

### D. Cluster : Exécution sur Tous les Nœuds (Avancé)

Si vous avez un cluster Proxmox (multi-nœuds), ces scripts ne s'exécutent que sur le nœud où la tâche cron est définie.

  * **Action :** Pour mettre à jour tous les invités, vous devez définir la tâche cron **sur chaque nœud** de votre cluster.
  * **Alternative :** Exécuter le script sur un seul nœud et utiliser SSH pour se connecter aux autres nœuds et y exécuter le même script. Cependant, cette méthode complexifie la gestion des logs et est déconseillée pour les débutants.