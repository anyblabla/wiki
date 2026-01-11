---
title: Guide Markdown — Syntaxe essentielle
description: Apprenez à maîtriser le Markdown pour documenter vos projets. Ce guide pratique détaille la syntaxe pour les titres, listes, blocs de code et tableaux afin d'uniformiser vos pages sur le wiki.
published: true
date: 2026-01-11T23:21:04.223Z
tags: wiki, tutoriel, documantation, markdown, syntax
editor: markdown
dateCreated: 2026-01-11T23:21:04.223Z
---

Le **Markdown** est un langage de balisage léger permettant de formater du texte avec une syntaxe simple et lisible. Cette page présente les éléments les plus courants pour rédiger vos fiches techniques.

---

## 1. Titres

Utilisez le symbole `#` suivi d’un espace.  
Plus il y a de `#`, plus le niveau du titre est profond.

```markdown
# Titre de niveau 1
## Titre de niveau 2
### Titre de niveau 3
```

---

## 2. Emphase (gras, italique, barré)

- **Gras** : `**texte**`
- *Italique* : `*texte*`
- ~~Barré~~ : `~~texte~~`

Certaines implémentations acceptent aussi `_italique_` et `__gras__`.

---

## 3. Listes

### Listes à puces

```markdown
* Élément 1
* Élément 2
  * Sous-élément
```

Les symboles `-` et `+` fonctionnent également selon les moteurs Markdown.

### Listes ordonnées

```markdown
1. Premier
2. Deuxième
```

---

## 4. Liens et images

- **Lien :**  
  ```markdown
  [Texte du lien](https://url.com)
  ```

- **Lien avec titre au survol :**  
  ```markdown
  [Texte du lien](https://url.com "Titre au survol")
  ```

- **Image :**  
  ```markdown
  ![Description](https://url-image.com/photo.jpg)
  ```

Certaines plateformes nécessitent une ligne vide avant une image pour un rendu correct.

---

## 5. Blocs de code

### Code en ligne

Utilisez des accents graves (backticks) :  
`sudo apt update`

### Bloc de code multilignes

Utilisez trois accents graves suivis du nom du langage (optionnel mais recommandé).

```bash
# Mise à jour du système sous Debian
sudo apt update && sudo apt upgrade -y
```

---

## 6. Citations

Utilisez le chevron `>` suivi d’un espace.

```markdown
> Ceci est une citation
> sur plusieurs lignes.
```

Exemple :

> Le logiciel libre est une question de liberté, pas de prix.

---

## 7. Tableaux

```markdown
| Commande | Description |
| :---     | :---        |
| `ls -al` | Liste les fichiers |
| `top`    | Affiche les processus |
```

Alignement avancé :

```markdown
| Gauche | Centre | Droite |
| :---   | :---:  | ---:   |
| a      | b      | c      |
```