---
title: Debian - Activer les mises à jour automatiques avec unattended-upgrades
description: >-
  Ce guide explique comment configurer et activer l'outil unattended-upgrades
  sur Debian afin d'appliquer automatiquement les mises à jour de sécurité et
  d'autres paquets.
published: true
date: '2026-11-11T22:34:07.944Z'
tags:
  - debian
  - upgrade
  - unattended upgrades
editor: markdown
dateCreated: '2025-11-01T00:14:07.294Z'
---

## 1\. Installation du paquet

Le paquet `unattended-upgrades` doit d'abord être installé sur votre système Debian. Il est également recommandé d'installer **`apt-listchanges`** pour recevoir des notifications des modifications de paquets.

Exécutez la commande suivante en tant que `root` (ou avec `sudo`) :

```bash
apt update
apt install unattended-upgrades apt-listchanges -y
```

-----

## 2\. Activation initiale

Après l'installation, vous pouvez activer le service de base pour les mises à jour de sécurité.

### A. Méthode interactive (`dpkg-reconfigure`)

Exécutez cette commande et **confirmez** l'activation des mises à jour automatiques lorsque vous y êtes invité :

```bash
dpkg-reconfigure unattended-upgrades
```

### B. Méthode non-interactive (recommandée)

Cette méthode permet d'activer directement les mises à jour de sécurité sans passer par l'interface de `dpkg-reconfigure` :

```bash
echo unattended-upgrades unattended-upgrades/enable_auto_updates boolean true | debconf-set-selections
dpkg-reconfigure -f noninteractive unattended-upgrades
```

-----

## 3\. Configuration des fréquences

Le système `apt` utilise un fichier de configuration pour définir la fréquence des opérations de maintenance automatique : `/etc/apt/apt.conf.d/20auto-upgrades`.

Éditez (ou créez) ce fichier pour définir la fréquence des opérations. Les valeurs sont en jours.

| Option | Description | Valeur `1` (Quotidienne) | Valeur `7` (Hebdomadaire) | Valeur `0` (Désactivé) |
| :--- | :--- | :--- | :--- | :--- |
| `APT::Periodic::Update-Package-Lists` | Fréquence de la mise à jour des listes de paquets (`apt update`). | **1** | 7 | 0 |
| `APT::Periodic::Unattended-Upgrade` | Fréquence d'exécution de `unattended-upgrade`. | **1** | 7 | 0 |
| `APT::Periodic::AutocleanInterval` | Fréquence du nettoyage des archives de paquets téléchargés. | **1** | 7 | 0 |
| `APT::Periodic::Verbose` | Niveau de détail des logs (0=silencieux, 1=sommaire, 2=détaillé). | **1** | 1 | 1 |

Un bon point de départ pour une **mise à jour quotidienne** est le contenu suivant :

```bash
// /etc/apt/apt.conf.d/20auto-upgrades
APT::Periodic::Update-Package-Lists "1";
APT::Periodic::Unattended-Upgrade "1";
APT::Periodic::AutocleanInterval "1";
APT::Periodic::Verbose "1";
```

-----

## 4\. Personnalisation avancée (fichier `50unattended-upgrades`)

Le fichier `/etc/apt/apt.conf.d/50unattended-upgrades` permet d'affiner le comportement du processus de mise à jour.

### A. Sources autorisées (`Allowed-Origins`)

Par défaut, seules les **mises à jour de sécurité** sont activées. Pour autoriser d'autres dépôts, modifiez la section `Unattended-Upgrade::Allowed-Origins` :

```bash
Unattended-Upgrade::Allowed-Origins {
    "${distro_id}:${distro_codename}-security"; // La ligne de sécurité est toujours recommandée

    // Décommenter pour les mises à jour standards et les mises à jour proposées
    // "${distro_id}:${distro_codename}";
    // "${distro_id}:${distro_codename}-updates";
    // "${distro_id}:${distro_codename}-proposed-updates";
};
```

> **⚠️ Attention :** L'activation des mises à jour non sécuritaires peut augmenter le risque de problèmes de stabilité sur les serveurs critiques.

### B. Redémarrage automatique (`Automatic-Reboot`)

Pour que le système redémarre automatiquement si une mise à jour de noyau ou d'une bibliothèque critique le nécessite, modifiez les options suivantes :

```bash
// Activer le redémarrage automatique après les mises à jour
Unattended-Upgrade::Automatic-Reboot "true";

// Définir l'heure de redémarrage (ex: 4h00 du matin)
Unattended-Upgrade::Automatic-Reboot-Time "04:00";
```

> **Remarque :** Le système ne redémarrera que si le fichier `/var/run/reboot-required` est présent.

### C. Notifications par courriel (`Mail`)

Pour être informé en cas de problème ou après une mise à jour, configurez une adresse courriel (nécessite l'installation d'un serveur de messagerie local comme `postfix` ou `msmtp`) :

```bash
// Adresse e-mail de notification
Unattended-Upgrade::Mail "admin@example.com";

// Quand envoyer le rapport (on-change, always, never)
Unattended-Upgrade::MailReport "on-change";
```

-----

## 5\. Test et débogage

Pour simuler l'exécution du processus et vérifier si les mises à jour fonctionneront comme prévu, utilisez les options `-d` (debug) et `--dry-run` (simulation) :

```bash
unattended-upgrade -d --dry-run
```

### 🔎 Consultation des logs

Les logs de l'exécution de `unattended-upgrades` se trouvent dans :

  * `/var/log/unattended-upgrades/unattended-upgrades.log`
  * `/var/log/unattended-upgrades/unattended-upgrades-dpkg.log`

-----

## 6\. Vérification du service

`unattended-upgrades` est souvent exécuté via les *timers* de `systemd` appelés par les scripts périodiques d'APT. Vous pouvez vérifier l'état des *timers* associés :

```bash
systemctl list-timers apt-daily.timer apt-daily-upgrade.timer
```
