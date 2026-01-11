---
title: Automatiser Wiki.js avec l'API GraphQL et Python
description: Apprenez à piloter votre instance Wiki.js via l'API GraphQL : configuration du jeton d'accès, lecture de pages et mise à jour automatique du contenu Markdown avec des scripts Python concrets.
published: true
date: 2026-01-11T00:31:29.927Z
tags: wikijs, api, python, automatisation, graphql
editor: markdown
dateCreated: 2026-01-11T00:31:29.927Z
---

## Introduction

Wiki.js s'appuie sur une API **GraphQL** plutôt qu'une interface REST traditionnelle. Ce choix permet une grande flexibilité en permettant de requêter précisément les données souhaitées. Pour un administrateur système, c'est l'outil idéal pour automatiser les tâches répétitives ou synchroniser du contenu.

## 1. Configuration de l'accès

Pour interagir avec l'API, vous devez disposer d'un jeton d'accès (Token) :

1. Connectez-vous à l'espace **Administration** de votre Wiki.js.
2. Allez dans la section **API Keys**.
3. Cliquez sur **Generate New Token**.
4. Définissez les droits (ex: `read:pages`, `write:pages`) selon vos besoins.
5. Notez l'URL de votre point d'entrée (Endpoint), généralement : `https://wiki.exemple.com/graphql`.

## 2. Structure du script Python

L'utilisation de la bibliothèque `requests` est la méthode la plus simple pour envoyer des requêtes GraphQL.

```python
import requests

# Remplacez par vos informations réelles
WIKI_URL = "https://wiki.exemple.com/graphql"
WIKI_TOKEN = "votre_token_secret_ici"

def query_graphql(query, variables=None):
    headers = {"Authorization": f"Bearer {WIKI_TOKEN}"}
    response = requests.post(WIKI_URL, json={'query': query, 'variables': variables}, headers=headers)
    return response.json()

```

## 3. Exemples d'utilisation

### Lister les pages existantes

Utile pour obtenir les ID nécessaires à d'autres manipulations.

```python
query = "{ pages { list { id path title } } }"
resultat = query_graphql(query)

```

### Lire le contenu d'une page

Voici comment récupérer le Markdown brut d'une page via son identifiant unique.

```python
query = """
query($id: Int!) {
  pages {
    single(id: $id) {
      content
      title
      description
    }
  }
}
"""
# Remplacez l'ID par celui de votre page
resultat = query_graphql(query, {"id": 123})

```

### Modifier ou créer une page (Mutation)

En GraphQL, les actions d'écriture sont appelées des **mutations**.

```python
mutation = """
mutation($id: Int!, $content: String!, $title: String!) {
  pages {
    update(id: $id, content: $content, title: $title) {
      responseResult { succeeded, message }
    }
  }
}
"""
variables = {
    "id": 123,
    "content": "Mon nouveau contenu mis à jour via script.",
    "title": "Titre mis à jour"
}
resultat = query_graphql(mutation, variables)

```

## 4. Conseils d'administration

* **Gestion du cache** : Les modifications via l'API sont immédiates en base de données, mais peuvent mettre du temps à apparaître sur le site. N'hésitez pas à utiliser l'outil **Flush Pages and Assets Cache** dans l'administration si nécessaire.
* **IDs uniques** : Notez que les ID de pages sont définitifs. Si vous créez et supprimez une page, l'ID utilisé ne sera plus jamais attribué par la base de données.
* **Sécurité** : Le jeton d'accès donne un contrôle total sur votre contenu. Stockez-le de manière sécurisée (par exemple dans des variables d'environnement) et ne le publiez jamais en clair sur un dépôt public.