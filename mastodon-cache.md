---
title: Maintenance Mastodon - Nettoyage et optimisation des caches avec tootctl
description: Maintenance de Mastodon en installation classique (bare-metal) : automatisation du nettoyage du cache, des médias et des comptes inactifs avec notifications Gotify.
published: true
date: 2026-08-21T18:43:53.133Z
tags: mastodon, cache, delete, gotify
editor: markdown
dateCreated: 2024-05-06T22:29:10.684Z
---

> ⚠️ Ce guide concerne une installation **classique de Mastodon, sans Docker**, généralement installée directement sur une base Debian.
>
> Si votre instance Mastodon fonctionne sous **Docker**, consultez la page dédiée : [Maintenance Mastodon sous Docker](https://wiki.blablalinux.be/fr/maintenance-mastodon-docker).

Mastodon accumule au fil du temps différentes données : médias provenant d'instances distantes, prévisualisations de liens, anciens statuts, comptes distants et fichiers médias devenus orphelins.

Une maintenance périodique permet de contrôler l'utilisation du stockage et de supprimer les données qui ne sont plus nécessaires.

Le script présenté dans ce guide reprend la même logique que celui utilisé pour une installation Mastodon sous Docker. La différence principale concerne uniquement la manière d'exécuter `tootctl` : ici, Mastodon est installé directement sur le système.

Le script permet notamment de :

* supprimer les anciens médias ;
* supprimer les anciennes prévisualisations de liens ;
* supprimer les anciens statuts ;
* nettoyer les comptes distants devenus inutiles ;
* rechercher et supprimer périodiquement les fichiers médias orphelins ;
* utiliser plusieurs threads pour accélérer le nettoyage des médias ;
* empêcher deux exécutions simultanées du script ;
* journaliser l'intégralité des opérations ;
* afficher l'utilisation des médias après le nettoyage ;
* envoyer des notifications Gotify en cas d'erreur ou à la fin de la maintenance.

> ℹ️ Contrairement à la version Docker, les commandes `tootctl` sont exécutées directement sur le système avec l'utilisateur `mastodon`.

---

# 1. Préparation

Cette procédure suppose que :

* Mastodon est installé directement sur Debian ;
* l'utilisateur système `mastodon` existe ;
* Ruby et Mastodon sont correctement installés ;
* `tootctl` fonctionne déjà manuellement ;
* `sudo` est disponible ;
* `curl` est installé si vous souhaitez utiliser Gotify ;
* `flock` est disponible.

Le script utilise deux paramètres permettant de retrouver l'installation Mastodon :

```bash
MASTODON_DIR="/var/www/mastodon/live"
RBENV_PATH="/opt/rbenv/versions/mastodon/bin"
```

Ces valeurs correspondent à une installation utilisant **rbenv**.

> ⚠️ Les chemins peuvent varier selon la méthode utilisée lors de l'installation de Mastodon. Vérifiez votre installation avant d'utiliser le script.

Le répertoire Mastodon doit notamment contenir :

```text
bin/tootctl
```

Vous pouvez vérifier sa présence avec :

```bash
ls -l /var/www/mastodon/live/bin/tootctl
```

---

# 2. Script de maintenance

Le script doit être placé, par exemple, dans :

```text
/root/scripts/mastodon-cleanup.sh
```

Il est prévu pour être exécuté avec les privilèges `root`.

## Script `mastodon-cleanup.sh`

La version ci-dessous reprend la logique du script de maintenance Docker, adaptée à une installation classique.

> ⚠️ Le token Gotify n'est volontairement pas publié dans cette documentation. Utilisez votre propre token.

```bash
#!/bin/bash
# Script de maintenance Mastodon pour installation classique
# Auteur : Amaury Libert (Blabla Linux)
# v2.1 - Adaptation de la version Docker pour installation classique

# --- VERROU ANTI-CHEVAUCHEMENT ---
LOCKFILE="/tmp/mastodon-cleanup.lock"
exec 200>"$LOCKFILE"
flock -n 200 || { echo "Une instance du script tourne déjà, abandon."; exit 1; }

# --- PARAMÈTRES DE GOTIFY ---
GOTIFY_URL=""
GOTIFY_TOKEN=""

# --- PARAMÈTRES DE MAINTENANCE ---
MASTODON_DIR="/var/www/mastodon/live"
RBENV_PATH="/opt/rbenv/versions/mastodon/bin"
LOGFILE="/var/log/mastodon-cleanup.log"
HOSTNAME=$(hostname)
DAYS_MEDIA=7
THREADS=4

# Semaine du mois (1-4/5), pour ne lancer remove-orphans
# qu'une fois par mois
WEEK_OF_MONTH=$(( ($(date +%-d) - 1) / 7 + 1 ))

exec 1>>"$LOGFILE" 2>&1

# --- FONCTION DE NOTIFICATION GOTIFY ---
send_gotify_notification() {
    if [ -n "$GOTIFY_URL" ] && [ -n "$GOTIFY_TOKEN" ]; then
        curl -k -s -X POST "${GOTIFY_URL}/message?token=${GOTIFY_TOKEN}" \
            -F "title=$1" -F "message=$2" -F "priority=$3" > /dev/null 2>&1
    fi
}

echo "======================================================"
echo "Début de la maintenance Mastodon sur $HOSTNAME : $(date)"
echo "======================================================"

# 1. Vérification du répertoire Mastodon
if [ ! -d "$MASTODON_DIR" ]; then
    MSG="Erreur : Le dossier Mastodon $MASTODON_DIR est introuvable."
    echo "$MSG"
    send_gotify_notification "❌ Mastodon Cleanup ÉCHEC" "$MSG sur $HOSTNAME." 8
    exit 1
fi

cd "$MASTODON_DIR" || {
    MSG="Erreur : Impossible d'accéder au dossier Mastodon."
    echo "$MSG"
    send_gotify_notification "❌ Mastodon Cleanup ÉCHEC" "$MSG sur $HOSTNAME." 8
    exit 1
}

ERRORS=0

# 2. Nettoyage des médias et vignettes
echo "--- Étape 1 : Nettoyage des médias et vignettes ---"

sudo -u mastodon RAILS_ENV=production PATH="$RBENV_PATH:$PATH" \
    bin/tootctl media remove --days="$DAYS_MEDIA" --concurrency="$THREADS"

[ $? -ne 0 ] && {
    ERRORS=1
    send_gotify_notification \
        "⚠️ Mastodon Cleanup ALERTE" \
        "Échec media remove sur $HOSTNAME." \
        5
}

sudo -u mastodon RAILS_ENV=production PATH="$RBENV_PATH:$PATH" \
    bin/tootctl preview_cards remove --days="$DAYS_MEDIA"

[ $? -ne 0 ] && {
    ERRORS=1
    send_gotify_notification \
        "⚠️ Mastodon Cleanup ALERTE" \
        "Échec preview_cards remove sur $HOSTNAME." \
        5
}

# 3. Nettoyage des anciens statuts et comptes inactifs
echo "--- Étape 2 : Nettoyage statuts et comptes ---"

sudo -u mastodon RAILS_ENV=production PATH="$RBENV_PATH:$PATH" \
    bin/tootctl statuses remove --days=30

[ $? -ne 0 ] && {
    ERRORS=1
    send_gotify_notification \
        "❌ Mastodon Cleanup ERREUR" \
        "Échec critique statuses remove sur $HOSTNAME." \
        8
}

sudo -u mastodon RAILS_ENV=production PATH="$RBENV_PATH:$PATH" \
    bin/tootctl accounts prune

[ $? -ne 0 ] && {
    ERRORS=1
    send_gotify_notification \
        "⚠️ Mastodon Cleanup ALERTE" \
        "Échec accounts prune sur $HOSTNAME." \
        5
}

# 4. Purge des orphelins (une fois par mois seulement)
if [ "$WEEK_OF_MONTH" -eq 1 ]; then
    echo "--- Étape 3 : Purge des fichiers orphelins (mensuel) ---"

    sudo -u mastodon RAILS_ENV=production PATH="$RBENV_PATH:$PATH" \
        bin/tootctl media remove-orphans

    [ $? -ne 0 ] && {
        ERRORS=1
        send_gotify_notification \
            "⚠️ Mastodon Cleanup ALERTE" \
            "Échec remove-orphans sur $HOSTNAME." \
            5
    }
fi

# 5. Rapport d'usage final
echo "--- Rapport d'utilisation médias ---"

USAGE=$(sudo -u mastodon RAILS_ENV=production PATH="$RBENV_PATH:$PATH" \
    bin/tootctl media usage)

echo "$USAGE"

echo "======================================================"
echo "Maintenance terminée : $(date)"
echo "======================================================"

if [ $ERRORS -eq 0 ]; then
    send_gotify_notification \
        "✅ Mastodon Cleanup TERMINÉ" \
        "Maintenance sur $HOSTNAME réussie sans erreur.

$USAGE" \
        4
else
    send_gotify_notification \
        "⚠️ Mastodon Cleanup TERMINÉ AVEC ALERTES" \
        "Maintenance sur $HOSTNAME finie, mais des erreurs ont été rencontrées. Vérifiez $LOGFILE." \
        5
fi

exit 0
```

---

# 3. Différence avec la version Docker

La logique de maintenance est volontairement identique.

Dans la version Docker, une commande ressemble à :

```bash
docker exec -u mastodon $CONTAINER_NAME bin/tootctl ...
```

Dans cette version classique, elle devient :

```bash
sudo -u mastodon RAILS_ENV=production PATH="$RBENV_PATH:$PATH" \
    bin/tootctl ...
```

Le reste du fonctionnement est identique.

Cette approche permet d'avoir une maintenance cohérente entre les deux types d'installation.

---

# 4. Verrou anti-chevauchement

Le script utilise `flock` :

```bash
LOCKFILE="/tmp/mastodon-cleanup.lock"
exec 200>"$LOCKFILE"
flock -n 200 || { echo "Une instance du script tourne déjà, abandon."; exit 1; }
```

Cela empêche deux instances du script de fonctionner simultanément.

C'est particulièrement utile lorsqu'une opération `tootctl` prend exceptionnellement beaucoup de temps.

Si une seconde exécution est lancée alors que la première est toujours active, elle s'arrête immédiatement :

```text
Une instance du script tourne déjà, abandon.
```

---

# 5. Configuration de Gotify

Les notifications Gotify sont optionnelles.

Par défaut :

```bash
GOTIFY_URL=""
GOTIFY_TOKEN=""
```

Le script n'envoie alors aucune notification.

Pour activer Gotify :

```bash
GOTIFY_URL="https://gotify.example.com"
GOTIFY_TOKEN="xxxxxxxxxxxxxxxx"
```

Le script envoie des notifications lorsqu'une opération échoue et à la fin de la maintenance.

## Notification d'échec

Si le répertoire Mastodon est introuvable, par exemple :

```text
❌ Mastodon Cleanup ÉCHEC
```

## Notification d'alerte

Lorsqu'une opération individuelle échoue :

```text
⚠️ Mastodon Cleanup ALERTE
```

Le message indique l'opération concernée.

## Notification de fin

Si tout s'est correctement déroulé :

```text
✅ Mastodon Cleanup TERMINÉ
```

Si une ou plusieurs opérations ont échoué :

```text
⚠️ Mastodon Cleanup TERMINÉ AVEC ALERTES
```

> ⚠️ Le token Gotify est une information sensible. Ne le publiez pas dans cette page, dans un dépôt Git ou dans une capture d'écran.

> ⚠️ La fonction utilise `curl -k`, qui désactive la vérification du certificat TLS. Si le certificat de votre serveur Gotify est correctement reconnu par le système, il est préférable de supprimer `-k`.

---

# 6. Vérification de l'installation Mastodon

Avant d'exécuter le script, vérifiez que le répertoire est correct :

```bash
ls -la /var/www/mastodon/live
```

Puis vérifiez `tootctl` :

```bash
ls -l /var/www/mastodon/live/bin/tootctl
```

Il est également recommandé de tester directement :

```bash
cd /var/www/mastodon/live
sudo -u mastodon RAILS_ENV=production PATH="/opt/rbenv/versions/mastodon/bin:$PATH" bin/tootctl --help
```

Si cette commande fonctionne, les paramètres utilisés par le script sont probablement corrects.

---

# 7. Nettoyage des médias

Le script commence le nettoyage avec :

```bash
bin/tootctl media remove --days=7 --concurrency=4
```

Les paramètres sont définis ici :

```bash
DAYS_MEDIA=7
THREADS=4
```

Le nombre de threads peut être adapté en fonction des performances du serveur.

> ⚠️ Plus de threads signifie potentiellement plus de performances, mais également davantage de charge CPU et I/O. Quatre threads constituent une valeur raisonnable pour commencer.

En cas d'erreur, le script positionne :

```bash
ERRORS=1
```

et envoie une notification Gotify.

---

# 8. Nettoyage des prévisualisations

Les anciennes cartes de prévisualisation sont supprimées avec :

```bash
bin/tootctl preview_cards remove --days=7
```

Cette opération est distincte du nettoyage des médias.

Une erreur entraîne également :

```bash
ERRORS=1
```

et une notification Gotify.

---

# 9. Nettoyage des anciens statuts

Le script utilise :

```bash
bin/tootctl statuses remove --days=30
```

La rétention configurée est donc de **30 jours**.

Cette opération peut être particulièrement lourde sur une instance ayant accumulé beaucoup de données.

En cas d'échec, une notification Gotify de niveau élevé est envoyée :

```text
❌ Mastodon Cleanup ERREUR
```

La maintenance continue néanmoins afin de permettre aux opérations suivantes de s'exécuter.

---

# 10. Nettoyage des comptes distants

Le script lance :

```bash
bin/tootctl accounts prune
```

Cette opération permet de nettoyer les comptes distants qui répondent aux critères de purge de Mastodon.

Comme pour les autres opérations, un échec est enregistré dans `ERRORS` et déclenche une notification Gotify.

---

# 11. Suppression des fichiers médias orphelins

La commande :

```bash
bin/tootctl media remove-orphans
```

est volontairement exécutée **une seule fois par mois**.

Le script calcule la semaine du mois :

```bash
WEEK_OF_MONTH=$(( ($(date +%-d) - 1) / 7 + 1 ))
```

Puis exécute `remove-orphans` uniquement lorsque :

```bash
WEEK_OF_MONTH -eq 1
```

La logique est donc :

```text
Chaque semaine :
    media remove
    preview_cards remove
    statuses remove
    accounts prune

Première semaine du mois :
    + media remove-orphans
```

Cette organisation évite de lancer chaque semaine une opération potentiellement lourde en I/O.

---

# 12. Rapport d'utilisation des médias

Après les opérations de nettoyage, le script exécute :

```bash
bin/tootctl media usage
```

Le résultat est stocké dans :

```bash
USAGE=$(...)
```

puis écrit dans le journal.

Si la maintenance s'est terminée sans erreur, ce rapport est également inclus dans la notification Gotify finale.

---

# 13. Gestion des erreurs

Le script utilise une variable globale :

```bash
ERRORS=0
```

Chaque opération importante est contrôlée.

Lorsqu'une commande échoue :

```bash
ERRORS=1
```

est positionné.

Le script ne s'arrête donc pas nécessairement à la première erreur. Il continue la maintenance et fournit ensuite un état global.

À la fin :

```text
ERRORS=0
```

signifie que les opérations surveillées se sont terminées sans erreur.

Si :

```text
ERRORS=1
```

la notification finale indique que la maintenance s'est terminée avec des alertes.

---

# 14. Journalisation

Le script écrit toutes ses sorties dans :

```text
/var/log/mastodon-cleanup.log
```

La redirection est effectuée directement dans le script :

```bash
exec 1>>"$LOGFILE" 2>&1
```

Cela permet notamment de conserver :

* les opérations effectuées ;
* les messages de `tootctl` ;
* les éventuelles erreurs ;
* le rapport `media usage` ;
* les dates de début et de fin.

Consulter le journal :

```bash
tail -n 100 /var/log/mastodon-cleanup.log
```

Suivre son évolution :

```bash
tail -f /var/log/mastodon-cleanup.log
```

---

# 15. Rotation du journal avec logrotate

Le fichier journal ne doit pas grossir indéfiniment.

Créer :

```text
/etc/logrotate.d/mastodon-cleanup
```

avec :

```text
/var/log/mastodon-cleanup.log {
    weekly
    rotate 8
    compress
    missingok
    notifempty
}
```

## Explication

### `weekly`

Le journal est géré selon une rotation hebdomadaire.

### `rotate 8`

Jusqu'à huit anciennes rotations sont conservées.

### `compress`

Les anciens journaux sont compressés afin de réduire leur occupation disque.

### `missingok`

L'absence du journal n'est pas considérée comme une erreur.

### `notifempty`

Un journal vide n'est pas inutilement soumis à une rotation.

On pourra donc retrouver des fichiers tels que :

```text
/var/log/mastodon-cleanup.log
/var/log/mastodon-cleanup.log.1.gz
/var/log/mastodon-cleanup.log.2.gz
...
```

---

# 16. Tester logrotate

Pour simuler une rotation sans modifier les fichiers :

```bash
logrotate -d /etc/logrotate.d/mastodon-cleanup
```

Pour forcer une rotation :

```bash
logrotate -f /etc/logrotate.d/mastodon-cleanup
```

> ⚠️ `-f` force réellement la rotation. Cette commande est donc principalement utile pour tester la configuration.

---

# 17. Installation du script

Créer le répertoire :

```bash
mkdir -p /root/scripts
```

Créer le script :

```bash
nano /root/scripts/mastodon-cleanup.sh
```

Coller le contenu du script puis enregistrer.

Rendre le fichier exécutable uniquement par `root` :

```bash
chmod 700 /root/scripts/mastodon-cleanup.sh
```

Vérifier les permissions :

```bash
ls -l /root/scripts/mastodon-cleanup.sh
```

Le résultat attendu est similaire à :

```text
-rwx------ 1 root root ... /root/scripts/mastodon-cleanup.sh
```

Cette restriction est particulièrement importante lorsqu'un token Gotify est présent dans le script.

---

# 18. Tester manuellement

Avant de programmer Cron, exécutez le script manuellement :

```bash
/root/scripts/mastodon-cleanup.sh
```

Puis consultez le journal :

```bash
tail -n 100 /var/log/mastodon-cleanup.log
```

Vérifiez notamment :

* le chemin de l'installation Mastodon ;
* le fonctionnement de `tootctl` ;
* l'utilisateur `mastodon` ;
* l'environnement Ruby/rbenv ;
* le nettoyage des médias ;
* le nettoyage des prévisualisations ;
* le nettoyage des statuts ;
* `accounts prune` ;
* le rapport `media usage` ;
* les éventuelles notifications Gotify.

---

# 19. Planification avec Cron

Éditer la crontab de `root` :

```bash
crontab -e
```

Pour exécuter la maintenance chaque dimanche à 03h00 :

```cron
00 03 * * 0 /bin/bash /root/scripts/mastodon-cleanup.sh
```

Le script gère lui-même la redirection vers :

```text
/var/log/mastodon-cleanup.log
```

Il n'est donc pas nécessaire d'ajouter :

```text
>> /var/log/mastodon-cleanup.log 2>&1
```

à la ligne Cron.

---

# 20. Vérifier la tâche Cron

Afficher la crontab :

```bash
crontab -l
```

Vous devez retrouver :

```cron
00 03 * * 0 /bin/bash /root/scripts/mastodon-cleanup.sh
```

Après l'exécution planifiée, vérifiez :

```bash
tail -n 100 /var/log/mastodon-cleanup.log
```

---

# 21. Résumé de la maintenance

| Opération                 |                   Configuration |
| ------------------------- | ------------------------------: |
| Médias                    |                         7 jours |
| Threads médias            |                               4 |
| Prévisualisations         |                         7 jours |
| Statuts                   |                        30 jours |
| Comptes distants          |                `accounts prune` |
| Médias orphelins          |                 1 fois par mois |
| Rapport médias            |                   `media usage` |
| Verrou anti-chevauchement |                         `flock` |
| Journal                   | `/var/log/mastodon-cleanup.log` |
| Rotation                  |                    Hebdomadaire |
| Historique                |                     8 rotations |
| Compression               |                             Oui |
| Gotify                    |                       Optionnel |
| Exécution                 |                Dimanche à 03h00 |

---

# 22. Différences avec l'installation Docker

Les opérations de maintenance sont volontairement identiques entre les deux pages.

La différence est uniquement liée à l'environnement d'exécution.

### Installation Docker

```bash
docker exec -u mastodon mastodon-web bin/tootctl ...
```

### Installation classique

```bash
sudo -u mastodon RAILS_ENV=production PATH="$RBENV_PATH:$PATH" \
    bin/tootctl ...
```

La logique de nettoyage, la gestion des erreurs, les notifications, le rapport d'utilisation, le verrouillage et la rotation des journaux restent identiques.

Cela permet de conserver une documentation cohérente pour les deux types d'installation.

---

# 23. Points importants

Cette automatisation ne remplace pas les sauvegardes de Mastodon.

Il est notamment recommandé de disposer d'une stratégie de sauvegarde pour :

* PostgreSQL ;
* les médias ;
* la configuration Mastodon ;
* les secrets et clés nécessaires à l'instance.

Le script est uniquement destiné à effectuer des opérations de maintenance et de nettoyage.

> ⚠️ Les commandes de suppression de statuts, de médias et de comptes doivent être utilisées avec prudence. Les données supprimées ne doivent pas être considérées comme récupérables.

---

## Mastodon

Vous pouvez retrouver mon instance Mastodon ici :

[https://mastodon.blablalinux.be/@blablalinux](https://mastodon.blablalinux.be/@blablalinux)
