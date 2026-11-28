---
title: GardeTicket - Keep your receipts and warranties on your phone
description: >-
  GardeTicket: free and private app to keep your receipts and track your
  warranties. Photo or PDF, alerts before expiration, works offline. Everything
  stays on your phone. Code on GitHub and Gitea, Debian + nginx deployment.
published: true
date: '2026-11-28T16:25:46.556Z'
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

**GardeTicket** is a web application (PWA) to store your receipts and track your warranties. Everything is saved **exclusively in your phone's browser**: no data is ever sent to a server.

👉 **Use the application: [gardeticket.blablalinux.be](https://gardeticket.blablalinux.be)** (open from a smartphone or tablet)

## What GardeTicket does

- 📷 Snap a photo of a receipt (automatic cropping, black-and-white "scanner" filter for faded receipts) or import a PDF invoice
- 📅 Calculate the warranty expiration date (1, 2, 3, 5 years, or lifetime) and display color indicators: green, orange (less than 3 months left), gray (expired)
- 🔔 Display a warning banner before expiration and create a reminder in your calendar (`.ics` file) 30 days before the warranty ends
- 🔍 Search for a purchase by item name, store, or serial number
- 📊 View your purchase statistics (number of receipts, total tracked amount, value still under warranty, breakdown by year and by store)
- 📤 Share a receipt or invoice directly with customer support / after-sales service
- 💾 Back up your receipts to a file (`.json`), send them to your cloud (`.txt`), and restore them
- 🎨 Choose from 8 color themes, with automatic light or dark mode
- 📴 Work seamlessly even without an internet connection

## Install the app on your phone

GardeTicket opens in your browser, and you can install it just like a native app:

- **Android** (Chrome, Vivaldi, Brave…): menu (the three dots) → *Add to Home screen* or *Install app*
- **iPhone** (Safari): *Share* button → *Add to Home Screen*

> ℹ️ The application is designed specifically for smartphones and tablets: access from desktop computers is intentionally blocked.

## Privacy and backups

Your receipts remain stored inside your phone's browser. If you clear your browser data, uninstall your browser, or switch phones, **they will be erased**.

Make sure to regularly use the **💾 Sauvegarder** (Backup) button and keep the resulting file in a safe place. The **📂 Restaurer** (Restore) button lets you recover everything (`.json` or `.txt` files) without creating duplicates.

## Self-hosting GardeTicket

The application is a simple static website (HTML, CSS, JavaScript) with no database or external dependencies. You can host it yourself to maintain full control over your instance.

**What you need:**

- a server (a virtual machine, an LXC container, or a Raspberry Pi is plenty);
- a **dedicated domain or subdomain**: the application must be served at the root (`/`);
- **HTTPS**, which is mandatory to install the app on your phone, enable offline mode, and access the camera.

**Which option should you choose?**

| Option | Who is it for? |
| --- | --- |
| **1. Docker Compose** | Recommended if you already self-host your services with Docker |
| **2. Script behind a reverse proxy** | Debian 13 server using Nginx Proxy Manager (or similar) for HTTPS |
| **3. Standalone script** | Debian 13 server handling its own HTTPS via Let's Encrypt |

### Option 1: with Docker Compose (recommended)

The multi-architecture image (`linux/amd64` and `linux/arm64`) is built and published automatically on the GitHub Container Registry (`ghcr.io/anyblabla/gardeticket:latest`) and on my Gitea (`gitea.blablalinux.be/blablalinux/gardeticket:latest`).

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

> ℹ️ Even when self-hosted via Docker, the application remains **100% client-side**: no volume is needed (and adding one would prevent automatic image updates), as receipts remain stored in each phone's browser.

<details>
<summary>Customize application files with Docker (advanced)</summary>

If you want to modify the HTML/CSS/JS code yourself while using Docker, first extract the web directory from the container before mounting it as a volume (note that automatic UI updates will no longer apply):

```bash
docker cp gardeticket:/var/www/gardeticket ./www
```

Then add the volume to your `docker-compose.yml` and restart the container:

```yaml
    volumes:
      - ./www:/var/www/gardeticket:ro
```

</details>

The application then listens over HTTP on port `8080` of your server. Place the container behind a reverse proxy with **HTTPS** enabled (see [Nginx Proxy Manager settings](#nginx-proxy-manager-settings)).

### Option 2: with the installation script, behind a reverse proxy (Debian 13)

The repository includes a script that installs nginx and deploys the files into `/var/www/gardeticket`:

```bash
sudo apt-get install -y git
git clone [https://github.com/anyblabla/gardeticket.git](https://github.com/anyblabla/gardeticket.git)
cd gardeticket
sudo bash deploy/install.sh
```

The server then serves the application over HTTP on port `80` with a minimal vhost. All that remains is to configure your reverse proxy (see [Nginx Proxy Manager settings](#nginx-proxy-manager-settings)).

### Option 3: with the installation script, as a standalone server (HTTPS Let's Encrypt)

```bash
sudo bash deploy/install.sh --standalone --domain tickets.example.com --email you@example.com
```

The nginx vhost will then handle security headers, CSP, desktop blocking, and the Let's Encrypt SSL certificate on its own.

<details>
<summary>Options for the <code>install.sh</code> script</summary>

| Option | Effect |
| --- | --- |
| `--domain NAME` | domain name for the vhost (default: default site) |
| `--standalone` | standalone vhost (security headers, CSP, desktop blocking) |
| `--email ADDRESS` | Let's Encrypt HTTPS (implies `--standalone`, requires `--domain`) |
| `--web-root PATH` | web root directory (default `/var/www/gardeticket`) |
| `--allow-desktop` | with `--standalone`: do not block desktop computers |
| `--update` | redeploy only the web files without touching nginx |
| `--force-nginx` | regenerate the vhost even if it already exists |
| `--uninstall` | remove the site (nginx remains installed) |

The script is idempotent: an existing vhost is not overwritten (unless `--force-nginx` is passed), and the previous site content is backed up in `/var/backups/gardeticket/previous`.

</details>

### Nginx Proxy Manager settings

For options 1 and 2, configure your proxy host in NPM as follows:

- **Details**: `http`, server IP, port `8080` (Docker) or `80` (script);
- **SSL**: certificate, *Force SSL*, *HTTP/2 Support*, *HSTS Enabled*;
- **Custom Nginx Configuration**: paste the contents of [`deploy/npm-advanced-config.conf`](https://github.com/anyblabla/gardeticket/blob/main/deploy/npm-advanced-config.conf) (security headers, CSP, desktop computer blocking);
- **403 "mobile only" page**: copy [`deploy/403_gardeticket/index.html`](https://github.com/anyblabla/gardeticket/blob/main/deploy/403_gardeticket/index.html) into NPM's `data/403_gardeticket/` directory (visible inside the container at `/data/403_gardeticket/`). This is the page desktop visitors will see;
- **Custom Locations**: add `/sw.js` (forwarding to `http://<SERVER_IP>:<PORT>`) with `Cache-Control "no-cache, no-store, must-revalidate"` and `X-Content-Type-Options "nosniff"`, to prevent an outdated service worker from remaining cached.

## Updating

**With Docker Compose:**

```bash
docker compose pull && docker compose up -d
```

Also compatible with Watchtower for automatic updates.

**With the installation script:**

```bash
cd gardeticket
git pull
sudo bash deploy/install.sh --update
```

> ⚠️ The service worker caches files for offline use. If you modify files inside `app/` yourself, make sure to increment `CACHE_NAME` in `app/sw.js` (for example `gardeticket-v28`), otherwise phones will keep serving the older cached version.

## Important notes

- **HTTPS is mandatory**: without it, the service worker and home screen installation will not work.
- **Serve at the root of a domain or subdomain**: `manifest.json` and `sw.js` use absolute paths (`/`, `/icon-192.png`…).
- **Strict CSP** (`script-src 'self'`): no inline JavaScript in HTML pages, which is why `theme-init.js` is a separate file. No `onclick="…"` attributes either.
- **Mobile only**: desktop blocking relies on User-Agent detection, so it can be bypassed. An `Uptime-Kuma` exception allows monitoring checks through.

## Troubleshooting

- **The "Install app" button does not appear, or the camera does not work**: verify that the site is served over **HTTPS**.
- **The old version still appears after an update**: check the `/sw.js` custom location (no-cache headers) and, if you edited the files yourself, verify the `CACHE_NAME` version in `app/sw.js`.
- **An "access denied" page appears on a computer**: this is the intentional desktop block. In standalone mode, you can disable it with `--allow-desktop`.
- **My receipts have disappeared**: they are stored locally in your browser, so clearing browser data removes them. Use **📂 Restaurer** (Restore) with your latest backup file.

## Source code

The code is free and open source, available on two identical repositories:

- 🐙 **GitHub**: [github.com/anyblabla/gardeticket](https://github.com/anyblabla/gardeticket)
- 🍵 **Gitea** (my instance): [gitea.blablalinux.be/blablalinux/gardeticket](https://gitea.blablalinux.be/blablalinux/gardeticket)

Suggestions and bug reports are welcome on either platform.

## License

GardeTicket is free software distributed under the **GNU AGPL v3** license.

---

> **Author**: this guide is proposed by **Amaury aka BlablaLinux**. Find all my services on [blablalinux.be/mes-services-publics/](https://blablalinux.be/mes-services-publics/).
