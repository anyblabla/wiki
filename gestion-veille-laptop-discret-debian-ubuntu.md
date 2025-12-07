---
title: Dépannage de la veille et de l'hibernation sur ordinateur portable (dGPU)
description: Mon guide expert pour résoudre les échecs de Veille (S3) et d'Hibernation (S4) sur les ordinateurs portables sous Debian/Ubuntu/Mint équipés d'une carte graphique discrète (NVIDIA ou AMD).
published: false
date: 2025-12-07T19:10:09.548Z
tags: debian, ubuntu, linux, laptop, veille, suspension, hibernation, acpi, nvidia, amd, mint, pilotes
editor: markdown
dateCreated: 2025-12-07T19:08:15.640Z
---

Introduction : les défis de l'énergie sur les laptops Linux
Lorsque je reconditionne du matériel ou que je dépanne des systèmes pour le collectif Emmabuntüs, je rencontre souvent un problème récurrent : le refus d'un ordinateur portable de passer en mode Veille (Suspension à la RAM, S3) ou en Hibernation (S4).
Ce phénomène touche particulièrement les machines équipées d'une Carte Graphique Discrète (dGPU), qu'elle soit NVIDIA ou AMD. Le cœur du problème réside dans la gestion de l'ACPI (Advanced Configuration and Power Interface) par le noyau Linux, qui doit ordonner à tous les périphériques d'entrer en mode basse consommation.
Je partage ici ma méthodologie de diagnostic et mes solutions éprouvées sur des bases Debian (Debian, Ubuntu, Mint, etc.).
Partie I : La suspension (Veille S3) – mon analyse du matériel
Si mon système ne parvient pas à dormir ou s'il s'éteint mais ne reprend pas, le coupable est presque toujours un composant qui refuse de se conformer à l'ordre S3. Le suspect principal reste le GPU discret ou son pilote.
1.1. J'ai isolé la carte graphique discrète (dGPU)
Sur les systèmes hybrides (Intel/AMD + NVIDIA/AMD), la dGPU est le point de friction.
 * Mon Test Rapide : J'utilise les outils de ma distribution (comme nvidia-prime ou d'autres gestionnaires de puissance) pour basculer temporairement sur la Carte Intégrée (iGPU) ou le mode "Power Saving".
 * Mon Diagnostic : Si la Suspension fonctionne alors, je sais que le problème est lié à la gestion de la dGPU lorsqu'elle est alimentée ou non correctement mise en veille par le pilote propriétaire. Je dois alors m'assurer que j'utilise les pilotes les plus stables (package nvidia-driver sous Debian/Ubuntu, par exemple).
1.2. J'ai forcé le mode Veille S3 via le noyau
Parfois, le système utilise par défaut un mode de veille moins stable (comme le "Modern Standby"). Je préfère forcer l'utilisation du mode S3 (Deep).
 * J'édite le fichier de configuration de GRUB :
   sudo nano /etc/default/grub

 * J'ajoute le paramètre mem_sleep_default=deep dans GRUB_CMDLINE_LINUX_DEFAULT.
   Exemple :
   GRUB_CMDLINE_LINUX_DEFAULT="quiet splash mem_sleep_default=deep"

 * J'actualise GRUB et redémarre pour appliquer la modification :
   sudo update-grub
reboot

1.3. J'ai diagnostiqué l'échec avec journalctl
La méthode la plus fiable pour identifier le périphérique bloquant est d'examiner les logs du noyau après un échec.
 * Je tente la Suspension.
 * Au retour, j'utilise le Terminal pour afficher les messages du cycle de démarrage précédent relatifs à la veille :
   journalctl -b -1 | grep -i suspend

   Je recherche une ligne indiquant un échec de la suspension d'un périphérique (device) ou d'un module (module). Cela me pointe directement le pilote ou le composant responsable (ex : xhci_hcd pour l'USB, un module réseau, etc.).
Partie II : L'hibernation (Veille S4) – ma configuration du swap
Si je veux que mon laptop puisse hiberner (enregistrer l'état de la RAM sur le disque dur), je dois m'assurer que l'espace de Swap est suffisant et correctement référencé par le noyau.
2.1. J'ai vérifié l'espace Swap et son emplacement
Pour l'Hibernation (S4), je m'assure que l'espace de Swap (partition ou fichier) est au moins égal à la quantité de RAM.
 * Vérification de la taille :
   free -h

 * Si j'utilise une Partition Swap : Je récupère l'UUID de la partition (sudo blkid).
 * Si j'utilise un Fichier Swap (courant) :
   * Je trouve le nom de la partition hôte : findmnt -no SOURCE -T /swapfile
   * Je trouve l'offset du fichier (sa position physique) :
     sudo filefrag -v /swapfile | awk '$1=="0:" {print substr($4, 1, length($4)-2)}'

2.2. J'ai configuré le noyau pour la reprise (Resume)
L'Hibernation ne fonctionne que si le noyau sait où chercher l'état sauvegardé au démarrage.
 * J'édite de nouveau /etc/default/grub.
 * J'ajoute les paramètres resume=UUID=... (pour une partition) ou resume=PARTITION et resume_offset=OFFSET (pour un fichier Swap) à la ligne GRUB_CMDLINE_LINUX_DEFAULT.
   Exemple (pour un Fichier Swap) :
   GRUB_CMDLINE_LINUX_DEFAULT="quiet splash mem_sleep_default=deep resume=/dev/nvme0n1p2 resume_offset=65536"

 * J'actualise GRUB et je redémarre.
Partie III : Dernières pistes – les périphériques "éveilleurs"
Même si le GPU est souvent la cause, les périphériques USB peuvent empêcher le système de s'endormir.
 * Périphériques USB et Réseau : Je vérifie quels périphériques sont autorisés à réveiller le système et je les désactive si nécessaire.
   cat /proc/acpi/wakeup

   Pour désactiver un périphérique listé (ex : XHC pour le contrôleur USB), j'utilise : echo XHC > /proc/acpi/wakeup.
 * Mise à jour du firmware : Enfin, j'explore toujours l'option de mettre à jour l'UEFI/BIOS si les problèmes ACPI persistent, car les fabricants peuvent publier des correctifs pour Linux.
En suivant cette méthodologie, je parviens systématiquement à diagnostiquer et à résoudre les problèmes de veille sur la grande majorité des ordinateurs portables sous Linux.