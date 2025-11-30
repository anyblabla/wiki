---
title: Injection d'un logo personnalisé via Nginx Proxy Manager (NPM)
description: Guide pas-à-pas pour injecter un logo flottant cliquable dans vos applications web via Nginx Proxy Manager (NPM). Inclut les scripts de base, les corrections anti-FOUC et les solutions aux erreurs de CSP.
published: true
date: 2025-11-30T23:33:47.977Z
tags: nginx, proxy, npm, injection, css, logo, csp
editor: markdown
dateCreated: 2025-11-30T23:16:50.068Z
---

Ce guide explique comment injecter un logo cliquable (taille recommandée : $50 \times 50$ pixels) dans le coin de vos applications web, en utilisant **Nginx Proxy Manager (NPM)**.

-----

## 1\. Objectif et méthode d'injection

Le script utilise la directive Nginx **`sub_filter`** pour rechercher la balise de fin d'en-tête HTML (`</head>`) dans la réponse du serveur proxifié, et y insérer votre code HTML (`<a>`, `<img>`) et le style CSS (`<style>`).

L'injection doit être réalisée via l'onglet **"Code Nginx Personnalisé"** de chaque hôte proxy dans l'interface de NPM (méthode privilégiée pour sa fiabilité).

-----

## 2\. Choix de la méthode d'injection : globale contre individuelle

| Méthode d'injection | Avantages | Problèmes rencontrés (Pourquoi nous utilisons l'injection individuelle) |
| :--- | :--- | :--- |
| **Fichier global** (`location_proxy.conf`) | Maintenance centralisée : Une seule modification met à jour tous les hôtes. | **Échec de l'exécution** : NPM ignore ou écrase souvent les directives `sub_filter` critiques des fichiers globaux. |
| **Hôte individuel** (méthode retenue) | **Fiabilité garantie**. **Contrôle granulaire** (choix de l'emplacement/position du logo par service). | Maintenance manuelle : Nécessite de copier/coller le code pour chaque hôte. |

-----

## 3\. Script de base pour l'injection (bas à droite)

Ce script est le point de départ pour l'injection. Il intègre la correction **Anti-FOUC** (Flash of Unstyled Content) en définissant la taille de l'image directement en HTML.

### Placement : Bas à droite

```nginx
# --- INJECTION DU LOGO DE LA COMMUNAUTÉ (Bas à Droite) ---

# 1. PARAMÈTRES NGINX OBLIGATOIRES
# Désactive la compression (obligatoire pour que sub_filter fonctionne)
proxy_set_header Accept-Encoding ""; 
sub_filter_types text/html;

# 2. INJECTION DU CODE HTML/CSS
sub_filter '</head>' '
    </head>
    <a href="https://VOTRE-URL-DE-DESTINATION.COM" id="custom-logo" target="_blank">
        <img src="https://VOTRE-DOMAINE.COM/CHEMIN/VERS/VOTRE-LOGO.PNG" alt="Aller à la page Communauté" width="50" height="50">
    </a>
    <style>
        #custom-logo { 
            position: fixed; 
            bottom: 15px; 
            right: 15px; /* ANCRAGE À DROITE */
            z-index: 10000; 
            transition: transform 0.3s ease; 
        } 
        #custom-logo:hover { 
            transform: scale(1.1); 
        } 
        #custom-logo img { 
            height: auto; 
            border-radius: 5px; 
            box-shadow: 0 2px 4px rgba(0,0,0,0.2); 
            cursor: pointer; 
        }
    </style>
';
# Force l'exécution du filtre sur toute la page 
sub_filter_once off;
```

-----

### ⚠️ **ATTENTION : URL à changer**

Vous devez impérativement remplacer ces deux placeholders dans le code ci-dessus :

1.  **Lien cliquable :** **`https://VOTRE-URL-DE-DESTINATION.COM`**
2.  **Source de l'image :** **`https://VOTRE-DOMAINE.COM/CHEMIN/VERS/VOTRE-LOGO.PNG`**

-----

## 4\. Décorticage du script

| Ligne de code | Rôle | Explication |
| :--- | :--- | :--- |
| `proxy_set_header Accept-Encoding "";` | Anti-compression | Indique au serveur d'origine de ne pas compresser la réponse pour que Nginx puisse modifier le contenu. |
| `sub_filter_types text/html;` | Cible | Dit à Nginx de ne chercher le motif que dans les documents HTML. |
| `width="50" height="50"` (dans `<img>`) | **Anti-FOUC** | Définit la taille de l'image **avant** le chargement du CSS pour éviter le clignotement. |
| `right: 15px;` | Positionnement | Ancre le logo à 15 pixels du bord **droit** de la fenêtre. |
| `sub_filter_once off;` | Force d'exécution | Assure que Nginx parcourt l'intégralité du document et applique le filtre. |

-----

## 5\. Scripts d'ajustement de position

Pour changer l'emplacement du logo, modifiez uniquement la section `position: fixed;` dans le bloc `<style>` :

### Placement : Bas à gauche

Pour un ancrage en **bas à gauche**, remplacez la ligne `right: 15px;` par la ligne suivante :

```css
            left: 15px; /* ANCRAGE À GAUCHE */
```

-----

## 6\. Ajout de headers de sécurité (CSP)

Certaines applications (Nextcloud, PrivateBin, etc.) bloquent le chargement d'images externes via le Content Security Policy (CSP). Si le logo n'apparaît pas (seul le texte `ALT` est visible), vous devez ajouter le bloc suivant **AVANT** le bloc d'injection du logo.

```nginx
# --- CORRECTION 1 : AUTORISATION DE L'IMAGE (Content Security Policy - CSP) ---
# Masque la politique de sécurité Content-Security-Policy originale
proxy_hide_header Content-Security-Policy;
# Ajoute une nouvelle politique pour autoriser les images depuis VOTRE-DOMAINE.COM et les styles en ligne.
add_header Content-Security-Policy "img-src 'self' https://VOTRE-DOMAINE.COM; style-src 'self' 'unsafe-inline';";
```

*Note : N'oubliez pas de remplacer **`https://VOTRE-DOMAINE.COM`** par le domaine où votre logo est hébergé.*

-----

## 7\. Conclusion : La réalité de l'injection de code

**La réalité est qu'il n'existe pas un seul script qui fonctionnera sur 100 % des pages web.**

Chaque application a des spécificités qui peuvent nécessiter des ajustements :

1.  **Content-Type (exemple : BentoPDF) :** L'application ne renvoie pas un `Content-Type: text/html` standard, nécessitant l'ajustement **`sub_filter_types *;`** (à la place de `text/html;`).
2.  **CSP Strict (exemple : Nextcloud/PrivateBin) :** Les headers de sécurité sont trop stricts, nécessitant l'ajout du bloc **CSP** ci-dessus.

L'approche recommandée est d'utiliser le **Script de base** et d'ajouter les correctifs (CSP ou `sub_filter_types *`) **uniquement si l'affichage échoue** sur un hôte donné.

-----

## 8\. Résultats d'affichage sur les services

Voici ce que donne l'injection du logo flottant sur quelques-uns de mes services **Blabla Linux** :

![joplin.png](/injection-logo-flottant-nginx-npm/joplin.png)

  * [https://joplin.blablalinux.be](https://joplin.blablalinux.be)
  
![jitsimeet.png](/injection-logo-flottant-nginx-npm/jitsimeet.png)

  * [https://jitsimeet.blablalinux.be](https://jitsimeet.blablalinux.be)
  
![convertx.png](/injection-logo-flottant-nginx-npm/convertx.png)

  * [https://convertx.blablalinux.be](https://convertx.blablalinux.be)
  
![it-tools.png](/injection-logo-flottant-nginx-npm/it-tools.png)

  * [https://it-tools.blablalinux.be](https://it-tools.blablalinux.be)
  
![signpdf.png](/injection-logo-flottant-nginx-npm/signpdf.png)

  * [https://signpdf.blablalinux.be](https://signpdf.blablalinux.be)
  
![libre-translate.png](/injection-logo-flottant-nginx-npm/libre-translate.png)

  * [https://ltranslate.blablalinux.be](https://ltranslate.blablalinux.be)
  
![pwpush.png](/injection-logo-flottant-nginx-npm/pwpush.png)

  * [https://pwpush.blablalinux.be](https://pwpush.blablalinux.be)