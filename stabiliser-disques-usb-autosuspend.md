---
title: Stabiliser les disques USB - Désactiver l'autosuspend
description: Guide pour stabiliser vos disques durs externes sur serveur Linux. Apprenez à désactiver l'autosuspend USB et à configurer une activité de maintien pour éviter les erreurs fatales Input/Output.
published: false
date: 2026-03-16T16:16:16.017Z
tags: sauvegarde, debian, ubuntu, linux, maintenance, usb, erreur i/o
editor: markdown
dateCreated: 2026-03-16T16:16:16.017Z
---

Ce guide explique comment empêcher Linux de couper l'alimentation de vos ports USB ou de mettre vos disques durs externes en veille profonde, ce qui provoque souvent des erreurs de type **Input/Output (I/O) error**.

## 1. Comprendre le problème

Par défaut, Linux utilise une fonction appelée **USB Autosuspend** pour économiser l'énergie. Si un disque USB n'est pas sollicité pendant quelques secondes, le système réduit sa tension ou coupe le port.

**Pourquoi est-ce un problème pour un serveur ?**

* **Disques durs mécaniques :** Un disque 2,5" (5400 rpm) met du temps à relancer la rotation de ses plateaux. Si le système demande une donnée alors qu'il est arrêté, le délai de réponse est trop long et Linux déclare une erreur I/O.
* **Déconnexion sauvage :** Le contrôleur du boîtier USB peut mal interpréter la baisse de tension et se déconnecter totalement, rendant vos sauvegardes (Timeshift, Rsync) impossibles.

---

## 2. Diagnostic : votre disque est-il concerné ?

Avant de modifier quoi que ce soit, vérifiez la valeur actuelle du délai d'autosuspend (exprimée en secondes) :

```bash
cat /sys/module/usbcore/parameters/autosuspend

```

> 💡 **Interprétation :**
> * **2** (ou plus) : l'autosuspend est actif (réglage par défaut).
> * **-1** : l'autosuspend est désactivé (réglage cible).
> 
> 

---

## 3. Procédure de correction

### Étape 1 : application immédiate (sans redémarrer)

Pour tester la stabilité tout de suite, forcez la valeur à -1 en mémoire vive :

```bash
echo -1 | sudo tee /sys/module/usbcore/parameters/autosuspend

```

### Étape 2 : rendre le réglage permanent via Grub

Pour que ce réglage survive à un redémarrage, il faut modifier les paramètres de démarrage du noyau.

1. Ouvrez le fichier de configuration de Grub :
`sudo nano /etc/default/grub`
2. Cherchez la ligne `GRUB_CMDLINE_LINUX_DEFAULT`.
3. Ajoutez `usbcore.autosuspend=-1` à l'intérieur des guillemets.
*Exemple :* `GRUB_CMDLINE_LINUX_DEFAULT="quiet usbcore.autosuspend=-1"`
4. Enregistrez (`Ctrl+O`, `Entrée`) et quittez (`Ctrl+X`).
5. **Mettez à jour Grub :**
`sudo update-grub`

---

## 4. La sécurité "Keep alive" (optionnel mais recommandé)

Certains boîtiers USB ont leur propre micro-logiciel (firmware) qui force la mise en veille malgré les réglages de Linux. Pour contrer cela, on crée une activité artificielle très légère.

Ajoutez une tâche planifiée dans la crontab de l'utilisateur root :
`sudo crontab -e`

Ajoutez cette ligne tout en bas :

```cron
# Empêcher la mise en veille physique du disque USB toutes les 15 min
*/15 * * * * ls /media/VOTRE_UTILISATEUR/NOM_DU_DISQUE > /dev/null 2>&1

```

---

## 💡 Conseils supplémentaires

* **Alimentation :** si les erreurs persistent, utilisez un **hub USB avec alimentation externe**. Les disques 2,5" tirent énormément sur le port USB au démarrage des plateaux.
* **Ports USB 3.0 :** privilégiez les ports bleus (USB 3.0/3.1) qui délivrent une intensité plus stable (900mA contre 500mA pour l'USB 2.0).