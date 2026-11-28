---
title: GardeTicket - Keep your receipts and warranties on your phone
description: >-
  GardeTicket: free and private app to keep your receipts and track your
  warranties. Photo or PDF, alerts before expiration, works offline. Everything
  stays on your phone. Code on GitHub and Gitea, Debian + nginx deployment.
published: true
date: '2026-11-28T21:02:53.195Z'
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

**GardeTicket** is a Progressive Web App (PWA) for keeping your warranties and grocery receipts. Everything is stored **locally in your phone's browser**: no data is ever sent to a server.

👉 **Use the app: [gardeticket.blablalinux.be](https://gardeticket.blablalinux.be)** (open from a smartphone or tablet)

## What GardeTicket Does

- 🗂️ Separate your purchases into two dedicated spaces: **🛡️ Garanties** (*Warranties* for covered appliances and invoices) and **🛒 Courses** (*Groceries* for supermarket receipts, with automatic monthly budget tracking and one-tap cleanup for receipts older than 3 months)
- 📷 Snap a receipt photo (auto-cropping and a black-and-white "scanner" filter are enabled by default for crisp, lightweight scans) or import a PDF invoice
- 📅 Automatically calculate warranty expiration dates (1, 2, 3, 5 years, or lifetime) with color-coded badges: green, orange (under 3 months left), and gray (expired)
- 🔔 Display an alert banner before a warranty expires and generate a calendar reminder (`.ics` file) set 30 days ahead
- 🔍 Search purchases by item name, store, or serial number
- 📊 View purchase statistics (calculated separately for warranties and groceries: total receipts, total tracked spending, value still under warranty, and breakdowns by year and store)
- 📤 Share a receipt or invoice directly with customer support
- 💾 Back up your receipts to a local file (`.json`), send them to your cloud (`.txt`), and restore without duplicates
- 🎨 Choose from 8 color themes with automatic light or dark mode
- 📴 Works completely offline

## Installing the App on Your Phone

GardeTicket runs in your web browser, and you can install it like a native app on your home screen:

- **Android** (Chrome, Vivaldi, Brave…): open the browser menu (three dots) → *Add to Home screen* or *Install app*
- **iPhone** (Safari): tap the *Share* button → *Add to Home Screen*

> ℹ️ The app is designed specifically for smartphones and tablets: access from desktop browsers is intentionally blocked.

## Privacy and Backups

Your receipts stay inside your phone's browser storage. If you clear your browser data, uninstall the browser, or switch phones, **they will be erased**.

Remember to regularly tap **💾 Sauvegarder** (*Save*) or **🌐 Vers le cloud** (*To cloud*) and store the exported file in a safe place. Tap **📂 Restaurer** (*Restore*) to recover your receipts (`.json` or `.txt` files) without creating duplicates.

## Self-Hosting GardeTicket

GardeTicket is a lightweight static web app (HTML, CSS, JavaScript) with no database or backend dependencies. You can easily self-host your own instance.

**What you need:**

- a server (a VM, an LXC container, or a Raspberry Pi is plenty);
- a **dedicated domain or subdomain**: the app must be served from the root path (`/`);
- **HTTPS**, required for PWA installation, offline caching, and camera access.

**Which deployment option should you choose?**

| Option | Best for |
| --- | --- |
| **1. Docker Compose** | Recommended if you already self-host services with Docker |
| **2. Install script behind a reverse proxy** | Debian 13 server using Nginx Proxy Manager (or another reverse proxy) for HTTPS |
| **3. Standalone install script** | Debian 13 server handling HTTPS directly with Let's Encrypt |

### Option 1: Using Docker Compose (Recommended)

The multi-architecture image (`linux/amd64` and `linux/arm64`) is automatically published on GitHub Container Registry (`ghcr.io/anyblabla/gardeticket:latest`) and on my Gitea instance (`gitea.blablalinux.be/blablalinux/gardeticket:latest`).

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

> ℹ️ Even when self-hosted with Docker, the app runs **100% client-side**: no persistent volume is needed (and mounting one would block automatic UI updates when pulling new images), as all receipts remain stored in each user's phone browser.

<details>
<summary>Customizing the app files with Docker (advanced)</summary>

If you want to customize the HTML/CSS/JS files yourself while using Docker, first extract the web directory from the container before mounting it as a volume (note that automatic UI updates from image pulls will no longer apply):

```bash
docker cp gardeticket:/var/www/gardeticket ./www
```

Then add the volume to `docker-compose.yml` and restart the container:

```yaml
    volumes:
      - ./www:/var/www/gardeticket:ro
```

</details>

The container listens in HTTP on port `8080`. Place it behind an **HTTPS** reverse proxy (see [Nginx Proxy Manager Settings](#nginx-proxy-manager-settings)).

### Option 2: Using the Install Script Behind a Reverse Proxy (Debian 13)

The repository includes an installation script that sets up nginx and deploys the files to `/var/www/gardeticket`:

```bash
sudo apt-get install -y git
git clone [https://github.com/anyblabla/gardeticket.git](https://github.com/anyblabla/gardeticket.git)
cd gardeticket
sudo bash deploy/install.sh
```

The server will serve the app over HTTP on port `80` with a minimal vhost. Next, configure your reverse proxy (see [Nginx Proxy Manager Settings](#nginx-proxy-manager-settings)).

### Option 3: Using the Install Script in Standalone Mode (Let's Encrypt HTTPS)

```bash
sudo bash deploy/install.sh --standalone --domain tickets.example.com --email you@example.com
```

The nginx vhost will handle security headers, CSP, desktop browser blocking, and the Let's Encrypt SSL certificate directly.

<details>
<summary>Available options for <code>install.sh</code></summary>

| Option | Effect |
| --- | --- |
| `--domain NAME` | Domain name for the vhost (default: default site) |
| `--standalone` | Standalone vhost (security headers, CSP, desktop blocking) |
| `--email ADDRESS` | Enable Let's Encrypt HTTPS (implies `--standalone`, requires `--domain`) |
| `--web-root PATH` | Web root directory (default: `/var/www/gardeticket`) |
| `--allow-desktop` | With `--standalone`: allow desktop browsers |
| `--update` | Redeploy web files only without touching nginx config |
| `--force-nginx` | Regenerate the nginx vhost even if it already exists |
| `--uninstall` | Remove the site configuration (nginx remains installed) |

The script is idempotent: an existing vhost will not be overwritten (unless `--force-nginx` is passed), and previous web files are backed up to `/var/backups/gardeticket/previous`.

</details>

### Nginx Proxy Manager Settings

For options 1 and 2, configure your Proxy Host in NPM as follows:

- **Details**: `http`, server IP, port `8080` (Docker) or `80` (script);
- **SSL**: select your certificate, enable *Force SSL*, *HTTP/2 Support*, and *HSTS Enabled*;
- **Custom Nginx Configuration**: paste the contents of [`deploy/npm-advanced-config.conf`](https://github.com/anyblabla/gardeticket/blob/main/deploy/npm-advanced-config.conf) (security headers, CSP, and desktop browser blocking);
- **Custom 403 "mobile-only" page**: copy [`deploy/403_gardeticket/index.html`](https://github.com/anyblabla/gardeticket/blob/main/deploy/403_gardeticket/index.html) into NPM's `data/403_gardeticket/` directory (mounted inside the container at `/data/403_gardeticket/`). This page is displayed to desktop visitors;
- **Custom Locations**: add `/sw.js` (pointing to `http://<SERVER_IP>:<PORT>`) with `Cache-Control "no-cache, no-store, must-revalidate"` and `X-Content-Type-Options "nosniff"` so browsers never serve a stale service worker from cache.

## Updating

**With Docker Compose:**

```bash
docker compose pull && docker compose up -d
```

Also compatible with Watchtower for automated image updates.

**With the install script:**

```bash
cd gardeticket
git pull
sudo bash deploy/install.sh --update
```

> ⚠️ The service worker caches files for offline use. If you modify files inside `app/` yourself, bump `CACHE_NAME` in `app/sw.js` (for example `gardeticket-v30`), otherwise phones will keep using the cached version.

## Important Notes

- **HTTPS is mandatory**: without HTTPS, the service worker and home-screen installation will not work.
- **Serve at the domain or subdomain root**: `manifest.json` and `sw.js` rely on root-relative paths (`/`, `/icon-192.png`, etc.).
- **Strict CSP** (`script-src 'self'`): no inline JavaScript is allowed in HTML pages (hence `theme-init.js`), and no inline `onclick="…"` handlers are used.
- **Mobile-only**: desktop blocking checks the User-Agent string and can be bypassed. An exception is included for `Uptime-Kuma` monitoring.

## Troubleshooting

- **The "Install app" prompt does not appear, or the camera won't open**: make sure the site is served over **HTTPS**.
- **An older version still loads after an update**: check the `/sw.js` custom location (no-cache headers) and, if you edited files manually, verify `CACHE_NAME` in `app/sw.js`.
- **An "Access Denied" page appears on desktop**: this is the intentional desktop browser block. In standalone mode, you can disable it with `--allow-desktop`.
- **My receipts disappeared**: receipts are stored in browser storage, so clearing browser data erases them. Use **📂 Restaurer** (*Restore*) with your latest backup file.

## Source Code

GardeTicket is open-source and mirrored across two identical repositories:

- 🐙 **GitHub**: [github.com/anyblabla/gardeticket](https://github.com/anyblabla/gardeticket)
- 🍵 **Gitea** (self-hosted instance): [gitea.blablalinux.be/blablalinux/gardeticket](https://gitea.blablalinux.be/blablalinux/gardeticket)

Suggestions and bug reports are welcome on either platform.

## License

GardeTicket is free software licensed under the **GNU AGPL v3**.

---

> **Author**: this guide is proposed by **Amaury aka BlablaLinux**. Find all my services on [blablalinux.be/mes-services-publics/](https://blablalinux.be/mes-services-publics/).
