---
title: Maîtriser Fastfetch : le guide de personnalisation par Blabla Linux
description: Je vous présente mon guide complet pour installer et personnaliser Fastfetch. Découvrez mes quatre modèles de configuration exclusifs, du style classique au look néon cyberpunk pour Linux.
published: true
date: 2026-01-03T02:32:53.330Z
tags: debian, ubuntu, personnalisation, fastfetch, terminal, ressource
editor: markdown
dateCreated: 2026-01-03T00:55:28.798Z
---

Je vous partage ici mon expérience avec **Fastfetch**, l'outil que j'utilise au quotidien pour afficher les informations système de mes machines. C'est, selon moi, la meilleure alternative à Neofetch pour allier rapidité et design.

## Installation de l'outil

Pour commencer, je vous explique comment l'installer sur vos systèmes basés sur Debian ou Ubuntu.

* **Sur Debian (Trixie/Sid) et Ubuntu (24.04+) :**

```bash
sudo apt update && sudo apt install fastfetch

```

* **Via PPA pour les versions plus anciennes :**

```bash
sudo add-apt-repository ppa:zhangsongcui3371/fastfetch
sudo apt update && sudo apt install fastfetch

```

## Gestion de ma configuration

Je centralise toutes mes modifications dans un seul fichier. Voici comment je procède pour appliquer mes réglages.

### Créer le fichier de base

Si vous voulez voir toutes les options disponibles, je vous conseille de générer le fichier par défaut :

```bash
fastfetch --gen-config

```

Le fichier se trouvera dans votre dossier personnel : `~/.config/fastfetch/config.jsonc`.

### Utiliser mes réglages personnalisés

1. Je m'assure que le dossier existe : `mkdir -p ~/.config/fastfetch`
2. J'édite le fichier avec mon éditeur favori : `nano ~/.config/fastfetch/config.jsonc`
3. Je remplace tout le contenu par l'un de mes modèles ci-dessous.

---

## 🎨 Aller plus loin : Les Nerd Fonts (Icônes HD)

Si vous voulez remplacer les émojis classiques par des icônes haute définition (logos de distribs, processeurs, etc.), vous devez installer une **Nerd Font**.

### 1. Installation rapide (Script)

Voici une commande pour installer la police **Hack Nerd Font** (très lisible) sur votre système :

```bash
mkdir -p ~/.local/share/fonts && cd ~/.local/share/fonts
wget https://github.com/ryanoasis/nerd-fonts/releases/latest/download/Hack.zip
unzip Hack.zip && rm Hack.zip && fc-cache -fv

```

### 2. L'astuce pour un rendu parfait

Inutile de forcer la police dans les réglages de votre terminal ! Dans les préférences de votre terminal (comme GNOME Terminal), **décochez "Police personnalisée"**. Le système utilisera sa police par défaut pour le texte et piochera automatiquement dans la Nerd Font fraîchement installée pour afficher les icônes.

> 💡 **Voici le résultat du rendu avec les Nerd Fonts sur mon terminal :**
> ![fastfetch-neon-cyber-nerd.jpg](/guide-personnalisation-fastfetch-linux/fastfetch-neon-cyber-nerd.jpg)

---

## Mes modèles de configuration

### 1. Style classique et épuré

C'est le modèle le plus complet en termes de données brutes. Je l'utilise quand j'ai besoin d'un rapport détaillé sans fioritures.

<details>
<summary>Voir le code (Classique)</summary>

```jsonc
// # Modifications apportées par Blabla Linux : https://link.blablalinux.be
{
    "$schema": "https://github.com/fastfetch-cli/fastfetch/raw/dev/doc/json_schema.json",
    "display": { "separator": " ➜  " },
    "modules": [
        "title", "separator",
        { "type": "os", "key": "Système     " },
        { "type": "host", "key": "Machine     " },
        { "type": "kernel", "key": "Noyau       " },
        { "type": "uptime", "key": "Activité    " },
        { "type": "packages", "key": "Paquets     " },
        { "type": "shell", "key": "Shell       " },
        { "type": "display", "key": "Écran       " },
        { "type": "de", "key": "Bureau      " },
        { "type": "terminal", "key": "Terminal    " },
        { "type": "cpu", "key": "Processeur  ", "temp": true },
        { "type": "gpu", "key": "Graphique   ", "hideType": "all", "format": "{1} {2}" },
        { "type": "memory", "key": "Mémoire     " },
        { "type": "swap", "key": "Swap        " },
        { "type": "disk", "key": "Disque      " },
        { "type": "localip", "key": "IP locale   ", "showIpv6": false },
        "break",
        { "type": "battery", "key": "Batterie    " },
        { "type": "poweradapter", "key": "Secteur     " },
        "colors"
    ]
}

```

</details>

![fastfetch-classic.jpg](/guide-personnalisation-fastfetch-linux/fastfetch-classic.jpg)

### 2. Style moderne avec émojis

Parfait pour une identification visuelle instantanée des composants.

<details>
<summary>Voir le code (Émojis)</summary>

```jsonc
// # Modifications apportées par Blabla Linux : https://link.blablalinux.be
{
    "$schema": "https://github.com/fastfetch-cli/fastfetch/raw/dev/doc/json_schema.json",
    "display": { "separator": " ➜  " },
    "modules": [
        "title", "separator",
        { "type": "os", "key": "🐧 Système     " },
        { "type": "host", "key": "💻 Machine     " },
        { "type": "kernel", "key": "⚙️  Noyau       " },
        { "type": "uptime", "key": "⏱️  Activité    " },
        { "type": "packages", "key": "📦 Paquets     " },
        { "type": "shell", "key": "🐚 Shell       " },
        { "type": "cpu", "key": "🧠 Processeur  ", "temp": true },
        { "type": "gpu", "key": "🎮 Graphique   ", "hideType": "all", "format": "{1} {2}" },
        { "type": "memory", "key": "💾 Mémoire     " },
        { "type": "disk", "key": "💽 Disque      " },
        { "type": "localip", "key": "🌐 IP locale   ", "showIpv6": false },
        "break",
        { "type": "battery", "key": "🔋 Batterie    " },
        { "type": "poweradapter", "key": "🔌 Power       " },
        "colors"
    ]
}

```

</details>

![fastfetch-emoji.jpg](/guide-personnalisation-fastfetch-linux/fastfetch-emoji.jpg)

### 3. Style tableau de bord (Dashboard)

Ce modèle organise l'information par catégories.

<details>
<summary>Voir le code (Tableau de bord)</summary>

```jsonc
// # Modifications apportées par Blabla Linux : https://link.blablalinux.be
{
    "$schema": "https://github.com/fastfetch-cli/fastfetch/raw/dev/doc/json_schema.json",
    "display": { "separator": " ", "color": { "keys": "magenta" } },
    "modules": [
        "title",
        { "type": "custom", "format": "┌─ Informations système ────────────────────────────────" },
        { "type": "os", "key": "│ 🐧 Système  ", "format": "{3} {8}" },
        { "type": "kernel", "key": "│ ⚙️  Noyau    ", "format": "{1} {2}" },
        { "type": "uptime", "key": "│ ⏱️  Activité ", "format": "{1}{2} {3}{4}" },
        { "type": "custom", "format": "├─ Matériel et température ─────────────────────────────" },
        { "type": "host", "key": "│ 💻 Machine  " },
        { "type": "cpu", "key": "│ 🧠 CPU      ", "temp": true, "format": "{6} @ {7} - {8}" },
        { "type": "gpu", "key": "│ 🎮 GPU      ", "hideType": "all", "format": "{2}" },
        { "type": "memory", "key": "│ 💾 Mémoire  " },
        { "type": "custom", "format": "├─ Stockage et réseau ──────────────────────────────────" },
        { "type": "disk", "key": "│ 💽 Disque   ", "folders": "/" },
        { "type": "localip", "key": "│ 🌐 IP v4    ", "showIpv6": false },
        { "type": "custom", "format": "└───────────────────────────────────────────────────────" },
        "break", "colors"
    ]
}

```

</details>

![fastfetch-open-dashboard.jpg](/guide-personnalisation-fastfetch-linux/fastfetch-open-dashboard.jpg)

### 4. Style Néon Cyberpunk

Ma création favorite avec des bannières de couleurs ANSI.

#### Option A : Version Émojis (Universelle)

<details>
<summary>Voir le code (Néon Émojis)</summary>

```jsonc
// # Modifications apportées par Blabla Linux : https://link.blablalinux.be
{
    "$schema": "https://github.com/fastfetch-cli/fastfetch/raw/dev/doc/json_schema.json",
    "display": { "separator": " ➜ ", "color": { "keys": "cyan", "output": "white" } },
    "modules": [
        "title",
        { "type": "custom", "format": " \u001b[46m\u001b[30m ARCHITECTURE SYSTÈME \u001b[0m", "key": " " },
        { "type": "os", "key": "  🐧 Système  ", "format": "{3} {8}" },
        { "type": "kernel", "key": "  ⚙️  Noyau    ", "format": "{1} {2}" },
        { "type": "shell", "key": "  🐚 Shell    " },
        { "type": "packages", "key": "  📦 Paquets   " },
        "break",
        { "type": "custom", "format": " \u001b[45m\u001b[30m RESSOURCES MATÉRIELLES \u001b[0m", "key": " " },
        { "type": "host", "key": "  💻 Machine  " },
        { "type": "cpu", "key": "  🧠 CPU      ", "temp": true, "format": "{6} @ {7} - {8}" },
        { "type": "gpu", "key": "  🎮 GPU      ", "hideType": "all", "format": "{2}" },
        { "type": "memory", "key": "  💾 Mémoire  ", "format": "{1} / {2} ({3})" },
        "break",
        { "type": "custom", "format": " \u001b[42m\u001b[30m RÉSEAU ET STATUT      \u001b[0m", "key": " " },
        { "type": "localip", "key": "  🌐 IP v4    ", "showIpv6": false },
        { "type": "battery", "key": "  🔋 Énergie  ", "format": "{4} ({5})" },
        "break", "colors"
    ]
}

```

</details>

![fastfetch-neon-cyber.jpg](/guide-personnalisation-fastfetch-linux/fastfetch-neon-cyber.jpg)

#### Option B : Version Nerd Fonts (Icônes HD)

<details>
<summary>Voir le code (Néon Nerd Fonts)</summary>

```jsonc
// # Modifications apportées par Blabla Linux : https://link.blablalinux.be
{
    "$schema": "https://github.com/fastfetch-cli/fastfetch/raw/dev/doc/json_schema.json",
    "display": { "separator": " ➜ ", "color": { "keys": "cyan", "output": "white" } },
    "modules": [
        "title",
        { "type": "custom", "format": " \u001b[46m\u001b[30m ARCHITECTURE SYSTÈME \u001b[0m", "key": " " },
        { "type": "os", "key": "   Système  ", "format": "{3} {8}" },
        { "type": "kernel", "key": "  󰒋 Noyau    ", "format": "{1} {2}" },
        { "type": "shell", "key": "  󱆃 Shell    " },
        { "type": "packages", "key": "  󰏖 Paquets   " },
        "break",
        { "type": "custom", "format": " \u001b[45m\u001b[30m RESSOURCES MATÉRIELLES \u001b[0m", "key": " " },
        { "type": "host", "key": "  󰌢 Machine  " },
        { "type": "cpu", "key": "  󰻠 CPU      ", "temp": true, "format": "{6} @ {7} - {8}" },
        { "type": "gpu", "key": "  󰢮 GPU      ", "hideType": "all", "format": "{2}" },
        { "type": "memory", "key": "  󰍛 Mémoire  ", "format": "{1} / {2} ({3})" },
        "break",
        { "type": "custom", "format": " \u001b[42m\u001b[30m RÉSEAU ET STATUT      \u001b[0m", "key": " " },
        { "type": "localip", "key": "  󰩟 IP v4    ", "showIpv6": false },
        { "type": "battery", "key": "  󰁹 Énergie  ", "format": "{4} ({5})" },
        "break", "colors"
    ]
}

```

</details>

> ☝️ Je vous invite à retrouver ces différents fichiers de configuration Fastfetch sur <a href="[https://bytestash.blablalinux.be/public/snippets](https://bytestash.blablalinux.be/public/snippets)" target="_blank" rel="noopener noreferrer">mon instance ByteStash</a> ✔️

> ☝️ Vous trouverez d'autres exemples de fichiers `config.jsonc` pour Fastfetch sur la **[page du projet GitHub](https://github.com/fastfetch-cli/fastfetch){.target-blank}** : [https://github.com/fastfetch-cli/fastfetch/tree/dev/presets/examples](https://github.com/fastfetch-cli/fastfetch/tree/dev/presets/examples){.target-blank}