---
title: Maintenance Mastodon - Nettoyage et optimisation des caches avec tootctl
description: Maintenance de Mastodon en installation classique (bare-metal) : automatisation du nettoyage du cache, des médias et des comptes inactifs avec notifications Gotify.
published: true
date: 2025-12-29T12:25:20.332Z
tags: mastodon, cache, delete, gotify
editor: markdown
dateCreated: 2024-05-06T22:29:10.684Z
---

> ⚠️ Ce guide concerne une installation "classique" (non Docker) de Mastodon sur base Debian. Si vous utilisez une installation tournant sous **Docker**, je vous invite à consulter cette page dédiée : [Maintenance Mastodon sous Docker](https://wiki.blablalinux.be/fr/maintenance-mastodon-docker).

Mastodon accumule divers types de données et de caches au fil du temps (images, médias, comptes distants, etc.). L'utilisation régulière des commandes `tootctl` est essentielle pour libérer de l'espace disque et maintenir la performance de votre instance.

---

## 1. Niveaux de nettoyage principaux

Voici les commandes de base pour vider les différents niveaux de cache et supprimer les données obsolètes :

| Nettoyage | Commande `tootctl` | Description |
| --- | --- | --- |
| **Vignettes en cache** | `tootctl cache clear` | Supprime les petites vignettes provenant d'autres serveurs. |
| **Vieux médias** | `tootctl media remove` | Supprime les images/vidéos plus anciennes que la période de rétention. |
| **Médias orphelins** | `tootctl media remove-orphans` | Supprime les fichiers médias qui ne sont plus liés à rien. |
| **Comptes obsolètes** | `tootctl accounts prune` | Supprime les profils distants inactifs (lourd). |
| **Statuts obsolètes** | `tootctl statuses remove` | Supprime les toots qui n'existent plus sur la fédération (très lourd). |

---

## 2. Automatisation par script (avec Gotify optionnel)

Ce script centralise les tâches de nettoyage pour votre installation bare-metal. Il peut vous envoyer une notification via **Gotify** s'il est configuré.

> 🔴 **Vous n'utilisez pas Gotify ?**
> Laissez simplement les variables `GOTIFY_URL` et `GOTIFY_TOKEN` vides. Le script détectera l'absence de configuration et ignorera l'envoi des messages sans faire d'erreur.

### Contenu du script : `mastodon-cleanup.sh`

```bash
#!/bin/bash
# Script de maintenance Mastodon (installation classique)
# Auteur : Amaury aka BlablaLinux

# --- PARAMÈTRES DE GOTIFY (Optionnel) ---
GOTIFY_URL=""
GOTIFY_TOKEN=""

# --- PARAMÈTRES DE MAINTENANCE ---
MASTODON_DIR="/var/www/mastodon/live"
RBENV_PATH="/opt/rbenv/versions/mastodon/bin"
LOGFILE="/var/log/mastodon-cleanup.log"
HOSTNAME=$(hostname)
DAYS_MEDIA=7
THREADS=4

# Redirection de toute la sortie vers le fichier journal
exec 1>>$LOGFILE 2>&1

# --- FONCTION DE NOTIFICATION GOTIFY ---
send_gotify_notification() {
    if [ -n "$GOTIFY_URL" ] && [ -n "$GOTIFY_TOKEN" ]; then
        local title="$1"
        local message="$2"
        local priority="$3"

        curl -k -s -X POST "$GOTIFY_URL/message?token=$GOTIFY_TOKEN" \
            -F "title=$title" \
            -F "message=$message" \
            -F "priority=$priority" > /dev/null 2>&1
    fi
}

echo "======================================================"
echo "Début de la maintenance Mastodon sur $HOSTNAME : $(date)"
echo "======================================================"

cd $MASTODON_DIR || {
    echo "Erreur : dossier Mastodon introuvable."
    send_gotify_notification "❌ Mastodon Cleanup ÉCHEC" "Dossier $MASTODON_DIR introuvable sur $HOSTNAME." 8
    exit 1
}

# 1. Nettoyage des médias et vignettes
echo "--- Étape 1 : Nettoyage des médias et vignettes ---"
sudo -u mastodon RAILS_ENV=production PATH=$RBENV_PATH bin/tootctl media remove --days=$DAYS_MEDIA --concurrency=$THREADS
sudo -u mastodon RAILS_ENV=production PATH=$RBENV_PATH bin/tootctl cache clear

# 2. Nettoyage des comptes et statuts
echo "--- Étape 2 : Nettoyage des comptes et statuts ---"
sudo -u mastodon RAILS_ENV=production PATH=$RBENV_PATH bin/tootctl accounts prune
sudo -u mastodon RAILS_ENV=production PATH=$RBENV_PATH bin/tootctl statuses remove

# 3. Optimisation de la mémoire RAM
echo "--- Étape 3 : Libération du cache RAM ---"
sync; echo 3 > /proc/sys/vm/drop_caches

echo "======================================================"
echo "Maintenance terminée avec succès : $(date)"
echo "======================================================"

# Envoi de la notification de succès
send_gotify_notification "✅ Mastodon Cleanup SUCCÈS" "La maintenance hebdomadaire sur $HOSTNAME s'est terminée correctement." 4

exit 0

```

---

## 3. Planification avec Crontab

1. Rendre le script exécutable : `chmod 700 /root/scripts/mastodon-cleanup.sh`
2. Ajouter à la crontab (`crontab -e`) pour une exécution le dimanche à 3h00 :

```cron
00 03 * * 0 /bin/bash /root/scripts/mastodon-cleanup.sh >> /var/log/mastodon-cleanup.log 2>&1

```

---

## 4. Notes importantes

**Optimisation de la mémoire :** la commande `drop_caches` permet de libérer les ressources mobilisées par PostgreSQL et Ruby pendant le nettoyage. Laissez le noyau Linux gérer le SWAP naturellement après cette opération.

**Stratégie hybride :** gardez les réglages de l'interface (Rétention du contenu) actifs avec des valeurs de sécurité (14 ou 30 jours) comme filet de sécurité si le script ne s'exécute pas.

[https://mastodon.blablalinux.be/@blablalinux](https://mastodon.blablalinux.be/@blablalinux)