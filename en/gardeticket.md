---
title: GardeTicket - Keep your receipts and warranties on your phone
description: >-
  GardeTicket: free and private app to keep your receipts and track your
  warranties. Photo or PDF, alerts before expiration, works offline. Everything
  stays on your phone. Code on GitHub and Gitea, Debian + nginx deployment.
published: true
date: '2026-11-24T22:46:31.339Z'
tags:
  - gardeticket
  - pwa
  - warranty
  - receipts
  - privacy
  - open source
  - self hosting
  - nginx
  - debian
  - debian 13
  - nginx proxy manager
editor: markdown
dateCreated: '2026-11-24T22:46:31.339Z'
---

**GardeTicket** is a web application (PWA) to keep your receipts and track your warranties. Everything is stored **exclusively in your phone's memory**: no data is sent to a server.

👉 **Use the application: [gardeticket.blablalinux.be](https://gardeticket.blablalinux.be)** (to be opened from a smartphone or tablet)

## What GardeTicket does

- 📷 Photograph a receipt (automatic cropping, black and white "scanner" filter for faded receipts) or import a PDF invoice
- 📅 Calculate warranty expiration (1, 2, 3, 5 years or lifetime) and display color cues: green, orange (less than 3 months), grey (expired)
- 🔔 Display an alert banner before expiration and create a calendar reminder 30 days before expiration
- 🔍 Find a purchase by item, store or serial number
- 📤 Share the receipt or invoice with after-sales service
- 💾 Backup and restore your tickets in a file
- 🎨 Choose from 8 color themes, with automatic light or dark mode
- 📴 Work even without an internet connection

## Installing the application on your phone

GardeTicket opens in your browser, but you can install it like a real application:

- **Android** (Chrome, Vivaldi, Brave...): menu (three dots) → *Add to Home screen* or *Install app*
- **iPhone** (Safari): *Share* button → *Add to Home Screen*

> ℹ️ The application is designed for smartphones and tablets: access from a desktop computer is intentionally blocked.

## Privacy and backups

Your receipts stay in your phone's browser. If you clear your browser data, uninstall it, or switch phones, **they will be erased**. Remember to regularly use the **💾 Backup** button and keep the resulting file in a safe place. The **📂 Restore** button allows you to recover everything without duplicates.

## Hosting GardeTicket yourself

The application is a simple static site (HTML, CSS, JavaScript), with no database or dependencies. The repository contains a script that installs nginx and deploys files on Debian 13:

```bash
git clone https://github.com/anyblabla/gardeticket.git
cd gardeticket
sudo bash deploy/install.sh
```

- The script offers a "behind a reverse proxy" mode (default) and a standalone mode with Let's Encrypt HTTPS (`--standalone --domain … --email …`).
- **HTTPS is mandatory** to install the application on the phone and use offline mode.
- To update: `git pull && sudo bash deploy/install.sh --update`

All details are in the repository's `README.md` file.

## Source code

The code is open source and available on two identical repositories:

- 🐙 **GitHub**: [github.com/anyblabla/gardeticket](https://github.com/anyblabla/gardeticket)
- 🍵 **Gitea** (my instance): [gitea.blablalinux.be/blablalinux/gardeticket](https://gitea.blablalinux.be/blablalinux/gardeticket)

Suggestions and issue reports are welcome on either platform.

## License

GardeTicket is free software distributed under the **GNU AGPL v3** license.

---

**Author**: this guide is proposed by **Amaury aka BlablaLinux**. Find all my services on [blablalinux.be/mes-services-publics/](https://blablalinux.be/mes-services-publics/).
