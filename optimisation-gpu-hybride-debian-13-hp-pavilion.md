---
title: Optimisation du GPU hybride (Intel/AMD) sur HP Pavilion dv7
description: Apprenez à optimiser le GPU hybride Intel/AMD sur Debian 13. Guide complet pour gérer le rendu 3D et le décodage vidéo matériel sur un laptop HP Pavilion dv7.
published: true
date: 2026-01-17T20:45:05.249Z
tags: trixie, debian 13, proxmox 9, va-api, amd radeon, intel hd graphics, accélération matérielle, hp pavilion
editor: markdown
dateCreated: 2026-01-17T19:38:21.964Z
---

## 1. Le concept des deux tuyaux (comprendre l'hybride)

Pour optimiser cette machine, il faut comprendre que le traitement graphique passe par deux circuits (ou "tuyaux") distincts qui peuvent travailler de concert :

### 🏗️ Le tuyau 3D (l'atelier de construction)

* **Technologie :** OpenGL / Mesa.
* **Usage :** Calcul des polygones et des textures (ex : SuperTuxKart, Blender, GNOME).
* **Pilotage :** Géré automatiquement par le système ou forcé manuellement via la commande `DRI_PRIME=1` pour solliciter la carte **AMD Radeon**, bien plus puissante.

### 📽️ Le tuyau vidéo (l'atelier de traduction)

* **Technologie :** VA-API (Video Acceleration API).
* **Usage :** Décodage des vidéos YouTube, Netflix ou fichiers MKV.
* **Pilotage :** Verrouillé sur l'**Intel HD 3000** via `LIBVA_DRIVER_NAME=i965`. Ce choix garantit une stabilité maximale, car sur cette génération, le décodeur Intel est plus fiable que le vieux moteur UVD d'AMD.

---

## 2. La transparence sous GNOME (Debian 13)

Bonne nouvelle : sur **Debian 13 Trixie** avec l'environnement **GNOME**, le système est devenu intelligent. La gestion manuelle est souvent facultative.

* **Détection automatique :** GNOME identifie les applications gourmandes et les lance souvent d'office sur le GPU AMD.
* **Le menu contextuel :** Vous pouvez forcer le choix d'un clic droit sur l'icône d'une application dans le menu :
* **"Lancer avec la carte graphique dédiée"** : Pour envoyer le travail vers l'AMD.
* **"Lancer avec la carte graphique intégrée"** : Pour rester sur l'Intel et économiser la batterie.



---

## 3. Installation et configuration

Passez en mode superutilisateur pour installer la base logicielle et fixer l'aiguillage du "tuyau vidéo" :

```bash
apt update
# Firmware et microcode (indispensables pour la stabilité et la sécurité)
apt install -y firmware-amd-graphics intel-microcode

# Pilotes VA-API et outils de diagnostic
apt install -y i965-va-driver vainfo mesa-utils

# Fixation de la variable d'environnement globale
echo "LIBVA_DRIVER_NAME=i965" | tee -a /etc/environment

```

---

## 4. Bilan de la répartition de charge (load balancing)

Votre laptop HP répartit la charge sans effort :

1. **L'AMD** s'occupe de la puissance brute (Jeux / 3D).
2. **L'Intel** gère la logistique (Affichage / Décodage vidéo).

C'est la configuration idéale pour préserver le processeur i7 et éviter les surchauffes inutiles.

---

## 5. Vérification finale

Après un redémarrage, validez la configuration avec ces deux tests :

1. `vainfo` -> Doit confirmer l'utilisation du pilote `i965` (Intel).
2. `DRI_PRIME=1 glxinfo | grep "renderer"` -> Doit confirmer l'utilisation de `AMD TURKS`.

<div style="text-align: center;"> <iframe title="Démonstration de l'accélération hybride (Intel/AMD) sur HP Pavilion dv7 - Debian 13" width="560" height="315" src="https://peertube.blablalinux.be/videos/embed/54V2zuHDJi2tjc8J8hxUxD" allow="fullscreen" sandbox="allow-same-origin allow-scripts allow-popups allow-forms" style="border: 0px;"></iframe> </div>