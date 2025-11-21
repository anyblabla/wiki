---
title: AppArmor sur Proxmox 9 - Comment j'ai réparé mes conteneurs Docker (LXC imbriqué)
description: Conflit AppArmor/runc sur Proxmox 9 : réparez les erreurs de démarrage des conteneurs Docker dans les LXC imbriqués après la mise à jour de containerd.io. Solutions et retour arrière.
published: true
date: 2025-11-21T14:02:05.701Z
tags: docker, lxc, proxmox, pve, proxmox9, apparmor, conteneur, imbrication, runc, containerd.io, erreur, sécurité, dépannage
editor: markdown
dateCreated: 2025-11-20T00:13:16.158Z
---

> **✅ MISE À JOUR IMPORTANTE (RÉSOLU) :** Ce problème de conflit AppArmor/runc a été corrigé dans **Proxmox Virtual Environment 9.1.1**, ou déjà peut-être dans **9.0.1**. Si votre hôte Proxmox est à jour, les solutions de contournement ci-dessous ne devraient plus être nécessaires. Cet article reste pertinent pour ceux qui utilisent encore Proxmox 9.0 ou qui rencontrent le problème malgré la mise à jour.

-----

## Introduction : le réveil douloureux

Si, comme moi, vous utilisez **Docker** dans des **Conteneurs LXC imbriqués** sur **Proxmox 9**, vous avez probablement eu une mauvaise surprise après une mise à jour récente de `containerd.io`. Cette mise à jour corrigeait une faille critique (CVE-2025-52881), mais elle a complètement brisé mes conteneurs \!

Après cette mise à jour, mes conteneurs LXC ne voulaient plus démarrer Docker, me laissant avec des erreurs de permission frustrantes liées à AppArmor.

## Le problème rencontré (l'erreur typique)

Au lancement, j'obtenais cette longue erreur déroutante, qui pointait vers un **refus de permission** lors de l'ouverture d'un fichier `sysctl` :

```
Error response from daemon: failed to create task for container: failed to create shim task: OCI runtime create failed: runc create failed: unable to start container process: error during container init: open sysctl net.ipv4.ip_unprivileged_port_start file: reopen fd 8: permission denied: unknown
```

## 🧠 L'explication technique (le conflit AppArmor)

J'ai passé du temps à creuser, et il s'avère que c'est un **conflit entre deux mesures de sécurité** :

1.  **runc et les montages détachés :** L'outil d'exécution de conteneurs (`runc`) utilise des montages "détachés" (*detached mounts*) pour renforcer la sécurité.
2.  **AppArmor (le gardien basé sur les chemins) :** AppArmor, intégré dans Proxmox 9, prend ses décisions de permission sur la base des **chemins d'accès complets**.

**Le Blocage :** La mise à jour de sécurité modifie la façon dont `runc` interagit avec le système de fichiers `/proc`. AppArmor n'arrive pas à identifier correctement le chemin du fichier dans ce montage détaché et refuse la permission, bloquant le démarrage du conteneur.

## 🛑 Recommandation principale : ne pas mettre à jour (pour l'instant)

Avant d'appliquer un contournement, **la meilleure solution actuelle est d'éviter la mise à jour des paquets à l'intérieur de vos conteneurs LXC** qui incluent le paquet **`containerd.io`** en version **1.7.28-2 (ou supérieure)**.

Cette mise à jour s'effectue au niveau du système d'exploitation invité **à l'intérieur du conteneur LXC** où Docker est installé. Si vos conteneurs fonctionnent toujours, mettez en attente ces mises à jour (par exemple, en utilisant la commande `apt-mark hold containerd.io` dans le conteneur, si vous utilisez Debian/Ubuntu) jusqu'à ce qu'un profil AppArmor corrigé soit fourni.

## Ma solution : désactiver AppArmor (le contournement si vous avez déjà mis à jour)

> **Remarque importante :** Si vous disposez d'un **instantané (snapshot) ou d'une sauvegarde** du conteneur LXC avant l'installation du paquet `containerd.io` problématique, **revenir à cette version antérieure est la solution la plus propre et la plus recommandée** pour préserver la sécurité de votre conteneur.

Si le retour en arrière n'est pas possible et que vos conteneurs sont bloqués, la seule façon que j'ai trouvée pour que Docker fonctionne à nouveau correctement est de **désactiver AppArmor** spécifiquement pour le conteneur LXC qui pose problème.

> **Attention :** Ce contournement est un compromis sur la sécurité.

### 🛠️ Méthode : configuration du LXC

J'ai modifié le fichier de configuration de mon conteneur LXC sur l'hôte Proxmox (généralement sous `/etc/pve/lxc/<ID_DU_CT>.conf`).

J'ai ajouté les deux lignes suivantes :

```ini
lxc.apparmor.profile: unconfined
lxc.mount.entry: /dev/null sys/module/apparmor/parameters/enabled none bind 0 0
```

N'oubliez pas de **redémarrer le conteneur LXC** après la modification pour que les changements soient appliqués.

### Alternative (que je n'ai pas utilisée)

Une autre astuce est de monter `/dev/null` sur le paramètre qui indique l'état d'AppArmor, forçant Docker à penser qu'il est désactivé :

```bash
mount --bind /dev/null /sys/module/apparmor/parameters/enabled
systemctl restart docker
```

-----

J'espère que ce guide vous aidera à faire fonctionner vos conteneurs en attendant un correctif officiel \!