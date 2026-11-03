---
title: Traitement d'Images (JPG) avec imgoptim
description: >-
  Cette page documente la fonction Bash imgoptim, un outil essentiel pour le
  redimensionnement et la compression des fichiers d'images JPG/JPEG.
published: true
date: '2026-11-03T00:07:24.085Z'
tags:
  - imgoptim
  - jpg
  - jpeg
  - convert
editor: markdown
dateCreated: '2025-10-29T23:38:04.792Z'
---

a## 💾 Installation : Où Placer le Code

La fonction `imgoptim` doit être placée dans le fichier de configuration de vos alias : **`~/.bash_aliases`** (dans votre répertoire personnel).

  * Vous pouvez ouvrir et modifier ce fichier directement avec la commande : `nano ~/.bash_aliases` (ou en utilisant votre alias `ba`).
  * Placez le bloc de code de la fonction avant la section des alias rapides (les lignes commençant par `alias jpgoptim...`).

### 📝 Le Bloc de Code à Insérer

Insérez le code suivant dans votre fichier **`~/.bash_aliases`** :

```bash
imgoptim () {
    if [ -z "$1" ] || [ -z "$2" ]; then
        echo "Erreur : Arguments manquants."
        echo "Utilisation : imgoptim [TAILLE_MAX_PX] [QUALITE_PERCENTAGE] [MODE_RECHERCHE(optionnel)]"
        return 1
    fi

    local RESIZE_CMD="-resize ${1}x${1}\>"
    local QUALITY_CMD="-quality ${2}%"
    local SEARCH_MODE="${3:-name}" # 'name' par défaut

    if [ "$SEARCH_MODE" = "regex" ]; then
        echo "Mode REGEX : Recherche *.jpg, *.jpeg (casse indifférente)."
        FIND_CMD="find . -regextype posix-extended -iregex '.*(jpeg|jpg)' -print0"
        # Utilisation de xargs pour gérer les espaces dans les noms de fichiers
        eval "$FIND_CMD | xargs -0 -I {} convert $RESIZE_CMD $QUALITY_CMD {} {}"
    else
        echo "Mode NAME : Recherche uniquement *.jpg (minuscules)."
        FIND_CMD="find . -name \"*.jpg\" -exec convert $RESIZE_CMD $QUALITY_CMD {} {} \;"
        eval "$FIND_CMD"
    fi

    echo "Optimisation terminée : ${1}px, ${2}% de qualité, Mode : $SEARCH_MODE."
}
```

-----

## ⚙️ Utilisation de la Fonction

Une fois le code inséré et le terminal actualisé (`source ~/.bash_aliases`), la fonction est utilisable directement ou via les alias rapides.

**Syntaxe :**

```bash
imgoptim [TAILLE_MAX_PX] [QUALITE_PERCENTAGE] [MODE_RECHERCHE (optionnel)]
```

| Argument | Rôle | Exemple | Note |
| :--- | :--- | :--- | :--- |
| **`TAILLE_MAX_PX`** | Résolution maximale du côté le plus long. | `2000` | Le redimensionnement n'est effectué que si l'image est **plus grande** que cette valeur (`>`). |
| **`QUALITE_PERCENTAGE`** | Qualité de la compression JPG (1-100%). | `80` | Utiliser `100` si vous voulez juste redimensionner sans perte de qualité. |
| **`MODE_RECHERCHE`** | Type de recherche de fichiers. | `regex` | `name` (Défaut) : Cherche uniquement `*.jpg` (minuscules). **`regex`** : Cherche `*.jpg`, `*.jpeg`, `*.JPG`, `*.JPEG`. |

-----

## 🚀 Raccourcis Simplifiés

Ces alias sont à placer *après* le bloc de la fonction `imgoptim` dans votre fichier `.bash_aliases` :

```bash
# Alias rapides appelant la fonction
alias jpgoptim2000='imgoptim 2000 80 name'
alias jpgoptim3000='imgoptim 3000 80 regex'
alias jpgoptimnextcloud='imgoptim 3468 80 regex'
```

**Exemple d'exécution :**

```bash
# Pour optimiser toutes les images du répertoire actuel avec une qualité de 80%
# et une taille maximale de 3000 pixels (en incluant les fichiers JPEG et JPG)
jpgoptim3000
```
