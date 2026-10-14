# 🛡️ VPS Infrastructure & PaaS Deployment

This repository contains the complete, step-by-step documentation, configuration files, and automation scripts for provisioning, hardening, and maintaining a production-ready Ubuntu 26.04 VPS. 

The primary objective of this infrastructure is to provide a highly secure, automated, and self-hosted Platform as a Service (PaaS) environment using **Coolify** as a wrapper over Docker, backed by robust monitoring, disaster recovery policies, and edge security.

---

## 🏗️ Architecture & Rationale (Why this stack?)

Every component in this stack was selected to balance developer experience (DX), security, and performance:

*   **Ubuntu 26.04 LTS:** Provides a stable, long-term supported foundation with a modern kernel optimized for containerized workloads.
*   **Coolify & Docker:** Instead of managing raw Docker Compose files, Coolify acts as our self-hosted PaaS. It dramatically speeds up deployment cycles, handles automated SSL generation, and provides a Vercel-like experience while keeping data strictly on our own servers.
*   **Traefik:** Acts as an edge router and reverse proxy. It dynamically discovers Docker containers and handles routing without requiring manual configuration restarts.
*   **Cloudflare Edge:** Offloads malicious traffic (WAF), enforces strict SSL, and hides the origin server's IP address, mitigating direct DDoS attacks before they reach our VPS.
*   **Cloudflare R2:** Provides S3-compatible, cost-effective, and highly durable off-site storage for our automated database and configuration backups.

---

## ⚙️ Tech Stack

*   **OS & Core:** Ubuntu Server 26.04 LTS, UFW (Uncomplicated Firewall), Fail2Ban, SSH Key-Based Auth
*   **PaaS & Containers:** Coolify, Docker Engine, Traefik (Reverse Proxy)
*   **Monitoring & Observability:** Uptime Kuma, Dozzle (Log Viewer), ntfy (Push Alerts)
*   **Edge & DNS:** Cloudflare (WAF, DNSSEC, Strict SSL, Proxy)
*   **Backups & Recovery:** Cloudflare R2 (Off-site Object Storage), Cron scheduling

---

## 📂 Repository Structure

```text
vps-linux-hardening-paas/
├── docs/       # Detailed step-by-step documentation and operational guides
├── scripts/    # Bash scripts for automation (e.g., initial setup, backups)
├── config/     # Static configuration files (e.g., sshd_config, jail.local)
└── README.md   # Project overview and indexg
```

## 📖 Documentation (Table of Contents)

Detailed deployment and configuration guides are located in the `/docs` directory. Please follow them in sequence for a new server deployment:

0. [Server Provisioning](docs/00-server-provisioning.md) - Initial server setup, user creation, and fundamental OS configuration.
1. [Security & Hardening](docs/01-security-hardening.md) - Firewall rules (UFW), Fail2Ban implementation, SSH hardening (disabling root login, password auth), and kernel network tweaks
2. [Coolify PaaS Setup](02-coolify-paas-setup.md) - Container engine installation, secure networking, Traefik reverse proxy configuration, and Coolify deployment.
3. [Monitoring & Alerts](docs/03-monitoring-alerts.md) - Real-time container log aggregation (Dozzle), up time monitoring (Uptime Kuma), and critical system push notifications (ntfy).
4. [Backup & Disaster Recovery](docs/04-backup-policy.md) - Automated, encrypted off-site database and configuration backups to Cloudflare R2 using cron jobs.
5. [DNS & Traffic Routing](docs/05-dns-and-traffic-routing.md) - Cloudflare Edge configuration, WAF custom rules, rate limiting, and origin traffic filtering.
6. [Routine Maintenance & Cheatsheet](docs/06-routine-maintenance.md) - Essential commands for troubleshooting, system updates, and resource management.