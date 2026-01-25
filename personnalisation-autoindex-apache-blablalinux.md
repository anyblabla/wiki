---
title: Personnalisation de l'explorateur de fichiers Apache (thème Blabla Linux)
description: Apprenez à personnaliser l'explorateur de fichiers Apache (mod_autoindex) avec le thème sombre de Blabla Linux. Guide technique pour un rendu moderne, responsive et aux couleurs de votre charte.
published: true
date: 2026-01-25T23:18:10.508Z
tags: apache, css, webdesign, tutoriel, autoindex
editor: markdown
dateCreated: 2026-01-25T22:50:47.081Z
---

Ce guide détaille la transformation technique de l'interface `mod_autoindex` d'Apache utilisée sur **fichiers.blablalinux.be**. L'objectif est de passer d'un listing brut à une interface moderne, sombre et responsive.

## 1. La structure : injection de code

Apache génère une table HTML de manière automatique. Pour la styliser, nous utilisons deux fichiers externes que nous injectons via le serveur :

* **`header.html`** : contient le `<head>`, le CSS et le début du conteneur. Il s'arrête juste après la balise `<h1>`. Apache insère ensuite sa table.
* **`footer.html`** : sert à refermer les balises `</div>`, `</body>` et `</html>`.

## 2. Décortiquage du CSS (la logique Blabla Linux)

### Les variables (`:root`)

C'est le cerveau du design. Si vous changez de charte graphique, tout se passe ici. Nous utilisons ici les couleurs de la charte **Blabla Linux**.

```css
:root {
    --primary-color: #f77f11;    /* L'identité orange de Blabla Linux */
    --bg-color: #1a1a1a;         /* Fond sombre pour le confort visuel */
    --card-bg: #2d2d2d;          /* Gris anthracite pour détacher le contenu */
    --link-color: #4da3ff;       /* Bleu clair pour une lecture facile des fichiers */
}

```

### Le conteneur dynamique (`.index-container`)

Contrairement à une page statique, une liste de fichiers peut être très courte ou très longue.

* **`height: auto`** et **`min-height: fit-content`** : ces propriétés garantissent que le cadre anthracite s'étirera toujours pour englober la totalité de la table générée par Apache, évitant que le texte ne déborde sur le fond noir.

### Le hack des icônes (`img`)

Apache utilise de vieilles icônes colorées qui jurent avec un thème sombre. Au lieu de les remplacer manuellement (fastidieux), on applique un filtre :

```css
filter: sepia(1) saturate(3) hue-rotate(10deg) brightness(1.2);

```

### La grille responsive (`@media`)

Les tables HTML sont l'ennemi du mobile. Pour corriger cela, nous utilisons des requêtes média pour épurer l'affichage :

```css
th:nth-child(n+3), td:nth-child(n+3) { display: none; }

```

**Comparaison du rendu Desktop vs Mobile :**

**Comparaison du rendu bureau vs mobile :**

<div align="center" style="margin: 30px 0;">
  <p style="margin-bottom: 15px; color: #f77f11; font-weight: bold; font-size: 1.2rem; text-shadow: 1px 1px 2px rgba(0,0,0,0.5);">
    🖥️ Rendu bureau (Interface complète)
  </p>
  <img src="/personnalisation-autoindex-apache-blablalinux/personnalisation-autoindex-apache-blablalinux-desktop.png" 
       alt="Rendu Desktop" 
       style="width: 100%; max-width: 850px; border-radius: 8px; box-shadow: 0 0 25px rgba(247, 127, 17, 0.4); border: 2px solid #444;">
  
  <div style="margin-top: 50px;"></div>

  <p style="margin-bottom: 15px; color: #f77f11; font-weight: bold; font-size: 1.2rem; text-shadow: 1px 1px 2px rgba(0,0,0,0.5);">
    📱 Rendu smartphone (Vue optimisée)
  </p>
  <img src="/personnalisation-autoindex-apache-blablalinux/personnalisation-autoindex-apache-blablalinux-mobile.jpg" 
       alt="Rendu Mobile" 
       style="width: 100%; max-width: 320px; border-radius: 12px; box-shadow: 0 0 25px rgba(247, 127, 17, 0.4); border: 2px solid #444;">
</div>

## 3. Configuration serveur (`.htaccess`)

C'est le chef d'orchestre. Sans ces lignes, le code HTML ne sera jamais lu par Apache :

```apache
# Force l'affichage en mode tableau propre
IndexOptions FancyIndexing HTMLTable NameWidth=*

# Définit les fichiers à utiliser pour l'enrobage
HeaderName /header.html
ReadmeName /footer.html

# Cache les fichiers techniques pour ne pas polluer la liste
IndexIgnore header.html footer.html .htaccess favicon.ico

```

---

## 4. Code HTML complet (`header.html`)

```html
<!DOCTYPE html>
<html lang="fr">
<head>
    <meta charset="utf-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <style>
        :root {
            /* Configuration des couleurs Blabla Linux */
            --primary-color: #f77f11;
            --bg-color: #1a1a1a;
            --card-bg: #2d2d2d;
            --text-color: #e0e0e0;
            --link-color: #4da3ff;
            --border-color: #444;
        }

        body {
            font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
            background-color: var(--bg-color);
            color: var(--text-color);
            margin: 0;
            padding: 20px 10px;
            display: flex;
            justify-content: center;
            min-height: 100vh;
        }

        .index-container {
            max-width: 900px;
            width: 100%;
            background: var(--card-bg);
            padding: 25px;
            border-radius: 12px;
            box-shadow: 0 10px 30px rgba(0,0,0,0.5);
            box-sizing: border-box;
            height: auto;
            min-height: fit-content;
            margin-bottom: 40px;
        }

        h1 { 
            color: #ffffff; 
            font-size: 1.5rem; 
            border-left: 5px solid var(--primary-color); 
            padding-left: 15px; 
            margin: 10px 0 25px 0;
            text-transform: uppercase;
        }

        table {
            width: 100%;
            border-collapse: collapse;
        }

        th {
            text-align: left;
            padding: 12px;
            border-bottom: 2px solid var(--primary-color);
            color: var(--primary-color);
            text-transform: uppercase;
            font-size: 0.85rem;
        }

        td {
            padding: 12px;
            border-bottom: 1px solid var(--border-color);
            vertical-align: middle;
        }

        tr:hover { background-color: #383838; }

        a { color: var(--link-color); text-decoration: none; font-weight: 500; }
        a:hover { text-decoration: underline; }

        img {
            margin-right: 10px;
            vertical-align: middle;
            filter: sepia(1) saturate(3) hue-rotate(10deg) brightness(1.2);
        }

        .btn-back { 
            text-decoration: none; 
            color: var(--primary-color); 
            font-weight: bold; 
            border: 2px solid var(--primary-color); 
            padding: 8px 15px; 
            border-radius: 8px; 
            display: inline-block;
            margin-bottom: 20px;
            transition: 0.3s;
        }
        .btn-back:hover { background: var(--primary-color); color: white; }
        
        @media (max-width: 650px) {
            th:nth-child(n+3), td:nth-child(n+3) { display: none; }
            th:nth-child(1), td:nth-child(1) { width: 40px; }
            th { font-size: 0.75rem; }
        }
    </style>
</head>
<body>
    <div class="index-container">
        <a href="https://fichiers.blablalinux.be" class="btn-back">⬅ Retour à l'accueil</a>
        <h1>Explorateur de fichiers</h1>

```