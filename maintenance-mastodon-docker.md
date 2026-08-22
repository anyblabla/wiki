---
title: Maintenance et nettoyage de Mastodon sous Docker
description: Maintenance de Mastodon sous Docker : nettoyage automatique du cache média, des comptes inactifs et des vieux messages avec notifications Gotify optionnelles.
published: true
date: 2026-08-22T10:37:51.540Z
tags: mastodon, docker, lxc, proxmox, cron, crontab, script, bash, pve, gotify, maintenance, automatisation
editor: markdown
dateCreated: 2025-12-25T13:00:52.896Z
---

> ⚠️ Ce guide est spécifiquement conçu pour une installation de **Mastodon sous Docker**, notamment lorsqu'elle fonctionne dans un conteneur LXC sur Proxmox.
>
> Si vous utilisez une installation Mastodon classique, sans Docker, consultez la page dédiée : [Maintenance Mastodon (installation classique)](https://wiki.blablalinux.be/fr/mastodon-cache).

---

## 1. Pourquoi automatiser la maintenance de Mastodon ?

Mastodon accumule au fil du temps différentes données : médias provenant d'instances distantes, avatars et bannières de comptes distants, miniatures de prévisualisation, anciens statuts, comptes distants, fichiers médias devenus orphelins, etc.

Une partie de ces opérations est gérée nativement par Mastodon. Cependant, une maintenance périodique permet de regrouper plusieurs opérations d'entretien et de contrôler plus facilement leur résultat.

Le script présenté dans ce guide permet notamment de :

* supprimer les anciens médias ;
* supprimer les anciennes prévisualisations de liens ;
* supprimer les avatars et bannières distants mis en cache localement ;
* supprimer les anciens statuts ;
* nettoyer les comptes distants devenus inutiles ;
* rechercher et supprimer périodiquement les fichiers médias orphelins ;
* utiliser plusieurs threads lors du nettoyage des médias ;
* empêcher deux exécutions simultanées du script ;
* journaliser l'intégralité des opérations ;
* mesurer l'utilisation des médias après le nettoyage ;
* envoyer des notifications Gotify en cas d'erreur ou à la fin de la maintenance.

Le nettoyage des fichiers orphelins est volontairement effectué **une seule fois par mois**, car cette opération peut être beaucoup plus lourde en entrées/sorties disque.

> ℹ️ Le script est exécuté **depuis l'hôte Docker**. Les commandes Mastodon sont exécutées dans le conteneur avec `docker exec` et l'utilisateur `mastodon`.

> ℹ️ Sur une instance active, la part la plus importante du stockage `public/system` n'est généralement pas constituée de vos propres médias, mais du **cache des médias fédérés** — avatars, bannières et pièces jointes des comptes distants que vos utilisateurs suivent ou croisent. La commande `tootctl media usage` permet de distinguer ce qui est local (`local`) du reste (cache fédéré). C'est justement ce cache que les étapes 1 et 1bis ci-dessous ciblent en priorité.

---

# 2. Préparation

Le script suppose que :

* Docker est installé et fonctionnel ;
* le conteneur Mastodon est en fonctionnement ;
* l'utilisateur exécutant le script dispose des droits nécessaires pour utiliser Docker ;
* `curl` est installé si les notifications Gotify sont utilisées ;
* `flock` est disponible, ce qui est normalement le cas sur une installation Debian/Ubuntu standard.

Dans le script, le conteneur est défini par :

```bash
CONTAINER_NAME="mastodon-web"
```

Adaptez cette valeur au nom réel de votre conteneur.

Vous pouvez obtenir la liste des conteneurs avec :

```bash
docker ps --format 'table {{.Names}}\t{{.Image}}\t{{.Status}}'
```

> ⚠️ Avec Docker Compose, le nom peut être différent selon votre projet. Vérifiez toujours le nom réellement utilisé avant de modifier `CONTAINER_NAME`.

---

# 3. Script de maintenance

Le script peut être placé dans :

```text
/root/scripts/mastodon-cleanup.sh
```

Il est prévu pour être exécuté avec les privilèges `root`.

## Script `mastodon-cleanup.sh`

Le script ci-dessous correspond à la **version 2.2** utilisée pour cette documentation.

> ⚠️ Le token Gotify n'est volontairement pas inclus dans cette page. Renseignez votre propre token dans le script si vous utilisez Gotify.

```bash
#!/bin/bash
# Script de maintenance Mastodon pour Docker (Version sécurisée avec alertes)
# Auteur : Amaury Libert (Blabla Linux)
# v2.2 - Ajout de --prune-profiles sur media remove (syntaxe correcte,
#        remplace la fausse commande "media prune-profile-media" de la v2.1)

# --- VERROU ANTI-CHEVAUCHEMENT ---
LOCKFILE="/tmp/mastodon-cleanup.lock"
exec 200>"$LOCKFILE"
flock -n 200 || { echo "Une instance du script tourne déjà, abandon."; exit 1; }

# --- PARAMÈTRES DE GOTIFY ---
GOTIFY_URL=""
GOTIFY_TOKEN=""

# --- PARAMÈTRES DE NETTOYAGE ---
CONTAINER_NAME="mastodon-web"
LOGFILE="/var/log/mastodon-cleanup.log"
HOSTNAME=$(hostname)
DAYS_MEDIA=7
DAYS_PROFILES=14
THREADS=4
# Semaine du mois (1-4/5), pour ne lancer remove-orphans qu'une fois par mois
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

# 1. Vérification de la présence du conteneur
if [ ! "$(docker ps -q -f name=$CONTAINER_NAME)" ]; then
    MSG="Erreur : Le conteneur $CONTAINER_NAME est introuvable."
    echo "$MSG"
    send_gotify_notification "❌ Mastodon Cleanup ÉCHEC" "$MSG sur $HOSTNAME." 8
    exit 1
fi

ERRORS=0

# 2. Nettoyage des médias et vignettes
echo "--- Étape 1 : Nettoyage des médias et vignettes ---"
docker exec -u mastodon $CONTAINER_NAME bin/tootctl media remove --days=$DAYS_MEDIA --concurrency=$THREADS
[ $? -ne 0 ] && { ERRORS=1; send_gotify_notification "⚠️ Mastodon Cleanup ALERTE" "Échec media remove sur $HOSTNAME." 5; }

docker exec -u mastodon $CONTAINER_NAME bin/tootctl preview_cards remove --days=$DAYS_MEDIA
[ $? -ne 0 ] && { ERRORS=1; send_gotify_notification "⚠️ Mastodon Cleanup ALERTE" "Échec preview_cards remove sur $HOSTNAME." 5; }

# 2bis. Nettoyage des avatars/headers distants en cache (comptes non suivis)
echo "--- Étape 1bis : Purge avatars/headers distants (--prune-profiles) ---"
docker exec -u mastodon $CONTAINER_NAME bin/tootctl media remove --days=$DAYS_PROFILES --prune-profiles
[ $? -ne 0 ] && { ERRORS=1; send_gotify_notification "⚠️ Mastodon Cleanup ALERTE" "Échec media remove --prune-profiles sur $HOSTNAME." 5; }

# 3. Nettoyage des anciens statuts et comptes inactifs
echo "--- Étape 2 : Nettoyage statuts et comptes ---"
docker exec -u mastodon $CONTAINER_NAME bin/tootctl statuses remove --days=30
[ $? -ne 0 ] && { ERRORS=1; send_gotify_notification "❌ Mastodon Cleanup ERREUR" "Échec critique statuses remove sur $HOSTNAME." 8; }

docker exec -u mastodon $CONTAINER_NAME bin/tootctl accounts prune
[ $? -ne 0 ] && { ERRORS=1; send_gotify_notification "⚠️ Mastodon Cleanup ALERTE" "Échec accounts prune sur $HOSTNAME." 5; }

# 4. Purge des orphelins (une fois par mois seulement, plus lourd en I/O)
if [ "$WEEK_OF_MONTH" -eq 1 ]; then
    echo "--- Étape 3 : Purge des fichiers orphelins (mensuel) ---"
    docker exec -u mastodon $CONTAINER_NAME bin/tootctl media remove-orphans
    [ $? -ne 0 ] && { ERRORS=1; send_gotify_notification "⚠️ Mastodon Cleanup ALERTE" "Échec remove-orphans sur $HOSTNAME." 5; }
fi

# 5. Rapport d'usage final
echo "--- Rapport d'utilisation médias ---"
USAGE=$(docker exec -u mastodon $CONTAINER_NAME bin/tootctl media usage)
echo "$USAGE"

echo "======================================================"
echo "Maintenance terminée : $(date)"
echo "======================================================"

if [ $ERRORS -eq 0 ]; then
    send_gotify_notification "✅ Mastodon Cleanup TERMINÉ" "Maintenance sur $HOSTNAME réussie sans erreur.

$USAGE" 4
else
    send_gotify_notification "⚠️ Mastodon Cleanup TERMINÉ AVEC ALERTES" "Maintenance sur $HOSTNAME finie, mais des erreurs ont été rencontrées. Vérifiez $LOGFILE." 5
fi

exit 0
```

---

# 4. Le verrou anti-chevauchement

Le script commence par créer un verrou :

```bash
LOCKFILE="/tmp/mastodon-cleanup.lock"
exec 200>"$LOCKFILE"
flock -n 200 || { echo "Une instance du script tourne déjà, abandon."; exit 1; }
```

Ce mécanisme empêche plusieurs instances du script de fonctionner simultanément.

C'est particulièrement important avec une exécution planifiée par Cron.

Par exemple, si un nettoyage prend exceptionnellement beaucoup de temps et qu'une nouvelle exécution est déclenchée avant la fin de la précédente, la deuxième instance s'arrête immédiatement.

Cela évite d'avoir deux opérations `tootctl` lourdes travaillant simultanément sur la même instance Mastodon.

---

# 5. Configuration de Gotify

Gotify est **optionnel**.

Les paramètres se trouvent au début du script :

```bash
GOTIFY_URL=""
GOTIFY_TOKEN=""
```

## Sans Gotify

Laissez simplement les deux variables vides :

```bash
GOTIFY_URL=""
GOTIFY_TOKEN=""
```

Le script détecte automatiquement l'absence de configuration et n'envoie aucune notification.

## Avec Gotify

Renseignez l'adresse de votre serveur et le token de votre application :

```bash
GOTIFY_URL="https://gotify.example.com"
GOTIFY_TOKEN="xxxxxxxxxxxxxxxx"
```

Le script peut alors envoyer plusieurs types de notifications.

### Échec du conteneur

Si le conteneur Mastodon n'est pas trouvé :

```text
❌ Mastodon Cleanup ÉCHEC
```

### Erreur pendant une opération

Par exemple :

```text
⚠️ Mastodon Cleanup ALERTE
Échec media remove...
```

ou :

```text
❌ Mastodon Cleanup ERREUR
Échec critique statuses remove...
```

### Fin sans erreur

```text
✅ Mastodon Cleanup TERMINÉ
```

La notification contient également le résultat de :

```bash
tootctl media usage
```

### Fin avec erreurs

Si au moins une opération a échoué :

```text
⚠️ Mastodon Cleanup TERMINÉ AVEC ALERTES
```

Le script indique alors de consulter le journal.

> ⚠️ Le token Gotify est une donnée sensible. Ne le publiez jamais dans une page Wiki publique, un dépôt Git ou une capture d'écran.

> ⚠️ La fonction utilise actuellement `curl -k`, qui désactive la vérification du certificat TLS. Si votre serveur Gotify utilise un certificat valide reconnu par le système, vous pouvez supprimer `-k`.

---

# 6. Vérification du conteneur

Avant d'exécuter `tootctl`, le script vérifie que le conteneur est actif :

```bash
if [ ! "$(docker ps -q -f name=$CONTAINER_NAME)" ]; then
```

Si le conteneur n'est pas trouvé, aucune commande de maintenance n'est lancée.

Le script :

1. écrit l'erreur dans le journal ;
2. envoie une notification Gotify si configurée ;
3. quitte avec le code `1`.

Cette vérification évite notamment de lancer une série de commandes `docker exec` sur un conteneur arrêté ou inexistant.

---

# 7. Nettoyage des médias

La première opération utilise :

```bash
docker exec -u mastodon $CONTAINER_NAME bin/tootctl media remove --days=$DAYS_MEDIA --concurrency=$THREADS
```

Avec les valeurs configurées :

```bash
DAYS_MEDIA=7
THREADS=4
```

La commande devient donc :

```bash
bin/tootctl media remove --days=7 --concurrency=4
```

Le nettoyage utilise quatre threads.

Cette valeur peut être adaptée en fonction des performances du serveur :

```bash
THREADS=4
```

> ⚠️ Augmenter le nombre de threads peut accélérer l'opération, mais augmente également la charge CPU, disque et potentiellement PostgreSQL. Il vaut mieux adapter cette valeur à la machine plutôt que de mettre un nombre arbitrairement élevé.

> ℹ️ Sans option supplémentaire, `media remove` ne supprime que les **pièces jointes** (attachments). Les avatars et bannières distants ne sont pas concernés par cette commande seule — voir l'étape suivante.

---

# 8. Nettoyage des avatars et bannières distants (`--prune-profiles`)

Sur une instance active, les avatars et bannières des comptes distants représentent souvent la part la plus importante du cache fédéré, bien plus que les pièces jointes elles-mêmes.

Le script exécute une seconde commande dédiée :

```bash
docker exec -u mastodon $CONTAINER_NAME bin/tootctl media remove --days=$DAYS_PROFILES --prune-profiles
```

Avec la configuration actuelle :

```bash
DAYS_PROFILES=14
```

La commande devient :

```bash
bin/tootctl media remove --days=14 --prune-profiles
```

Le flag `--prune-profiles` (disponible depuis Mastodon 4.1.0) indique à `media remove` de traiter les avatars et bannières en cache plutôt que les pièces jointes.

### Comportement par défaut

Par défaut, seuls les comptes **non suivis localement** sont concernés : les avatars/bannières des comptes que vos utilisateurs suivent activement restent en cache, pour éviter de les re-télécharger en boucle.

### Rétention plus longue que les médias

`DAYS_PROFILES` est volontairement plus élevé que `DAYS_MEDIA`, car un avatar ou une bannière change beaucoup moins souvent qu'une pièce jointe postée dans un statut. Une rétention trop courte forcerait des re-téléchargements inutiles.

### Options plus agressives (non utilisées dans ce script)

Deux options existent pour aller plus loin, mais ne sont pas incluses ici car plus radicales :

* `--remove-headers` : cible uniquement les bannières (headers), pas les avatars. Ne peut pas être combiné avec `--prune-profiles`.
* `--include-follows` : à utiliser avec `--prune-profiles` ou `--remove-headers`, supprime également le cache des comptes **suivis**, ce qui forcera leur re-téléchargement à la prochaine interaction.

Ces options peuvent être envisagées ponctuellement (par exemple en exécution manuelle, ou intégrées à l'étape mensuelle aux côtés de `remove-orphans`) si le cache reste élevé malgré le nettoyage hebdomadaire standard.

> ⚠️ Comme pour `media remove` classique, cette opération peut prendre du temps sur une instance avec beaucoup de comptes distants connus — elle itère sur chaque compte.

---

# 9. Nettoyage des prévisualisations de liens

Le script exécute ensuite :

```bash
docker exec -u mastodon $CONTAINER_NAME bin/tootctl preview_cards remove --days=$DAYS_MEDIA
```

Avec la configuration actuelle :

```bash
DAYS_MEDIA=7
```

les prévisualisations sont donc nettoyées avec une rétention de 7 jours.

Cette opération est indépendante du nettoyage des médias et de la purge des profils.

---

# 10. Nettoyage des statuts

Le script lance :

```bash
docker exec -u mastodon $CONTAINER_NAME bin/tootctl statuses remove --days=30
```

Les statuts sont donc nettoyés avec une période de 30 jours.

Cette opération est considérée comme particulièrement importante à surveiller, car elle peut être beaucoup plus lourde que le simple nettoyage des médias.

Si elle échoue, le script :

```bash
ERRORS=1
```

et envoie une notification Gotify de niveau élevé :

```text
❌ Mastodon Cleanup ERREUR
```

La maintenance continue néanmoins avec l'étape suivante.

---

# 11. Nettoyage des comptes distants

Le script exécute :

```bash
docker exec -u mastodon $CONTAINER_NAME bin/tootctl accounts prune
```

Cette opération permet de nettoyer les comptes distants devenus inutiles selon les critères de Mastodon.

Si cette opération échoue, le script positionne également :

```bash
ERRORS=1
```

et envoie une alerte Gotify.

Le script ne s'arrête cependant pas immédiatement : les opérations restantes continuent.

---

# 12. Suppression des fichiers médias orphelins

Une particularité importante de cette version du script est que :

```bash
tootctl media remove-orphans
```

**n'est pas exécuté chaque semaine.**

Le script calcule d'abord la semaine du mois :

```bash
WEEK_OF_MONTH=$(( ($(date +%-d) - 1) / 7 + 1 ))
```

Puis :

```bash
if [ "$WEEK_OF_MONTH" -eq 1 ]; then
```

lance :

```bash
docker exec -u mastodon $CONTAINER_NAME bin/tootctl media remove-orphans
```

L'opération est donc exécutée pendant la **première semaine de chaque mois**.

### Pourquoi ?

La recherche et la suppression des fichiers orphelins peuvent être beaucoup plus lourdes en I/O que le nettoyage courant des médias.

Il est donc inutile de faire cette opération à chaque exécution hebdomadaire du script.

La stratégie retenue est :

```text
Chaque semaine :
    media remove
    preview_cards remove
    media remove --prune-profiles
    statuses remove
    accounts prune

Première semaine du mois :
    + media remove-orphans
```

Si `remove-orphans` échoue, le script signale également l'erreur via Gotify et conserve l'état :

```bash
ERRORS=1
```

---

# 13. Rapport d'utilisation des médias

À la fin du nettoyage, le script exécute :

```bash
USAGE=$(docker exec -u mastodon $CONTAINER_NAME bin/tootctl media usage)
```

Puis affiche le résultat dans le journal :

```bash
echo "$USAGE"
```

Cela permet d'avoir une indication de l'utilisation actuelle des médias après les opérations de nettoyage, avec la distinction entre volume total et volume local (`tootctl media usage` affiche les deux, par exemple `Headers 14.5 GB / 79 KB local`) — ce qui permet de vérifier rapidement que le cache fédéré reste maîtrisé sans confondre avec vos propres médias.

Ce même rapport est inclus dans la notification Gotify lorsque la maintenance se termine sans erreur.

---

# 14. Gestion globale des erreurs

Le script utilise :

```bash
ERRORS=0
```

Chaque opération importante est ensuite contrôlée.

Par exemple :

```bash
[ $? -ne 0 ] && { ERRORS=1; send_gotify_notification ...; }
```

Cela permet au script de **continuer la maintenance même lorsqu'une opération individuelle échoue**, tout en conservant l'information sur l'erreur.

À la fin :

```bash
if [ $ERRORS -eq 0 ]; then
```

le script distingue deux situations.

### Aucune erreur

```text
✅ Mastodon Cleanup TERMINÉ
```

### Une ou plusieurs erreurs

```text
⚠️ Mastodon Cleanup TERMINÉ AVEC ALERTES
```

Cette distinction est importante : recevoir une notification de fin ne signifie donc pas automatiquement que toutes les opérations ont réussi.

---

# 15. Journalisation

Toutes les sorties du script sont enregistrées dans :

```text
/var/log/mastodon-cleanup.log
```

La redirection est réalisée ici :

```bash
exec 1>>"$LOGFILE" 2>&1
```

Les sorties standard **et** les erreurs sont donc enregistrées dans le même fichier.

Pour consulter les dernières lignes :

```bash
tail -n 100 /var/log/mastodon-cleanup.log
```

Pour suivre l'exécution en temps réel :

```bash
tail -f /var/log/mastodon-cleanup.log
```

---

# 16. Rotation du journal avec logrotate

Le fichier `/var/log/mastodon-cleanup.log` doit être soumis à une rotation afin d'éviter qu'il ne grossisse indéfiniment.

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

## Signification

### `weekly`

Rotation hebdomadaire.

### `rotate 8`

Conservation de huit anciennes rotations.

On pourra donc retrouver :

```text
mastodon-cleanup.log
mastodon-cleanup.log.1.gz
mastodon-cleanup.log.2.gz
mastodon-cleanup.log.3.gz
...
mastodon-cleanup.log.8.gz
```

### `compress`

Les anciens journaux sont compressés.

### `missingok`

L'absence du fichier journal n'est pas considérée comme une erreur.

### `notifempty`

Un fichier journal vide n'est pas inutilement soumis à une rotation.

---

# 17. Tester logrotate

Pour vérifier la configuration sans effectuer de rotation :

```bash
logrotate -d /etc/logrotate.d/mastodon-cleanup
```

Pour forcer une rotation :

```bash
logrotate -f /etc/logrotate.d/mastodon-cleanup
```

> ⚠️ `-f` force réellement la rotation. Cette commande est donc principalement destinée aux tests.

---

# 18. Installation du script

Créer le répertoire :

```bash
mkdir -p /root/scripts
```

Créer le fichier :

```bash
nano /root/scripts/mastodon-cleanup.sh
```

Coller le script puis enregistrer.

Rendre le script accessible uniquement à `root` :

```bash
chmod 700 /root/scripts/mastodon-cleanup.sh
```

Vérifier :

```bash
ls -l /root/scripts/mastodon-cleanup.sh
```

Le résultat doit être similaire à :

```text
-rwx------ 1 root root ... /root/scripts/mastodon-cleanup.sh
```

Cette restriction est particulièrement importante si un token Gotify est configuré directement dans le script.

---

# 19. Tester manuellement

Avant de programmer l'exécution automatique, il est recommandé de lancer le script manuellement :

```bash
/root/scripts/mastodon-cleanup.sh
```

Puis consulter le journal :

```bash
tail -n 100 /var/log/mastodon-cleanup.log
```

Cette première exécution permet de vérifier :

* que le conteneur est correctement détecté ;
* que `docker exec` fonctionne ;
* que l'utilisateur `mastodon` existe dans le conteneur ;
* que les commandes `tootctl` fonctionnent, y compris `media remove --prune-profiles` ;
* que les notifications Gotify fonctionnent si elles sont activées ;
* que le rapport `media usage` est correctement généré.

---

# 20. Planification avec Cron

Une fois le test manuel effectué, éditer la crontab de `root` :

```bash
crontab -e
```

Pour exécuter la maintenance chaque dimanche à 03h00 :

```cron
00 03 * * 0 /bin/bash /root/scripts/mastodon-cleanup.sh
```

Il n'est **pas nécessaire** de rajouter :

```text
>> /var/log/mastodon-cleanup.log 2>&1
```

à la ligne Cron.

Le script effectue déjà lui-même la redirection vers :

```text
/var/log/mastodon-cleanup.log
```

Cela évite d'avoir deux mécanismes différents pour gérer le journal.

---

# 21. Vérifier la tâche Cron

Afficher la crontab :

```bash
crontab -l
```

Vous devez retrouver :

```cron
00 03 * * 0 /bin/bash /root/scripts/mastodon-cleanup.sh
```

Après une exécution, vérifier le journal :

```bash
tail -n 100 /var/log/mastodon-cleanup.log
```

---

# 22. Résumé

| Opération                 |                                Configuration |
| ------------------------- | --------------------------------------------: |
| Médias (pièces jointes)   |                                       7 jours |
| Threads médias            |                                             4 |
| Avatars/bannières distants (`--prune-profiles`) |                        14 jours |
| Prévisualisations         |                                       7 jours |
| Statuts                   |                                      30 jours |
| Comptes distants          |                              `accounts prune` |
| Fichiers orphelins        |                               1 fois par mois |
| Rapport médias            |                                 `media usage` |
| Verrou anti-chevauchement |                                       `flock` |
| Journal                   |               `/var/log/mastodon-cleanup.log` |
| Rotation                  |                                  Hebdomadaire |
| Historique                |                                   8 rotations |
| Compression               |                                           Oui |
| Gotify                    |                                     Optionnel |
| Exécution                 |                              Dimanche à 03h00 |

---

# 23. Points importants à retenir

Le script n'est pas simplement une succession de commandes `tootctl`.

Il apporte également plusieurs mécanismes de sécurité et de surveillance :

* **`flock`** empêche deux nettoyages simultanés ;
* chaque opération importante est contrôlée ;
* les erreurs sont mémorisées dans `ERRORS` ;
* une notification Gotify spécifique est envoyée lorsqu'une opération échoue ;
* la notification finale distingue un nettoyage réussi d'un nettoyage terminé avec alertes ;
* `media remove --prune-profiles` cible spécifiquement le cache d'avatars/bannières distants, généralement la plus grosse source de croissance du stockage ;
* `remove-orphans` est limité à une exécution mensuelle ;
* `media usage` fournit un état de l'utilisation des médias, avec la distinction local/total ;
* le journal est conservé et automatiquement compressé avec `logrotate`.

> ⚠️ Le script est une automatisation de maintenance. Il ne remplace évidemment pas une stratégie de sauvegarde PostgreSQL et des fichiers médias.

---

# 24. Captures

![maintenance-mastodon-docker.jpg](/maintenance-mastodon-docker/maintenance-mastodon-docker.jpg)

![maintenance-mastodon-docker-02.jpg](/maintenance-mastodon-docker/maintenance-mastodon-docker-02.jpg)

---

## Mastodon

Vous pouvez retrouver mon instance Mastodon ici :

[https://mastodon.blablalinux.be/@blablalinux](https://mastodon.blablalinux.be/@blablalinux)