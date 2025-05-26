portfolio-public

Step-by-step guide to migrate, configure, secure, and deploy a WordPress portfolio site—local development to live Azure VM—with CI/CD.

Table of Contents

Overview

Prerequisites

Phase 1: Local Backup & Private Repo

1.1. Configure Git & Push Full Backup

Phase 2: Prep Rocky Linux VM

2.1. System Update & LAMP Install

2.2. Database Setup & Import

2.3. WordPress Configuration & Permissions

2.4. Reset Admin Password if Login Loop

Phase 3: Cloud Deployment on Azure

3.1. Provision VM & Network

3.2. Clone Private Repo & Import Dump

3.3. VirtualHost & DNS

Phase 4: Security & HTTPS

4.1. Domain Registration & DNS Setup

4.2. Let's Encrypt SSL with Certbot

4.3. Firewall & NSG Rules

4.4. ModSecurity & Hardening

Phase 5: CI/CD with GitHub Actions

Next Steps

References

Overview

This repository documents the full lifecycle of building a WordPress portfolio site:

Local development on XAMPP (Windows)

Migration to Rocky Linux VM

Private backup on GitHub

Cloud deployment on Azure

Security hardening (HTTPS, firewall, ModSecurity)

CI/CD automation via GitHub Actions

By following this guide, you can reproduce the setup, deploy your own WordPress site, and demonstrate best practices.

Prerequisites

Basic Linux (Rocky/CentOS) and Apache skills

Azure subscription (able to create VMs and DNS records)

GitHub account

A domain name (registered separately)
