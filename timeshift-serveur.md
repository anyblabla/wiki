---
title: Timeshift sur serveur
description: Nous allons installer Timeshift sur une distribution serveur pour ensuite créer des sauvegardes automatiques.
published: true
date: 2025-04-19T22:48:26.440Z
tags: timeshift, sauvegarde, serveur, cron
editor: markdown
dateCreated: 2024-06-05T18:52:33.986Z
---

## Explication

*Beaucoup installent et utilisent l'utilitaire de créations d'instantané* [Timeshift](https://github.com/linuxmint/timeshift) *sur distribution desktop, ici, nous allons l'installer et l'utiliser sur une distribution serveur. Donc, sans interface graphique.*

*Il existe de nombreux tutoriels sur la toile. La plupart, pour ne pas dire tous, sont incomplets ! Ils ne montrent pas l'édition et l'initialisation du fichier* [JSON](https://www.json.org/json-en.html) *pour une prise automatique d'instantanés grâce aux tâches* [cron](https://fr.wikipedia.org/wiki/Cron)*.*

*Après la lecture de la documentation, après beaucoup de recherches et de tests, voici la procédure qui fonctionne.*

*Nous allons partir d'une distribution* [Ubuntu Server](https://ubuntu.com/download/server)*.*

*Les commandes ci-dessous seront exécutées avec “*[sudo](https://fr.wikipedia.org/wiki/Sudo)*”.*

## Installation / Paramétrage

-   Mise à jour des dépôts…

```plaintext
sudo apt update
```

-   Installation de Timeshift…

```plaintext
sudo apt install timeshift -y
```

Après l'installation, nous devons exécuter manuellement Timeshift et créer notre premier instantané.

-   Nous devons d'abord connaître les détails du système de fichiers…

```plaintext
df -h
```

Exemple de sortie…

![](/timeshift-serveur/df-h.png)

-   Je vais réaliser le premier instantané sur /dev/sda1…

```plaintext
sudo timeshift --create --comments "First Snapshot" --snapshot-device /dev/sda1
```

Exemple de sortie…

![](/timeshift-serveur/timeshift-create.png)

-   Après la création du premier instantané, le fichier “timeshift.json” a été créé. Vous le modifiez selon vos besoins.

```plaintext
sudo nano /etc/timeshift/timeshift.json
```

Mon fichier…

![](/timeshift-serveur/timeshift.json.png)

Lignes à personnaliser…

Cette ligne spécifie le numéro UUID du disque cible, donc du disque qui accueille les sauvegardes (ici /dev/sda1)…

`"backup_device_uuid" : "d00a20aa-cdd1-486d-9e6f-e3f87b3b6aff",` 

-   Si l'UUID n'est pas spécifié, utilisez cette commande pour l'obtenir et le spécifier…

```plaintext
lsblk -fs
```

Cette ligne arrête les emails cron pour les tâches terminées…

`"stop_cron_emails" : "true",`

Ces lignes spécifient les calendriers à activer (mensuel/hebdomadaire/quotidien/horaire/amorçage)…

`"schedule_monthly" : "true",`

`"schedule_weekly" : "true",`

`"schedule_daily" : "true",`

`"schedule_hourly" : "false",`

`"schedule_boot" : "true",`

Ces lignes spécifient le nombre de sauvegardes à conserver pour chaque calendrier…

`"count_monthly" : "1",`

`"count_weekly" : "2",`

`"count_daily" : "3",`

`"count_hourly" : "4",`

`"count_boot" : "4",`

Une fois votre fichier JSON personnalisé, vous sauvegardez (CTRL+X / ENTER / O).

**Maintenant, la partie la plus importante...**

-   Nous allons faire vérifier notre fichier JSON par Timeshift. Ainsi, les tâches calendrier seront activées via chaque fichier cron (cron.d/cron.daily/cron.hourly/cron.monthly) qui se trouvent dans /etc/…

```plaintext
sudo timeshift --check
```

Exemple de sortie…

![](/timeshift-serveur/timeshift-check.png)

-   Pour lister les instantanés…

```plaintext
sudo timeshift --list
```

Exemple de sortie…

![](/timeshift-serveur/timeshift-list.png)

Vous savez à présent comment fonctionne la prise automatique d'instantanés grâce à Timeshift sur distribution serveur.

## Bonus

Quelques commandes supplémentaires (à adapter)…

-   Lister les instantanés qui se trouvent sur le périphérique /dev/sda1…

```plaintext
sudo timeshift --list --snapshot-device /dev/sda
```

-   Créer manuellement un instantané qui sera nommé “after update”, et qui portera le tag “D” pour “Daily”…

```plaintext
sudo timeshift --create --comments "after update" --tags D
```

-   Restaurer un instantané (commande interactive)…

```plaintext
sudo timeshift --restore
```

-   Restaurer un instantané spécifique à partir d'un périphérique spécifique…

```plaintext
sudo timeshift --restore --snapshot '2014-10-12_16-29-08' --target /dev/sda1
```

-   Supprimer un instantané spécifique…

```plaintext
sudo timeshift --delete  --snapshot '2014-10-12_16-29-08'
```

-   Supprimer **TOUS** les instantanés…

```plaintext
sudo timeshift --delete-all
```

-   Et bien plus encore…

```plaintext
timeshift
```

## Avertissement

Je dois vous prévenir que Timeshift est utilisé pour la partie fichiers système. Pas pour la partie fichier personnels. Les répertoires utilisateurs ne sont pas pris en compte. Si vous décidez d'inclure les répertoires utilisateurs personnels, **_attention_** ! À la restauration d'un instantané, les fichiers contenus dans les répertoires utilisateurs personnels **_reviendront à un état antérieur, voir, seront supprimés_** ! Timeshift sert à prendre des instantanés du système. **_On n'utilise pas Timeshift comme utilitaire de sauvegarde_**. Des instantanés et des sauvegardes, c'est différent, on utilise donc un logiciel différent.