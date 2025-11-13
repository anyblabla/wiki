---
title: Traduction unifiée pour Nautilus, Nemo, Thunar, Dolphin et Texte sélectionné avec LibreTranslate
description: Script Bash pour traduction unifiée (texte ou fichier) avec LibreTranslate. Intégration facile dans Nautilus, Nemo, Thunar, et Dolphin. Affiche les traductions complètes dans une fenêtre Zenity redimensionnable. Installation simple.
published: true
date: 2025-11-13T18:42:53.286Z
tags: 
editor: markdown
dateCreated: 2025-11-10T17:35:33.344Z
---

Ce guide explique comment créer un script unifié pour traduire des documents entiers ou du texte sélectionné en utilisant votre instance LibreTranslate. Le résultat complet de la traduction est affiché dans une fenêtre Zenity dédiée.

> ⚠️ Bien que l'outil notify-send soit couramment utilisé pour les actions rapides (comme la confirmation ou les courtes notifications), il n'a pas été privilégié ici car la plupart des systèmes de notification d'environnement de bureau (GNOME, Plasma, XFCE) limitent fortement le nombre de caractères affichés, empêchant de lire les traductions longues. Zenity permet, lui, de visualiser le texte intégral dans une fenêtre redimensionnable.

-----

## 1\. Prérequis et dépendances

Assurez-vous que les paquets suivants sont installés sur votre système :

| Outil | Description | Commande d'installation (Debian/Ubuntu) |
| :--- | :--- | :--- |
| **`curl`** | Outil de transfert de données pour l'appel à l'API. | `sudo apt install curl` |
| **`jq`** | Processeur JSON pour extraire la traduction de la réponse de l'API. | `sudo apt install jq` |
| **`zenity`** | Outil pour l'affichage graphique du texte traduit dans une fenêtre sans limite de caractères. | `sudo apt install zenity` |
| **`wl-clipboard`** | Outils de gestion du presse-papiers sous Wayland (`wl-paste`). | `sudo apt install wl-clipboard` |
| **`xclip`** | Outils de gestion du presse-papiers sous X11 (Alternative à `wl-clipboard`). | `sudo apt install xclip` |

-----

## 2\. Création du script unifié

Le script suivant gère trois modes de fonctionnement : la traduction d'un **fichier sélectionné comme argument** (Thunar, Dolphin), la traduction d'un **fichier via variable d'environnement** (Nautilus, Nemo), et la traduction du **texte sélectionné/presse-papiers** (via un raccourci clavier).

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
# Script unifié : Traduit un fichier (via argument ou variable Nautilus/Nemo) OU le contenu du presse-papiers/sélection (Raccourci).

# --- 1. Configuration de LibreTranslate ---
LIBRETRANSLATE_URL="http://127.0.0.1:5000/translate"
SOURCE_LANG="auto" # Détection automatique
TARGET_LANG="en"   # Langue cible (ajustez si besoin, ex: "fr")
# -------------------------------------

TEXT_TO_TRANSLATE=""
SOURCE_NAME="Texte Sélectionné"

# 2. Détection du contexte

# SCÉNARIO 1 : Lancé depuis un gestionnaire de fichiers comme argument (Thunar, Dolphin)
if [ $# -gt 0 ]; then
    # Prend le premier argument (le chemin du fichier)
    SELECTED_FILE="$1" 
    
    if [ -f "$SELECTED_FILE" ]; then
        TEXT_TO_TRANSLATE=$(cat "$SELECTED_FILE" | tr '\n' ' ')
        SOURCE_NAME="Fichier $(basename "$SELECTED_FILE")"
    fi

# SCÉNARIO 2 : Lancé depuis Nautilus/Nemo (utilise la variable d'environnement)
elif [ -n "$NAUTILUS_SCRIPT_SELECTED_FILE_PATHS" ]; then
    SELECTED_FILE=$(echo "$NAUTILUS_SCRIPT_SELECTED_FILE_PATHS" | head -n 1)
    
    if [ -f "$SELECTED_FILE" ]; then
        # Lit le contenu du fichier et le met sur une seule ligne
        TEXT_TO_TRANSLATE=$(cat "$SELECTED_FILE" | tr '\n' ' ')
        SOURCE_NAME="Fichier $(basename "$SELECTED_FILE")"
    fi
else
    # SCÉNARIO 3 : Lancé par un raccourci personnalisé (Sélection/Presse-papiers)
    
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
    MESSAGE="Le contenu est vide. Sélectionnez un fichier, ou copiez/sélectionnez du texte."
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

### 4.1. Pour la traduction de texte sélectionné (raccourci clavier)

Pour lancer le script depuis n'importe quelle application :

1.  Ouvrez les **Paramètres** de votre environnement de bureau (GNOME ou Cinnamon).
2.  Allez dans la section **Clavier** → **Raccourcis personnalisés**.
3.  Ajoutez un nouveau raccourci :
      * **Nom :** `Traduire texte sélectionné`
      * **Commande :** Entrez le chemin complet du script (remplacez `VOTRE_NOM_UTILISATEUR`) :
        ```
        /home/VOTRE_NOM_UTILISATEUR/.local/bin/translate_unified.sh
        ```
      * **Raccourci :** Définissez la combinaison de touches souhaitée (par exemple, **`Super`+`T`**).

**Utilisation :** Sélectionnez du texte dans n'importe quelle application et appuyez sur le raccourci clavier défini. La traduction complète apparaîtra dans une fenêtre Zenity.

### 4.2. Intégration dans les gestionnaires de fichiers

Le script unifié fonctionne différemment selon le gestionnaire de fichiers : Nautilus/Nemo utilisent une variable d'environnement, tandis que Thunar/Dolphin utilisent des arguments de ligne de commande.

#### 4.2.1. GNOME Files (Nautilus)

Nautilus utilise le répertoire `scripts`. Créez un lien symbolique vers votre script unifié :

```bash
mkdir -p ~/.local/share/nautilus/scripts/
# Crée un lien symbolique vers le script unifié
ln -sf ~/.local/bin/translate_unified.sh ~/.local/share/nautilus/scripts/Traduire_fichier_entier
```

**Utilisation :** Clic droit sur un fichier → **Scripts** → **Traduire fichier entier**.

#### 4.2.2. Nemo (Linux Mint)

Nemo est un fork de Nautilus et utilise la même logique et la même variable d'environnement, mais un chemin différent :

```bash
mkdir -p ~/.local/share/nemo/scripts/
# Crée un lien symbolique vers le script unifié
ln -sf ~/.local/bin/translate_unified.sh ~/.local/share/nemo/scripts/Traduire_fichier_entier
```

**Utilisation :** Clic droit sur un fichier → **Scripts** → **Traduire fichier entier**.

#### 4.2.3. Thunar (XFCE)

Thunar utilise les **Actions personnalisées** (Custom Actions) qui appellent directement le script avec le chemin du fichier en argument (`%f`). L'approche la plus simple est d'utiliser l'interface graphique :

1.  Ouvrez Thunar.
2.  Allez dans le menu **Édition** → **Configurer les actions personnalisées...**
3.  Cliquez sur le bouton **Ajouter (+)**.
4.  **Onglet "De base" :**
      * **Nom :** `Traduire fichier entier (LibreTranslate)`
      * **Commande :** Entrez le chemin complet du script, en utilisant `%f` pour passer le fichier sélectionné comme premier argument :
        ```bash
        /home/VOTRE_NOM_UTILISATEUR/.local/bin/translate_unified.sh %f
        ```
      * **Icône :** Choisissez une icône appropriée, par exemple `accessories-text-editor`.
5.  **Onglet "Conditions d'apparition" :**
      * **Modèles de fichier :** `*` (pour tous les fichiers).
      * **Conditions d'apparition :** Cochez uniquement `Fichiers` et éventuellement `Autres` si vous voulez que l'option apparaisse sur le bureau.

**Utilisation :** Clic droit sur un fichier → **Traduire fichier entier (LibreTranslate)**.

#### 4.2.4. Dolphin (KDE)

Dolphin utilise des **Menus de service** (`.desktop` files) pour les actions contextuelles. Créez un fichier `.desktop` dans le répertoire des services :

```bash
mkdir -p ~/.local/share/kio/servicemenus/
nano ~/.local/share/kio/servicemenus/libretranslate_file.desktop
```

**Contenu du fichier `libretranslate_file.desktop` :**

```desktop
[Desktop Entry]
Type=Service
Name=Traduire Fichier Entier (LibreTranslate)
Comment=Envoie le fichier sélectionné à LibreTranslate et affiche le résultat
Icon=edit-paste
Exec=/home/VOTRE_NOM_UTILISATEUR/.local/bin/translate_unified.sh %f
Path=
Terminal=false
Actions=Translate
Selection-File=true
MimeType=text/plain;application/xml;text/html;
```

> **Note :** N'oubliez pas de remplacer `/home/VOTRE_NOM_UTILISATEUR/` par votre chemin réel.

**Utilisation :** Clic droit sur un fichier texte → **Actions** → **Traduire Fichier Entier (LibreTranslate)**.

-----

## 5\. Débogage : résoudre le problème du "presse-papiers vide" (Ubuntu/Wayland)

Si, lors de l'utilisation du raccourci clavier, vous obtenez systématiquement un message d'erreur indiquant que la sélection est vide, cela est souvent lié à la gestion de la **sélection primaire** (texte surligné) sous Wayland ou GNOME, qui ne remonte pas correctement au script.

### Solution 1 : Utiliser le presse-papiers classique (recommandé)

Le presse-papiers classique (copier/coller : **`Ctrl+C`** et **`Ctrl+V`**) est généralement plus fiable que la sélection primaire.

**Instruction :** Assurez-vous d'avoir bien appuyé sur **`Ctrl+C`** sur le texte désiré avant de lancer votre raccourci de traduction (**`Super`+`T`** par exemple).

### Solution 2 : Modifier le script pour forcer le presse-papiers standard

Si vous souhaitez que le script se rabatte immédiatement sur le presse-papiers classique (`Ctrl+C`) au lieu de tenter la sélection primaire (surbrillance), vous pouvez modifier le script `~/.local/bin/translate_unified.sh`.

1.  Ouvrez le fichier : `nano ~/.local/bin/translate_unified.sh`

2.  Localisez la section **SCÉNARIO 3** (lignes 44 à 66 environ) et remplacez-la par cette version modifiée et plus robuste qui priorise le presse-papiers standard :

<!-- end list -->

```bash
else
    # SCÉNARIO 3 : Lancé par un raccourci personnalisé (Sélection/Presse-papiers)
    
    # 2.1. Essayer Wayland (wl-clipboard)
    if command -v wl-paste &> /dev/null; then
        # On force la lecture du presse-papiers standard (copié via Ctrl+C)
        TEXT_TO_TRANSLATE=$(wl-paste) 
        
        # Si vide, on tente tout de même la sélection primaire par sécurité
        if [ -z "$TEXT_TO_TRANSLATE" ]; then
             TEXT_TO_TRANSLATE=$(wl-paste --primary) 
        fi

    # 2.2. Essayer X11 (xclip)
    elif command -v xclip &> /dev/null; then
        # On force la lecture du presse-papiers standard
        TEXT_TO_TRANSLATE=$(xclip -selection clipboard -o) 
        
        # Si vide, on se rabat sur la sélection primaire
        if [ -z "$TEXT_TO_TRANSLATE" ]; then
             TEXT_TO_TRANSLATE=$(xclip -selection primary -o)
        fi
    else
        notify-send "Traduction LibreTranslate" "Erreur : Dépendance 'wl-clipboard' ou 'xclip' manquante."
        exit 1
    fi
fi
```

Cette modification assure que le presse-papiers standard est priorisé, ce qui est souvent plus fiable pour les grandes quantités de texte.

-----

## 6\. 💡 Options non présentes mais utiles

LibreTranslate prend en charge d'autres paramètres pour affiner la requête. Si votre instance LibreTranslate nécessite une clé API, vous devez ajuster le script.

### 6.1. Gestion du format de contenu

Par défaut, le script considère le texte comme du texte brut (`text`). Si vous traduisez du contenu HTML, vous devez ajouter le paramètre `"format": "html"` à la requête `curl` (étape 4).

### 6.2. Utilisation d'une clé API

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

## 7\. Exemple de script complet avec clé API

Pour référence, si vous souhaitez utiliser immédiatement la version du script avec une clé API requise, voici le contenu complet.

Ce script doit être sauvegardé sous `~/.local/bin/translate_unified.sh`.

```bash
#!/bin/bash
# Script unifié : Traduit un fichier (via argument ou variable Nautilus/Nemo) OU le contenu du presse-papiers/sélection (Raccourci).

# --- 1. Configuration de LibreTranslate ---
LIBRETRANSLATE_URL="http://127.0.0.1:5000/translate"
SOURCE_LANG="auto"        # Détection automatique
TARGET_LANG="en"          # Langue cible (ajustez si besoin, ex: "fr")
LIBRETRANSLATE_API_KEY="VOTRE_CLÉ_SECRÈTE" # <-- N'OUBLIEZ PAS DE REMPLACER VOTRE_CLÉ_SECRÈTE
# -------------------------------------

TEXT_TO_TRANSLATE=""
SOURCE_NAME="Texte Sélectionné"

# 2. Détection du contexte

# SCÉNARIO 1 : Lancé depuis un gestionnaire de fichiers comme argument ($1 pour Thunar/Dolphin)
if [ $# -gt 0 ]; then
    SELECTED_FILE="$1" 
    
    if [ -f "$SELECTED_FILE" ]; then
        TEXT_TO_TRANSLATE=$(cat "$SELECTED_FILE" | tr '\n' ' ')
        SOURCE_NAME="Fichier $(basename "$SELECTED_FILE")"
    fi

# SCÉNARIO 2 : Lancé depuis Nautilus/Nemo (utilise la variable d'environnement)
elif [ -n "$NAUTILUS_SCRIPT_SELECTED_FILE_PATHS" ]; then
    SELECTED_FILE=$(echo "$NAUTILUS_SCRIPT_SELECTED_FILE_PATHS" | head -n 1)
    
    if [ -f "$SELECTED_FILE" ]; then
        TEXT_TO_TRANSLATE=$(cat "$SELECTED_FILE" | tr '\n' ' ')
        SOURCE_NAME="Fichier $(basename "$SELECTED_FILE")"
    fi
else
    # SCÉNARIO 3 : Lancé par un raccourci personnalisé (Sélection/Presse-papiers)
    
    # 2.1. Essayer Wayland (wl-clipboard)
    if command -v wl-paste &> /dev/null; then
        TEXT_TO_TRANSLATE=$(wl-paste --primary) 
        if [ -z "$TEXT_TO_TRANSLATE" ]; then
             TEXT_TO_TRANSLATE=$(wl-paste) 
        fi

    # 2.2. Essayer X11 (xclip)
    elif command -v xclip &> /dev/null; then
        TEXT_TO_TRANSLATE=$(xclip -selection primary -o) 
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
    MESSAGE="Le contenu est vide. Sélectionnez un fichier, ou copiez/sélectionnez du texte."
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

-----

## 8\. Vidéo

Une vidéo de démonstration existe, et celle-ci à été publié sur les réseaux-sociaux Blabla Linux 😎

  - [Sur Facebook](https://www.facebook.com/share/v/17K7SU2Db4/)
  - [Sur Twitter (X)](https://x.com/BlablaLinux/status/1987910094017753265?s=20)
  - [Sur Bluesky](https://bsky.app/profile/blablalinux.be/post/3m5bxvgholk2q)
  - [Sur Mastodon](https://mastodon.blablalinux.be/@blablalinux/115526187793773014)