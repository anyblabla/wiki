---
title: Automatisation du cycle électrique d'un serveur PBS physique
description: Allumer et éteindre automatiquement un serveur Proxmox Backup Server physique via Wake-on-LAN, SSH et cron, avec surveillance des sauvegardes et notifications Gotify.
published: false
date: 2026-06-02T15:46:58.408Z
tags: proxmox, cron, ssh, bash, pbs, gotify, automatisation, homelab, wakeonlan
editor: markdown
dateCreated: 2026-06-02T15:46:58.408Z
---

## Présentation

Ce guide explique comment allumer et éteindre automatiquement un serveur **Proxmox Backup Server (PBS) physique** selon un calendrier défini, tout en s'assurant qu'aucune sauvegarde n'est interrompue au moment de l'extinction.

Le script gère les cas suivants :

- Allumage à distance via **Wake-on-LAN**
- Extinction immédiate si aucune tâche n'est en cours
- Surveillance en arrière-plan si des sauvegardes tournent encore, avec extinction automatique dès leur fin
- Extinction forcée après 4 heures si les tâches ne se terminent pas
- Notifications à chaque étape via **Gotify**
- Annulation manuelle de la surveillance si nécessaire

---

## Prérequis

### Sur le serveur qui exécute le script (ex. : nœud Proxmox VE)

Les outils suivants doivent être installés :

```bash
apt install -y wakeonlan iputils-ping openssh-client curl
```

### Sur le serveur PBS physique

L'outil `jq` est nécessaire pour analyser les réponses JSON de PBS :

```bash
apt install -y jq
```

### Authentification SSH sans mot de passe

Le script utilise SSH en mode non-interactif (`BatchMode`). Il faut que la clé SSH du nœud source soit autorisée sur le PBS.

Si ce n'est pas encore fait :

```bash
# Générer une clé SSH (si elle n'existe pas)
ssh-keygen -t ed25519 -C "pvenode-to-pbs"

# Copier la clé publique vers le PBS
ssh-copy-id root@<IP_DU_PBS>
```

Vérification :

```bash
ssh -o BatchMode=yes root@<IP_DU_PBS> "echo OK"
```

La commande doit retourner `OK` sans demander de mot de passe.

### Wake-on-LAN activé sur le PBS

Dans le BIOS/UEFI du serveur PBS, activer l'option **Wake-on-LAN** (souvent sous "Power Management" ou "Network Boot").

Vérifier également que la carte réseau le supporte :

```bash
ethtool <interface> | grep Wake-on
```

La valeur `g` doit apparaître dans les options activées.

---

## Installation du script

### Création du fichier

```bash
nano /root/scripts/gestion-pbs-physique.sh
```

Coller le contenu suivant, en adaptant les variables de configuration à votre environnement :

```bash
#!/bin/bash
# Automatisation cycle électrique PBS Physique (Amaury aka BlablaLinux)

# --- CONFIGURATION ---
PBS_IP="192.168.2.206"           # Adresse IP fixe du PBS physique
PBS_MAC="78:e7:d1:80:87:f9"      # Adresse MAC pour le Wake-on-LAN
GOTIFY_URL="https://gotify.exemple.com"  # URL de votre instance Gotify
GOTIFY_TOKEN="votre_token_ici"   # Token d'application Gotify
MAX_RETRIES=8                    # 8 × 30 min = 4 heures max de surveillance
WATCHER_PID_FILE="/var/run/pbs-cycle-watcher.pid"

# --- FONCTION DE NOTIFICATION ---
send_gotify_notification() {
    local title="$1"
    local message="$2"
    local priority="$3"
    curl -s -X POST "$GOTIFY_URL/message?token=$GOTIFY_TOKEN" \
        -F "title=$title" \
        -F "message=$message" \
        -F "priority=$priority" > /dev/null 2>&1
}

# --- FONCTION DE VÉRIFICATION DE JOIGNABILITÉ ---
check_reachable() {
    ping -c 1 -W 2 "$PBS_IP" > /dev/null 2>&1
}

# --- FONCTION DE COMPTAGE DES TÂCHES EN COURS ---
get_running_tasks() {
    ssh -o ConnectTimeout=5 -o BatchMode=yes root@"$PBS_IP" \
        "proxmox-backup-manager task list --output-format json \
         | jq '[.[] | select(.status == \"running\")] | length'" 2>/dev/null
}

case "$1" in
    start)
        /usr/bin/wakeonlan "$PBS_MAC" > /dev/null 2>&1
        send_gotify_notification "⚡ PBS Physique" "Le serveur de sauvegarde physique a été mis sous tension." "5"
        ;;

    stop)
        # Vérification que PBS est joignable
        if ! check_reachable; then
            send_gotify_notification "💤 PBS Physique" "Serveur déjà injoignable (déjà éteint ?)." "4"
            echo "PBS ($PBS_IP) injoignable. Déjà éteint ?"
            exit 0
        fi

        # Vérification initiale des tâches
        TASKS=$(get_running_tasks)

        if [ -z "$TASKS" ]; then
            send_gotify_notification "🚨 PBS Physique" "PBS joignable mais SSH KO. Extinction annulée." "9"
            echo "PBS répond au ping mais SSH échoue. Abandon."
            exit 1
        fi

        if [ "$TASKS" -gt 0 ]; then
            send_gotify_notification "⚠️ PBS Physique" "Sauvegardes en cours ($TASKS). Surveillance activée jusqu'à extinction (max 4h)." "8"

            # Boucle en arrière-plan avec garde-fou
            (
                RETRIES=0
                while [ "$RETRIES" -lt "$MAX_RETRIES" ]; do
                    sleep 1800
                    RETRIES=$((RETRIES + 1))

                    TASKS_REMAINING=$(get_running_tasks)

                    if [ -z "$TASKS_REMAINING" ]; then
                        send_gotify_notification "🚨 PBS Physique" "Erreur SSH lors de la vérification (tentative $RETRIES/$MAX_RETRIES)." "8"
                        continue
                    fi

                    if [ "$TASKS_REMAINING" -eq 0 ]; then
                        ssh -o ConnectTimeout=5 -o BatchMode=yes root@"$PBS_IP" "poweroff" > /dev/null 2>&1
                        send_gotify_notification "💤 PBS Physique" "Sauvegardes terminées, extinction effectuée." "5"
                        rm -f "$WATCHER_PID_FILE"
                        exit 0
                    fi
                done

                # Timeout atteint : extinction forcée
                send_gotify_notification "🚨 PBS Physique" "Timeout 4h atteint, extinction forcée malgré tâches potentiellement actives." "10"
                ssh -o ConnectTimeout=5 -o BatchMode=yes root@"$PBS_IP" "poweroff" > /dev/null 2>&1
                rm -f "$WATCHER_PID_FILE"
            ) &

            echo $! > "$WATCHER_PID_FILE"
            echo "Watcher lancé en arrière-plan (PID: $(cat $WATCHER_PID_FILE))."

        else
            ssh -o ConnectTimeout=5 -o BatchMode=yes root@"$PBS_IP" "poweroff" > /dev/null 2>&1
            send_gotify_notification "💤 PBS Physique" "Aucune tâche active, extinction immédiate." "5"
        fi
        ;;

    status)
        if [ -f "$WATCHER_PID_FILE" ]; then
            PID=$(cat "$WATCHER_PID_FILE")
            if kill -0 "$PID" 2>/dev/null; then
                echo "Watcher actif (PID: $PID)."
            else
                echo "Watcher PID $PID introuvable (processus terminé)."
                rm -f "$WATCHER_PID_FILE"
            fi
        else
            echo "Aucun watcher en cours."
        fi
        ;;

    cancel)
        if [ -f "$WATCHER_PID_FILE" ]; then
            PID=$(cat "$WATCHER_PID_FILE")
            if kill "$PID" 2>/dev/null; then
                send_gotify_notification "🛑 PBS Physique" "Surveillance annulée manuellement. Aucune extinction effectuée." "6"
                echo "Watcher (PID: $PID) arrêté."
                rm -f "$WATCHER_PID_FILE"
            else
                echo "Impossible d'arrêter le PID $PID (déjà terminé ?)."
                rm -f "$WATCHER_PID_FILE"
            fi
        else
            echo "Aucun watcher à annuler."
        fi
        ;;

    *)
        echo "Usage: $0 {start|stop|status|cancel}"
        exit 1
        ;;
esac
```

### Rendre le script exécutable

```bash
chmod +x /root/scripts/gestion-pbs-physique.sh
```

---

## Explication du script

### Variables de configuration

| Variable | Rôle |
|---|---|
| `PBS_IP` | Adresse IP fixe du PBS physique sur le réseau local |
| `PBS_MAC` | Adresse MAC nécessaire pour le Wake-on-LAN |
| `GOTIFY_URL` | URL de l'instance Gotify pour les notifications push |
| `GOTIFY_TOKEN` | Token d'application Gotify (généré dans l'interface web) |
| `MAX_RETRIES` | Nombre maximum de vérifications (8 × 30 min = 4 h) |
| `WATCHER_PID_FILE` | Fichier stockant le PID du processus de surveillance |

### Commande `start`

Envoie un paquet **Wake-on-LAN** à l'adresse MAC du PBS, puis envoie une notification Gotify confirmant la mise sous tension.

> Le Wake-on-LAN fonctionne en envoyant un "magic packet" sur le réseau local. Le serveur PBS doit être connecté en filaire et avoir le WoL activé dans son BIOS.

### Commande `stop`

Le comportement dépend de l'état du PBS au moment de l'appel :

**Cas 1 — PBS déjà éteint ou injoignable**
Le ping échoue. Le script notifie discrètement et se termine proprement (`exit 0`). Ce cas est normal si le script est appelé alors que le PBS est déjà éteint.

**Cas 2 — PBS joignable mais SSH non fonctionnel**
Le ping réussit mais la connexion SSH échoue. C'est un vrai problème (clé SSH manquante, service SSH arrêté). Une alerte prioritaire est envoyée et l'extinction est annulée pour éviter une coupure non contrôlée.

**Cas 3 — Aucune tâche en cours**
SSH fonctionne et `jq` retourne `0`. Le script envoie immédiatement la commande `poweroff` au PBS et notifie l'extinction.

**Cas 4 — Sauvegardes en cours**
`jq` retourne un nombre supérieur à `0`. Un processus de surveillance est lancé **en arrière-plan** (avec `&`) pour ne pas bloquer le terminal ou le job cron. Ce watcher :

- Attend 30 minutes
- Recompte les tâches actives
- Recommence jusqu'à ce que le compteur tombe à `0`
- Lance `poweroff` et notifie dès que les sauvegardes sont terminées
- Force l'extinction après 4 heures (8 tentatives) si les tâches ne se terminent pas

Le PID du watcher est sauvegardé dans `/var/run/pbs-cycle-watcher.pid` pour permettre un suivi ou une annulation manuelle.

### Commande `status`

Vérifie si un watcher est actuellement actif en lisant le fichier PID. Utile pour savoir si une surveillance est en cours après un `stop` avec sauvegardes actives.

```bash
/root/scripts/gestion-pbs-physique.sh status
# → Watcher actif (PID: 12345).
# → Aucun watcher en cours.
```

### Commande `cancel`

Arrête le watcher en arrière-plan sans éteindre le PBS. Pratique si on souhaite annuler une extinction programmée.

```bash
/root/scripts/gestion-pbs-physique.sh cancel
# → Watcher (PID: 12345) arrêté.
```

---

## Automatisation via cron

Éditer la crontab du nœud Proxmox VE :

```bash
crontab -e
```

Ajouter les deux lignes suivantes :

```cron
# Allumage automatique du PBS physique le 28 de chaque mois à 08h00
0 8 28 * * /root/scripts/gestion-pbs-physique.sh start

# Extinction automatique du PBS physique le 2 de chaque mois à 08h00
0 8 2 * * /root/scripts/gestion-pbs-physique.sh stop
```

Dans cet exemple, le PBS s'allume le 28 de chaque mois et s'éteint le 2 du mois suivant, ce qui lui laisse environ 5 jours pour effectuer toutes ses sauvegardes. Adaptez les dates et heures selon votre propre calendrier de sauvegardes.

> **Remarque :** Si des sauvegardes sont encore en cours le 2 à 08h00, le watcher prend le relais et retarde l'extinction jusqu'à leur fin, ou au maximum 4 heures plus tard.

---

## Tableau des notifications Gotify

| Icône | Situation | Priorité |
|---|---|---|
| ⚡ | Mise sous tension via WoL | 5 |
| 💤 | Extinction effectuée (normale ou différée) | 5 |
| 💤 | PBS déjà injoignable au moment du stop | 4 |
| ⚠️ | Sauvegardes en cours, surveillance activée | 8 |
| 🚨 | Erreur SSH malgré ping OK | 9 |
| 🚨 | Erreur SSH lors d'une vérification du watcher | 8 |
| 🚨 | Timeout 4h atteint, extinction forcée | 10 |
| 🛑 | Surveillance annulée manuellement | 6 |

---

## Dépannage

### Le Wake-on-LAN ne fonctionne pas

- Vérifier que le WoL est activé dans le BIOS/UEFI du PBS
- Vérifier que la carte réseau supporte le WoL (`ethtool eth0 | grep Wake-on`)
- S'assurer que le PBS est connecté en **filaire** (le WoL ne fonctionne pas en Wi-Fi)
- Tester manuellement : `wakeonlan 78:e7:d1:80:87:f9`

### Le SSH échoue en BatchMode

```bash
# Tester la connexion SSH sans mot de passe
ssh -o ConnectTimeout=5 -o BatchMode=yes root@<PBS_IP> "echo OK"
```

Si la commande demande un mot de passe ou échoue, copier la clé SSH :

```bash
ssh-copy-id root@<PBS_IP>
```

### `jq` introuvable sur le PBS

```bash
ssh root@<PBS_IP> "apt install -y jq"
```

### Vérifier manuellement le comptage de tâches

```bash
ssh root@<PBS_IP> \
  "proxmox-backup-manager task list --output-format json \
   | jq '[.[] | select(.status == \"running\")] | length'"
```

Doit retourner un entier (`0` si aucune tâche active).

### Le watcher tourne encore après une extinction manuelle

```bash
/root/scripts/gestion-pbs-physique.sh status
/root/scripts/gestion-pbs-physique.sh cancel
```
