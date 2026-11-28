---
title: GardeTicket - Keep your receipts and warranties on your phone
description: >-
  GardeTicket: free and private app to keep your receipts and track your
  warranties. Photo or PDF, alerts before expiration, works offline. Everything
  stays on your phone. Code on GitHub and Gitea, Debian + nginx deployment.
published: true
date: '2026-11-28T02:45:21.027Z'
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

**GardeTicket** is a web application (PWA) that keeps your receipts and tracks your warranties. Everything is stored **only in your phone's browser**: no data is ever sent to a server.

👉 **Use the app: [gardeticket.blablalinux.be](https://gardeticket.blablalinux.be)** (open it from a smartphone or tablet)

## What GardeTicket does

- 📷 Photograph a receipt (automatic cropping, black-and-white "scanner" filter for faded receipts) or import a PDF invoice
- 📅 Calculate the warranty end date (1, 2, 3, 5 years or lifetime) and show colour markers: green, orange (less than 3 months left), grey (expired)
- 🔔 Show an alert banner before expiry and create a calendar reminder (`.ics` file) 30 days before the end
- 🔍 Find a purchase by item, store or serial number
- 📤 Share the receipt or invoice with an after-sales service
- 💾 Back up your receipts to a file (`.json`), send them to your cloud (`.txt`) and restore them
- 🎨 Choose from 8 colour themes, with automatic light or dark mode
- 📴 Work even without an internet connection

## Install the app on your phone

GardeTicket opens in the browser, but you can install it like a real app:

- **Android** (Chrome, Vivaldi, Brave…): menu (the three dots) → *Add to Home screen* or *Install app*
- **iPhone** (Safari): *Share* button → *Add to Home Screen*

> ℹ️ The app is designed for smartphones and tablets: access from a desktop computer is deliberately blocked.

## Privacy and backups

Your receipts stay in your phone's browser. If you clear the browser data, uninstall the browser or switch phones, **they will be erased**.

Remember to use the **💾 Back up** button regularly and keep the resulting file somewhere safe. The **📂 Restore** button brings everything back (`.json` or `.txt` files), without duplicates.

## Host GardeTicket yourself

The app is a simple static site (HTML, CSS, JavaScript), with no database and no dependencies. You can host it yourself to stay in control of your own instance.

**What you need:**

- a server (a virtual machine, an LXC container or a Raspberry Pi is enough);
- a **dedicated domain or subdomain**: the app must be published at the root (`/`);
- **HTTPS**, required to install the app on the phone, use offline mode and access the camera.

**Which option should you choose?**

| Option | Who is it for? |
| --- | --- |
| **1. Docker Compose** | Recommended if you already self-host your services with Docker |
| **2. Script behind a reverse proxy** | Debian 13 server with Nginx Proxy Manager (or another proxy) handling HTTPS |
| **3. Standalone script** | Debian 13 server that handles HTTPS itself with Let's Encrypt |

### Option 1: with Docker Compose (recommended)

The multi-architecture image (`linux/amd64` and `linux/arm64`) is published automatically on the GitHub Container Registry (`ghcr.io/anyblabla/gardeticket:latest`) and on my Gitea (`gitea.blablalinux.be/blablalinux/gardeticket:latest`).

```bash
mkdir -p gardeticket && cd gardeticket
cat << 'YAML' > docker-compose.yml
services:
  gardeticket:
    image: ghcr.io/anyblabla/gardeticket:latest
    container_name: gardeticket
    restart: unless-stopped
    ports:
      - "8080:80"
YAML
docker compose up -d
```

> ℹ️ Even when hosted with Docker, the app stays **100% client-side**: no volume is needed, and receipts stay stored in each phone's browser.

The app then answers over HTTP on port `8080` of your server. Put the container behind a reverse proxy with **HTTPS** (see [Nginx Proxy Manager settings](#nginx-proxy-manager-settings)).

### Option 2: with the install script, behind a reverse proxy (Debian 13)

The repository contains a script that installs nginx and deploys the files:

```bash
sudo apt-get install -y git
git clone https://github.com/anyblabla/gardeticket.git
cd gardeticket
sudo bash deploy/install.sh
```

The server then serves the app over HTTP on port `80` with a minimal vhost. All that is left is to configure your reverse proxy (see [Nginx Proxy Manager settings](#nginx-proxy-manager-settings)).

### Option 3: with the install script, standalone (Let's Encrypt HTTPS)

```bash
sudo bash deploy/install.sh --standalone --domain tickets.example.org --email you@example.org
```

The nginx vhost then handles the security headers, the CSP, the blocking of desktop computers and the Let's Encrypt certificate by itself.

<details>
<summary>The <code>install.sh</code> script options</summary>

| Option | Effect |
| --- | --- |
| `--domain NAME` | domain name of the vhost (default: default site) |
| `--standalone` | standalone vhost (headers, CSP, desktop blocking) |
| `--email ADDRESS` | Let's Encrypt HTTPS (implies `--standalone`, requires `--domain`) |
| `--web-root PATH` | publishing folder (default `/var/www/gardeticket`) |
| `--allow-desktop` | with `--standalone`: does not block desktop computers |
| `--update` | only redeploys the files, without touching nginx |
| `--force-nginx` | regenerates the vhost even if it already exists |
| `--uninstall` | removes the site (nginx stays installed) |

The script can be run again safely: an existing vhost is not overwritten (unless you use `--force-nginx`) and the previous site content is kept in `/var/backups/gardeticket/previous`.

</details>

### Nginx Proxy Manager settings

For options 1 and 2, here are the host settings in NPM:

- **Details**: `http`, server IP, port `8080` (Docker) or `80` (script);
- **SSL**: certificate, *Force SSL*, *HTTP/2*, *HSTS*;
- **Custom Nginx configuration**: paste the content of [`deploy/npm-advanced-config.conf`](https://github.com/anyblabla/gardeticket/blob/main/deploy/npm-advanced-config.conf) (security headers, CSP, desktop blocking);
- **"Mobile only" 403 page**: copy [`deploy/403_gardeticket/index.html`](https://github.com/anyblabla/gardeticket/blob/main/deploy/403_gardeticket/index.html) into NPM's `data/403_gardeticket/` folder (visible in the container under `/data/403_gardeticket/`). This is what visitors on a desktop computer will see;
- **Custom locations**: add `/sw.js` (to `http://<SERVER_IP>:<PORT>`) with `Cache-Control "no-cache, no-store, must-revalidate"` and `X-Content-Type-Options "nosniff"`, so that an old service worker does not stay cached.

## Update

**With Docker Compose:**

```bash
docker compose pull && docker compose up -d
```

Also compatible with Watchtower for automatic updates.

**With the install script:**

```bash
cd gardeticket
git pull
sudo bash deploy/install.sh --update
```

> ⚠️ The service worker works in "cache first" mode. If you modify the files in `app/` yourself, increment `CACHE_NAME` in `app/sw.js` (for example `gardeticket-v26`), otherwise phones will keep the old version.

## Important points

- **HTTPS is mandatory**: without it, the service worker and installing the app do not work.
- **Publish at the root of a domain or subdomain**: `manifest.json` and `sw.js` use absolute paths (`/`, `/icon-192.png`…).
- **Strict CSP** (`script-src 'self'`): no inline JavaScript in the pages, hence the `theme-init.js` file. No `onclick="…"` either.
- **Mobile only**: desktop blocking relies on the User-Agent, so it can be bypassed. The `Uptime-Kuma` exception lets monitoring through.

## Something wrong?

- **The "Install app" button does not appear, or the camera does not work**: check that the site is served over **HTTPS**.
- **The old version still shows after an update**: check the `/sw.js` custom location (no cache) and, if you modified the files, the `CACHE_NAME` number in `app/sw.js`.
- **An "access denied" page shows on a desktop computer**: this is the deliberate blocking of desktop computers. In standalone mode, you can remove it with `--allow-desktop`.
- **My receipts have disappeared**: they are stored in the browser, so they are erased if its data was cleared. Use **📂 Restore** with your latest backup.

## Source code

The code is free software and available on two identical repositories:

- 🐙 **GitHub**: [github.com/anyblabla/gardeticket](https://github.com/anyblabla/gardeticket)
- 🍵 **Gitea** (my instance): [gitea.blablalinux.be/blablalinux/gardeticket](https://gitea.blablalinux.be/blablalinux/gardeticket)

Suggestions and issue reports are welcome on either one.

## License

GardeTicket is free software released under the **GNU AGPL v3** license.

---

Auteur : ce guide est proposé par Amaury alias BlablaLinux. Retrouvez tous mes services sur [Mes services publics](https://blablalinux.be/mes-services-publics/).
