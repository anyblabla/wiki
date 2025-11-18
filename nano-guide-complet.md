---
title: Nano - L'éditeur léger, mais aux fonctionnalités robustes
description: Guide complet de GNU nano, l'éditeur de texte léger pour Linux. Maîtrisez les commandes de base, les fonctions avancées, et apprenez à configurer la coloration syntaxique et les numéros de ligne via le fichier .nanorc.
published: true
date: 2025-11-18T00:20:23.912Z
tags: nano, éditeur, nanorc
editor: markdown
dateCreated: 2025-11-18T00:08:30.133Z
---

## Introduction

**GNU nano** est un éditeur de texte en mode console (terminal) pour les systèmes de type **Unix** et **Linux**. C'est un clone libre de l'éditeur **Pico** de l'Université du Kansas. Il a été conçu pour être une alternative **simple à utiliser** et plus accessible que des éditeurs plus complexes comme **Vi/Vim** ou **Emacs**.

L'un des avantages majeurs de nano est l'affichage constant d'une ligne d'aide en bas de l'écran, listant les raccourcis les plus courants, ce qui le rend idéal pour les modifications rapides et pour les débutants.

-----

## Utilisation et commandes

### Utilisation de base

Pour lancer nano, ouvrez un terminal et tapez `nano`. Pour ouvrir un fichier spécifique ou en créer un nouveau, utilisez la syntaxe :

```bash
nano [nom_du_fichier]
```

Dans l'éditeur, les commandes sont indiquées par la notation **`^X`** (**Ctrl+X**) ou **`M-X`** (**Alt+X** ou Échappe+X).

### Commandes essentielles

| Commande (Raccourci) | Description |
| :--- | :--- |
| **`^G` (Ctrl+G)** | Affiche l'aide complète. |
| **`^O` (Ctrl+O)** | **Écrire** (sauvegarder) le fichier en cours. |
| **`^X` (Ctrl+X)** | **Quitter** nano. |
| **`^W` (Ctrl+W)** | Lancer une **recherche** de texte. |
| **`^K` (Ctrl+K)** | **Couper** (supprimer) la ligne courante. |
| **`^U` (Ctrl+U)** | **Décoller** (coller) le contenu du tampon. |
| **`^C` (Ctrl+C)** | Afficher la position courante du curseur (ligne, colonne). |

### Commandes avancées pour la manipulation de texte

| Commande (Raccourci) | Description |
| :--- | :--- |
| **`^R` (Ctrl+R)** | **Insérer** un autre fichier à l'emplacement du curseur. |
| **`M-R` (Alt+R)** | **Remplacer** une chaîne de caractères (après recherche). |
| **`M-A` (Alt+A)** | Commencer/terminer le **marquage de la région** (sélection de texte). |
| **`M-6` (Alt+6)** | **Copier** la région de texte sélectionnée. |
| **`M-\` (Alt+)** | Aller au début du fichier. |
| **`M-/` (Alt+/)** | Aller à la fin du fichier. |
| **`^A` (Ctrl+A)** | Aller au début de la ligne. |
| **`^E` (Ctrl+E)** | Aller à la fin de la ligne. |

-----

## Configuration avancée

La personnalisation de nano se fait via le fichier **`.nanorc`** situé dans le répertoire personnel de l'utilisateur (`~/.nanorc`).

### Exemple de fichier `.nanorc` complet et détaillé

```ini
## FICHIER DE CONFIGURATION POUR GNU NANO (~/.nanorc)

# =================================================================
# 1. AFFICHAGE et COMPORTEMENT GÉNÉRAL
# =================================================================
# Active l'affichage des numéros de ligne sur le côté gauche.
set linenumbers
# Affiche la position du curseur (ligne/colonne) de manière constante.
set constantshow
# Active le "softwrap" : les longues lignes sont coupées visuellement.
set softwrap
# Utilise 4 espaces au lieu d'une tabulation.
set tabsize 4
set tabstospaces
# Affiche le caractère d'espace de fin (trailing whitespace).
set afterends

# =================================================================
# 2. INDENTATION et SAUVEGARDE
# =================================================================
# Active l'indentation automatique des nouvelles lignes.
set autoindent
# Utilise des fichiers de sauvegarde (.nom_du_fichier.save).
set backup
set backupdir "/tmp"

# =================================================================
# 3. COLORATION SYNTAXIQUE (SYNTAX HIGHLIGHTING)
# =================================================================
# Pour inclure automatiquement tous les schémas de coloration disponibles sur le système.
include "/usr/share/nano/*.nanorc"

# ... (Le reste des personnalisations de couleurs ou de syntaxe spécifiques)
```

### Activer la coloration syntaxique

#### Importer toutes les configurations de syntaxe

Pour initialiser rapidement votre fichier `.nanorc` avec la coloration syntaxique et l'indentation automatique, vous pouvez utiliser la commande suivante :

```bash
echo "set autoindent" > ~/.nanorc ; ls -c1 /usr/share/nano | sed "s/^/include \"\/usr\/share\/nano\//g" | sed "s/$/\"/g" >> ~/.nanorc
```

> **⭐ Remarque Importante :** Utilisez soit la méthode de **copie de l'exemple complet** (dans la section ci-dessus), soit la méthode de la **commande rapide**, mais **jamais les deux** simultanément, car la commande écrase le fichier existant.

> **⚠️ Note Technique :** Les fichiers de coloration syntaxique sont inclus dans le paquet **`nano`** sur Debian 13. Si la coloration ne fonctionne pas, assurez-vous que la ligne `include "/usr/share/nano/*.nanorc"` est présente et décommentée dans votre fichier `~/.nanorc`.

#### Forcer un langage spécifique

Si le fichier n'a pas d'extension reconnue, vous pouvez forcer la coloration d'une syntaxe lors du lancement :

```bash
nano -Y langage fichier
```

-----

## 🚀 Cas d'usage courants

### 1\. Éditer un fichier système (avec sudo)

```bash
sudo nano /etc/ssh/sshd_config
```

### 2\. Commenter rapidement plusieurs lignes

1.  Appuyez sur **`M-A` (Alt+A)** pour commencer la sélection.
2.  Déplacez le curseur pour sélectionner le bloc.
3.  Appuyez sur **`M-3` (Alt+3)** pour commenter le bloc sélectionné.

### 3\. Aller directement à une ligne spécifique

```bash
nano +ligne,colonne nom_du_fichier
```

**Exemple :** `nano +42,5 mon_script.sh`

-----

## ❓ Foire aux questions (FAQ)

### Q : Comment afficher les numéros de ligne ?

**R :** Utilisez le raccourci **`M-#` (Alt+\#)**, ou ajoutez `set linenumbers` dans votre fichier `~/.nanorc`.

### Q : Pourquoi les longs fichiers apparaissent-ils sur une seule ligne ?

**R :** Le saut de ligne automatique (*softwrap*) est désactivé par défaut. Utilisez **`M-$` (Alt+$)** pour l'activer temporairement, ou ajoutez `set softwrap` dans `~/.nanorc`.

### Q : Comment annuler (undo) ma dernière modification ?

**R :** Utilisez le raccourci **`M-U` (Alt+U)** pour annuler, et **`M-E` (Alt+E)** pour rétablir.