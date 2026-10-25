---
title: Notesnook - Utiliser l’instance BlablaLinux
description: Utilisez l'instance Notesnook de BlablaLinux : configurez les serveurs, créez votre compte et synchronisez vos notes chiffrées de bout en bout sur navigateur, ordinateur et Android.
published: true
date: 2026-10-25T19:26:38.807Z
tags: linux, auto-hébergement, tutoriel, blablalinux, notesnook, notes, chiffrement, e2ee, synchronisation, android
editor: markdown
dateCreated: 2026-10-16T17:27:34.741Z
---

Notesnook est une application de prise de notes **chiffrées de bout en bout** : vos notes sont chiffrées sur votre appareil avant d'être envoyées au serveur, qui ne peut pas les lire. Cette page explique comment utiliser **mon instance auto-hébergée** plutôt que le service officiel.

> **À retenir :** il faut configurer les adresses de mes serveurs **avant** de créer votre compte. Sinon, le compte est créé chez Notesnook (le service officiel) et non ici.
> {.is-warning}

## Les 4 adresses à connaître

Copiez-les telles quelles, avec le `https://` :

```
https://notes-sync.blablalinux.be
https://notes-auth.blablalinux.be
https://notes-events.blablalinux.be
https://notes-share.blablalinux.be

```

| Champ dans l'application | Adresse | 
| ----- | ----- | 
| Serveur de synchronisation (*Sync server*) | https://notes-sync.blablalinux.be | 
| Serveur d'authentification (*Auth server*) | https://notes-auth.blablalinux.be | 
| Serveur d'événements (*Events server*) | https://notes-events.blablalinux.be | 
| Serveur de monographies (*Monograph server*) | https://notes-share.blablalinux.be | 

## 1. Installer ou ouvrir l'application

* **Navigateur** : [app.notesnook.com](https://app.notesnook.com)

* **Ordinateur** (Windows, macOS, Linux) et **Android** : téléchargez l'application depuis le [site officiel de Notesnook](https://notesnook.com), qui indique les sources d'installation disponibles.

> La procédure ci-dessous a été testée sur le client de bureau Linux (identique à la version web) et sur Android. Sur iOS, elle devrait être très proche, mais elle n'a pas été vérifiée.
> {.is-info}

## 2. Configurer les serveurs, puis créer son compte

### Navigateur et ordinateur

1. Sur l'écran **Créer un compte** (*Create account*), repérez en bas à gauche la mention *Utilisation de l'instance officielle Notesnook* et cliquez sur **Configurer** (*Configure*).

   ![Écran de création de compte avec le lien Configurer](/tutoriels-notesnook-utiliser-instance-blablalinux/notesnook-app-desktop-linux-1.png)

2. Dans **Serveurs**, remplissez les 4 champs avec les adresses ci-dessus.

   ![Paramètres des serveurs, champs vides](/tutoriels-notesnook-utiliser-instance-blablalinux/notesnook-app-desktop-linux-2.png)

3. Cliquez sur **Tester la connexion** (*Test connection*). Le message vert *Connexion établie avec succès à tous les serveurs* doit apparaître. Le bouton **Enregistrer** (*Save*) reste grisé tant que ce test n'a pas réussi.

   ![Test de connexion réussi](/tutoriels-notesnook-utiliser-instance-blablalinux/notesnook-app-desktop-linux-3.png)

4. Cliquez sur **Enregistrer**. L'application se recharge d'elle-même en quelques secondes.

   ![Rechargement de l'application après l'enregistrement](/tutoriels-notesnook-utiliser-instance-blablalinux/notesnook-app-desktop-linux-4.png)

5. De retour sur l'écran de création de compte, saisissez votre **adresse e-mail**, votre **mot de passe** et sa confirmation, puis cliquez sur **Créer un compte**.

> **Note concernant la connexion et la sécurité (2FA) :**
> Lorsque l'on crée un compte, la double authentification (2FA) par e-mail est directement activée. Par conséquent, la connexion s'effectue dans l'ordre suivant : saisie de l'**adresse e-mail**, validation par le **code reçu par e-mail**, puis saisie du **mot de passe**.
> Dans les paramètres du compte, vous pouvez ensuite choisir d'utiliser une application 2FA comme méthode primaire, tout en conservant l'adresse e-mail en tant que méthode de secours. L'ensemble de ce processus (création, 2FA par e-mail et application 2FA) a été rigoureusement testé depuis l'extérieur et validé comme étant pleinement fonctionnel.
> {.is-info}

### Android

1. Sur l'écran d'accueil, touchez **Connectez-vous pour chiffrer et synchroniser vos notes**. Un panneau s'ouvre en bas de l'écran : touchez **Paramètres**.

   <img src="/tutoriels-notesnook-utiliser-instance-blablalinux/notesnook-app-mobile-android-1.png" alt="Écran d'accueil non connecté" width="300" />

   <img src="/tutoriels-notesnook-utiliser-instance-blablalinux/notesnook-app-mobile-android-3.png" alt="Panneau avec le menu Paramètres" width="300" />

2. Dans **Paramètres**, touchez **Serveurs** (*Configurer les URL de serveur pour Notesnook*).

   <img src="/tutoriels-notesnook-utiliser-instance-blablalinux/notesnook-app-mobile-android-4.png" alt="Menu Paramètres" width="300" />

3. Remplissez les 4 champs, **dans cet ordre** : synchronisation, authentification, événements, monographies. Touchez **Tester la connexion** : le message *Connexion établie avec succès à tous les serveurs* doit s'afficher. Touchez ensuite **Enregistrer**.

   <img src="/tutoriels-notesnook-utiliser-instance-blablalinux/notesnook-app-mobile-android-5.png" alt="Écran Serveurs rempli et testé" width="300" />

4. Un message vous informe que l'URL du serveur est modifiée et vous invite à redémarrer l'application. Touchez **Terminé** et redémarrez Notesnook.

   <img src="/tutoriels-notesnook-utiliser-instance-blablalinux/notesnook-app-mobile-android-6.png" alt="URL du serveur modifiée" width="300" />

5. Revenez à l'écran d'accueil, touchez **Connectez-vous pour chiffrer et synchroniser vos notes**, puis choisissez la création d'un compte : saisissez votre adresse e-mail, votre mot de passe et procédez à l'authentification par code e-mail.

## 3. Vérifier que tout fonctionne

Une fois connecté, l'application doit indiquer que vos notes sont **synchronisées** (sur Android : mention *Synchronisé* avec un point vert dans le panneau du compte). Créez une note de test sur un appareil et vérifiez qu'elle apparaît sur l'autre.

<img src="/tutoriels-notesnook-utiliser-instance-blablalinux/notesnook-app-mobile-android-7.png" alt="Compte connecté et synchronisé sur Android" width="300" />

## 4. Extension de navigateur : Web Clipper

Notesnook propose une extension officielle, le **Web Clipper**, pour enregistrer des pages web, des articles ou une sélection de contenu directement dans vos notes. Les clips sont chiffrés sur votre appareil, puis synchronisés comme n'importe quelle note. L'extension est **open source** et disponible pour **Firefox** et pour les navigateurs basés sur **Chromium** (Chrome, Edge…). Elle ne vous demande aucun identifiant.

> **Comment ça marche avec mon instance :** l'extension ne se connecte pas directement à un serveur. D'après l'aide officielle de Notesnook, elle s'appuie sur l'**application web** ([app.notesnook.com](https://app.notesnook.com)) pour synchroniser et enregistrer vos clips. Il suffit donc que l'application web de votre navigateur soit **configurée avec mes serveurs** (étape 2) et **connectée à votre compte**. Ce fonctionnement a été testé avec mon instance.
> {.is-info}

### Installer l'extension

**Chrome et navigateurs Chromium** : installez [Notesnook Web Clipper depuis le Chrome Web Store](https://chrome.google.com/webstore/detail/notesnook-web-clipper/kljhpemdlcnjohmfmkogahelkcidieaj).

**Firefox** :

1. Téléchargez le fichier `.xpi` depuis la [page officielle du Web Clipper](https://notesnook.com/notesnook-web-clipper).
2. Ouvrez `about:addons` dans Firefox.
3. Glissez-déposez le fichier `.xpi` dans la page pour terminer l'installation.

Les navigateurs regroupent les extensions dans un menu : pensez à **épingler** Notesnook à la barre d'outils pour y accéder facilement. Les détails sont dans l'[aide officielle d'installation](https://notesnook.com/help/web-clipper/installation).

### Connecter l'extension (une seule fois)

1. Vérifiez que l'application web ([app.notesnook.com](https://app.notesnook.com)) est configurée avec mes serveurs et connectée à votre compte, **dans le même navigateur** que l'extension.
2. Cliquez sur l'icône Notesnook de la barre d'outils, puis sur **Connect with Notesnook** (intitulé donné par l'aide officielle, il peut varier selon la langue).
3. L'application web s'ouvre dans un nouvel onglet, en arrière-plan. Patientez quelques secondes : l'extension se connecte automatiquement.

Une fois la connexion établie, la mention **Web clipper connected** apparaît dans la barre d'état, en bas à gauche de l'application web, et l'extension affiche ses options de clip.

![Le Web Clipper connecté à l'application web, avec les zones et les modes de clip](/tutoriels-notesnook-utiliser-instance-blablalinux/notesnook-webclipper.png)

### Clipper une page

1. **Choisissez la zone à clipper :**

   | Zone | Ce qui est enregistré |
   |---|---|
   | *Full page* | La page entière, de haut en bas, avec tous les détails (plus lourd et plus long) |
   | *Article* | Le contenu principal de la page, comme le mode lecture de Firefox : idéal pour les blogs et les articles |
   | *Visible area* | Uniquement ce qui est affiché à l'écran |
   | *Selected nodes* | Les éléments sur lesquels vous cliquez, un par un |

2. **Choisissez le mode de clip :**

   | Mode | Résultat |
   |---|---|
   | *Bookmark* | Seulement l'adresse et le titre de la page : idéal pour lire plus tard |
   | *Simplified* | Le contenu sans les styles, directement dans la note : idéal pour les articles |
   | *Screenshot* | Une image de la page, non interactive |
   | *Complete with styles* | Toute la page avec ses styles, intégrée à la note comme un élément interactif |

   Par défaut, l'extension propose la zone *Article* et le mode *Simplified*, ce qui convient à la plupart des pages.

3. **Classez votre clip (facultatif)** : ajoutez-le à une note existante, à un carnet, ou attribuez-lui des étiquettes.
4. Cliquez sur **Start clip** pour lancer le clip, puis sur **Save** pour l'enregistrer et le synchroniser avec votre compte.

> Les modes *Screenshot* et *Complete with styles* ne sont proposés que lorsque vous êtes connecté à l'application web. Leur contenu est enregistré comme **pièce jointe**, donc sur le stockage de mon serveur : privilégiez *Bookmark* ou *Simplified* quand cela vous suffit.
> {.is-info}

### Si l'extension ne se connecte pas

1. Vérifiez que vous êtes **connecté** sur l'application web et que ses serveurs sont bien les miens (**Paramètres → Serveurs**).
2. Gardez l'application web **ouverte en arrière-plan**.
3. L'extension ne gère pas encore plusieurs fenêtres de navigateur : fermez les fenêtres supplémentaires.
4. Redémarrez le navigateur.

Pour aller plus loin : [aide officielle du Web Clipper](https://notesnook.com/help/web-clipper/clipping-your-first-web-page-with-web-clipper) et [dépannage](https://notesnook.com/help/web-clipper/troubleshooting).

## Se connecter depuis un autre appareil

Sur chaque nouvel appareil : **configurez d'abord les 4 serveurs** (étape 2), puis connectez-vous avec le même compte.

> Pour modifier les adresses des serveurs, vous devez d'abord **vous déconnecter**. L'application refuse le changement tant que vous êtes connecté.
> {.is-info}

## Bon à savoir

* **Mot de passe** : vos notes sont chiffrées avec lui. Ne pouvant pas les déchiffrer, je ne peux pas les récupérer si vous le perdez. Choisissez-en un solide et conservez-le dans un gestionnaire de mots de passe.

* **Sauvegardes** : faites des sauvegardes régulières depuis **Paramètres → Sauvegarde et exportation** (*Backup & export*).

* **Données stockées** : votre adresse e-mail est enregistrée sur mon serveur. Le contenu de vos notes est chiffré et illisible pour moi.

* **Monographies** : cette fonction publie une note sur une page publique accessible à toute personne disposant du lien. Ne publiez que ce que vous acceptez de rendre public.

* **Service auto-hébergé** : il est fourni sans garantie de disponibilité. D'où l'intérêt de sauvegarder vos notes.

## Questions fréquentes

**Le bouton « Enregistrer » reste grisé ou le test de connexion échoue.**
Vérifiez qu'il n'y a pas de faute de frappe ni d'espace dans les 4 adresses, et que le `https://` est présent. Le message d'erreur indique en général le serveur qui pose problème. Si l'erreur persiste, contactez-moi en précisant ce message.

**J'ai créé mon compte sur le service officiel par erreur.**
Les deux comptes sont distincts. Exportez vos notes depuis **Sauvegarde et exportation**, déconnectez-vous, configurez mes serveurs, créez un compte chez moi, puis importez vos notes avec les outils de restauration ou d'importation de l'application. À ma connaissance, il n'existe pas de migration automatique.

**Je veux revenir au service officiel.**
Déconnectez-vous, puis utilisez **Réinitialiser** (sur Android : *Réinitialiser les URL des serveurs*) dans les paramètres des serveurs.

**Comment supprimer mon compte ?**
Sur Android : **Paramètres → Compte → Supprimer les données** (action irréversible : sauvegardez vos notes avant). Vous pouvez aussi me demander la suppression par e-mail.

## Héberger votre propre instance

Vous préférez installer votre propre serveur Notesnook plutôt que d'utiliser le mien ? Mon guide pas à pas, avec des commandes à copier-coller (Docker Compose dans un LXC Debian, MongoDB sécurisé, stockage S3, SMTP et Nginx Proxy Manager), est disponible ici : [Notesnook - Installer son propre serveur (Docker, LXC, NPM)](/fr/tutoriels-notesnook-installer-sur-son-serveur).

## Contact

Une question, un souci d'inscription, une demande de suppression de compte :

* **Formulaire de contact** : [blablalinux.be/contact](https://blablalinux.be/contact)

* **Me contacter via Matrix** : [@blablalinux:blablalinux.be](https://matrixto.blablalinux.be/#/@blablalinux:blablalinux.be)

---

> **Auteur** : ce guide est proposé par **Amaury aka BlablaLinux**. Retrouvez l'ensemble de mes services sur [blablalinux.be/mes-services-publics/](https://blablalinux.be/mes-services-publics/).