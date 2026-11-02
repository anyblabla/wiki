---
title: Guide complet - Installation et utilisation de wp-cli
description: >-
  Apprenez à piloter WordPress depuis votre terminal Linux. Ce guide pas à pas
  détaille l'installation du script wp-cli pour créer, lister et supprimer vos
  articles via l'API REST de votre blog.
published: true
date: '2026-11-02T23:42:13.462Z'
tags:
  - bash
  - linux
  - wordpress
  - automatisation
  - cli
  - api rest
editor: markdown
dateCreated: '2026-01-18T14:00:02.479Z'
---

aCe guide détaille l'installation, la configuration et l'utilisation du script **wp-cli**, un outil interactif en Bash pour piloter votre instance WordPress via l'API REST native.

> Le projet est hébergé sur deux plateformes. Choisissez votre source préférée :
> * **Gitea (Principal) :** [gitea.blablalinux.be/blablalinux/wp-cli-bash](https://gitea.blablalinux.be/blablalinux/wp-cli-bash)
> * **GitHub (Miroir) :** [github.com/blablalinux/wp-cli-bash](https://github.com/anyblabla/wp-cli-bash)

## 1. Présentation de l'outil

Le script `wp-cli` permet d'effectuer les actions suivantes sans ouvrir de navigateur :

* **Vérifier le statut :** affiche instantanément le nombre total d'articles (publiés et brouillons).
* **Lister le contenu :** affiche les 10 derniers articles avec leur identifiant unique (ID) et leur état.
* **Rédiger et publier :** crée un nouvel article en demandant le titre et le contenu directement dans le prompt.
* **Mettre à la corbeille :** supprime un article rapidement grâce à son ID.

## 2. Prérequis : générer le jeton WordPress

Avant de configurer le script, vous devez autoriser votre terminal à communiquer avec votre site.

1. Connectez-vous à votre administration WordPress.
2. Allez dans le menu **Utilisateurs** > **Profil**.
3. Descendez jusqu'à la section **Mots de passe d'application**.
4. Saisissez un nom (ex: `Terminal-Linux`) et cliquez sur **Ajouter un nouveau mot de passe d'application**.
5. **Important :** copiez le code de 24 caractères qui s'affiche. C'est votre jeton unique.

## 3. Installation pas à pas

### Étape A : installation des dépendances

Le script utilise `curl` pour les requêtes web et `jq` pour traiter les données JSON.

```bash
sudo apt update && sudo apt install jq curl -y

```

### Étape B : création de l'arborescence

```bash
mkdir -p ~/Scripts
cd ~/Scripts
touch wp-cli.sh
chmod +x wp-cli.sh

```

### Étape C : rédaction du script

Ouvrez le fichier avec l'éditeur `nano` :

```bash
nano wp-cli.sh

```

Copiez le code suivant et **personnalisez les trois variables** de la section de configuration :

```bash
#!/bin/bash

# ==============================================================================
# Script : wp-cli.sh (Interactif)
# Auteur : Amaury aka BlablaLinux
# ==============================================================================

# --- ⚠️ CONFIGURATION À PERSONNALISER ⚠️ ---
URL_SITE="https://votre-site.com"          # L'URL de votre blog (sans / à la fin)
USER_WP="votre_login"                     # Votre nom d'utilisateur WordPress
APP_PASS="xxxx xxxx xxxx xxxx xxxx xxxx"  # Le jeton généré à l'étape 2
# -------------------------------------------

AUTH=$(echo -n "$USER_WP:$APP_PASS" | base64)

# Vérification de la présence de jq
if ! command -v jq &> /dev/null; then echo "Erreur : jq est requis." ; exit 1 ; fi

# --- Fonctions ---
list_posts() {
    clear
    echo "--- 10 DERNIERS ARTICLES ---"
    curl -s --header "Authorization: Basic $AUTH" "$URL_SITE/wp-json/wp/v2/posts?status=any&per_page=10" | jq -r '.[] | "ID: \(.id) | \(.title.rendered) [\(.status)]"'
    echo "----------------------------"
}

create_post() {
    clear
    echo "--- RÉDACTION D'UN NOUVEL ARTICLE ---"
    read -p "Titre de l'article : " title
    read -p "Contenu (texte brut) : " content
    echo "Action : 1) Publier immédiatement | 2) Enregistrer en Brouillon"
    read -p "Choix : " c
    local st="publish" ; [[ "$c" == "2" ]] && st="draft"
    
    JSON_TMP=$(mktemp)
    jq -n --arg ti "$title" --arg co "$content" --arg st "$st" '{title: $ti, content: $co, status: $st}' > "$JSON_TMP"
    
    RESPONSE=$(curl -s --request POST --url "$URL_SITE/wp-json/wp/v2/posts" --header "Authorization: Basic $AUTH" --header "Content-Type: application/json" --data-binary @"$JSON_TMP")
    rm "$JSON_TMP"
    
    local id=$(echo "$RESPONSE" | jq -r '.id')
    [[ "$id" != "null" ]] && echo "✅ Succès ! Article créé avec l'ID : $id" || echo "❌ Échec de la création."
    read -p "Appuyez sur Entrée pour continuer..."
}

delete_post() {
    clear ; list_posts
    read -p "Entrez l'ID de l'article à supprimer : " id
    if [[ -n "$id" ]]; then
        echo "🗑️ Suppression en cours..."
        curl -s --request DELETE --url "$URL_SITE/wp-json/wp/v2/posts/$id" --header "Authorization: Basic $AUTH" | jq -r '"✅ ID \(.id) a été déplacé vers la corbeille."'
    fi
    read -p "Appuyez sur Entrée pour continuer..."
}

# --- Menu Principal ---
while true; do
    clear
    echo "========================="
    echo "    WP-CLI MANAGER       "
    echo "========================="
    echo "1) Voir le statut global (Total articles)"
    echo "2) Lister les 10 derniers articles"
    echo "3) Créer un nouvel article / Brouillon"
    echo "4) Supprimer un article via son ID"
    echo "5) Quitter"
    echo "========================="
    read -p "Votre choix [1-5] : " choice
    case $choice in
        1) clear ; echo "📊 Statut du site :" ; curl -s --header "Authorization: Basic $AUTH" "$URL_SITE/wp-json/wp/v2/posts?status=any&per_page=1" -I | grep -i 'x-wp-total' ; read -p "Entrée..." ;;
        2) list_posts ; read -p "Appuyez sur Entrée..." ;;
        3) create_post ;;
        4) delete_post ;;
        5) exit 0 ;;
        *) echo "Choix invalide..." ; sleep 1 ;;
    esac
done

```

## 4. Création de l'alias

Pour lancer la commande `wp-cli` depuis n'importe quel emplacement dans votre terminal :

1. Éditez votre fichier d'alias : `nano ~/.bash_aliases`
2. Ajoutez cette ligne : `alias wp-cli='bash ~/Scripts/wp-cli.sh'`
3. Rechargez votre session : `source ~/.bash_aliases`

## 5. Sécurité

Puisque ce script contient vos accès en clair, il est impératif d'en restreindre la lecture :

```bash
chmod 700 ~/Scripts/wp-cli.sh

```
