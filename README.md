# 🍍 Pineapple Cloud

![Stars](https://img.shields.io/github/stars/jaganganesh/pineapple-cloud?style=flat)
![Forks](https://img.shields.io/github/forks/jaganganesh/pineapple-cloud?style=flat)
![License: GPL v3](https://img.shields.io/github/license/jaganganesh/pineapple-cloud)
![Last Commit](https://img.shields.io/github/last-commit/jaganganesh/pineapple-cloud)
[![Sponsor](https://img.shields.io/badge/Sponsor-GitHub%20Sponsors-ea4aaa?logo=githubsponsors&logoColor=white)](https://github.com/sponsors/jaganganesh)

The minimalist, ultra-lightweight, self-hosted privacy alternative to iCloud.

<img src="./assets/images/pineapple-cloud.png" alt="Pineapple Cloud" width="400">

**pineapple-cloud** unifies your **native macOS & iOS Apps (Contacts, Calendar, Reminders, and Notes)** into a single, highly efficient Docker stack running completely on your local network.

Unlike heavy groupware solutions like Nextcloud, which demand over 1GB of RAM and put a heavy load on budget systems, **pineapple-cloud runs under 100MB of idle RAM**. It is optimized to keep mini-servers, Raspberry Pis, and budget NAS units (like the Synology J-series) running cool and responsive.

## 🚀 Why pineapple-cloud?

Apple's native device ecosystem splits accounts across two entirely distinct sync protocols:

1. **CalDAV & CardDAV:** Controls native Calendar timelines, Reminders checklists, and Contact cards.
2. **IMAP (Mail Server Storage):** Operates quietly behind the scenes to sync Apple Notes.

**pineapple-cloud** bridges this gap inside an isolated Docker network by pairing a local **IMAP Mailserver Container** with the streamlined **Radicale DAV Engine**. No bloated databases or heavy web UI dashboards—just clean, protocol-native syncing.

## ✨ Features & Ecosystem Benefits

- 📝 **Native Apple Notes Sync:** Handled natively via local IMAP. No public mail routing or domain MX records required.
- 🗓️ **Native Apple Calendar & Reminders:** Managed directly by Radicale's lightning-fast CalDAV engine.
- 👤 **Native Apple Contacts:** Instant address book populating via CardDAV.
- 🪶 **Low-Resource Engineering:** Under 100MB RAM usage. Minimizes hard drive swap-writes, making it highly safe during sudden power interruptions.
- 📦 **Fully Portable Volume Layout:** Relative pathing structures (`./data`) let you migrate your entire setup between hosts instantly.

## 🛠️ Complete Installation Blueprint

### 1. Initialize Your Directory Structure

Open your terminal and create the configuration tree:

```bash
mkdir -p pineapple-cloud/data/mail-data pineapple-cloud/data/mail-config pineapple-cloud/data/radicale-data pineapple-cloud/data/radicale-config pineapple-cloud/data/radicale-certs
cd pineapple-cloud
```

### 2. Configure Your Docker Compose Environment

Create a file named `docker-compose.yml` and add the verified container service definition layer:

```yaml
services:
  # IMAP Server - Apple Notes
  imap-server:
    image: mailserver/docker-mailserver:latest
    container_name: apple-imap
    ports:
      - "143:143"
    environment:
      OVERRIDE_HOSTNAME: pineapple.cloud
      ENABLE_POP3: "0"
      ENABLE_SMTP: "0"
      ENABLE_SPAMASSASSIN: "0"
      ENABLE_CLAMAV: "0"
      ENABLE_FAIL2BAN: "0"
      ONE_DIR: "1"
    cap_add:
      - NET_ADMIN
    volumes:
      - ./data/mail-data:/var/mail
      - ./data/mail-config:/tmp/docker-mailserver
    restart: unless-stopped

  # DAV Server - Apple Contacts, Calendar and Reminders
  radicale:
    image: tomsquest/docker-radicale:latest
    container_name: apple-dav
    ports:
      - "5232:5232"
    volumes:
      - ./data/radicale-data:/data
      - ./data/radicale-config:/config:ro
      - ./data/radicale-certs:/certs:ro
    restart: unless-stopped
```

### 3. Spin Up the Containers

Launch the core architecture in detached background mode:

```bash
docker compose up -d
```

### 4. Provisioning the Apple Notes IMAP Mailbox

To add your account, drop into the running container's bash shell and use the internal interactive account provisioning script:

Step A: Access the container shell:

```bash
docker exec -it apple-imap /bin/bash
```

Step B: Inside the container prompt, execute the setup script with your chosen identity details and password:

```bash
setup email add your_name@pineapple.cloud
your_secure_password
```

Type `exit` to return to your host terminal when completed.

## 🔐 SSL/TLS Certificate Setup Guide

By default, Apple devices prefer encrypted connections. You have two options to manage your setup:

### Option A: Using Self-Signed Certificates (Secure & Recommended)

To prevent clear-text transfer warnings, generate local self-signed SSL certificates using OpenSSL:

```bash
openssl req -x509 -newkey rsa:4096 -keyout ./data/radicale-certs/server.key -out ./data/radicale-certs/server.cert -sha256 -days 3650 -nodes -subj "/CN=127.0.0.1"
```

Once generated, double-click the `server.cert` file on your Mac to open **Keychain Access**, locate the certificate, and change its properties to **"Always Trust"**.

### Option B: Using Unencrypted HTTP/Plain text

If you run pineapple-cloud strictly on `127.0.0.1` (localhost) or an isolated home router subnet without setting up SSL, macOS will flag the plain-text traffic. When connecting, you must confirm the warning exceptions manually.

## 🖥️ macOS Internet Accounts Configuration Guide

Open **System Settings ➔ Internet Accounts** on your Mac and bind each native service step-by-step using your screenshots as references:

### 1. Connecting Contacts (CardDAV Layer)

- Navigate to **Add Account... ➔ Add Other Account... ➔ CardDAV Account**.
- Change the **Account Type** selector from Automatic to **Manual**.
- **User Name:** Your Radicale username.
- **Server Address:** `127.0.0.1` (or your mini-server network IP).
- **Server Path:** `/` | **Port:** `5232`
- _Note:_ Uncheck **Use SSL** if accessing via HTTP. Check it if you generated local keys.

<img src="./assets/images/cardDAV.png" alt="CardDAV - Apple Contacts" width="400">

### 2. Connecting Calendars & Reminders (CalDAV Layer)

- Navigate to **Add Account... ➔ Add Other Account... ➔ CalDAV Account**.
- Change the **Account Type** selector to **Manual**.
- Input your username and password database configurations as shown:

<img src="./assets/images/calDAV.png" alt="CalDAV - Apple Calendar and Reminders" width="400">

### 3. Connecting Apple Notes (IMAP Layer)

- Navigate to **Add Account... ➔ Add Other Account... ➔ Mail Account**.
- Use your configured full email domain address (e.g., `your_name@pineapple.cloud`).
- When the "Unable to verify" prompt displays, pass the local loopback server IPs:
  - **Incoming Mail Server:** `127.0.0.1`
  - **Outgoing Mail Server (SMTP):** `127.0.0.1`
- Finalize the profile layer by **unchecking Mail** and exclusively **checking Notes**.

<img src="./assets/images/noteIMAP.png" alt="IMAP - Apple Notes" width="400">

## ⚠️ Known Apple Ecosystem Limitations (IMAP Restrictions)

Because Apple strips rich canvas properties when a note is stored over open IMAP protocols rather than inside iCloud core servers, the following features will be modified:

- 🚫 Detailed font-size selectors and custom heading typography templates are disabled.
- 🚫 Native interactive checkbox bubbles, vector sketches, and inline database tables are restricted.
- _💡 Pro-Tip:_ Utilize traditional typography elements like **Bold (`Cmd+B`)**, _Italics (`Cmd+I`)_, or standard hyphens/asterisks (`* `) to create clean, readable lists that render perfectly into plain text HTML folders.

## 🤝 Contributing & Star Support

Have improvements or configuration profiles to share?

1. Fork the codebase.
2. Cut an active development branch (`git checkout -b feature/AmazingFeature`).
3. Commit your adjustments (`git commit -m 'feat: optimize connection handling'`).
4. Push to your branch and open a Pull Request targeting our `develop` tracking branch.

**If this project helped you reclaim ownership of your personal data, drop a ⭐ to help other Apple power-users discover us!**

## ⚖️ License

**pineapple-cloud** is open-source software distributed under the terms of the [GNU GPLv3 (or later)](LICENSE) license.
