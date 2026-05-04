# 🦅 Pterodactyl Panel — Full Installation Guide

> **Authored by [闇 𝐋𝐎𝐑𝐃 𝐃𝐄𝐕𝐈𝐍𝐄 闇](https://github.com/Heis-Devine)** · Battle-tested on Ubuntu 24.04 · No auto-scripts — pure manual install with real explanations.

---

```
██████╗ ████████╗███████╗██████╗  ██████╗ ██████╗  █████╗  ██████╗████████╗██╗   ██╗██╗
██╔══██╗╚══██╔══╝██╔════╝██╔══██╗██╔═══██╗██╔══██╗██╔══██╗██╔════╝╚══██╔══╝╚██╗ ██╔╝██║
██████╔╝   ██║   █████╗  ██████╔╝██║   ██║██║  ██║███████║██║        ██║    ╚████╔╝ ██║
██╔═══╝    ██║   ██╔══╝  ██╔══██╗██║   ██║██║  ██║██╔══██║██║        ██║     ╚██╔╝  ██║
██║        ██║   ███████╗██║  ██║╚██████╔╝██████╔╝██║  ██║╚██████╗   ██║      ██║   ███████╗
╚═╝        ╚═╝   ╚══════╝╚═╝  ╚═╝ ╚═════╝ ╚═════╝ ╚═╝  ╚═╝ ╚═════╝  ╚═╝      ╚═╝   ╚══════╝
```

---

## 📋 Table of Contents

- [Requirements](#-requirements)
- [Phase 1 — Domain & SSL First](#-phase-1--domain--ssl-first)
- [Phase 2 — System Preparation](#-phase-2--system-preparation)
- [Phase 3 — Install the Panel](#-phase-3--install-the-panel)
- [Phase 4 — Configure Nginx with HTTPS](#-phase-4--configure-nginx-with-https)
- [Phase 5 — Panel Setup & Admin User](#-phase-5--panel-setup--admin-user)
- [Phase 6 — Install Wings Daemon](#-phase-6--install-wings-daemon)
- [Phase 7 — Connect Wings to Panel](#-phase-7--connect-wings-to-panel)
- [Troubleshooting](#-troubleshooting)

---

## ✅ Requirements

| Requirement | Details |
|-------------|---------|
| **OS** | Ubuntu 22.04 or 24.04 |
| **RAM** | 2GB minimum (4GB+ recommended) |
| **CPU** | 2+ vCPU |
| **Disk** | 20GB+ free space |
| **Domain** | A domain/subdomain pointing to your server IP |
| **Ports Open** | 80, 443, 8080, 2022 |

> 💡 **Free Domain Tip:** Go to [duckdns.org](https://duckdns.org), create a free subdomain like `yourname.duckdns.org` and point it to your server IP **before you begin**. SSL won't work without a domain.

---

## 🟣 Phase 1 — Domain & SSL First

> **Why first?** Setting up SSL before everything else means Nginx, Wings, and the panel all start life on HTTPS. This avoids the painful back-and-forth of switching from HTTP to HTTPS mid-setup.

### 1.1 — Point Your Domain to Your Server

Log into your domain provider (or DuckDNS) and set your domain's IP to your server's public IP address. Wait a few minutes for it to propagate.

Verify it works:
```bash
ping yourdomain.duckdns.org
```

You should see your server IP in the response.

### 1.2 — Install Certbot & Nginx

```bash
apt update && apt install -y certbot python3-certbot-nginx nginx
```

### 1.3 — Get Your SSL Certificate

```bash
certbot certonly --standalone -d yourdomain.duckdns.org \
  --non-interactive --agree-tos -m youremail@gmail.com
```

> ⚠️ Make sure **port 80 is open** on your server before running this. Certbot uses it to verify domain ownership.

Your certificates will be saved at:
- **Certificate:** `/etc/letsencrypt/live/yourdomain.duckdns.org/fullchain.pem`
- **Private Key:** `/etc/letsencrypt/live/yourdomain.duckdns.org/privkey.pem`

Keep these paths handy — you'll use them in multiple places.

---

## 🟣 Phase 2 — System Preparation

### 2.1 — Update System

```bash
apt update && apt upgrade -y
```

### 2.2 — Install Core Dependencies

```bash
apt install -y curl wget git tar unzip
curl -sL https://deb.nodesource.com/setup_20.x | bash -
apt install -y nodejs
```

### 2.3 — Install Docker

Wings uses Docker to run server containers.

```bash
curl -sSL https://get.docker.com/ | SHELL=bash sh
systemctl enable --now docker
```

### 2.4 — Install Redis

Redis handles the panel's queue jobs.

```bash
apt install -y redis-server
systemctl enable --now redis-server
```

### 2.5 — Install MariaDB

```bash
apt install -y mariadb-server
mysql_secure_installation
```

Answer the prompts:

| Question | Answer |
|----------|--------|
| Switch to unix_socket authentication | **N** |
| Change root password | **Y** → choose a strong password |
| Remove anonymous users | **Y** |
| Disallow root login remotely | **Y** |
| Remove test database | **Y** |
| Reload privilege tables | **Y** |

### 2.6 — Create the Panel Database

```bash
mysql -u root -p
```

Run these SQL commands one by one:

```sql
CREATE USER 'pterodactyl'@'127.0.0.1' IDENTIFIED BY 'YourDBPassword';
CREATE USER 'pterodactyl'@'localhost' IDENTIFIED BY 'YourDBPassword';
CREATE DATABASE panel;
GRANT ALL PRIVILEGES ON panel.* TO 'pterodactyl'@'127.0.0.1' WITH GRANT OPTION;
GRANT ALL PRIVILEGES ON panel.* TO 'pterodactyl'@'localhost' WITH GRANT OPTION;
FLUSH PRIVILEGES;
EXIT;
```

> ⚠️ **Critical:** Always create the user for BOTH `127.0.0.1` AND `localhost`. Missing either one causes a frustrating "Access denied" error during panel setup.

---

## 🟣 Phase 3 — Install the Panel

### 3.1 — Install PHP & Nginx

```bash
apt install -y php8.3 php8.3-cli php8.3-gd php8.3-mysql php8.3-pdo \
php8.3-mbstring php8.3-tokenizer php8.3-bcmath php8.3-xml \
php8.3-fpm php8.3-curl php8.3-zip nginx
```

### 3.2 — Install Composer

```bash
curl -sS https://getcomposer.org/installer | php -- \
  --install-dir=/usr/local/bin --filename=composer
```

### 3.3 — Download the Panel

```bash
mkdir -p /var/www/pterodactyl
cd /var/www/pterodactyl
curl -Lo panel.tar.gz https://github.com/pterodactyl/panel/releases/latest/download/panel.tar.gz
tar -xzvf panel.tar.gz
chmod -R 755 storage/* bootstrap/cache/
```

### 3.4 — Setup Environment

```bash
cp .env.example .env
composer install --no-dev --optimize-autoloader
php artisan key:generate --force
```

### 3.5 — Run Environment Wizard

```bash
php artisan p:environment:setup
```

| Question | Answer |
|----------|--------|
| App URL | `https://yourdomain.duckdns.org` |
| App timezone | `Africa/Lagos` (or your timezone) |
| Cache driver | `file` |
| Session driver | `file` |
| Queue driver | `redis` |
| Enable UI settings editor | `yes` |
| Redis Host | Press **Enter** (default 127.0.0.1) |
| Redis Password | Press **Enter** (none) |
| Redis Port | Press **Enter** (default 6379) |

### 3.6 — Configure Database Connection

```bash
php artisan p:environment:database
```

| Question | Answer |
|----------|--------|
| Database host | `127.0.0.1` |
| Database port | `3306` |
| Database name | `panel` |
| Database username | `pterodactyl` |
| Database password | `YourDBPassword` |

### 3.7 — Run Migrations

```bash
php artisan migrate --seed --force
```

---

## 🟣 Phase 4 — Configure Nginx with HTTPS

### 4.1 — Remove Default Config

```bash
rm /etc/nginx/sites-enabled/default
```

### 4.2 — Create Panel Config

```bash
nano /etc/nginx/sites-available/pterodactyl.conf
```

Paste this (replace `yourdomain.duckdns.org` with your actual domain):

```nginx
server {
    listen 80;
    server_name yourdomain.duckdns.org;
    return 301 https://$server_name$request_uri;
}

server {
    listen 443 ssl http2;
    server_name yourdomain.duckdns.org;

    root /var/www/pterodactyl/public;
    index index.html index.htm index.php;
    charset utf-8;

    ssl_certificate /etc/letsencrypt/live/yourdomain.duckdns.org/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/yourdomain.duckdns.org/privkey.pem;
    ssl_session_cache shared:SSL:10m;
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers "ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256";
    ssl_prefer_server_ciphers on;

    location / {
        try_files $uri $uri/ /index.php?$query_string;
    }

    location = /favicon.ico { access_log off; log_not_found off; }
    location = /robots.txt  { access_log off; log_not_found off; }

    access_log off;
    error_log  /var/log/nginx/pterodactyl.app-error.log error;

    client_max_body_size 100m;
    client_body_timeout 120s;

    sendfile off;

    location ~ \.php$ {
        fastcgi_split_path_info ^(.+\.php)(/.+)$;
        fastcgi_pass unix:/run/php/php8.3-fpm.sock;
        fastcgi_index index.php;
        include fastcgi_params;
        fastcgi_param PHP_VALUE "upload_max_filesize = 100M \n post_max_size=100M";
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        fastcgi_param HTTP_PROXY "";
        fastcgi_intercept_errors off;
        fastcgi_buffer_size 16k;
        fastcgi_buffers 4 16k;
        fastcgi_connect_timeout 300;
        fastcgi_send_timeout 300;
        fastcgi_read_timeout 300;
    }

    location ~ /\.ht {
        deny all;
    }
}
```

### 4.3 — Enable and Restart Nginx

```bash
ln -s /etc/nginx/sites-available/pterodactyl.conf /etc/nginx/sites-enabled/
nginx -t
systemctl restart nginx
chown -R www-data:www-data /var/www/pterodactyl/*
```

---

## 🟣 Phase 5 — Panel Setup & Admin User

### 5.1 — Create Admin User

```bash
php artisan p:user:make
```

| Question | Answer |
|----------|--------|
| Is this user an administrator | `yes` |
| Email | your email |
| Username | your username |
| First Name | your name |
| Last Name | your name |
| Password | **min 8 chars, 1 capital, 1 number** |

### 5.2 — Setup Queue Worker Service

```bash
nano /etc/systemd/system/pteroq.service
```

Paste:

```ini
[Unit]
Description=Pterodactyl Queue Worker
After=redis-server.service

[Service]
User=www-data
Group=www-data
Restart=always
ExecStart=/usr/bin/php /var/www/pterodactyl/artisan queue:work --queue=high,standard,low --sleep=3 --tries=3
StartLimitInterval=180
StartLimitBurst=30
RestartSec=5s

[Install]
WantedBy=multi-user.target
```

```bash
systemctl enable --now pteroq.service
```

### 5.3 — Access Your Panel

Open your browser and visit:

```
https://yourdomain.duckdns.org
```

Login with the admin credentials you just created. 🎉

---

## 🟣 Phase 6 — Install Wings Daemon

Wings is what actually runs your game servers/containers.

### 6.1 — Install Wings Binary

```bash
mkdir -p /etc/pterodactyl
curl -L -o /usr/local/bin/wings \
  "https://github.com/pterodactyl/wings/releases/latest/download/wings_linux_amd64"
chmod u+x /usr/local/bin/wings
```

### 6.2 — Create a Node on the Panel

1. Go to **Admin → Locations → Create** — name it anything (e.g. `Nigeria`)
2. Go to **Admin → Nodes → Create New**

Fill in:

| Field | Value |
|-------|-------|
| Name | `Node1` (or anything you like) |
| Location | Select the location you just created |
| FQDN | `yourdomain.duckdns.org` |
| Communicate Over SSL | **Use SSL Connection** |
| Behind Proxy | **Not Behind Proxy** |
| Total Memory | Your RAM in MB minus ~2000 for system (e.g. `28000` for 30GB) |
| Memory Over-Allocation | `0` |
| Total Disk Space | Your disk in MB (e.g. `200000` for 200GB) |
| Disk Over-Allocation | `0` |
| Daemon Port | `8080` |
| Daemon SFTP Port | `2022` |

Click **Create Node**.

### 6.3 — Add Allocations

Go to the **Allocation** tab on your new node and add:

| Field | Value |
|-------|-------|
| IP Address | Your server's public IP |
| Ports | `25565-25575, 8080, 2022, 3000-3010` |

Click **Submit**.

### 6.4 — Auto-Configure Wings

Go to the **Configuration** tab on your node and click **Generate Token**. Copy the command shown and run it on your server:

```bash
cd /etc/pterodactyl && wings configure --panel-url https://yourdomain.duckdns.org \
  --token YOUR_TOKEN_HERE --node YOUR_NODE_ID
```

> ⚠️ **Never share your token publicly.** Treat it like a password.

---

## 🟣 Phase 7 — Connect Wings to Panel

### 7.1 — Enable Wings SSL

Since the panel runs on HTTPS, Wings **must also run on HTTPS**. Edit the Wings config:

```bash
nano /etc/pterodactyl/config.yml
```

Find the `api:` section and make it look like this:

```yaml
api:
  host: 0.0.0.0
  port: 8080
  ssl:
    enabled: true
    cert: /etc/letsencrypt/live/yourdomain.duckdns.org/fullchain.pem
    key: /etc/letsencrypt/live/yourdomain.duckdns.org/privkey.pem
```

Also make sure the `remote:` line uses HTTPS:

```yaml
remote: https://yourdomain.duckdns.org
```

### 7.2 — Create Wings System Service

```bash
nano /etc/systemd/system/wings.service
```

Paste:

```ini
[Unit]
Description=Pterodactyl Wings Daemon
After=docker.service
Requires=docker.service
PartOf=docker.service

[Service]
User=root
WorkingDirectory=/etc/pterodactyl
LimitNOFILE=4096
PIDFile=/var/run/wings/daemon.pid
ExecStart=/usr/local/bin/wings
Restart=on-failure
StartLimitInterval=180
StartLimitBurst=30
RestartSec=5s

[Install]
WantedBy=multi-user.target
```

```bash
systemctl enable --now wings
```

### 7.3 — Fix Same-Server Routing (Critical Step!)

> 🔴 **This step is required when the panel and Wings run on the same server.** When the panel tries to reach Wings via your public domain, it loops externally and times out. This one-liner fixes it by routing that domain locally instead.

```bash
echo "127.0.0.1 yourdomain.duckdns.org" >> /etc/hosts
systemctl restart wings
systemctl restart pteroq
systemctl restart nginx
```

### 7.4 — Verify Everything Works

Go to **Admin → Nodes → Your Node → About**

You should see:

| Field | Expected |
|-------|----------|
| 💚 Heart icon | **Green** |
| Daemon Version | `v1.x.x (Latest)` |
| System Information | `Linux (amd64)` |
| Total CPU Threads | Your CPU count |

If all four show correctly — **you're fully done!** 🎉

---

## 🔧 Troubleshooting

### ❌ "Access denied for user 'pterodactyl'@'localhost'"
You only created the user for `127.0.0.1`. Fix:
```bash
mysql -u root -p
GRANT ALL PRIVILEGES ON panel.* TO 'pterodactyl'@'localhost' IDENTIFIED BY 'YourDBPassword';
FLUSH PRIVILEGES;
EXIT;
```

### ❌ "Connection refused [tcp://127.0.0.1:6379]"
Redis isn't running. Fix:
```bash
apt install -y redis-server
systemctl enable --now redis-server
```

### ❌ Node heart is green but system info keeps spinning
The panel can't reach Wings API. This is the same-server routing issue. Fix:
```bash
echo "127.0.0.1 yourdomain.duckdns.org" >> /etc/hosts
systemctl restart wings pteroq nginx
```

### ❌ "Could not establish a connection to the machine running this server"
Wings is running HTTP but panel expects HTTPS. Make sure:
1. `api.ssl.enabled: true` in `/etc/pterodactyl/config.yml`
2. SSL cert paths are correct
3. `remote:` uses `https://` not `http://`

### ❌ Node heart is red after setup
Check Wings logs:
```bash
journalctl -u wings --no-pager -n 50
```
Also check panel logs:
```bash
tail -n 100 /var/www/pterodactyl/storage/logs/laravel-$(date +%F).log
```

---

## 📌 Quick Reference — All Services

```bash
# Check status of all services
systemctl status wings pteroq nginx php8.3-fpm redis-server mariadb

# Restart everything
systemctl restart wings pteroq nginx

# View Wings logs live
journalctl -u wings -f

# View Panel logs
tail -f /var/www/pterodactyl/storage/logs/laravel-$(date +%F).log
```

---

## ⚡ Credits

```
Guide written from real experience by:

 ██╗      ██████╗ ██████╗ ██████╗     ██████╗ ███████╗██╗   ██╗██╗███╗   ██╗███████╗
 ██║     ██╔═══██╗██╔══██╗██╔══██╗    ██╔══██╗██╔════╝██║   ██║██║████╗  ██║██╔════╝
 ██║     ██║   ██║██████╔╝██║  ██║    ██║  ██║█████╗  ██║   ██║██║██╔██╗ ██║█████╗
 ██║     ██║   ██║██╔══██╗██║  ██║    ██║  ██║██╔══╝  ╚██╗ ██╔╝██║██║╚██╗██║██╔══╝
 ███████╗╚██████╔╝██║  ██║██████╔╝    ██████╔╝███████╗ ╚████╔╝ ██║██║ ╚████║███████╗
 ╚══════╝ ╚═════╝ ╚═╝  ╚═╝╚═════╝     ╚═════╝ ╚══════╝  ╚═══╝  ╚═╝╚═╝  ╚═══╝╚══════╝
```

> Every step in this guide was learned the hard way so you don't have to.
> If this helped you, drop a ⭐ on the repo!

---

*Made with 🔥 by Lord Devine · [github.com/Heis-Devine](https://github.com/Heis-Devine)*
