---
title: Lignes directrices pour les assistants IA
description: >-
  Règles éditoriales, structure des dossiers, conventions de nommage et mise en
  page à respecter par les assistants IA connectés en MCP sur
  wiki.blablalinux.be.
published: true
date: '2026-11-21T22:25:22.146Z'
tags:
  - mcp
  - ia
  - guidelines
  - meta
  - documentation
editor: markdown
dateCreated: '2026-11-21T22:25:22.146Z'
---

Ce document définit les règles éditoriales et techniques à respecter lors de la création, de la modification ou de l'organisation des pages sur **wiki.blablalinux.be**.

## 1. Langues et arborescence

Le wiki est strictement séparé par langue à la racine :
* **`/fr/...`** : articles rédigés en français.
* **`/en/...`** : articles rédigés en anglais (traductions des guides français).

**Règles de classement :**
* Ne crée jamais de page directement à la racine du site. Place toujours l'article dans `/fr/` ou `/en/` selon sa langue.
* Respecte les catégories et sous-dossiers existants avant d'en créer de nouveaux.
* Si tu crées ou modifies un guide en français, vérifie si son équivalent existe dans `/en/` (et inversement) pour garder la même structure de chemin.

## 2. Nommage des URL (slugs) et titres

* **Chemins (URL) :** uniquement en minuscules, sans accents, sans espaces et sans caractères spéciaux. Sépare les mots par des tirets (`-`).
  * *Exemple valide :* `/fr/docker/installer-portainer`
  * *Exemple invalide :* `/fr/Docker/Installer_Portainer`
* **Titres :** clairs, directs et orientés action (ex. : *Installer et configurer Nginx Proxy Manager*). Ne répète pas le titre principal en `# H1` dans le corps de la page, car Wiki.js l'affiche déjà automatiquement.

## 3. Ton et style de rédaction

* **Simplicité :** utilise des mots simples, des phrases courtes et un ton pédagogique et direct (comme un partage d'expérience entre passionnés). Évite le jargon inutile ou explique-le brièvement.
* **Adresse au lecteur :** utilise le tutoiement en français (*tu*, *ton*, *tes*).
* **Structure :**
  * Commence par une courte introduction (2 à 3 phrases) expliquant à quoi sert l'outil ou le tutoriel.
  * Ajoute une section **Prérequis** si nécessaire (système, paquets, accès).
  * Découpe les étapes avec des sous-titres clairs (`##`, `###`) et des listes numérotées pour les actions chronologiques.

## 4. Mise en forme Markdown et médias

* **Blocs de code :** précise toujours le langage après les trois accents graves (`bash`, `yaml`, `json`, `ini`, `nginx`, etc.) pour activer la coloration syntaxique.
* **Commandes et variables :** mets les noms de fichiers, chemins système et commandes courtes en code en ligne (`` `docker compose up -d` ``). Signale clairement les valeurs que l'utilisateur doit personnaliser (ex. : `votre-domaine.be`).
* **Images et ressources :**
  * Chaque dossier de langue (`/fr/` et `/en/`) dispose de son propre dossier médias.
  * Utilise toujours des **chemins relatifs** pour intégrer les images dans les pages.
  * Renseigne systématiquement un texte alternatif (`alt`) descriptif.

## 5. Signature en fin d'article

Termine chaque nouveau guide par la signature standard selon la langue de la page :

* **Pour un article en français (`/fr/`) :**
> **Auteur** : ce guide est proposé par **Amaury aka BlablaLinux**. Retrouvez l'ensemble de mes services sur [blablalinux.be/mes-services-publics/](https://blablalinux.be/mes-services-publics/).

* **Pour un article en anglais (`/en/`) :**
> **Author**: this guide is proposed by **Amaury aka BlablaLinux**. Find all my services on [blablalinux.be/mes-services-publics/](https://blablalinux.be/mes-services-publics/).

## 6. Règles de modification

* **Pas de suppression destructive :** lors de la mise à jour d'un guide existant, conserve la structure globale, les commentaires ou notes spécifiques déjà présents, et ne modifie que les sections concernées.
* **Raison de modification :** renseigne toujours un résumé court et explicite dans l'historique des versions à chaque sauvegarde.
