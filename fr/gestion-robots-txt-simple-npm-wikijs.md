---
title: Gérer un robots.txt simple pour le référencement grâce à Nginx Proxy Manager
description: >-
  Guide simple et stable pour implémenter un fichier robots.txt statique pour
  Wiki.js grâce à Nginx Proxy Manager (NPM). Autorise le référencement public et
  sécurise le tableau de bord d'administration.
published: true
date: '2026-11-11T22:36:56.289Z'
tags:
  - nginx
  - proxy
  - npm
  - sécurité
  - seo
  - robots txt
  - wikijs
editor: markdown
dateCreated: '2025-11-29T18:04:26.219Z'
---

Ce guide utilise la méthode du **fichier statique** (`alias`) pour implémenter un `robots.txt` simple et stable pour votre site derrière **Nginx Proxy Manager (NPM)**. Nous prenons comme **exemple l'application Wiki.js** et le domaine **`wiki.blablalinux.be`** pour illustrer la protection des chemins d'administration sensibles.

-----

## 1\. Principe de la méthode du fichier statique

Le fait de demander à Nginx de servir un **fichier statique** existant sur le disque via l'instruction `alias` est la solution la plus stable pour contourner les problèmes de limite de longueur de chaîne et garantir un fonctionnement sans erreur dans **NPM**.

### Pré-requis

1.  Avoir accès au système de fichiers du serveur ou au volume Docker où **NPM** stocke ses données.

-----

## 2\. Contenu du fichier robots.txt simple (Exemple Wiki.js)

Vous devez créer ou mettre à jour le fichier sur votre système de fichiers.

  * **Chemin suggéré (à adapter) :** `/data/seo/votre-application/robots.txt`

Ce contenu est la **configuration de base idéale** : elle autorise tous les robots à indexer le contenu public, tout en protégeant un chemin d'administration sensible (le tableau de bord Wiki.js dans cet exemple).

```text
Sitemap: https://votre-domaine.com/sitemap.xml

# 1. RÈGLES POUR LES ROBOTS STANDARDS (AUTORISÉS & SÉCURISÉS)
# Cette règle autorise le référencement de tout le contenu public.
User-agent: *
Disallow: /a/dashboard # Bloque l'accès au panneau d'administration de Wiki.js (À ADAPTER)
Allow: /
```

> **👉 N'oubliez pas d'adapter la ligne `Sitemap:` avec votre propre nom de domaine et le chemin `Disallow:` avec les chemins sensibles de votre application.**

> **Exemple spécialisé :** Pour voir un exemple de `robots.txt` qui **bloque également les robots d'intelligence artificielle**, vous pouvez consulter celui de Wiki.js Blabla Linux : [https://wiki.blablalinux.be/robots.txt](https://wiki.blablalinux.be/robots.txt).

-----

## 3\. Configuration dans Nginx Proxy Manager

Collez ce bloc Nginx dans l'onglet **Advanced** de l'hôte proxy de votre site. **Il est crucial d'adapter la directive `alias` pour qu'elle corresponde exactement au chemin où vous avez créé votre fichier statique.**

```nginx
# Bloc pour servir le fichier robots.txt statique (Solution stable via NPM)
location = /robots.txt {
    # Alias : ADAPTEZ CE CHEMIN avec le chemin réel de votre fichier sur le serveur
    alias /data/seo/votre-application/robots.txt; 
    
    # Assurer le bon Content-Type pour le SEO
    add_header Content-Type text/plain;
    charset utf-8;
    
    # Correction des problèmes potentiels de compression/artefacts
    gzip off;
    proxy_set_header Accept-Encoding ""; 
}

> **⚠️ REMARQUE IMPORTANTE :** Dans la ligne `alias`, vous devez impérativement remplacer `/votre-application` par le nom de dossier que vous avez choisi pour stocker votre fichier `robots.txt`. Par exemple : `/data/seo/wikijs/robots.txt`.
```

-----

## 4\. Vérification finale

Après avoir créé le fichier statique et sauvegardé la configuration dans NPM :

1.  Accédez à l'URL : `https://votre-domaine.com/robots.txt`.
2.  Le contenu doit s'afficher parfaitement, confirmant que votre site est prêt pour l'indexation, avec les chemins sensibles protégés.
