---
title: Conception d'un outil en ligne de commande pour Wiki.js
description: Apprenez à créer et installer un script Python personnalisé pour gérer votre instance Wiki.js (CRUD) directement depuis le terminal Debian, avec gestion automatique des langues et des slugs.
published: true
date: 2026-05-23T21:00:30.785Z
tags: wikijs, api, python, graphql, cli
editor: markdown
dateCreated: 2026-01-11T01:38:11.338Z
---

L'interface web de Wiki.js est puissante, mais pour un utilisateur habitué au terminal, la création ou la modification rapide de pages peut être plus efficace via un script. Cette page détaille la mise en place d'un **CLI (Command Line Interface)** sur mesure pour piloter votre Wiki.

> ☝️ Dépôt disponible sur [GitHub](https://github.com/anyblabla/wikijs-cli) et [Gitea](https://gitea.blablalinux.be/blablalinux/wikijs-cli) 👌

## 1. Prérequis et dépendances

Pour que le script fonctionne, votre système doit disposer de Python 3 et de la bibliothèque `requests` (pour les appels API).

### Installation des paquets

Sur Debian, Ubuntu ou Linux Mint, ouvrez votre terminal et tapez :

```bash
sudo apt update
sudo apt install python3 python3-requests -y

```

## 2. Configuration de Wiki.js

Avant de manipuler le script, vous devez autoriser l'accès à votre instance :

1. Connectez-vous à votre Wiki.js en tant qu'administrateur.
2. Allez dans l'**Administration** > **Clés API**.
3. Créez une nouvelle clé avec les droits suivants : `pages:read`, `pages:write`.
4. Notez précieusement votre **Token API** et l'**URL GraphQL** (ex: `https://wiki.votre-domaine.be/graphql`).

## 3. Création du script

Nous allons créer le fichier dans un dossier `Scripts` de votre répertoire personnel.

### Étape A : Créer le dossier et le fichier

```bash
mkdir -p ~/Scripts
nano ~/Scripts/wiki_cli.py

```

### Étape B : Le code source complet

Copiez le code suivant et veillez à bien remplacer les variables `WIKI_URL` et `WIKI_TOKEN` au début du script.

```python
import requests
import sys
import os
import tempfile
import subprocess
import readline

# Activation du support de l'édition de ligne (flèches, backspace, etc.)
readline.parse_and_bind("tab: complete")

# --- Configuration Blabla Linux ---
# Script conçu par BlablaLinux
# Remplacez les valeurs ci-dessous par vos propres accès API
WIKI_URL = "https://wiki.votre-domaine.be/graphql"
WIKI_TOKEN = "VOTRE_TOKEN_API_ICI"

def clear_screen():
    os.system('clear' if os.name == 'posix' else 'cls')

def query_graphql(query, variables=None):
    headers = {"Authorization": f"Bearer {WIKI_TOKEN}"}
    try:
        response = requests.post(WIKI_URL, json={'query': query, 'variables': variables}, headers=headers)
        return response.json()
    except Exception as e:
        print(f"Erreur de connexion : {e}")
        return {}

def get_text_from_editor(initial_content=""):
    """Ouvre l'éditeur par défaut pour éditer le contenu Markdown."""
    editor = os.environ.get('EDITOR', 'nano')
    with tempfile.NamedTemporaryFile(suffix=".md", delete=False, mode='w+', encoding='utf-8') as tf:
        tf.write(initial_content)
        temp_path = tf.name
    subprocess.call([editor, temp_path])
    with open(temp_path, 'r', encoding='utf-8') as tf:
        content = tf.read()
    if os.path.exists(temp_path):
        os.remove(temp_path)
    return content

def list_pages(filter_str=None):
    query = "{ pages { list { id path title locale isPublished } } }"
    res = query_graphql(query)
    all_pages = res.get('data', {}).get('pages', {}).get('list', [])
    if filter_str:
        return [p for p in all_pages if filter_str.lower() in p['title'].lower() or filter_str.lower() in p['path'].lower()]
    return all_pages

def get_single_page(page_id):
    query = """query($id: Int!) { pages { single(id: $id) { id title description content path locale editor isPublished isPrivate tags { tag } } } }"""
    return query_graphql(query, {"id": page_id}).get('data', {}).get('pages', {}).get('single')

def select_page_ui(action_name):
    print(f"\n{'─'*10} {action_name} {'─'*10}")
    search = input("Rechercher un mot-clé (ou Entrée pour tout lister) : ")
    pages = list_pages(search)
    
    if not pages:
        print("Aucune page trouvée.")
        return None

    for i, p in enumerate(pages):
        pub = "✔" if p['isPublished'] else "✘"
        print(f"{i+1:2d}) [{p['locale'].upper()}] [{pub}] {p['title']} ({p['path']})")
    
    idx_input = input("\nNuméro de la page (ou 'q' pour annuler) : ")
    if idx_input.lower() == 'q' or not idx_input: return None
    try:
        return get_single_page(pages[int(idx_input)-1]['id'])
    except:
        return None

def create_page():
    print("\n" + "─"*40 + "\n📄 CRÉATION D'UNE PAGE\n" + "─"*40)
    title = input("Titre : ")
    path = input("Slug (ex: mon-tuto ou en/my-tuto) : ")
    
    if path.startswith("en/"):
        locale = "en"
        path = path[3:]
    else:
        locale = "fr"
        if path.startswith("fr/"):
            path = path[3:]

    desc = input("Description : ")
    print("Ouverture de l'éditeur pour le contenu...")
    content = get_text_from_editor()
    tags_raw = input("Tags (séparés par des virgules) : ")
    tags = [t.strip() for t in tags_raw.split(',') if t.strip()]
    is_published = (input("Publier immédiatement ? (O/n) : ") or "o").lower() == "o"

    mutation = """
    mutation($content: String!, $description: String!, $editor: String!, $isPublished: Boolean!, $isPrivate: Boolean!, $locale: String!, $path: String!, $tags: [String]!, $title: String!) {
      pages { create(content: $content, description: $description, editor: $editor, isPublished: $isPublished, isPrivate: $isPrivate, locale: $locale, path: $path, tags: $tags, title: $title) {
          responseResult { succeeded, message }
        } } }"""
    
    vars = {
        "content": content, "description": desc, "editor": "markdown", 
        "isPublished": is_published, "isPrivate": False, 
        "locale": locale, "path": path, "tags": tags, "title": title
    }
    res = query_graphql(mutation, vars)
    if 'errors' in res:
        print(f"❌ Erreur : {res['errors'][0]['message']}")
    else:
        print(f"✅ Résultat : {res['data']['pages']['create']['responseResult']['message']}")
    input("\nAppuyez sur Entrée...")

def update_page():
    current = select_page_ui("✏️ MODIFICATION")
    if not current: return

    print("\n(Laissez vide pour conserver la valeur actuelle)")
    title = input(f"Titre [{current['title']}] : ") or current['title']
    desc = input(f"Description [{current['description']}] : ") or current['description']
    path = input(f"Slug [{current['path']}] : ") or current['path']
    
    if path.startswith("en/"):
        locale = "en"
        path = path[3:]
    else:
        locale = "fr"
        if path.startswith("fr/"): 
            path = path[3:]

    pub_choice = input(f"Publier ? (actuel: {current['isPublished']}) (o/n/Entrée) : ").lower()
    is_published = current['isPublished'] if not pub_choice else (pub_choice == 'o')

    content = get_text_from_editor(current['content']) if input("Modifier le contenu ? (o/N) : ").lower() == 'o' else current['content']
    tags = [t.strip() for t in input("Tags : ").split(',')] if input("Changer les tags ? (o/N) : ").lower() == 'o' else [t['tag'] for t in current['tags']]

    mutation = """
    mutation($id: Int!, $content: String!, $description: String!, $editor: String!, $isPublished: Boolean!, $isPrivate: Boolean!, $locale: String!, $path: String!, $tags: [String]!, $title: String!) {
      pages { update(id: $id, content: $content, description: $description, editor: $editor, isPublished: $isPublished, isPrivate: $isPrivate, locale: $locale, path: $path, tags: $tags, title: $title) {
          responseResult { succeeded, message }
        } } }"""
    
    vars = {
        "id": current['id'], "content": content, "description": desc, "editor": current['editor'], 
        "isPublished": is_published, "isPrivate": current['isPrivate'], 
        "locale": locale, "path": path, "tags": tags, "title": title
    }
    res = query_graphql(mutation, vars)
    print(f"✅ Résultat : {res['data']['pages']['update']['responseResult']['message']}")
    input("\nAppuyez sur Entrée...")

def delete_page():
    current = select_page_ui("🗑️ SUPPRESSION")
    if not current: return
    
    if input(f"⚠️ Confirmer la suppression définitive de '{current['title']}' ? (oui/non) : ").lower() == 'oui':
        res = query_graphql("mutation($id: Int!) { pages { delete(id: $id) { responseResult { succeeded, message } } } }", {"id": current['id']})
        print(f"✅ Résultat : {res['data']['pages']['delete']['responseResult']['message']}")
    input("\nAppuyez sur Entrée...")

def main():
    while True:
        clear_screen()
        print("\n" + "═"*30 + "\n      WIKI.JS CLI MANAGER\n          by BlablaLinux\n" + "═"*30)
        print("  1. Lister / Rechercher les pages")
        print("  2. Créer une nouvelle page")
        print("  3. Modifier une page existante")
        print("  4. Supprimer une page")
        print("  q. Quitter le programme")
        print("─"*30)
        
        choix = input("Action : ").lower()
        if choix == '1':
            s = input("Recherche (Entrée pour tout) : ")
            pages = list_pages(s)
            print("\n" + "─"*40)
            for p in pages:
                st = "✔" if p['isPublished'] else "✘"
                print(f"• [{p['locale'].upper()}] [{st}] {p['title']} ({p['path']})")
            print("─"*40)
            input("\nAppuyez sur Entrée...")
        elif choix == '2': create_page()
        elif choix == '3': update_page()
        elif choix == '4': delete_page()
        elif choix == 'q': 
            clear_screen()
            user_name = os.environ.get('USER', 'Ami')
            print(f"Au revoir {user_name} ! Merci d'utiliser Wiki CLI by BlablaLinux.")
            break

if __name__ == "__main__":
    main()
    
```

## 4. Présentation des fonctionnalités du menu

Une fois lancé via la commande `wiki`, le script présente un menu numéroté :

### 1️⃣ Lister / Rechercher

Permet de consulter votre contenu. Vous pouvez filtrer par mot-clé pour voir le statut (✅ publié, 📝 brouillon), la langue et le chemin des pages.

### 2️⃣ Créer une page

* **Automatisation** : Si le slug commence par `en/`, la page est créée en anglais, sinon elle est en français.
* **Édition** : Ouvre votre éditeur local (Nano par défaut) pour rédiger en Markdown.
* **Métadonnées** : Définit le titre, la description et les étiquettes en une étape.

### 3️⃣ Modifier une page

Recherchez la page, choisissez son numéro, et modifiez ce que vous souhaitez. Si vous laissez un champ vide et appuyez sur Entrée, la valeur actuelle est conservée.

### 4️⃣ Supprimer une page

Supprime définitivement une page après une confirmation explicite (`oui`).

## 5. Installation et utilisation du script

Une fois le fichier enregistré, il existe deux manières de l'utiliser confortablement.

### Option 1 : Utilisation par un alias (Recommandé)

Cette méthode permet de lancer le script via une commande courte sans modifier les permissions du fichier.

1. Ouvrez votre fichier d'alias : `nano ~/.bash_aliases`.
2. Ajoutez cette ligne :
`alias wiki='python3 ~/Scripts/wiki_cli.py'`
3. Rechargez vos alias : `source ~/.bashrc`.
4. Tapez simplement `wiki` pour lancer le programme.

### Option 2 : Rendre le script exécutable

Si vous préférez appeler le script directement, vous devez lui donner les droits d'exécution.

1. Ajoutez le "shebang" en toute première ligne du fichier : `#!/usr/bin/env python3`.
2. Rendez le fichier exécutable :

```bash
chmod +x ~/Scripts/wiki_cli.py

```

3. Vous pouvez maintenant le lancer avec : `~/Scripts/wiki_cli.py`.
4. (Facultatif) Pour l'utiliser comme une commande système globale, créez un lien symbolique :

```bash
sudo ln -s ~/Scripts/wiki_cli.py /usr/local/bin/wiki

```

## 6. Guide d'utilisation rapide

* **Création** : Le script détecte si vous voulez une page en anglais. Si le slug commence par `en/`, il règle la langue sur anglais. Sinon, il choisit français par défaut.
* **Édition** : Pour le contenu, le script ouvre votre éditeur par défaut. Si vous n'en avez pas défini, il utilisera `nano`. Une fois votre texte rédigé, sauvegardez et quittez (`Ctrl+O` puis `Ctrl+X` sous Nano) pour que le script envoie les données au serveur.

## 7. Démonstration

![python-conception-cli-wikijs.gif](/python-conception-cli-wikijs/python-conception-cli-wikijs.gif)