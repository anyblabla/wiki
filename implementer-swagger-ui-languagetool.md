---
title: Implémenter Swagger UI pour une API sans interface (exemple LanguageTool)
description: Apprenez à déployer Swagger UI de manière indépendante pour documenter et tester l'API de LanguageTool, tout en configurant correctement les politiques de sécurité CORS sous Nginx Proxy Manager.
published: true
date: 2026-02-01T00:24:51.466Z
tags: docker, npm, api, swagger, languagetool, cors
editor: markdown
dateCreated: 2026-01-31T23:26:44.436Z
---

### Contenu du wiki

> **Note importante** : Les noms de domaine utilisés dans ce guide (ex: `blablalinux.be`) sont des exemples issus d'une infrastructure réelle. Vous devez impérativement les remplacer par vos propres noms de domaine lors de la configuration.

## Introduction

Certains services auto-hébergés, bien que puissants, ne proposent pas d'interface native pour explorer et tester leurs fonctionnalités. C'est le cas de **LanguageTool**, qui offre une API robuste mais nécessite un outil externe pour être manipulée visuellement.

L'objectif de ce guide est d'utiliser **Swagger UI** comme une interface universelle. Nous allons voir comment héberger nous-mêmes la spécification de l'API pour contourner les restrictions de sécurité et offrir un environnement de test complet à l'utilisateur.

## Liens officiels

* **LanguageTool** : [Site officiel](https://languagetool.org) | [GitHub](https://github.com/languagetool-org/languagetool)
* **Swagger UI** : [Site officiel](https://swagger.io/tools/swagger-ui/) | [GitHub](https://github.com/swagger-api/swagger-ui)
* **Spécification API** : [Fichier JSON direct (original)](https://languagetool.org/http-api/languagetool-swagger.json)

---

## 1. Pré-requis

Avant de commencer, assurez-vous de disposer des éléments suivants :

* Un serveur sous **Debian** (ou Ubuntu) avec Docker et Docker Compose installés.
* Une instance **LanguageTool** fonctionnelle (par exemple `languagetool.votre-domaine.fr`).
* Un reverse proxy comme **Nginx Proxy Manager** (NPM) pour gérer le SSL et les entêtes.
* Un sous-domaine dédié pour l'interface (par exemple `swagger.votre-domaine.fr`).

---

## 2. Adaptation de la spécification API

Par défaut, le fichier JSON de LanguageTool pointe vers leurs serveurs officiels. Pour utiliser votre propre instance, il est impératif d'héberger une version modifiée du fichier de spécification.

1. **Récupération locale** :
`curl -L -o specs/languagetool-api.json https://languagetool.org/http-api/languagetool-swagger.json`
2. **Modifications du fichier** :
Ouvrez le fichier `languagetool-api.json` et adaptez les premières lignes :
* **Host** : Indiquez votre propre domaine d'API.
* **Schemes** : Forcez l'utilisation du protocole `https`.
* **BasePath** : Vérifiez qu'il est bien réglé sur `/v2` pour correspondre à l'API.



---

## 3. Déploiement de Swagger UI avec Docker

Nous utilisons un volume pour injecter notre fichier JSON modifié directement dans le conteneur Swagger UI. L'utilisation du mode `restart: always` est recommandée pour la haute disponibilité.

```yaml
services:
  swagger-ui:
    image: swaggerapi/swagger-ui:latest
    container_name: swagger-ui
    restart: always  # Pour garantir le redémarrage automatique après un reboot
    ports:
      - "8085:8080"
    volumes:
      - ./specs:/usr/share/nginx/html/specs
    environment:
      # Chemin vers le fichier JSON monté dans le volume
      API_URL: "specs/languagetool-api.json"
      # Désactive le validateur externe pour plus de confidentialité
      VALIDATOR_URL: ""

```

---

## 4. Configuration des entêtes CORS (étape cruciale)

C'est ici que se situe la principale difficulté technique. Puisque Swagger appelle une API située sur un autre sous-domaine, le navigateur bloquera la requête par sécurité (CORS).

Il faut ajouter des entêtes spécifiques dans le **Proxy Host de l'API (LanguageTool)** (onglet **Advanced**) sur Nginx Proxy Manager :

```nginx
# Autorisation pour votre domaine Swagger
add_header 'Access-Control-Allow-Origin' 'https://swagger.votre-domaine.fr' always;
add_header 'Access-Control-Allow-Methods' 'GET, POST, OPTIONS, PUT, DELETE' always;
add_header 'Access-Control-Allow-Headers' 'DNT,User-Agent,X-Requested-With,If-Modified-Since,Cache-Control,Content-Type,Range,Authorization' always;

# Gestion du preflight (Méthode OPTIONS)
# Indispensable pour valider la communication avant l'envoi des données réelles
if ($request_method = 'OPTIONS') {
    add_header 'Access-Control-Allow-Origin' 'https://swagger.votre-domaine.fr' always;
    add_header 'Access-Control-Allow-Methods' 'GET, POST, OPTIONS, PUT, DELETE' always;
    add_header 'Access-Control-Allow-Headers' 'DNT,User-Agent,X-Requested-With,If-Modified-Since,Cache-Control,Content-Type,Range,Authorization' always;
    return 204;
}

```

---

## 5. Validation et tests

Une fois les services relancés :

1. Connectez-vous sur votre instance Swagger.
2. Recherchez la méthode `POST /check`.
3. Cliquez sur **Try it out**, saisissez un texte avec une faute (ex: "La joolie maison") et cliquez sur **Execute**.
4. Vous devriez recevoir une réponse **200 OK** contenant les suggestions de correction provenant de votre propre serveur.

## 6. Résultat attendu

Si la configuration est correcte, vous obtiendrez une réponse 200 OK contenant les données de correction de votre serveur.

![implementer-swagger-ui-languagetool.png](/implementer-swagger-ui-languagetool/implementer-swagger-ui-languagetool.png)