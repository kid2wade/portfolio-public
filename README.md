# portfolio-public

Step-by-step guide to migrate, configure, secure, and deploy a WordPress portfolio site—local development to live Azure VM—with CI/CD.

---

## Table of Contents

- [Overview](#overview)  
- [Prerequisites](#prerequisites)  
- [Phase 1: Local Backup & Private Repo](#phase-1-local-backup--private-repo)  
  - [1.1 Configure Git & Push Full Backup](#11-configure-git--push-full-backup)  
- [Phase 2: Prep Rocky Linux VM](#phase-2-prep-rocky-linux-vm)  
  - [2.1 System Update & LAMP Install](#21-system-update--lamp-install)  
  - [2.2 Database Setup & Import](#22-database-setup--import)  
  - [2.3 WordPress Configuration & Permissions](#23-wordpress-configuration--permissions)  
  - [2.4 Reset Admin Password if Login Loop](#24-reset-admin-password-if-login-loop)  
- [Phase 3: Cloud Deployment on Azure](#phase-3-cloud-deployment-on-azure)  
  - [3.1 Provision VM & Network](#31-provision-vm--network)  
  - [3.2 Clone Private Repo & Import Dump](#32-clone-private-repo--import-dump)  
  - [3.3 VirtualHost & DNS](#33-virtualhost--dns)  
- [Phase 4: Security & HTTPS](#phase-4-security--https)  
  - [4.1 Domain Registration & DNS Setup](#41-domain-registration--dns-setup)  
  - [4.2 Let’s Encrypt SSL with Certbot](#42-lets-encrypt-ssl-with-certbot)  
  - [4.3 Firewall & NSG Rules](#43-firewall--nsg-rules)  
  - [4.4 ModSecurity & Hardening](#44-modsecurity--hardening)  
- [Phase 5: CI/CD with GitHub Actions](#phase-5-cicd-with-github-actions)  
- [Next Steps](#next-steps)  
- [References](#references)  

---

## Overview

A complete workflow for:

1. Backing up an existing WordPress portfolio locally and into a **private** GitHub repo  
2. Standing up a Rocky Linux VM, installing LAMP, importing the site  
3. Migrating to Azure VM, configuring DNS & Apache VirtualHosts  
4. Locking down security: HTTPS, firewalls, ModSecurity, HSTS  
5. Preparing for continuous deployment via GitHub Actions  

---

## Prerequisites

- **Local machine** (Windows/macOS) with Git, WP-CLI, mysqldump  
- **Rocky Linux VM** accessible via SSH  
- **Azure subscription** for VM, NSG, public IP  
- **Domain name** (e.g. `example.com`) registered  
- **Basic Azure CLI** or Portal familiarity  

---

## Phase 1: Local Backup & Private Repo

### 1.1 Configure Git & Push Full Backup

1. Create a new **private** GitHub repo, e.g. `wordpress-portfolio-full`.  
2. On your local machine:
   ```bash
   cd ~/Sites/myportfolio
   git init
   git remote add origin git@github.com:you/wordpress-portfolio-full.git
   git add .
   git commit -m "Initial full backup"
   git push -u origin main
   ```
    
