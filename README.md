# Frappe Framework 16 & ERPNext 16 Installation Guide
## Ubuntu 24.04 LTS — Production Setup

This documentation explains how to install Frappe Framework v16 and ERPNext v16 on Ubuntu 24.04 LTS, configure the required dependencies, initialize a Bench environment, create a site, install ERPNext, and configure the server for production use with Nginx and SSL.

---

## 1. Prerequisites

Ensure that your server meets the following software requirements before starting the installation.

| Software | Required version |
|---|---|
| Operating System | Ubuntu 24.04 LTS |
| Python | 3.14 |
| pip | 25.3+ |
| Node.js | 24 |
| Yarn | 1.22+ |
| Redis | 6+ |
| MariaDB | 11.8 |

You should also have:
- SSH access to your Ubuntu server.
- A user account with sudo privileges.
- A domain or subdomain pointing to your server's public IP for production deployment.
- Sufficient RAM and disk space for ERPNext, database operations, and asset compilation.

---

## 2. Update the Ubuntu Server

Before installing any dependencies, update the system package lists to ensure the latest package information is available.

```bash
sudo apt update
```

---

## 3. Create the Frappe System User

For a production environment, it is recommended to run Frappe under a dedicated Linux user instead of the root account.

### 3.1 Create the Frappe user

```bash
sudo adduser frappe
sudo usermod -aG sudo frappe
```

**Explanation:**
- `adduser frappe` creates a dedicated Linux user named `frappe`.
- `usermod -aG sudo frappe` adds the Frappe user to the sudo group, allowing administrative commands when required.

### 3.2 Switch to the Frappe user

```bash
sudo su - frappe
```

All subsequent Bench installation and configuration commands should be executed using the Frappe user unless otherwise specified.

---

## 4. Install System Dependencies

Frappe requires several system libraries, development packages, background services, and utilities to support Python packages, database connectivity, PDF generation, frontend compilation, and production deployment.

### 4.1 Install the required packages

```bash
sudo apt install -y \
    git curl wget build-essential \
    software-properties-common \
    pkg-config \
    libffi-dev libssl-dev \
    libmariadb-dev \
    libmariadb-dev-compat \
    libjpeg-dev zlib1g-dev \
    liblcms2-dev libwebp-dev \
    libtiff5-dev libopenjp2-7-dev \
    libharfbuzz-dev libfribidi-dev \
    libxcb1-dev \
    xfonts-base xfonts-75dpi \
    redis-server \
    nginx \
    supervisor \
    cron \
    xvfb \
    libfontconfig \
    libfreetype6
```

### 4.2 Minimal dependency installation (optional)

If you only need Git, Redis, and MariaDB-related packages, use the following command instead.

```bash
sudo apt install git redis-server libmariadb-dev mariadb-server mariadb-client pkg-config
```

**Note:** The minimal installation does not include all the dependencies needed for the complete production setup. Install the full dependency list if you intend to configure PDF generation and production services.

---

## 5. Configure MariaDB

MariaDB is the database server used to store Frappe and ERPNext data, including users, transactions, configuration, and application records.

### 5.1 Secure the MariaDB installation

If the installation process does not prompt you to configure the MariaDB root account, initialize the security configuration manually.

```bash
sudo mariadb-secure-installation
```

**Important:** Run this command only if the security setup has not already been completed.

### 5.2 Restart MariaDB

```bash
sudo service mysql restart
```

This restarts the database service after configuration.

---

## 6. Install Node.js 24 and Yarn

Node.js is required for frontend development and asset compilation. Yarn is used to manage JavaScript dependencies in the Frappe Bench environment.

**Important:** Install NVM and Node.js as the normal Frappe user, not as root.

### 6.1 Switch to the Frappe user

```bash
sudo su - frappe
```

### 6.2 Install NVM

NVM (Node Version Manager) allows you to install and manage multiple Node.js versions without relying on the system-wide Node installation.

```bash
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.40.3/install.sh | bash
```

### 6.3 Load NVM

```bash
source ~/.bashrc
```

### 6.4 Install Node.js 24

```bash
nvm install 24
```

### 6.5 Install npm

```bash
sudo apt-get install npm -y
```

### 6.6 Install Yarn globally

```bash
sudo npm install -g yarn
```

This installs Yarn globally so it can be used for frontend asset builds.

---

## 7. Install wkhtmltopdf

`wkhtmltopdf` is used by Frappe and ERPNext to generate PDF documents from HTML, including invoices, quotations, purchase orders, and other print formats.

### 7.1 Install the required fonts

```bash
sudo apt-get install xfonts-75dpi
```

### 7.2 Download the appropriate wkhtmltopdf package

Visit the official download page and select the `.deb` package suitable for your server architecture (AMD64 or ARM64).

Reference: https://wkhtmltopdf.org/downloads.html

The following command downloads the AMD64 package:

```bash
wget https://github.com/wkhtmltopdf/packaging/releases/download/0.12.6.1-2/wkhtmltox_0.12.6.1-2.jammy_amd64.deb
```

### 7.3 Install wkhtmltopdf

```bash
sudo dpkg -i wkhtmltox_0.12.6.1-2.jammy_amd64.deb
```

### 7.4 Fix dependency errors if they occur

```bash
sudo apt install -f -y
```

This attempts to resolve missing package dependencies after installation.

---

### Install Chromium for PDF Print Format

Frappe/ERPNext uses Chromium to generate PDF documents from Print Formats. Follow these steps to install Chromium and its required dependencies.

**1. Install Chromium**

You can use either of the following methods.

**Option 1: Install using APT**

```bash
sudo apt-get update
sudo apt-get install -y chromium-browser
```

**Option 2: Install using Bench**

```bash
bench setup-chrome
```

**2. Install Required Dependencies**

If you encounter missing package errors or Chromium fails to generate PDF files, install the following dependencies:

```bash
sudo apt update && sudo apt install -y fonts-liberation libatk-bridge2.0-0 libgbm1 libgtk-3-0 libnspr4 libnss3 libvulkan1 libasound2t64 libxdamage1
```

**3. Restart Bench**

After installing Chromium and its dependencies, restart the Frappe services:

```bash
bench restart
```

**Note:** If PDF generation still fails, verify that Chromium is installed correctly and that all required dependencies are available.

## 8. Install uv and Python

`uv` is a fast Python package and environment manager. In this installation, it is used to install Python and manage the Bench CLI in an isolated environment.

### 8.1 Install uv

```bash
curl -LsSf https://astral.sh/uv/install.sh | sh
```

### 8.2 Refresh the terminal environment

```bash
source ~/.bashrc
```

### 8.3 Install Python 3.14

```bash
uv python install 3.14 --default
```

This installs Python 3.14 and sets it as the default Python version managed by uv.

---

## 9. Install Frappe Bench CLI

Frappe Bench is the command-line tool used to initialize development environments, manage sites and applications, build assets, and configure production deployments.

### 9.1 Install Bench using uv

```bash
uv tool install frappe-bench
```

### 9.2 Refresh the terminal environment

```bash
source ~/.bashrc
```

### 9.3 Verify the Bench installation

```bash
bench --version
```

If the command displays the installed Bench version, the CLI installation was successful.

---

## 10. Initialize the Frappe Bench

A Bench directory contains the Frappe framework, Python virtual environment, application directories, site configurations, and other resources required to manage your ERPNext installation.

### 10.1 Initialize a Bench using Frappe version 16

```bash
bench init --frappe-branch version-16 frappe-bench
```

**Explanation:**
- `--frappe-branch version-16` selects the Frappe Framework v16 branch.
- `frappe-bench` is the name of the Bench directory being created.

### 10.2 Navigate to the Bench directory

```bash
cd frappe-bench
```

### 10.3 Start Bench in development mode (optional)

```bash
bench start
```

This starts the development processes, including the web server and supporting services configured for development.

Use `Ctrl + C` to stop the development server.

**Note:** `bench start` is intended for development. For production, use the production setup described in Section 13.

---

## 11. Create a Site and Install ERPNext

A Frappe site is an individual instance that contains its own database, site configuration, and installed applications.

### 11.1 Create a new site

```bash
bench new-site erp.sambhusbabu.com
```

During site creation, Bench will prompt for the MariaDB root password and the Administrator password for the new ERPNext site.

### 11.2 Download ERPNext version 16

```bash
bench get-app erpnext --branch version-16
```

This downloads the ERPNext application source code corresponding to the version-16 branch.

### 11.3 Install ERPNext on the site

```bash
bench --site erp.sambhusbabu.com install-app erpnext
```

This command is intended to install ERPNext on the specified site. See the command verification notes at the end of this document.

---

## 12. Configure Bench for Production

Production setup configures the services required to run Frappe and ERPNext continuously in a production environment.

### 12.1 Enable the scheduler for all sites

```bash
bench --site all enable-scheduler
```

The scheduler executes scheduled tasks, background jobs, notifications, and other automated operations configured in Frappe.

### 12.2 Enable DNS-based multitenancy

```bash
bench config dns_multitenant on
```

This enables DNS-based site resolution, allowing Bench to identify sites based on the incoming hostname.

### 12.3 Configure production services

```bash
sudo env "PATH=$PATH" bench setup production frappe
```
if error run:
```bash
sudo env "PATH=$PATH:/usr/sbin:/usr/bin" bench setup production frappe
```

This configures the production environment for the `frappe` Linux user.

The production setup generates and configures the necessary service files for components such as Supervisor and Nginx.

---

## 13. Configure SSL for the ERPNext Site

SSL certificates enable HTTPS access to your ERPNext site and encrypt communication between the browser and server.

### 13.1 Install Certbot and the Nginx plugin

```bash
sudo apt install -y certbot python3-certbot-nginx
```

### 13.2 Generate and configure the SSL certificate

```bash
sudo certbot --nginx -d erp.sambhusbabu.com
```

Certbot will request the certificate and configure HTTPS for the specified domain, provided the DNS and server configuration are correct.

### 13.3 Restart Bench

```bash
bench restart
```

This restarts the Bench-managed production processes so that services can operate with the updated configuration.

---

# 14. Troubleshooting Common Installation Issues

## 14.1 Frappe loads without CSS or JavaScript

### Symptoms
- The website displays only the HTML skeleton.
- CSS styles are missing.
- JavaScript files are not loading.
- The login page appears unformatted.

### Possible cause
The web server may not have sufficient permissions to access the generated static assets, or the assets may not have been built correctly.

### Apply the following commands

```bash
sudo chmod 755 /home/frappe
sudo chmod -R o+rx /home/frappe/frappe-bench/sites/assets
bench build
bench restart
```

**Explanation:**
- The permission commands allow the web server to traverse the directory and read the generated assets.
- `bench build` recompiles the frontend assets.
- `bench restart` restarts the Bench-managed services.

**Security note:** Review filesystem permissions carefully in production. Prefer granting access to the required directories and users rather than applying broad permissions unnecessarily.

---

## 14.2 Ansible is missing during production setup

### Symptoms
The production setup process reports that Ansible is not installed.

### Install Ansible

```bash
sudo apt install -y ansible
```

After installation, retry the production setup command.

---

## 14.3 pip installation or upgrade issues

### Symptoms
- pip is missing from the Bench CLI environment.
- pip upgrade commands fail.
- The installation reports package management errors.

### Run the following commands

```bash
sudo /home/frappe/.local/share/uv/tools/frappe-bench/bin/python -m ensurepip --upgrade
sudo /home/frappe/.local/share/uv/tools/frappe-bench/bin/python -m pip install --upgrade pip
/home/frappe/.local/share/uv/tools/frappe-bench/bin/python -m pip --version
```

**Explanation:**
- `ensurepip` initializes pip in the Bench CLI's Python environment.
- The second command upgrades pip in that environment.
- The final command verifies the installed pip version.

---

# 15. Important Verification Notes Before Using This Guide

The following points need verification or correction. They have not been silently changed in the installation commands above.

### 15.1 ERPNext installation command contains a typo

Your original command is:

```bash
bench --site sambhusbabu.com instal-app erpnext
```

The standard Bench command is:

```bash
bench --site sambhusbabu.com install-app erpnext
```

`instal-app` is missing an `l`. Also, your site creation command uses `erp.sambhusbabu.com`, while the installation command references `sambhusbabu.com`. These site names must match the actual site you created.

### 15.2 wkhtmltopdf architecture mismatch

Your notes mention ARM64, but the actual download command uses:

```text
wkhtmltox_0.12.6.1-2.jammy_amd64.deb
```

This is the AMD64 package. If your server is ARM64, you need to verify and use the correct architecture-specific package.

### 15.3 Python 3.14 compatibility

Verify that the exact Frappe v16 branch and all required Python dependencies are compatible with Python 3.14 before proceeding with production deployment. Compatibility can vary as branches and dependencies evolve.

### 15.4 Node.js and npm installation

You install Node.js through NVM and then install npm through APT. This can potentially result in PATH or version conflicts because NVM manages its own Node.js and npm installation.

Verify which Node.js and npm executables are active after installation.

### 15.5 MariaDB version

Your prerequisites specify MariaDB 11.8, but the listed APT commands do not explicitly install or pin that version. Verify the actual installed version before initializing Bench.

### 15.6 Domain consistency

The following commands use two different domains:

- Site creation: `erp.sambhusbabu.com`
- ERPNext installation: `sambhusbabu.com`
- SSL certificate: `erp.sambhusbabu.com`

Use the same site identifier consistently when creating and installing ERPNext. The SSL hostname should also match the production site you intend to access.

### 15.7 Production setup and permissions

The production setup command should be executed from the correct Bench directory with the necessary sudo privileges. If it reports that Bench cannot be found under sudo, verify the PATH passed to the command and the location of the Bench executable.

---

## Conclusion

This guide provides a structured workflow for installing Frappe Framework v16 and ERPNext v16 on Ubuntu 24.04 LTS:

1. Update Ubuntu and create the Frappe system user.
2. Install the system dependencies.
3. Configure MariaDB.
4. Install Node.js, Yarn, wkhtmltopdf, Python, and uv.
5. Install Bench CLI and initialize the Frappe v16 environment.
6. Create the site and install ERPNext v16.
7. Configure production services using Bench, Supervisor, and Nginx.
8. Configure SSL using Certbot.
9. Troubleshoot common asset, permission, and pip issues.

Always verify the branch compatibility and system versions before using this guide for a new production server.
