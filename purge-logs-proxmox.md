---
title: Script de purge massive des journaux (purge-logs)
description: Script d'administration pour Proxmox et Debian permettant de purger massivement les journaux système, de vider les fichiers .log sans les supprimer et de libérer l'espace disque instantanément.
published: true
date: 2026-02-06T14:09:23.627Z
tags: proxmox, debian, script, bash, nettoyage, administration système
editor: markdown
dateCreated: 2026-02-06T14:09:23.627Z
---

Cet utilitaire permet de libérer rapidement de l'espace disque sur un hôte **Proxmox** ou n'importe quel système **Debian/Ubuntu** en nettoyant les journaux accumulés sans perturber les services en cours d'exécution.

## Installation rapide

Exécutez cette commande unique en tant que `root` pour créer le script, configurer les permissions et générer le raccourci système :

```bash
cat << 'EOF' > /root/scripts/purge_logs.sh && chmod +x /root/scripts/purge_logs.sh && ln -sf /root/scripts/purge_logs.sh /usr/local/bin/purge-logs
#!/bin/bash
# Script de purge massive des logs pour Amaury (BlablaLinux)
echo "--- Nettoyage de /var/log en cours... ---"
find /var/log -type f \( -name "*.log" -exec truncate -s 0 {} + \) -o \( -name "*.gz" -o -name "*.1" \) -delete
journalctl --vacuum-time=1s > /dev/null 2>&1
echo "--- Nettoyage terminé ! [OK] ---"
EOF

```

## Fonctionnement technique

Le script combine trois actions de nettoyage profond :

1. **Mise à zéro des fichiers .log** : Utilise `truncate -s 0` plutôt que `rm`. Cela vide le contenu des fichiers sans les supprimer, évitant ainsi de rompre les descripteurs de fichiers des services qui écrivent dedans.
2. **Suppression des archives de rotation** : Recherche et supprime définitivement les anciens fichiers compressés (`.gz`) et les copies de sauvegarde immédiates (`.1`).
3. **Nettoyage du journal système** : Utilise `journalctl --vacuum-time=1s` pour purger la base de données binaire des journaux de `systemd` (ne conserve que les entrées de la dernière seconde).

## Utilisation

Une fois installé, le script peut être appelé depuis n'importe quel dossier via son alias :

```bash
purge-logs

```

### Exemple de sortie

> --- Nettoyage de /var/log en cours... ---
> --- Nettoyage terminé ! [OK] ---

## Précautions d'usage

* **Historique** : L'exécution de cette commande rend tout dépannage (troubleshooting) difficile pour les événements passés, car les traces sont effacées.
* **Fréquence** : Recommandé avant de réaliser une sauvegarde complète (backup) du nœud Proxmox ou en cas de saturation imminente de la partition racine `/`.