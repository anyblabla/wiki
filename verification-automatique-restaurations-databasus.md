---
title: Mettre en place la vérification automatique des restaurations avec Databasus
description: Apprenez à configurer un agent de vérification Databasus dans un LXC Debian avec Docker pour tester automatiquement la restauration de vos bases de données après chaque sauvegarde réussie.
published: true
date: 2026-05-22T22:16:14.410Z
tags: docker, lxc, proxmox, sauvegarde, postgresql, databasus, mysql
editor: markdown
dateCreated: 2026-05-22T22:11:52.144Z
---

Un backup qui se termine sans erreur ne garantit pas qu'il est réellement restaurable (droits manquants, extensions absentes, corruption de données). La seule preuve réelle, c'est de tester la restauration.

Ce guide explique comment déployer un agent de vérification Databasus dédié dans un conteneur LXC sous Debian. L'agent récupère automatiquement les sauvegardes, les restaure dans un conteneur Docker éphémère, valide la cohérence des tables (comptage des lignes), puis détruit le conteneur.

---

## 1. Prérequis

* Un conteneur LXC Debian dédié avec Docker pré-installé (le nesting/emboîtement doit être activé dans les options du LXC sur Proxmox).
* Un accès réseau sortant depuis le LXC vers l'instance Databasus.
* **Ressources recommandées** : 2 vCPUs, 2 Go de RAM et un espace disque équivalent à au moins 2x la taille de votre plus gros backup (l'archive et la base restaurée cohabitent temporairement pendant le test).

---

## 2. Création de l'agent dans l'interface Databasus

1. Rendez-vous dans **Settings** → **Verification agents** et cliquez sur **Create verification agent**.
2. Donnez-lui un nom explicite (ex : `srv-verifier-01`).
3. Notez précieusement l'**Agent ID** et le **Token** affichés (le token ne s'affiche qu'une seule fois).

---

## 3. Installation et configuration sur le serveur (LXC)

Connectez-vous en SSH sur votre LXC dédié (en tant que `root` dans le répertoire `/root`) et suivez les étapes suivantes.

### Étape 3.1 : télécharger le binaire de l'agent

Exécutez la commande suivante en remplaçant `http://VOTRE_IP_DATABASUS:PORT` par l'adresse réelle de votre instance Databasus :

```bash
cd /root
curl -L -o verification-agent "http://VOTRE_IP_DATABASUS:PORT/api/v1/system/verification-agent?arch=amd64" && chmod +x verification-agent

```

*(Remplacez `arch=amd64` par `arch=arm64` si votre serveur utilise une architecture ARM).*

### Étape 3.2 : premier lancement manuel

Lancez l'agent une première fois pour valider la connexion et générer le fichier de configuration locale.

> ⚠️ **Note importante** : Si votre instance Databasus tourne en HTTP local (sans HTTPS), vous devez obligatoirement ajouter le flag `--allow-insecure-http` comme indiqué ci-dessous.

```bash
./verification-agent start \
  --databasus-host=http://VOTRE_IP_DATABASUS:PORT \
  --allow-insecure-http \
  --agent-id=VOTRE_AGENT_ID \
  --token=VOTRE_TOKEN \
  --max-cpu=2 \
  --max-ram-mb=2048 \
  --max-disk-gb=20 \
  --max-concurrent-jobs=1

```

**Les variables à adapter selon votre configuration :**

* `--databasus-host` : l'adresse URL complète de votre instance Databasus.
* `--agent-id` : l'ID généré à l'étape 2.
* `--token` : le jeton de sécurité généré à l'étape 2.
* `--max-cpu` / `--max-ram-mb` / `--max-disk-gb` : les limites matérielles maximales que l'agent a le droit d'allouer sur ce LXC pour les conteneurs Docker de test.

Vérifiez que l'agent est bien actif et qu'il détecte votre Docker local :

```bash
./verification-agent status

```

Une fois la mention `Agent is running` validée, arrêtez proprement l'agent en tâche de fond pour passer à l'étape d'automatisation :

```bash
./verification-agent stop

```

---

## 4. Automatisation avec Systemd

Pour que l'agent démarre automatiquement avec le conteneur LXC, nous allons créer un service système.

Créer le fichier de service :

```bash
nano /etc/systemd/system/databasus-verifier.service

```

Coller la configuration suivante (elle utilise le répertoire `/root` où se trouve le binaire) :

```ini
[Unit]
Description=Databasus Verification Agent
After=network.target docker.service
Requires=docker.service

[Service]
Type=simple
WorkingDirectory=/root
# L'argument 'run' permet à systemd de gérer le processus au premier plan
ExecStart=/root/verification-agent run
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target

```

Enregistrer et fermer le fichier (`Ctrl+O`, `Entrée`, puis `Ctrl+X`).

Activer et démarrer le service :

```bash
systemctl daemon-reload
systemctl enable --now databasus-verifier

```

Vérifier le bon fonctionnement du service :

```bash
systemctl status databasus-verifier

```

---

## 5. Activation des tests sur vos bases de données

L'agent apparaît désormais avec un badge vert **Online** dans l'onglet *Settings* de Databasus. Il ne reste plus qu'à planifier les vérifications :

1. Allez sur la fiche de la base de données à surveiller.
2. Dans la section **Restore verification**, cochez **Scheduled verification**.
3. Choisissez l'intervalle **After backup** (fortement recommandé si vos sauvegardes tournent sur un rythme classique type 1 ou 2x par jour). L'agent testera la base de données de manière séquentielle (une par une) juste après chaque dump réussi.
4. Cochez la case **Verification failed** dans les notifications pour être alerté uniquement en cas d'échec du test.
5. Cliquez sur **Save**.

Vous pouvez suivre le statut complet, la durée de restauration et le nombre de lignes importées par table directement depuis l'onglet **Verifications** de chaque base de données.

![verification-automatique-restaurations-databasus.png](/verification-automatique-restaurations-databasus/verification-automatique-restaurations-databasus.png)

---

> **Auteur** : ce guide est proposé par **Amaury aka BlablaLinux**. Retrouvez l'ensemble de mes services sur [blablalinux.be/mes-services-publics/](https://blablalinux.be/mes-services-publics/).
