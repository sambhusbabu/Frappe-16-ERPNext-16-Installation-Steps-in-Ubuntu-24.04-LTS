# Frappe 16 / ERPNext 16 Installation on Ubuntu 24.04 LTS

This guide includes both:

- **Development setup** (single server, bench start)
- **Full production setup** (Nginx + Supervisor + Redis + MariaDB)

---

## 1) Prerequisites (Ubuntu 24.04 LTS)

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y git curl wget nano software-properties-common apt-transport-https ca-certificates \
  build-essential libffi-dev libssl-dev zlib1g-dev libbz2-dev libreadline-dev libsqlite3-dev \
  libncursesw5-dev xz-utils tk-dev libxml2-dev libxmlsec1-dev liblzma-dev
```

### Create a dedicated user

```bash
sudo adduser frappe
sudo usermod -aG sudo frappe
su - frappe
```

### Install Python, pip, and virtualenv tools

```bash
sudo apt install -y python3-dev python3-pip python3-venv
```

### Install Node.js (LTS) and Yarn

```bash
curl -fsSL https://deb.nodesource.com/setup_20.x | sudo -E bash -
sudo apt install -y nodejs
sudo npm install -g yarn
```

### Install Redis and MariaDB

```bash
sudo apt install -y redis-server mariadb-server mariadb-client
sudo systemctl enable --now redis-server mariadb
```

### Secure MariaDB and set UTF8 settings for Frappe

```bash
sudo mysql_secure_installation
```

```bash
sudo nano /etc/mysql/mariadb.conf.d/50-server.cnf
```

Add this under `[mysqld]`:

```ini
character-set-client-handshake = FALSE
character-set-server = utf8mb4
collation-server = utf8mb4_unicode_ci
skip-character-set-client-handshake
```

Then restart MariaDB:

```bash
sudo systemctl restart mariadb
```

### Install wkhtmltopdf and cron

```bash
sudo apt install -y xvfb libfontconfig wkhtmltopdf cron
sudo systemctl enable --now cron
```

### Install Bench CLI

```bash
sudo pip3 install frappe-bench
```

---

## 2) Development Setup (Frappe 16 + ERPNext 16)

### Initialize bench for Frappe v16

```bash
bench init --frappe-branch version-16 frappe-bench --python python3
cd frappe-bench
```

### Create a new site

```bash
bench new-site dev.localhost
```

### Get ERPNext v16 app and install it

```bash
bench get-app --branch version-16 erpnext
bench --site dev.localhost install-app erpnext
```

### Start development server

```bash
bench start
```

Open in browser: `http://localhost:8000`

> Optional hosts entry for local DNS:
> ```bash
> echo "127.0.0.1 dev.localhost" | sudo tee -a /etc/hosts
> ```

---

## 3) Full Production Setup (Single Server)

Production uses:
- Gunicorn (managed by Supervisor)
- Nginx reverse proxy
- Redis services
- MariaDB

### Install production dependencies

```bash
sudo apt install -y nginx supervisor
sudo systemctl enable --now nginx supervisor
```

### From your bench directory, prepare production configs

```bash
cd ~/frappe-bench
sudo bench setup production frappe
```

This command configures:
- Supervisor programs for workers/web
- Nginx site config
- Systemd/Supervisor integration for services

### Enable HTTPS with Certbot (recommended)

```bash
sudo apt install -y certbot python3-certbot-nginx
sudo certbot --nginx
```

### Firewall rules (if UFW enabled)

```bash
sudo ufw allow OpenSSH
sudo ufw allow 'Nginx Full'
sudo ufw enable
```

### Production service checks

```bash
sudo supervisorctl status
sudo systemctl status nginx
bench --site dev.localhost doctor
```

### Useful production commands

```bash
cd ~/frappe-bench
bench --site dev.localhost migrate
bench --site dev.localhost clear-cache
bench restart
```

---

## 4) Notes

- Use strong MariaDB and Administrator passwords.
- Always run `bench update --reset` carefully on production, preferably during maintenance windows.
- Keep regular backups:

```bash
bench --site dev.localhost backup
```
