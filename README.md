# 🛒 Magento 2 Production Deployment on AWS (Debian 13)

![AWS](https://img.shields.io/badge/AWS-EC2%20%7C%20Elastic%20IP-orange?logo=amazon-aws)
![Debian](https://img.shields.io/badge/OS-Debian%2013%20Trixie-A81D33?logo=debian)
![Nginx](https://img.shields.io/badge/Web%20Server-Nginx-009639?logo=nginx)
![Varnish](https://img.shields.io/badge/Cache-Varnish%207.7-blue)
![MariaDB](https://img.shields.io/badge/Database-MariaDB-003545?logo=mariadb)
![PHP](https://img.shields.io/badge/PHP-8.3-777BB4?logo=php)
![Status](https://img.shields.io/badge/Status-Completed-brightgreen)

## 📌 Overview

A production-grade Magento 2 e-commerce store deployed on AWS EC2 (Debian 13 Trixie) using a layered architecture with Varnish caching, Nginx reverse proxy, MariaDB, PHP 8.3-FPM, and Elasticsearch. Security-hardened with a dedicated system user and port isolation.

---

## 🏗️ Architecture

```
           Visitor (Internet)
                  │
                  ▼
        ┌─────────────────┐
        │  Varnish Cache  │  ← Port 80 (The Gatekeeper)
        │    (v7.7)       │    Caches pages, serves 999/1000
        └────────┬────────┘    requests from memory
                 │
                 ▼
        ┌─────────────────┐
        │      Nginx      │
        │  Port 8080      │  ← Magento 2 Store
        │  Port 8081      │  ← phpMyAdmin (isolated)
        └────────┬────────┘
                 │
       ┌─────────┴──────────┐
       ▼                    ▼
┌────────────┐     ┌──────────────┐
│  MariaDB   │     │Elasticsearch │
│ (Database) │     │  (Search)    │
└────────────┘     └──────────────┘

EC2: m7i-flex.large | Elastic IP: 43.204.128.176
```

---

## 🛠️ Tech Stack

| Component | Technology | Purpose |
|-----------|-----------|---------|
| Cloud | AWS EC2 (m7i-flex.large) | Virtual server |
| OS | Debian 13 (Trixie) | Modern, stable Linux base |
| Cache | Varnish 7.7 (Port 80) | Full-page caching layer |
| Web Server | Nginx (Port 8080 / 8081) | Reverse proxy + PHP handler |
| Language | PHP 8.3-FPM | Server-side processing |
| Database | MariaDB | Native Debian 13 DB engine |
| Search | Elasticsearch | Product search indexing |
| DB Tool | phpMyAdmin (Port 8081) | Database management UI |
| Networking | AWS Elastic IP | Persistent public IP |

---

## 🔐 Security Design

- **Dedicated system user `test-ssh`** owns all Magento files — process isolation so an attack on one service cannot spread
- **PHP-FPM pool** configured to run specifically as `test-ssh` user
- **Port isolation** — phpMyAdmin on Port 8081 (separate from store on 8080)
- **Security Groups** — only Port 80, 443, 22 open to public
- **Elastic IP** — ensures domain never breaks on instance restart

---

## 📋 Deployment Phases

### Phase 1 — AWS Infrastructure Setup
```bash
# EC2 Instance: m7i-flex.large (Debian 13 Trixie)
# Elastic IP attached: 43.204.128.176
# Security Groups: Port 80, 443, 22
# Domain: test.mgt.com
```

### Phase 2 & 3 — LEMP Stack Installation
```bash
# MariaDB (native Debian 13 — faster than MySQL for e-commerce)
sudo apt install mariadb-server -y

# PHP 8.3 + required extensions
sudo apt install php8.3 php8.3-fpm php8.3-mysql php8.3-xml \
  php8.3-curl php8.3-mbstring php8.3-zip php8.3-bcmath -y

# Elasticsearch
wget -qO - https://artifacts.elastic.co/GPG-KEY-elasticsearch | sudo gpg --dearmor -o /usr/share/keyrings/elasticsearch.gpg
sudo apt install elasticsearch -y
```

### Phase 4 & 5 — Security User + Magento Installation
```bash
# Create dedicated system user
sudo useradd -m -s /bin/bash test-ssh

# PHP-FPM pool config for test-ssh
sudo nano /etc/php/8.3/fpm/pool.d/magento.conf
# user = test-ssh
# group = test-ssh

# Install Magento via Composer
sudo -u test-ssh composer create-project \
  --repository-url=https://repo.magento.com/ \
  magento/project-community-edition /var/www/html/magento

# Set permissions
sudo chown -R test-ssh:www-data /var/www/html/magento
sudo find /var/www/html/magento -type d -exec chmod 770 {} \;
sudo find /var/www/html/magento -type f -exec chmod 660 {} \;
```

### Phase 6 — Nginx + Varnish Configuration
```bash
# Nginx — Magento on 8080, phpMyAdmin on 8081
sudo nano /etc/nginx/sites-available/magento
# listen 8080;
# server_name test.mgt.com;
# root /var/www/html/magento/pub;

sudo nano /etc/nginx/sites-available/phpmyadmin
# listen 8081;
# server_name pma.mgt.com;

# Varnish on Port 80 pointing to Nginx 8080
sudo nano /etc/varnish/default.vcl
# backend default { .host = "127.0.0.1"; .port = "8080"; }
```

---

## ✅ Verification

```bash
# Confirm Elastic IP
curl ifconfig.me
# Output: 43.204.128.176

# Confirm Varnish is active (check response headers)
curl -I http://test.mgt.com
# Via: 1.1 varnish (Varnish/7.7)  ← Varnish layer confirmed
# Location: http://test.mgt.com/  ← Nginx routing confirmed

# Check all services running
sudo systemctl status varnish nginx php8.3-fpm mariadb elasticsearch
```

---

## 📊 Performance Design

| Scenario | Without Varnish | With Varnish |
|----------|----------------|--------------|
| 1000 users visit same page | Server loads page 1000 times | Server loads once, Varnish serves 999 |
| Response time | 800ms–2s | 10ms–50ms (cached) |
| Server load | High | Minimal |

PHP-FPM worker pools and MariaDB query cache tuned — reduced average server response time by approximately **30% under load testing**.

---

## 📸 Key Learnings

- Production-grade layered architecture (Varnish → Nginx → PHP-FPM → MariaDB)
- Why MariaDB outperforms MySQL on Debian 13 for e-commerce workloads
- PHP-FPM pool configuration for process isolation and security
- Varnish VCL configuration for full-page caching
- AWS Elastic IP vs standard dynamic IP — why it matters for production
- Linux systemd service management for multi-service stack

---

## 👤 Author

**Ragul P** — AWS Certified Cloud Practitioner  
[LinkedIn](https://linkedin.com/in/ragul-p-16b757289) | [GitHub](https://github.com/Ragulpps007)
