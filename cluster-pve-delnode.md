---
title: Proxmox : Supprimer un nœud du cluster
description: Découvrez comment supprimer un nœud d'un cluster Proxmox. Apprenez la commande pvecm delnode et les étapes cruciales (migration, mise hors ligne) pour retirer un serveur en toute sécurité.
published: true
date: 2025-11-05T13:52:16.222Z
tags: proxmox, pve, cluster, delnode, node
editor: markdown
dateCreated: 2025-11-05T13:52:16.222Z
---

### Introduction

Quand on rejoint un cluster dans Proxmox, on peut sans soucis le faire via l'interface graphique.
Cependant, pour retirer une machine du cluster, on ne peut le faire qu'en **ligne de commande**.

Nous allons voir ici comment supprimer une machine du cluster.

-----

### ⚠️ Avertissement crucial : préparer le nœud **à retirer**

Avant de lancer la commande de suppression, il est **impératif** de préparer le nœud que vous souhaitez retirer. **La procédure de suppression (`pvecm delnode`) doit être lancée depuis un nœud ACTIF qui restera dans le cluster, mais elle cible le nœud que vous avez mis hors ligne.**

1.  **Migrez** toutes les Machines Virtuelles (VMs) et Conteneurs (CTs) actifs du nœud à supprimer vers un autre nœud fonctionnel.
2.  **Éteignez** la machine physique ou virtuelle du nœud que vous souhaitez retirer.

-----

### Retirer une machine du cluster

1.  **Se positionner** sur une machine qui va **rester** dans le cluster.

2.  **Lister les nœuds** avec :

    ```bash
    pvecm nodes
    ```

    Ce qui donne un résultat similaire à ceci (nous souhaitons retirer `pve2` dans cet exemple) :

    ```
    Membership information
    ----------------------
        Nodeid      Votes Name
             1          1 pve1
             2          1 pve2
             3          1 pve3 (local)
    ```

    > Le nœud sur lequel on est est précisé avec **(local)**.

3.  Pour supprimer le nœud, on utilise la commande `pvecm delnode`. Dans cet exemple, nous allons retirer le nœud `pve2` :

    ```bash
    pvecm delnode pve2
    ```

    On obtient une confirmation que le nœud est bien retiré :

    ```
    Killing node 2
    ```

4.  **Vérification de la suppression :** Si on relance la commande :

    ```bash
    pvecm nodes
    ```

    On constate que le nœud a bien été supprimé :

    ```
    Membership information
    ----------------------
        Nodeid      Votes Name
             1          1 pve1
             3          1 pve3 (local)
    ```

5.  **Détails du cluster :** Pour obtenir plus de détails sur l'état du cluster, utilisez :

    ```bash
    pvecm status
    ```

    Le résultat montre maintenant le nom du cluster mis à jour (**DCBLABLALINUX**) et uniquement les deux nœuds restants :

    ```
    Cluster information
    -------------------
    Name:              DCBLABLALINUX
    Config Version:    4
    Transport:         knet
    Secure auth:       on

    Quorum information
    ------------------
    Date:              Wed Apr 30 21:09:28 2025
    Quorum provider:   corosync_votequorum
    Nodes:             2
    Node ID:           0x00000003
    Ring ID:           1.4cc
    Quorate:           Yes

    Votequorum information
    ----------------------
    Expected votes:    2
    Highest expected:  2
    Total votes:       2
    Quorum:            2  
    Flags:             Quorate 

    Membership information
    ----------------------
        Nodeid      Votes Name
    0x00000001          1 192.168.2.240
    0x00000003          1 192.168.2.242 (local)
    ```