---
title: Sécurité pdf - Pourquoi le masquage graphique est une illusion
description: Apprenez pourquoi masquer un texte avec un rectangle noir dans un PDF est inefficace. Ce guide montre comment extraire les données cachées sous OnlyOffice ou LibreOffice avec un script Python dédié.
published: false
date: 2025-12-24T12:40:16.902Z
tags: pdf, debian, sécurité, tutoriel, python
editor: markdown
dateCreated: 2025-12-24T12:40:16.902Z
---

## 1. Introduction

Il est fréquent de voir des utilisateurs masquer des informations sensibles (mots de passe, noms propres, coordonnées) en dessinant un rectangle noir par-dessus le texte dans une suite bureautique comme OnlyOffice ou LibreOffice.

Ce guide pour Blabla Linux démontre que cette pratique est totalement inefficace pour protéger vos données. Le texte reste présent dans la structure du fichier et peut être extrait en quelques secondes.

## 2. Le concept technique

Dans un fichier pdf, le texte et les formes graphiques (rectangles) résident sur des couches différentes. Dessiner un rectangle noir ne supprime pas les caractères situés en dessous ; cela revient simplement à poser un cache physique sur un document écrit. Un extracteur de texte ignore la couche graphique et lit directement le flux de données textuelles.

## 3. Prérequis pour Debian

Nous utilisons Python et la bibliothèque PyMuPDF, disponible directement dans les dépôts de Debian 13.

```bash
sudo apt update
sudo apt install python3-pymupdf

```

## 4. L'outil : le révélateur de secrets

Voici le script `reveler_secret.py`. Son rôle est d'ignorer les éléments graphiques et d'exporter tout le texte du pdf dans un fichier brut au format .txt.

```python
import sys
import os
try:
    import pymupdf as fitz
except ImportError:
    import fitz

def extraire_vers_txt(fichier_pdf):
    # Génération du nom de sortie
    nom_base = os.path.splitext(fichier_pdf)[0]
    fichier_txt = nom_base + "_REVELE.txt"
    
    try:
        doc = fitz.open(fichier_pdf)
        with open(fichier_txt, "w", encoding="utf-8") as f:
            for i, page in enumerate(doc):
                # Extraction du texte pur (ignore les rectangles et images)
                texte = page.get_text("text")
                
                f.write(f"--- PAGE {i+1} ---\n")
                f.write(texte)
                f.write("\n\n")
        
        doc.close()
        print(f"Succès ! Le contenu a été extrait dans : {fichier_txt}")
        
    except Exception as e:
        print(f"Erreur lors de l'analyse : {e}")

if __name__ == "__main__":
    if len(sys.argv) < 2:
        print("Usage : python3 reveler_secret.py mon-fichier.pdf")
    else:
        extraire_vers_txt(sys.argv[1])

```

## 5. Utilisation du script

Placez votre fichier pdf "masqué" dans le même dossier que le script et lancez la commande suivante dans votre terminal :

```bash
python3 reveler_secret.py confidentiel.pdf

```

Le fichier généré `confidentiel_REVELE.txt` contiendra l'intégralité du texte, révélant ainsi ce qui était caché sous les carrés noirs.

## 6. Recommandations pour une sécurité réelle

Pour supprimer réellement une information d'un pdf, deux méthodes sont fiables :

1. Utiliser un outil de caviardage (redaction) professionnel qui détruit physiquement les objets dans le code source du fichier.
2. Exporter la page concernée en image (rasterisation) avant de la reconvertir en pdf. Cette action transforme le texte en pixels, rendant l'extraction impossible.