---
title: Maintenance et nettoyage de Mastodon sous Docker
description: Maintenance de Mastodon sous Docker : nettoyage automatique du cache média, des comptes inactifs et des vieux messages avec notifications Gotify optionnelles.
published: true
date: 2026-08-21T18:17:35.952Z
tags: mastodon, docker, lxc, proxmox, cron, crontab, script, bash, pve, gotify, maintenance, automatisation
editor: markdown
dateCreated: 2025-12-25T13:00:52.896Z
---

# Maintenance Mastodon sous Docker

> ⚠️ Ce guide est spécifiquement conçu pour une installation de **Mastodon sous Docker**, notamment lorsqu'elle fonctionne dans un conteneur LXC sur Proxmox.
>
> Si vous utilisez une installation Mastodon classique, sans Docker, consultez la page dédiée : [Maintenance Mastodon (installation classique)](https://wiki.blablalinux.be/fr/mastodon-cache).

---

## 1. Pourquoi automatiser la maintenance de Mastodon ?

Mastodon génère et conserve une quantité importante de données au fil du temps : médias provenant d'instances distantes, miniatures de liens, anciens statuts, comptes distants devenus inactifs, etc.

Une partie de ces données est normalement gérée par les tâches de maintenance internes de Mastodon. Cependant, pour une instance auto-hébergée, il peut être intéressant de mettre en place une maintenance périodique afin de conserver un espace disque maîtrisé et d'effectuer certaines opérations explicitement.

Le script présenté ici automatise plusieurs opérations :

* nettoyage des médias anciens ;
* suppression des miniatures de prévisualisation de liens ;
* suppression des anciens statuts ;
* nettoyage des comptes distants devenus inactifs ;
* utilisation de plusieurs threads pour accélérer le nettoyage des médias ;
* tentative de libération du cache mémoire lorsque l'environnement l'autorise ;
* journalisation complète des opérations ;
* notification via Gotify en cas d'erreur ou à la fin de la maintenance.

> ℹ️ Le script est volontairement exécuté **depuis l'hôte Docker**, et les commandes Mastodon sont exécutées directement dans le conteneur avec `docker exec`.

---

# 2. Préparation

Le script suppose que :

* Docker est installé et fonctionnel ;
* l'utilisateur exécutant le script possède les droits nécessaires pour utiliser Docker ;
* le conteneur Mastodon est en fonctionnement ;
* la commande `docker exec` est disponible ;
* `curl` est installé si vous souhaitez utiliser les notifications Gotify.

Dans l'exemple ci-dessous, le conteneur Mastodon porte le nom :

```text
mastodon-web-1
```

Vous pouvez vérifier le nom réel de votre conteneur avec :

```bash
docker ps --format 'table {{.Names}}\t{{.Image}}\t{{.Status}}'
```

Si votre conteneur porte un autre nom, adaptez la variable `CONTAINER_NAME` dans le script.

---

# 3. Script de maintenance

Le script doit être placé, par exemple, dans :

```text
/root/scripts/mastodon-cleanup.sh
```

Il est conçu pour être exécuté avec les privilèges `root`.

## Script `mastodon-cleanup.sh`

```bash
#!/bin/bash
# Script de maintenance Mastodon pour Docker
# Auteur : Amaury aka BlablaLinux

# --- PARAMÈTRES DE GOTIFY (Optionnel) ---
GOTIFY_URL=""
GOTIFY_TOKEN=""

# --- PARAMÈTRES DE NETTOYAGE ---
CONTAINER_NAME="mastodon-web-1"
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

        curl -k -s -X POST "${GOTIFY_URL}/message?token=${GOTIFY_TOKEN}" \
            -F "title=${title}" \
            -F "message=${message}" \
            -F "priority=${priority}" > /dev/null 2>&1
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

# 2. Nettoyage des médias et miniatures de liens
echo "--- Étape 1 : Nettoyage des médias et vignettes ---"
docker exec -u mastodon $CONTAINER_NAME bin/tootctl media remove --days=$DAYS_MEDIA --concurrency=$THREADS
docker exec -u mastodon $CONTAINER_NAME bin/tootctl preview_cards remove --days=$DAYS_MEDIA

# 3. Nettoyage des anciens statuts et comptes inactifs
echo "--- Étape 2 : Nettoyage statuts et comptes ---"
docker exec -u mastodon $CONTAINER_NAME bin/tootctl statuses remove --days=30
if [ $? -ne 0 ]; then
    send_gotify_notification "⚠️ Mastodon Cleanup ALERTE" "Le nettoyage des statuts a rencontré une erreur sur $HOSTNAME." 5
fi
docker exec -u mastodon $CONTAINER_NAME bin/tootctl accounts prune

# 4. Optimisation RAM (Vérification des droits LXC)
echo "--- Étape 3 : Libération du cache RAM ---"
sync
if [ -w /proc/sys/vm/drop_caches ]; then
    echo 3 > /proc/sys/vm/drop_caches
    echo "Cache RAM libéré avec succès."
else
    echo "Note : Droits insuffisants pour drop_caches (LXC), ignoré."
fi

echo "======================================================"
echo "Maintenance terminée : $(date)"
echo "======================================================"

# 5. Envoi de la notification de succès final
send_gotify_notification "✅ Mastodon Cleanup TERMINÉ" "La maintenance sur $HOSTNAME est terminée avec succès." 4

exit 0
```

---

# 4. Configuration de Gotify

Les notifications Gotify sont **totalement optionnelles**.

Elles permettent notamment de recevoir une notification sur son téléphone ou ses autres clients Gotify lorsque le script rencontre un problème ou lorsque la maintenance est terminée.

Dans le script :

```bash
GOTIFY_URL=""
GOTIFY_TOKEN=""
```

### Sans Gotify

Laissez simplement les deux variables vides :

```bash
GOTIFY_URL=""
GOTIFY_TOKEN=""
```

Le script détecte automatiquement l'absence de configuration et n'effectue aucun appel vers Gotify.

### Avec Gotify

Renseignez l'URL de votre serveur et le token de l'application :

```bash
GOTIFY_URL="https://gotify.example.com"
GOTIFY_TOKEN="xxxxxxxxxxxxxxxx"
```

Le token est une information sensible. Le script doit donc rester accessible uniquement à `root`.

> ⚠️ Le script utilise actuellement `curl -k`, ce qui désactive la vérification du certificat TLS. Si votre serveur Gotify dispose d'un certificat correctement reconnu par le système, il est préférable de supprimer l'option `-k`.

---

# 5. Fonctionnement du nettoyage

Le script réalise plusieurs opérations successives.

## 5.1 Vérification du conteneur Docker

Avant toute opération, le script vérifie que le conteneur Mastodon est bien en fonctionnement :

```bash
docker ps -q -f name=$CONTAINER_NAME
```

Si le conteneur n'est pas trouvé, le script :

1. écrit une erreur dans le journal ;
2. envoie une notification Gotify si elle est configurée ;
3. arrête immédiatement son exécution.

Cela évite notamment de lancer inutilement les commandes `tootctl` lorsque Mastodon est arrêté.

---

## 5.2 Nettoyage des médias

La première opération utilise :

```bash
bin/tootctl media remove
```

avec :

```bash
--days=7
```

Les médias concernés par le nettoyage sont donc ceux qui répondent aux critères de rétention définis par Mastodon avec une ancienneté de 7 jours.

Le script utilise également :

```bash
--concurrency=4
```

Ce paramètre permet à `tootctl` de traiter plusieurs opérations en parallèle.

La valeur peut être adaptée :

```bash
THREADS=4
```

> ⚠️ Une valeur élevée n'est pas nécessairement meilleure. Sur une petite machine ou un stockage lent, augmenter fortement le nombre de threads peut au contraire augmenter la charge CPU et I/O.

---

## 5.3 Suppression des prévisualisations de liens

Le script lance ensuite :

```bash
bin/tootctl preview_cards remove --days=7
```

Cette commande nettoie les anciennes cartes de prévisualisation générées pour les liens publiés sur l'instance.

Cette opération est distincte du nettoyage des médias.

---

## 5.4 Suppression des anciens statuts

Les anciens statuts sont supprimés avec :

```bash
bin/tootctl statuses remove --days=30
```

La durée utilisée ici est donc de **30 jours**.

Cette valeur est indépendante de la rétention des médias :

```text
Médias                  → 7 jours
Prévisualisations       → 7 jours
Statuts                 → 30 jours
```

> ⚠️ Cette opération doit être configurée en connaissance de cause. Une fois les données supprimées, elles ne sont pas destinées à être récupérées par un simple retour arrière.

Si la commande rencontre une erreur, le script envoie une notification Gotify d'alerte lorsque Gotify est configuré.

---

## 5.5 Nettoyage des comptes distants

Le script exécute également :

```bash
bin/tootctl accounts prune
```

Cette commande permet à Mastodon de nettoyer certains comptes distants qui ne sont plus utilisés ou qui répondent aux critères de purge définis par Mastodon.

Cette opération est particulièrement intéressante pour les instances qui ont accumulé beaucoup de comptes provenant d'autres serveurs au fil du temps.

---

# 6. Libération du cache RAM

Après les opérations Mastodon, le script exécute :

```bash
sync
```

puis vérifie si le fichier suivant est accessible en écriture :

```text
/proc/sys/vm/drop_caches
```

Si c'est le cas :

```bash
echo 3 > /proc/sys/vm/drop_caches
```

est exécuté.

La valeur `3` demande au noyau de libérer les caches de pages ainsi que les caches dentries/inodes.

## Et avec un conteneur LXC ?

C'est ici que l'environnement Proxmox entre en jeu.

Un conteneur LXC, particulièrement lorsqu'il est non privilégié, peut ne pas avoir le droit de modifier :

```text
/proc/sys/vm/drop_caches
```

Le script vérifie donc explicitement les permissions avant de tenter l'opération.

Si l'accès n'est pas possible, il écrit simplement :

```text
Note : Droits insuffisants pour drop_caches (LXC), ignoré.
```

et continue normalement.

> ℹ️ L'échec de cette étape n'est donc **pas considéré comme une erreur bloquante**.

> ⚠️ `drop_caches` ne doit pas être considéré comme un mécanisme permettant de "récupérer de la RAM" de manière permanente. Le cache disque est une utilisation normale et utile de la mémoire Linux. Cette commande est surtout pertinente dans certains scénarios de maintenance ou de diagnostic.

---

# 7. Journalisation

Toutes les sorties du script sont enregistrées dans :

```text
/var/log/mastodon-cleanup.log
```

Cette redirection est effectuée directement par le script :

```bash
exec 1>>$LOGFILE 2>&1
```

Cela signifie que les messages normaux, les erreurs et les résultats des commandes `docker exec` sont enregistrés dans ce fichier.

Pour consulter le journal :

```bash
cat /var/log/mastodon-cleanup.log
```

ou, pour suivre son évolution en temps réel :

```bash
tail -f /var/log/mastodon-cleanup.log
```

---

# 8. Rotation des journaux avec logrotate

Un fichier de journal qui grossit indéfiniment finira forcément par devenir un problème.

Il est donc recommandé de confier sa rotation à `logrotate`.

Créer :

```text
/etc/logrotate.d/mastodon-cleanup
```

avec le contenu suivant :

```text
/var/log/mastodon-cleanup.log {
    weekly
    rotate 8
    compress
    missingok
    notifempty
}
```

## Explication de la configuration

### `weekly`

Le fichier journal est vérifié et, lorsqu'une rotation est nécessaire, celle-ci est effectuée chaque semaine.

### `rotate 8`

Conserve jusqu'à **8 anciennes rotations**.

On obtient donc environ :

```text
mastodon-cleanup.log
mastodon-cleanup.log.1.gz
mastodon-cleanup.log.2.gz
...
mastodon-cleanup.log.8.gz
```

Cela permet de conserver plusieurs semaines d'historique sans laisser le fichier grossir indéfiniment.

### `compress`

Les anciens journaux sont compressés, généralement avec gzip.

Un journal historique occupe ainsi beaucoup moins d'espace disque.

### `missingok`

Si le fichier n'existe pas, `logrotate` ne considère pas cela comme une erreur.

C'est pratique notamment si le script n'a encore jamais été exécuté.

### `notifempty`

Un fichier vide n'est pas inutilement soumis à une rotation.

---

## Tester la configuration logrotate

Avant d'attendre la prochaine rotation automatique, il est possible de tester la configuration avec :

```bash
logrotate -d /etc/logrotate.d/mastodon-cleanup
```

L'option `-d` effectue une simulation et ne modifie normalement aucun fichier.

Pour forcer une rotation :

```bash
logrotate -f /etc/logrotate.d/mastodon-cleanup
```

> ⚠️ La commande `-f` force réellement la rotation. À utiliser uniquement pour un test ou lorsque cela est nécessaire.

---

# 9. Installation du script

Créer le répertoire :

```bash
mkdir -p /root/scripts
```

Créer ensuite le script :

```bash
nano /root/scripts/mastodon-cleanup.sh
```

Coller le contenu du script puis enregistrer.

Le script doit être accessible uniquement à `root`, notamment parce qu'il peut contenir un token Gotify.

Appliquer les permissions :

```bash
chmod 700 /root/scripts/mastodon-cleanup.sh
```

Vérifier :

```bash
ls -l /root/scripts/mastodon-cleanup.sh
```

Le résultat attendu est similaire à :

```text
-rwx------ 1 root root ... /root/scripts/mastodon-cleanup.sh
```

---

# 10. Tester manuellement le script

Avant de créer une tâche Cron, il est fortement recommandé d'exécuter le script manuellement :

```bash
/root/scripts/mastodon-cleanup.sh
```

Puis consulter le journal :

```bash
tail -n 100 /var/log/mastodon-cleanup.log
```

Cela permet de vérifier :

* que le conteneur Docker est correctement détecté ;
* que `tootctl` fonctionne ;
* que l'utilisateur `mastodon` existe bien dans le conteneur ;
* que les commandes de nettoyage fonctionnent ;
* que Gotify reçoit correctement les notifications ;
* que l'accès à `drop_caches` est accepté ou correctement ignoré.

---

# 11. Automatisation avec Cron

Une fois le test manuel terminé, la maintenance peut être automatisée avec Cron.

Éditer la crontab de `root` :

```bash
crontab -e
```

Pour exécuter la maintenance chaque dimanche à 03h00 :

```cron
00 03 * * 0 /bin/bash /root/scripts/mastodon-cleanup.sh
```

Le script se charge lui-même de rediriger sa sortie vers :

```text
/var/log/mastodon-cleanup.log
```

Il n'est donc **pas nécessaire** d'ajouter :

```text
>> /var/log/mastodon-cleanup.log 2>&1
```

à la ligne Cron.

Cela éviter une double redirection inutile et garde la gestion du journal au même endroit, dans le script.

---

# 12. Vérifier la tâche Cron

Afficher la crontab de `root` :

```bash
crontab -l
```

La ligne doit apparaître :

```cron
00 03 * * 0 /bin/bash /root/scripts/mastodon-cleanup.sh
```

Pour vérifier après exécution que le script a bien été lancé :

```bash
tail -n 100 /var/log/mastodon-cleanup.log
```

Selon la configuration de votre système, les événements Cron peuvent également être visibles dans les journaux système.

---

# 13. Résumé de la maintenance

La configuration proposée utilise les valeurs suivantes :

| Opération                  |           Rétention / paramètre |
| -------------------------- | ------------------------------: |
| Médias                     |                         7 jours |
| Prévisualisations de liens |                         7 jours |
| Statuts                    |                        30 jours |
| Comptes distants           |                `accounts prune` |
| Threads médias             |                               4 |
| Fréquence                  |                Dimanche à 03h00 |
| Journal                    | `/var/log/mastodon-cleanup.log` |
| Rotation                   |                    Hebdomadaire |
| Historique conservé        |                     8 rotations |
| Compression                |                             Oui |
| Notification               |                Gotify optionnel |

---

# 14. Captures

![maintenance-mastodon-docker.jpg](/maintenance-mastodon-docker/maintenance-mastodon-docker.jpg)

![maintenance-mastodon-docker-02.jpg](/maintenance-mastodon-docker/maintenance-mastodon-docker-02.jpg)

---

## Mastodon

Vous pouvez retrouver mon instance Mastodon ici :

https://mastodon.blablalinux.be/@blablalinux
