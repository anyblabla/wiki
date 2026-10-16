---
title: Notesnook - Utiliser l’instance BlablaLinux
description: Utilisez l'instance Notesnook de BlablaLinux : configurez les serveurs, créez votre compte et synchronisez vos notes chiffrées de bout en bout sur navigateur, ordinateur et Android.
published: false
date: 2026-10-16T17:27:34.741Z
tags: linux, auto-hébergement, tutoriel, blablalinux, notesnook, notes, chiffrement, e2ee, synchronisation, android
editor: markdown
dateCreated: 2026-10-16T17:27:34.741Z
---

Notesnook est une application de prise de notes **chiffrées de bout en bout** : vos notes sont chiffrées sur votre appareil avant d'être envoyées au serveur, qui ne peut pas les lire. Cette page explique comment utiliser **mon instance auto-hébergée** plutôt que le service officiel.

> **À retenir :** il faut configurer les adresses de mes serveurs **avant** de créer votre compte. Sinon, le compte est créé chez Notesnook (le service officiel) et non ici.
{.is-warning}

## Les 4 adresses à connaître

Copiez-les telles quelles, avec le `https://` :

```
https://notes-sync.blablalinux.be
https://notes-auth.blablalinux.be
https://notes-events.blablalinux.be
https://notes-share.blablalinux.be
```

| Champ dans l'application | Adresse |
|---|---|
| Serveur de synchronisation (*Sync server*) | `https://notes-sync.blablalinux.be` |
| Serveur d'authentification (*Auth server*) | `https://notes-auth.blablalinux.be` |
| Serveur d'événements (*Events server*) | `https://notes-events.blablalinux.be` |
| Serveur de monographies (*Monograph server*) | `https://notes-share.blablalinux.be` |

## 1. Installer ou ouvrir l'application

- **Navigateur** : [app.notesnook.com](https://app.notesnook.com)
- **Ordinateur** (Windows, macOS, Linux) et **Android** : téléchargez l'application depuis le [site officiel de Notesnook](https://notesnook.com), qui indique les sources d'installation disponibles.

> La procédure ci-dessous a été testée sur le client de bureau Linux (identique à la version web) et sur Android. Sur iOS, elle devrait être très proche, mais elle n'a pas été vérifiée.
{.is-info}

## 2. Configurer les serveurs, puis créer son compte

# Choisissez votre appareil {.tabset}

### Navigateur et ordinateur

1. Sur l'écran **Créer un compte** (*Create account*), repérez en bas à gauche la mention *Utilisation de l'instance officielle Notesnook* et cliquez sur **Configurer** (*Configure*).

   ![Écran de création de compte avec le lien Configurer](/tutoriels/notesnook/notesnook-app-desktop-1.png)

2. Dans **Serveurs**, remplissez les 4 champs avec les adresses ci-dessus.

   ![Paramètres des serveurs, champs vides](/tutoriels/notesnook/notesnook-app-desktop-2.png)

3. Cliquez sur **Tester la connexion** (*Test connection*). Le message vert *Connected to all servers successfully* doit apparaître. Le bouton **Enregistrer** (*Save*) reste grisé tant que ce test n'a pas réussi.

   ![Test de connexion réussi](/tutoriels/notesnook/notesnook-app-desktop-3.png)

4. Cliquez sur **Enregistrer**. L'application se recharge d'elle-même en quelques secondes.

   ![Rechargement de l'application après l'enregistrement](/tutoriels/notesnook/notesnook-app-desktop-4.png)

5. De retour sur l'écran de création de compte, saisissez votre **adresse e-mail**, votre **mot de passe** et sa confirmation, puis cliquez sur **Créer un compte**.

   ![Application prête à l'emploi](/tutoriels/notesnook/notesnook-app-desktop-5.png)

### Android

1. Sur l'écran d'accueil, touchez **Connectez-vous pour chiffrer et synchroniser vos notes**. Un panneau s'ouvre en bas de l'écran : touchez **Paramètres**.

   ![Écran d'accueil non connecté](/tutoriels/notesnook/notesnook-app-mobile-1.png)
   ![Panneau avec le menu Paramètres](/tutoriels/notesnook/notesnook-app-mobile-3.png)

2. Dans **Paramètres**, touchez **Serveurs** (*Configurer les URL de serveur pour Notesnook*).

   ![Menu Paramètres](/tutoriels/notesnook/notesnook-app-mobile-4.png)

3. Remplissez les 4 champs, **dans cet ordre** : synchronisation, authentification, événements, monographies. Touchez **Tester la connexion** : le message *Connexion établie avec succès à tous les serveurs* doit s'afficher. Touchez ensuite **Enregistrer**.

   ![Écran Serveurs rempli et testé](/tutoriels/notesnook/notesnook-app-mobile-5.png)

4. Revenez à l'écran d'accueil, touchez **Connectez-vous pour chiffrer et synchroniser vos notes**, puis choisissez la création d'un compte : saisissez votre adresse e-mail et votre mot de passe.

---

## 3. Vérifier que tout fonctionne

Une fois connecté, l'application doit indiquer que vos notes sont **synchronisées** (sur Android : mention *Synchronisé* avec un point vert dans le panneau du compte). Créez une note de test sur un appareil et vérifiez qu'elle apparaît sur l'autre.

![Compte connecté et synchronisé sur Android](/tutoriels/notesnook/notesnook-app-mobile-6.png)

## Se connecter depuis un autre appareil

Sur chaque nouvel appareil : **configurez d'abord les 4 serveurs** (étape 2), puis connectez-vous avec le même compte.

> Pour modifier les adresses des serveurs, vous devez d'abord **vous déconnecter**. L'application refuse le changement tant que vous êtes connecté.
{.is-info}

## Bon à savoir

- **Mot de passe** : vos notes sont chiffrées avec lui. Ne pouvant pas les déchiffrer, je ne peux pas les récupérer si vous le perdez. Choisissez-en un solide et conservez-le dans un gestionnaire de mots de passe.
- **Sauvegardes** : faites des sauvegardes régulières depuis **Paramètres → Sauvegarde et exportation** (*Backup & export*).
- **Données stockées** : votre adresse e-mail est enregistrée sur mon serveur. Le contenu de vos notes est chiffré et illisible pour moi.
- **Monographies** : cette fonction publie une note sur une page publique accessible à toute personne disposant du lien. Ne publiez que ce que vous acceptez de rendre public.
- **Service auto-hébergé** : il est fourni sans garantie de disponibilité. D'où l'intérêt de sauvegarder vos notes.

## Questions fréquentes

**Le bouton « Enregistrer » reste grisé ou le test de connexion échoue.**
Vérifiez qu'il n'y a pas de faute de frappe ni d'espace dans les 4 adresses, et que le `https://` est présent. Le message d'erreur indique en général le serveur qui pose problème. Si l'erreur persiste, contactez-moi en précisant ce message.

**J'ai créé mon compte sur le service officiel par erreur.**
Les deux comptes sont distincts. Exportez vos notes depuis **Sauvegarde et exportation**, déconnectez-vous, configurez mes serveurs, créez un compte chez moi, puis importez vos notes avec les outils de restauration ou d'importation de l'application. À ma connaissance, il n'existe pas de migration automatique.

**Je veux revenir au service officiel.**
Déconnectez-vous, puis utilisez **Réinitialiser** (sur Android : *Réinitialiser les URL des serveurs*) dans les paramètres des serveurs.

**Comment supprimer mon compte ?**
Sur Android : **Paramètres → Compte → Supprimer les données** (action irréversible : sauvegardez vos notes avant). Vous pouvez aussi me demander la suppression par e-mail.

## Contact

Une question, un souci d'inscription, une demande de suppression de compte :
**[À COMPLÉTER : adresse e-mail ou salon Matrix de contact]**
