---
title: Migration de domaine YOURLS : de yourls à y.blablalinux.be
description: Procédure complète pour migrer YOURLS vers un domaine court comme y.blablalinux.be avec Nginx Proxy Manager, redirection 301, défi DNS OVH et sécurisation optionnelle via un plugin reCAPTCHA.
published: true
date: 2026-01-30T01:09:14.513Z
tags: nginx, npm, yourls, redirection 301, recaptcha
editor: markdown
dateCreated: 2026-01-30T01:09:14.513Z
---

Cette fiche documente la procédure technique pour raccourcir l'URL de base de YOURLS tout en maintenant la compatibilité des anciens liens partagés sur le web. **La procédure est testée et 100 % fonctionnelle.**

## 1. Stratégie de redirection

L'objectif est de faire de `y.blablalinux.be` le domaine principal. Grâce à la redirection **301 (permanente)**, tout le trafic arrivant sur l'ancien domaine est automatiquement redirigé vers le nouveau sans briser les liens existants.

## 2. Configuration dans Nginx Proxy Manager

### Paramètres du proxy host

* **Noms de domaine** : Ajouter `y.blablalinux.be` et conserver `yourls.blablalinux.be`.
* **Certificat SSL** :
* Puisque j'utilise le **défi DNS via l'API OVH**, le certificat wildcard gère automatiquement la validation.
* Aucun renouvellement manuel n'est requis pour le nouveau sous-domaine.



### Bloc de configuration avancée

```nginx
# Redirection chirurgicale de l'ancien sous-domaine vers le nouveau
if ($host = 'yourls.blablalinux.be') {
    return 301 https://y.blablalinux.be$request_uri;
}

```

## 3. Configuration de YOURLS avec Docker Compose

Modifier la variable d'environnement dans votre fichier `docker-compose.yml` :

```yaml
environment:
  - YOURLS_SITE=https://y.blablalinux.be

```

**Important** : Relancer le conteneur pour appliquer les changements :
`docker compose up -d`

> **Note** : Une fois le conteneur relancé, YOURLS met à jour dynamiquement l'affichage. Dans le panneau d'administration, tous les anciens liens créés avec `yourls.blablalinux.be` basculent automatiquement vers l'affichage en `y.blablalinux.be`.

## 4. Sécurisation avec reCAPTCHA (optionnel)

*Note : Cette section nécessite l'installation préalable d'un plugin reCAPTCHA dans YOURLS.*

* **Domaine autorisé (console Google)** : Utiliser `blablalinux.be` (couvre tous les sous-domaines par récursivité).
* **Validation du domaine** : **Activée** pour une sécurité optimale.
* **Option AMP** : **Désactivée** (inutile pour YOURLS).