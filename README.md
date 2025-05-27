# portfolio-public

**Public documentation & demo** for my WordPress portfolio project.  
_Code, DB dumps & secrets live in a separate private repo: Example:`wp-portfolio-repo`._

---

## Table of Contents

1. [Project Overview](#project-overview)  
2. [Prerequisites](#prerequisites)  
3. [Phase 1: Local Backup & Private Repo](#phase-1-local-backup--private-repo)  
4. [Phase 2: Prep CentOS VM (Local)](#phase-2-prep-centos-vm-local)  
   1. [2.1 System Update & LAMP](#21-system-update--lamp)  
   2. [2.2 MySQL Database & Import](#22-mysql-database--import)  
   3. [2.3 WP Config & Permissions](#23-wp-config--permissions)  
   4. [2.4 Fix WP Login Loop (reset admin password)](#24-fix-wp-login-loop-reset-admin-password)  
5. [Phase 3: Deploy to Azure CentOS VM](#phase-3-deploy-to-azure-centos-vm)  
   1. [3.1 Azure Networking & NSG](#31-azure-networking--nsg)  
   2. [3.2 DNS & Domain Setup](#32-dns--domain-setup)  
   3. [3.3 Clone & Sync Private Repo](#33-clone--sync-private-repo)  
   4. [3.4 Apache VirtualHost](#34-apache-virtualhost)  
6. [Phase 4: HTTPS & Security Hardening](#phase-4-https--security-hardening)  
   1. [4.1 Install & Run Certbot](#41-install--run-certbot)  
   2. [4.2 VirtualHost & Apache Config (80→443)](#42-virtualhost--apache-config-80443)  
   3. [4.3 Configure Firewall & NSG Rules](#43-configure-firewall--nsg-rules)  
   4. [4.4 Enable HSTS & Security Headers](#44-enable-hsts--security-headers)  
   5. [4.5 Install & Tune ModSecurity CRS](#45-install--tune-modsecurity-crs)  
7. [Phase 5: CI/CD Plan (GitHub Actions)](#phase-5-cicd-plan-github-actions)  
8. [Next Steps & Improvements](#next-steps--improvements)  
9. [References](#references)  

---

## Project Overview

This repo documents how to take a WordPress site from local backup → CentOS VM dev → hardened, live Azure hosting → (soon) CI/CD:

- **Phase 1**: Full local backup into a private GitHub repo  
- **Phase 2**: Prep a CentOS 8 VM (LAMP, import, fix login loops)  
- **Phase 3**: Deploy that VM to Azure, configure NSG, DNS, VirtualHost  
- **Phase 4**: Secure with Let’s Encrypt SSL, HTTP→HTTPS redirect, HSTS, Azure NSG, ModSecurity CRS  
- **Phase 5**: (Coming soon) Automate deploy via GitHub Actions  

---

## Prerequisites

- Local machine with Git, WP-CLI, mysqldump  
- CentOS 8 VM (tested on Azure + local)  
- Azure subscription (compute, public IP, NSG)  
- Registered domain (e.g. `yourdomain.com`)  
- DNS access (to create A/CNAME records)  

---

## Phase 1: Local Backup & Private Repo

1. **Create a private GitHub repo** named example: `wp-portfolio-repo`.  
2. On your local WordPress project directory:
   ```bash
   cd ~/Projects/myportfolio
   git init
   git remote add origin git@github.com:you/wp-portfolio-repo.git
   git add .
   git commit -m "Full site backup"
   git push -u origin main
   ```

3. Export database:
   ```bash
   mysqldump -u root -p myportfolio_db > myportfolio_dump.sql
   ```

4. Optionally: add myportfolio_dump.sql to .gitignore if you only want code in Git.

---

## Phase 2: Prep CentOS VM (Local)

###2.1 System Update & LAMP
   ```bash
   # Update & add EPEL if desired
   sudo dnf update -y
   # Install Apache, MariaDB, PHP
   sudo dnf install -y httpd mariadb-server php php-mysqlnd php-fpm php-opcache php-gd php-mbstring
   # Start & enable
   sudo systemctl enable --now httpd mariadb
   ```

###2.2 MySQL Database & Import
   ```bash
   # Secure MariaDB (follow prompts)
   sudo mysql_secure_installation

   # Create DB & user
   cat <<SQL | sudo mysql -u root -p
   CREATE DATABASE portfolio;
   CREATE USER 'wpuser'@'localhost' IDENTIFIED BY 'StrongPass!';
   GRANT ALL ON portfolio.* TO 'wpuser'@'localhost';
   FLUSH PRIVILEGES;
   SQL

   # Transfer & import
   scp myportfolio_dump.sql centos@LOCAL_VM_IP:~
   ssh centos@LOCAL_VM_IP "mysql -u wpuser -p portfolio < ~/myportfolio_dump.sql"
   ```

###2.3 WP Config & Permissions
   ```bash
   # Move your code into Apache docroot
   sudo mv ~/Projects/myportfolio /var/www/html/portfolio
   sudo chown -R apache:apache /var/www/html/portfolio
   sudo chmod -R 755 /var/www/html/portfolio

   # Edit /var/www/html/portfolio/wp-config.php:
   #   DB_NAME → portfolio
   #   DB_USER → wpuser
   #   DB_PASSWORD → StrongPass!
   #   DB_HOST → localhost
   ```

###2.4 Fix WP Login Loop (Reset Admin Password)
If you get stuck in a redirect loop logging into /wp-admin:
   ```bash
   sudo dnf install -y wp-cli
   wp user update admin --user_pass="NewPass123!" --path=/var/www/html/portfolio
   ```

---

## Phase 3: Deploy to Azure CentOS VM

###3.1 Azure Networking & NSG

  1. Provision a CentOS 8 VM in Azure (size, region, SSH key).

  2. Create a Network Security Group (NSG) with inbound rules:

    - SSH (22) — restricted to your IP
    
    - HTTP (80) — Any

    - HTTPS (443) — Any

  3. Associate NSG with the VM’s NIC.

###3.2 DNS & Domain Setup
  - At your registrar (example: Porkbun), point:

    - A @ → VM_PUBLIC_IP

    - CNAME www → @

###3.3 Clone & Sync Private Repo
   ```bash
   # On Azure VM:
   sudo dnf install -y git
   cd /var/www/html
   git clone git@github.com:you/wp-portfolio-repo.git portfolio
   cd portfolio
   # If you need to re-import DB, repeat mysqldump/import as above
   ```

###3.4 Apache VirtualHost
Create /etc/httpd/conf.d/portfolio.conf:
   ```apache
   <VirtualHost *:80>
     ServerName yourdomain.com
     ServerAlias www.yourdomain.com
     DocumentRoot /var/www/html/portfolio
     DirectoryIndex index.php
     <Directory /var/www/html/portfolio>
       AllowOverride All
       Require all granted
     </Directory>
     ErrorLog /var/log/httpd/portfolio-error.log
     CustomLog /var/log/httpd/portfolio-access.log combined
   </VirtualHost>
   ```
   ```bash
   sudo apachectl configtest && sudo systemctl restart httpd
   ```

---


## Phase 4: HTTPS & Security Hardening

###4.1 Install & Run Certbot
   ```bash
   sudo dnf install -y certbot python3-certbot-apache
   sudo certbot --apache \
     --agree-tos \
     --redirect \
     -m you@example.com \
     -d yourdomain.com,www.yourdomain.com
   ```

###4.2 VirtualHost & Apache Config (80 → 443)
Edit /etc/httpd/conf.d/portfolio-ssl.conf (or append below your <VirtualHost *:80> block)
   ```bash
    <IfModule mod_ssl.c>
     <VirtualHost *:443>
       ServerName yourdomain.com
       ServerAlias www.yourdomain.com

       DocumentRoot /var/www/html/portfolio
       DirectoryIndex index.php

       <Directory /var/www/html/portfolio>
         AllowOverride All
         Require all granted
       </Directory>

       SSLEngine on
       SSLCertificateFile /etc/letsencrypt/live/yourdomain.com/fullchain.pem
       SSLCertificateKeyFile  /etc/letsencrypt/live/yourdomain.com/privkey.pem

       # Security headers
       Header always set Strict-Transport-Security "max-age=63072000; includeSubDomains; preload"
       Header always set X-Content-Type-Options "nosniff"
       Header always set X-Frame-Options "SAMEORIGIN"
       Header always set Referrer-Policy "strict-origin-when-cross-origin"

       ErrorLog  /var/log/httpd/portfolio-ssl-error.log
       CustomLog /var/log/httpd/portfolio-ssl-access.log combined
     </VirtualHost>
   </IfModule>
   ```
   ```bash
   sudo apachectl configtest
   sudo systemctl restart httpd
   ```

###4.3 Configure Firewall & NSG Rules
- Confirm only ports 22, 80, 443 allowed inbound in your Azure NSG.

- On the VM you can also run:

   ```bash
   sudo firewall-cmd --permanent --add-service=http
   sudo firewall-cmd --permanent --add-service=https
   sudo firewall-cmd --reload
   ```

###4.4 Enable HSTS & Security Headers
Already included above in your SSL vhost:
   ```bash
   Header always set Strict-Transport-Security "max-age=63072000; includeSubDomains; preload"
   Header always set X-Content-Type-Options "nosniff"
   Header always set X-Frame-Options "SAMEORIGIN"
   Header always set Referrer-Policy "strict-origin-when-cross-origin"
   ```

###4.5 Install & Tune ModSecurity CRS
   ```bash
   sudo dnf install -y mod_security mod_security_crs
   sudo mkdir -p /etc/httpd/modsecurity.d/{activated_rules,local_rules}
   sudo ln -s /usr/share/mod_security_crs/base_rules/*.conf \
            /etc/httpd/modsecurity.d/activated_rules/

   # Load it by editing /etc/httpd/conf.d/mod_security.conf:
   cat <<EOF | sudo tee /etc/httpd/conf.d/mod_security.conf
   <IfModule security2_module>
     SecRuleEngine On
     SecRequestBodyAccess On
     IncludeOptional /etc/httpd/modsecurity.d/activated_rules/*.conf
     IncludeOptional /etc/httpd/modsecurity.d/local_rules/*.conf
   </IfModule>
   EOF

   # Add a local test rule:
   cat <<EOF | sudo tee /etc/httpd/modsecurity.d/local_rules/000-local-block-admin.conf
   SecRule REQUEST_URI "@streq /wp-admin/admin-post.php" \
     "id:9001001,phase:1,deny,status:403,log,msg:'Block admin-post.php URI'"
   EOF

   sudo apachectl configtest
   sudo systemctl restart httpd

   # Verify:
   curl -I "https://yourdomain.com/wp-admin/admin-post.php" \
     -H "User-Agent: () { :; }; echo; echo; /bin/bash -i >& /dev/tcp/127.0.0.1/80 0>&1"
   # → should return HTTP/1.1 403 Forbidden
   ```

---

## Phase 5: CI/CD Plan (GitHub Actions)

🚧 Work in progress — under testing!
In the future, we’ll maintain:

- Private repo (wordpress-portfolio-full) for code & DB dumps

- Public repo (portfolio-public) for docs & GitHub workflows

When ready, on every push to main in the private repo we will:

 1. Run PHP linting & WP-CLI health checks

 2. SSH-deploy updated files to the Azure VM

 3. Optionally run wp search-replace for URL changes

Secrets like AZURE_SSH_KEY and VM_IP will be stored as GitHub repo secrets.
A sample .github/workflows/deploy.yml will appear here soon!

---

## Next Steps & Improvements
- Automate daily backups (DB + files) to Azure Storage or S3

- Add uptime/SSL expiry monitoring

- Harden PHP (disable unused functions), further lock down SSH

- Migrate to containers (Docker + Kubernetes)

## References
- Let’s Encrypt on CentOS 8 (DigitalOcean)

- OWASP ModSecurity CRS Quickstart

- Azure VM + NSG Docs

