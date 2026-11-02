---
title: Gestion des clés API (API Keys) pour LibreTranslate
description: >-
  Gestion des clés API LibreTranslate : quotas par utilisateur, activation de
  --api-keys, et commandes ltmanage keys (ajouter, supprimer, afficher). Guide
  essentiel pour configurer des limites de requêtes personnalisées.
published: true
date: '2026-11-02T23:39:30.014Z'
tags:
  - translate
  - api
  - key
editor: markdown
dateCreated: '2025-11-07T18:57:09.915Z'
---

a## Introduction

Le système de gestion des clés API de **LibreTranslate** permet d'établir des **quotas de limites d'utilisation par utilisateur**. Cette fonctionnalité est conçue pour permettre à des utilisateurs spécifiques de bénéficier de limites de requêtes par minute (taux de rafraîchissement) plus élevées que la limite par défaut du serveur.

### Fonctionnement des limites

  * **Limite par défaut :** Par défaut, tous les utilisateurs sont soumis à la limitation de débit globale définie par l'option `--req-limit` de LibreTranslate.
  * **Limite personnalisée :** En transmettant le paramètre facultatif `api_key` aux points de terminaison REST, l'utilisateur concerné bénéficie de la limite de requêtes supérieure associée à cette clé.

-----

## ⚙️ Activation de la fonctionnalité

Pour activer la prise en charge de la gestion des clés API, le serveur LibreTranslate doit être démarré avec l'option suivante :

```bash
--api-keys
```

-----

## 🛠️ Commandes de gestion (`ltmanage keys`)

L'outil en ligne de commande `ltmanage keys` est utilisé pour émettre, supprimer et visualiser les clés API.

### 1\. Ajouter une nouvelle clé

Cette commande génère une nouvelle clé API et lui assigne une limite de requêtes par minute.

  * **Syntaxe :** `ltmanage keys add <limite_requetes_par_minute>`

  * **Exemple :** Pour émettre une nouvelle clé avec une limite fixée à **120 requêtes par minute** :

    ```bash
    ltmanage keys add 120
    ```

### 2\. Supprimer une clé

Cette commande révoque et supprime une clé API existante du système.

  * **Syntaxe :** `ltmanage keys remove <api-key>`

  * **Exemple :**

    ```bash
    ltmanage keys remove a1b2c3d4e5f6g7h8i9j0
    ```

### 3\. Afficher les clés

Cette commande liste toutes les clés API actuellement actives ainsi que la limite de requêtes associée à chacune.

  * **Syntaxe :** `ltmanage keys`

-----

## 💻 Exemple d'utilisation de la clé API

Pour bénéficier de la limite de requêtes supérieure, la clé API doit être transmise dans la requête de traduction HTTP, généralement dans le corps de la requête (méthode `POST`).

| Paramètre | Rôle |
| :--- | :--- |
| `q`, `source`, `target` | Paramètres de traduction standard. |
| **`api_key`** | **Clé unique pour bénéficier du quota élevé.** |

### Requête `curl` (Exemple)

Cet exemple montre une requête de traduction Anglais vers Français, incluant la clé API :

```bash
curl -X POST "http://localhost:5000/translate" \
     -H "Content-Type: application/json" \
     -d '{
         "q": "Hello World",
         "source": "en",
         "target": "fr",
         "api_key": "a1b2c3d4e5f6g7h8i9j0"
     }'
```
