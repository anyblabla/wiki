---
title: Traduction via LibreTranslate avec Nautilus et raccourci clavier
description: >-
  Ce guide complet permet de configurer la traduction instantanée de fichiers ou
  de texte sélectionné sur GNOME/Wayland via votre LibreTranslate local.
  Utilisez un raccourci clavier ou le menu Nautilus pour obtenir la traduction
  intégrale.
published: true
date: '2026-11-03T00:08:16.866Z'
tags:
  - script
  - translate
  - libretranslate
  - nautilus
  - raccourci
  - clavier
editor: markdown
dateCreated: '2025-11-10T13:08:12.330Z'
---

Ce guide explique comment créer un script unifié pour traduire des documents entiers ou du texte sélectionné en utilisant votre instance LibreTranslate. Le résultat complet de la traduction est affiché dans une fenêtre Zenity dédiée.

> ⚠️ Bien que l'outil notify-send soit couramment utilisé pour les actions rapides (comme la confirmation ou les courtes notifications), il n'a pas été privilégié ici car les notifications GNOME limitent fortement le nombre de caractères affichés, empêchant de lire les traductions longues. Zenity permet, lui, de visualiser le texte intégral dans une fenêtre redimensionnable.

☝️ Avant d'aller plus loin, voici une autre page qui pourrait vous intéresser 👇

[Traduction unifiée pour Nautilus, Nemo, Thunar, Dolphin et Texte sélectionné avec LibreTranslate](/libretranslate-traduction-unifiee-gestionnaires-fichiers)

-----

## 1\. Prérequis et dépendances

Assurez-vous que les paquets suivants sont installés sur votre système :

| Outil | Description | Commande d'installation (Debian/Ubuntu) |
| :--- | :--- | :--- |
| **`curl`** | Outil de transfert de données pour l'appel à l'API. | `sudo apt install curl` |
| **`jq`** | Processeur JSON pour extraire la traduction de la réponse de l'API. | `sudo apt install jq` |
| **`zenity`** | Outil pour l'affichage graphique du texte traduit dans une fenêtre sans limite de caractères. | `sudo apt install zenity` |
| **`wl-clipboard`** | Outils de gestion du presse-papiers sous Wayland (`wl-paste`). | `sudo apt install wl-clipboard` |

-----

## 2\. Création du script unifié

Le script suivant gère deux modes de fonctionnement : la traduction d'un **fichier** (via Nautilus) et la traduction du **texte sélectionné** (via un raccourci clavier).

### A. Création et sauvegarde du fichier

Créez le fichier script dans votre dossier personnel d'exécutables :

```bash
mkdir -p ~/.local/bin
nano ~/.local/bin/translate_unified.sh
```

### B. Contenu du script (`translate_unified.sh`)

**Ceci est la version sans clé API par défaut :**

```bash
#!/bin/bash
# Script unifié : Traduit un fichier sélectionné (Nautilus) OU le contenu du presse-papiers/sélection (Raccourci).

# --- 1. Configuration de LibreTranslate ---
LIBRETRANSLATE_URL="http://127.0.0.1:5000/translate"
SOURCE_LANG="auto" # Détection automatique
TARGET_LANG="en"   # Langue cible (ajustez si besoin, ex: "fr")
# -------------------------------------

TEXT_TO_TRANSLATE=""
SOURCE_NAME="Texte Sélectionné"

# 2. Détection du contexte
if [ -n "$NAUTILUS_SCRIPT_SELECTED_FILE_PATHS" ]; then
    # SCÉNARIO 1 : Lancé depuis Nautilus (Fichier)
    SELECTED_FILE=$(echo "$NAUTILUS_SCRIPT_SELECTED_FILE_PATHS" | head -n 1)
    
    if [ -f "$SELECTED_FILE" ]; then
        # Lit le contenu du fichier et le met sur une seule ligne
        TEXT_TO_TRANSLATE=$(cat "$SELECTED_FILE" | tr '\n' ' ')
        SOURCE_NAME="Fichier $(basename "$SELECTED_FILE")"
    fi
else
    # SCÉNARIO 2 : Lancé par un raccourci personnalisé (Sélection/Presse-papiers)
    
    # 2.1. Essayer Wayland (wl-clipboard)
    if command -v wl-paste &> /dev/null; then
        # Tenter la SÉLECTION SIMPLE (texte en surbrillance)
        TEXT_TO_TRANSLATE=$(wl-paste --primary) 
        
        # Si vide, se rabattre sur le PRESSE-PAPIERS (Ctrl+C)
        if [ -z "$TEXT_TO_TRANSLATE" ]; then
             TEXT_TO_TRANSLATE=$(wl-paste) 
        fi

    # 2.2. Essayer X11 (xclip)
    elif command -v xclip &> /dev/null; then
        # Tenter la SÉLECTION SIMPLE
        TEXT_TO_TRANSLATE=$(xclip -selection primary -o) 
        
        # Si vide, se rabattre sur le PRESSE-PAPIERS
        if [ -z "$TEXT_TO_TRANSLATE" ]; then
             TEXT_TO_TRANSLATE=$(xclip -selection clipboard -o)
        fi
    else
        notify-send "Traduction LibreTranslate" "Erreur : Dépendance 'wl-clipboard' ou 'xclip' manquante."
        exit 1
    fi
fi

# 3. Vérification du contenu à traduire
if [ -z "$TEXT_TO_TRANSLATE" ]; then
    MESSAGE="Le contenu est vide. Sélectionnez un fichier ou copiez/sélectionnez du texte."
    notify-send "Traduction LibreTranslate" "$MESSAGE"
    exit 1
fi

# 4. Exécution de la traduction via l'API (sans clé API)
TRANSLATION_JSON=$(curl -s -X POST "$LIBRETRANSLATE_URL" \
     -H 'Content-Type: application/json' \
     -d "{
      \"q\": \"$TEXT_TO_TRANSLATE\",
      \"source\": \"$SOURCE_LANG\",
      \"target\": \"$TARGET_LANG\"
     }")

# 5. Extraction du texte traduit
TRANSLATED_TEXT=$(echo "$TRANSLATION_JSON" | jq -r '.translatedText')

# 6. Affichage du résultat via Zenity (texte complet)
if [ "$TRANSLATED_TEXT" != "null" ] && [ -n "$TRANSLATED_TEXT" ]; then
    
    # Utilise Zenity pour afficher le texte intégral dans une fenêtre redimensionnable
    zenity --text-info \
        --title="Traduction ($SOURCE_NAME)" \
        --width=600 \
        --height=400 \
        --filename=<(echo -e "--- Traduction complète ---\n\n$TRANSLATED_TEXT")

else
    notify-send "Erreur de Traduction" "Impossible de traduire. Vérifiez votre service LibreTranslate ou la connexion."
fi
```

### C. Permissions d'exécution

Rendez le script exécutable :

```bash
chmod +x ~/.local/bin/translate_unified.sh
```

-----

## 3\. Configuration et personnalisation

### Explication des paramètres modifiables

| Variable | Description | Personnalisation |
| :--- | :--- | :--- |
| `LIBRETRANSLATE_URL` | L'**adresse complète** de l'API de traduction de votre instance LibreTranslate. | Modifiez pour utiliser une adresse locale (`http://192.168.1.10:5000/translate`) ou un **nom de domaine/adresse externe** (`https://traduction.mon-domaine.com/translate`). Le chemin doit se terminer par `/translate`. |
| `SOURCE_LANG` | Définit la **langue du texte source**. | Laissez `"auto"` pour la **détection automatique**, ou utilisez un code ISO 639-1 (ex: `"de"` pour l'allemand). |
| `TARGET_LANG` | Définit la **langue cible** de la traduction. | Remplacez le code actuel par le **code ISO 639-1** de la langue souhaitée (ex: `"fr"` pour le français). |

-----

## 4\. Intégration dans le système

### A. Pour la traduction de fichiers (Nautilus)

Pour ajouter l'option au menu contextuel du gestionnaire de fichiers :

```bash
mkdir -p ~/.local/share/nautilus/scripts/
# Crée un lien symbolique vers le script unifié
ln -sf ~/.local/bin/translate_unified.sh ~/.local/share/nautilus/scripts/Traduire_fichier_entier
```

**Utilisation :** Faites un clic droit sur n'importe quel fichier texte dans Nautilus, allez dans **Scripts** et choisissez **Traduire fichier entier**.

### B. Pour la traduction de texte sélectionné (Raccourci clavier)

Pour lancer le script depuis n'importe quelle application :

1.  Ouvrez les **Paramètres** de GNOME.
2.  Allez dans la section **Clavier** → **Raccourcis personnalisés**.
3.  Ajoutez un nouveau raccourci :
      * **Nom :** `Traduire texte sélectionné`
      * **Commande :** Entrez le chemin complet du script (remplacez `VOTRE_NOM_UTILISATEUR`) :
        ```
        /home/VOTRE_NOM_UTILISATEUR/.local/bin/translate_unified.sh
        ```
      * **Raccourci :** Définissez la combinaison de touches souhaitée (par exemple, **`Super`+`T`**).

**Utilisation :** Sélectionnez du texte dans n'importe quelle application et appuyez sur le raccourci clavier défini. La traduction complète apparaîtra dans une fenêtre Zenity.

-----

## 5\. 💡 Options non présentes mais utiles

LibreTranslate prend en charge d'autres paramètres pour affiner la requête. Si votre instance LibreTranslate nécessite une clé API, vous devez ajuster le script.

### 5.1. Gestion du format de contenu

Par défaut, le script considère le texte comme du texte brut (`text`). Si vous traduisez du contenu HTML, vous devez ajouter le paramètre `"format": "html"` à la requête `curl` (étape 4).

### 5.2. Utilisation d'une clé API

Si votre instance LibreTranslate nécessite une clé d'authentification, suivez ces étapes :

1.  **Ajoutez la variable `LIBRETRANSLATE_API_KEY` dans la section 1 du script :**
    ```bash
    LIBRETRANSLATE_API_KEY="VOTRE_CLÉ_SECRÈTE"
    ```
2.  **Modifiez la requête `curl` (étape 4) pour inclure la clé :**
    ```bash
    # (Extrait de l'étape 4 du script)
    TRANSLATION_JSON=$(curl -s -X POST "$LIBRETRANSLATE_URL" \
         -H 'Content-Type: application/json' \
         -d "{
          \"q\": \"$TEXT_TO_TRANSLATE\",
          \"source\": \"$SOURCE_LANG\",
          \"target\": \"$TARGET_LANG\",
          \"api_key\": \"$LIBRETRANSLATE_API_KEY\"
         }")
    ```

-----

## 6\. Exemple de script complet avec Clé API

Pour référence, si vous souhaitez utiliser immédiatement la version du script avec une clé API requise, voici le contenu complet.

Ce script doit être sauvegardé sous `~/.local/bin/translate_unified.sh`.

```bash
#!/bin/bash
# Script unifié : Traduit un fichier sélectionné (Nautilus) OU le contenu du presse-papiers/sélection (Raccourci).

# --- 1. Configuration de LibreTranslate ---
LIBRETRANSLATE_URL="http://127.0.0.1:5000/translate"
SOURCE_LANG="auto"        # Détection automatique
TARGET_LANG="en"          # Langue cible (ajustez si besoin, ex: "fr")
LIBRETRANSLATE_API_KEY="VOTRE_CLÉ_SECRÈTE" # <-- N'OUBLIEZ PAS DE REMPLACER VOTRE_CLÉ_SECRÈTE
# -------------------------------------

TEXT_TO_TRANSLATE=""
SOURCE_NAME="Texte Sélectionné"

# 2. Détection du contexte
if [ -n "$NAUTILUS_SCRIPT_SELECTED_FILE_PATHS" ]; then
    # SCÉNARIO 1 : Lancé depuis Nautilus (Fichier)
    SELECTED_FILE=$(echo "$NAUTILUS_SCRIPT_SELECTED_FILE_PATHS" | head -n 1)
    
    if [ -f "$SELECTED_FILE" ]; then
        TEXT_TO_TRANSLATE=$(cat "$SELECTED_FILE" | tr '\n' ' ')
        SOURCE_NAME="Fichier $(basename "$SELECTED_FILE")"
    fi
else
    # SCÉNARIO 2 : Lancé par un raccourci personnalisé (Sélection/Presse-papiers)
    
    # 2.1. Essayer Wayland (wl-clipboard)
    if command -v wl-paste &> /dev/null; then
        # Tenter la SÉLECTION SIMPLE
        TEXT_TO_TRANSLATE=$(wl-paste --primary) 
        
        # Si vide, se rabattre sur le PRESSE-PAPIERS
        if [ -z "$TEXT_TO_TRANSLATE" ]; then
             TEXT_TO_TRANSLATE=$(wl-paste) 
        fi

    # 2.2. Essayer X11 (xclip)
    elif command -v xclip &> /dev/null; then
        # Tenter la SÉLECTION SIMPLE
        TEXT_TO_TRANSLATE=$(xclip -selection primary -o) 
        
        # Si vide, se rabattre sur le PRESSE-PAPIERS
        if [ -z "$TEXT_TO_TRANSLATE" ]; then
             TEXT_TO_TRANSLATE=$(xclip -selection clipboard -o)
        fi
    else
        notify-send "Traduction LibreTranslate" "Erreur : Dépendance 'wl-clipboard' ou 'xclip' manquante."
        exit 1
    fi
fi

# 3. Vérification du contenu à traduire
if [ -z "$TEXT_TO_TRANSLATE" ]; then
    MESSAGE="Le contenu est vide. Sélectionnez un fichier ou copiez/sélectionnez du texte."
    notify-send "Traduction LibreTranslate" "$MESSAGE"
    exit 1
fi

# 4. Exécution de la traduction via l'API (AVEC CLÉ API)
TRANSLATION_JSON=$(curl -s -X POST "$LIBRETRANSLATE_URL" \
     -H 'Content-Type: application/json' \
     -d "{
      \"q\": \"$TEXT_TO_TRANSLATE\",
      \"source\": \"$SOURCE_LANG\",
      \"target\": \"$TARGET_LANG\",
      \"api_key\": \"$LIBRETRANSLATE_API_KEY\"
     }")

# 5. Extraction du texte traduit
TRANSLATED_TEXT=$(echo "$TRANSLATION_JSON" | jq -r '.translatedText')

# 6. Affichage du résultat via Zenity (texte complet)
if [ "$TRANSLATED_TEXT" != "null" ] && [ -n "$TRANSLATED_TEXT" ]; then
    
    zenity --text-info \
        --title="Traduction ($SOURCE_NAME)" \
        --width=600 \
        --height=400 \
        --filename=<(echo -e "--- Traduction complète ---\n\n$TRANSLATED_TEXT")

else
    notify-send "Erreur de Traduction" "Impossible de traduire. Vérifiez votre service LibreTranslate ou la clé API."
fi
```
---

## 7\. Vidéo

Une vidéo de démonstration existe, et celle-ci à été publié sur les réseaux-sociaux Blabla Linux 😎

- [Sur Facebook](https://www.facebook.com/share/v/17K7SU2Db4/)
- [Sur Twitter (X)](https://x.com/BlablaLinux/status/1987910094017753265?s=20)
- [Sur Bluesky](https://bsky.app/profile/blablalinux.be/post/3m5bxvgholk2q)
- [Sur Mastodon](https://mastodon.blablalinux.be/@blablalinux/115526187793773014)
