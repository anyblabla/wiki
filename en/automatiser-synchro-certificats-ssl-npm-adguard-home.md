---
title: Synchronize SSL Certificates from NPM to AdGuard Home
description: >-
  Step-by-step guide to automate copying and renewing Let's Encrypt SSL
  certificates from Nginx Proxy Manager (Docker) to one or multiple AdGuard Home
  instances via SSH and Cron.
published: true
date: '2026-11-20T21:50:08.055Z'
tags:
  - npm
  - nginx proxy manager
  - adguard
  - docker
  - ssl
  - tls
  - lets encrypt
  - certificate
  - linux
  - bash
  - cron
  - ssh
  - automation
editor: markdown
dateCreated: '2026-11-20T21:50:08.055Z'
---

Every three months, when Let's Encrypt certificates are automatically renewed by Nginx Proxy Manager (NPM), you normally have to manually retrieve the newly numbered files (`fullchain12.pem`, `privkey12.pem`, etc.) and update the configuration of AdGuard Home.

This guide explains how to fully automate this task: AdGuard Home will point to fixed file paths, and a script will check every night if the certificate has changed to push it over SSH and restart the service.

---

## Prerequisites and Architecture

* **NPM Server:** An LXC host or container running Nginx Proxy Manager under Docker, with the `letsencrypt` folder mounted locally (e.g., `/root/npm/letsencrypt`).
* **AdGuard Home Servers:** One or multiple AdGuard Home instances (here, two instances in high availability: `192.168.1.10` and `192.168.1.11`).
* **Network Access:** The NPM host must be able to reach the AdGuard Home servers over SSH.

---

## Step 1: Locate the Certificate Folder on the NPM Host

In the `live/` directory of Nginx Proxy Manager, Certbot constantly maintains unnumbered symlinks (`fullchain.pem` and `privkey.pem`) that always point to the latest valid version of the certificate.

On your machine hosting NPM, list the contents of the `live` directory:

```bash
ls -la /root/npm/letsencrypt/live/
```

Locate the subdirectory name corresponding to your certificate (for example `npm-1`). Verify that it is indeed the correct certificate by displaying its expiration date:

```bash
openssl x509 -in /root/npm/letsencrypt/live/npm-1/fullchain.pem -noout -enddate
```

> **To adapt:** Replace `/root/npm/letsencrypt` with the actual path to your NPM volume, and `npm-1` with the folder name shown on your system.

---

## Step 2: Configure Passwordless SSH Access

To allow the script to copy files without human intervention, generate a dedicated SSH key pair on the NPM host and deploy it to each AdGuard Home instance.

1. Generate the dedicated SSH key:

```bash
ssh-keygen -t ed25519 -N "" -f ~/.ssh/id_adguard_cert -C "sync-cert-adguard"
```

2. Copy the public key to your AdGuard Home servers (adapt IP addresses):

```bash
ssh-copy-id -i ~/.ssh/id_adguard_cert.pub root@192.168.1.10
ssh-copy-id -i ~/.ssh/id_adguard_cert.pub root@192.168.1.11
```

3. Test direct connectivity to verify that no password is requested:

```bash
ssh -i ~/.ssh/id_adguard_cert root@192.168.1.10 "hostname" && ssh -i ~/.ssh/id_adguard_cert root@192.168.1.11 "hostname"
```

---

## Step 3: Create the Synchronization Script

This script compares the SHA256 hash of the current certificate with the last known hash. If the certificate has not changed, the script stops immediately. If it was renewed, it copies the files to each AdGuard Home instance and restarts the service.

1. Create the scripts directory (if it does not exist) and the `sync-cert-adguard.sh` file:

```bash
mkdir -p /root/scripts
cat << 'EOF' > /root/scripts/sync-cert-adguard.sh
#!/bin/bash

# --- VARIABLES TO CUSTOMIZE ---
CERT_DIR="/root/npm/letsencrypt/live/npm-1"
ADGUARD_HOSTS=("192.168.1.10" "192.168.1.11")
# ------------------------------

SSH_KEY="/root/.ssh/id_adguard_cert"
DEST_DIR="/root/ssl"
HASH_FILE="/var/tmp/adguard_cert.sha256"

# Calculate current certificate hash (automatically follows symlinks)
CURRENT_HASH=$(sha256sum "$CERT_DIR/fullchain.pem" | awk '{print $1}')
LAST_HASH=$(cat "$HASH_FILE" 2>/dev/null)

# If the certificate hasn't changed or is missing, stop here
if [ -z "$CURRENT_HASH" ] || [ "$CURRENT_HASH" == "$LAST_HASH" ]; then
    exit 0
fi

SUCCESS=true

for HOST in "${ADGUARD_HOSTS[@]}"; do
    ssh -i "$SSH_KEY" "root@$HOST" "mkdir -p $DEST_DIR"
    scp -i "$SSH_KEY" -q "$CERT_DIR/fullchain.pem" "$CERT_DIR/privkey.pem" "root@$HOST:$DEST_DIR/"
    
    if [ $? -eq 0 ]; then
        ssh -i "$SSH_KEY" "root@$HOST" "systemctl restart AdGuardHome"
    else
        SUCCESS=false
    fi
done

# Save the new hash only if all copies succeeded
if [ "$SUCCESS" = true ]; then
    echo "$CURRENT_HASH" > "$HASH_FILE"
fi
EOF
```

> **What to customize in this script:**
> * `CERT_DIR`: The full path to the `npm-X` folder identified in Step 1.
> * `ADGUARD_HOSTS`: The list of IP addresses (or local hostnames) of your AdGuard Home servers, separated by a space. If you only have one server, simply set `ADGUARD_HOSTS=("192.168.1.10")`.
> * *Note:* If AdGuard Home runs in Docker on the remote server, replace `systemctl restart AdGuardHome` with `docker restart adguardhome`.

2. Make the script executable and run it for the first time to initialize the files:

```bash
chmod +x /root/scripts/sync-cert-adguard.sh
/root/scripts/sync-cert-adguard.sh
```

3. Verify that the files `fullchain.pem` and `privkey.pem` are present in `/root/ssl/` on your AdGuard Home servers:

```bash
ssh -i ~/.ssh/id_adguard_cert root@192.168.1.10 "ls -lh /root/ssl/"
```

---

## Step 4: Configure Fixed Paths in AdGuard Home

Now you just need to instruct AdGuard Home to read from these new fixed locations.

1. Log in to the **AdGuard Home** web interface.
2. Go to **Settings** > **Encryption settings**.
3. Check **Set path to certificate file** and enter:
   ```text
   /root/ssl/fullchain.pem
   ```
4. Check **Set path to private key file** and enter:
   ```text
   /root/ssl/privkey.pem
   ```
5. Click **Save configuration**.

> **Important:** If you use `adguardhome-sync` to replicate your configuration between two instances, note that encryption (TLS) settings are not synchronized. Remember to apply this change manually on each AdGuard Home instance.

### Optional Cleanup

If you used to place your old numbered certificates directly in `/root/` on your AdGuard Home servers, you can now safely remove them without affecting the `/root/ssl/` folder:

```bash
ssh -i ~/.ssh/id_adguard_cert root@192.168.1.10 "rm -f /root/cert*.pem /root/chain*.pem /root/fullchain*.pem /root/privkey*.pem"
ssh -i ~/.ssh/id_adguard_cert root@192.168.1.11 "rm -f /root/cert*.pem /root/chain*.pem /root/fullchain*.pem /root/privkey*.pem"
```

---

## Step 5: Schedule Automatic Execution with Cron

On the Nginx Proxy Manager host, add a cron job to run the script every day at 4:00 AM:

```bash
(crontab -l 2>/dev/null; echo "0 4 * * * /root/scripts/sync-cert-adguard.sh") | crontab -
```

Verify that the job is properly saved:

```bash
crontab -l
```

As soon as Nginx Proxy Manager renews your certificate, it will be automatically pushed to your AdGuard Home instances during the following night without any manual action required.

---

> **Author**: This guide is provided by **Amaury aka BlablaLinux**. Find all my public services at [blablalinux.be/mes-services-publics/](https://blablalinux.be/mes-services-publics/).
