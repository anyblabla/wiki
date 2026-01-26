---
title: Personnalisation de l'explorateur de fichiers Apache (thème Blabla Linux)
description: Apprenez à personnaliser l'explorateur de fichiers Apache (mod_autoindex) avec le thème sombre de Blabla Linux. Guide technique pour un rendu moderne, responsive et aux couleurs de votre charte.
published: true
date: 2026-01-26T00:17:05.339Z
tags: apache, css, webdesign, tutoriel, autoindex
editor: markdown
dateCreated: 2026-01-25T22:50:47.081Z
---

Ce guide détaille la transformation technique de l'interface `mod_autoindex` d'Apache utilisée sur **fichiers.blablalinux.be**. L'objectif est de passer d'un listing brut à une interface moderne, sombre, centrée et responsive.

## 1. La structure : Le principe du "Sandwich"

Apache génère une table HTML de manière automatique. Pour la styliser, nous utilisons deux fichiers externes que nous injectons pour "envelopper" le contenu généré :

* **`header.html`** : Contient les balises `<head>`, le CSS et ouvre le conteneur visuel.
* **Le contenu Apache** : La table des fichiers générée automatiquement par le serveur.
* **`footer.html`** : Referme proprement les balises et ajoute la signature (crédits).

## 2. Personnalisation : Adaptez les couleurs

Le design repose sur des variables CSS situées au début du fichier `header.html`. Pour adapter ce thème à votre propre charte graphique, modifiez simplement ces valeurs :

```css
:root {
    --primary-color: #f77f11;    /* Couleur d'accentuation (Orange Blabla Linux) */
    --bg-color: #1a1a1a;         /* Couleur du fond de page (Sombre) */
    --card-bg: #2d2d2d;          /* Couleur du bloc central (Anthracite) */
    --link-color: #4da3ff;       /* Couleur des liens de fichiers */
    --text-color: #e0e0e0;       /* Couleur du texte général */
}

```

> [!TIP]
> **Centrage vertical :** Le thème utilise `flexbox` sur le `body` avec `justify-content: center`. Cela permet au bloc de fichiers de rester parfaitement au milieu de l'écran, quelle que soit la taille de votre moniteur.

---

## 3. Étape par étape : Mise en place

### Étape 1 : Préparer les fichiers

Créez un dossier nommé `files` (ou utilisez la racine) sur votre serveur et déposez-y les trois fichiers détaillés ci-dessous.

### Étape 2 : Configurer le serveur (`.htaccess`)

C'est le fichier qui donne les instructions à Apache. Il doit être placé à la racine de votre répertoire de fichiers.

```apache
Options +Indexes
IndexOptions FancyIndexing HTMLTable NameWidth=* DescriptionWidth=* Charset=UTF-8

# Injection du haut et du bas de page
HeaderName /files/header.html
ReadmeName /files/footer.html

# Masquer les fichiers techniques de la liste publique
IndexIgnore header.html footer.html .htaccess favicon.ico

```

### Étape 3 : Créer le haut de page (`header.html`)

Copiez ce code pour définir le style et l'ouverture de la page :

```html
<!DOCTYPE html>
<html lang="fr">
<head>
    <meta charset="utf-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <style>
        :root {
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
            margin: 0; padding: 20px 10px;
            display: flex; flex-direction: column;
            justify-content: center; align-items: center;
            min-height: 100vh; box-sizing: border-box;
        }
        .index-container {
            max-width: 900px; width: 100%;
            background: var(--card-bg); padding: 25px;
            border-radius: 12px; box-shadow: 0 10px 30px rgba(0,0,0,0.5);
            box-sizing: border-box;
        }
        h1 { 
            color: #ffffff; font-size: 1.5rem; 
            border-left: 5px solid var(--primary-color); 
            padding-left: 15px; margin: 10px 0 25px 0;
            text-transform: uppercase;
        }
        table { width: 100%; border-collapse: collapse; }
        th {
            text-align: left; padding: 12px;
            border-bottom: 2px solid var(--primary-color);
            color: var(--primary-color); text-transform: uppercase; font-size: 0.85rem;
        }
        td { padding: 12px; border-bottom: 1px solid var(--border-color); vertical-align: middle; }
        tr:hover { background-color: #383838; }
        a { color: var(--link-color); text-decoration: none; font-weight: 500; }
        a:hover { text-decoration: underline; }
        img {
            margin-right: 10px; vertical-align: middle;
            filter: sepia(1) saturate(3) hue-rotate(10deg) brightness(1.2);
        }
        .btn-back { 
            text-decoration: none; color: var(--primary-color); 
            font-weight: bold; border: 2px solid var(--primary-color); 
            padding: 8px 15px; border-radius: 8px; 
            display: inline-block; margin-bottom: 20px; transition: 0.3s;
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

### Étape 4 : Créer le pied de page (`footer.html`)

Ce fichier ferme le design et ajoute vos informations personnelles :

```html
    </div> <footer style="text-align: center; width: 100%; margin: 20px 0; color: #777; font-size: 0.9rem; font-family: 'Segoe UI', sans-serif;">
        <p>Propulsé par le thème <strong>Blabla Linux</strong> — 2026</p>
        <p>
            <a href="https://blablalinux.be" target="_blank" style="color: #f77f11; text-decoration: none; font-weight: bold;">
                Visiter le site officiel
            </a>
        </p>
    </footer>
</body>
</html>

```

---

## 4. Rendu Final

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

<hr style="border: 0; height: 1px; background-image: linear-gradient(to right, rgba(0, 0, 0, 0), rgba(247, 127, 17, 0.75), rgba(0, 0, 0, 0)); margin: 40px 0;">