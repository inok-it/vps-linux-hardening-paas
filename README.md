# 🛡️ VPS Infrastructure & PaaS Deployment

This repository contains the complete, step-by-step documentation, configuration files, and automation scripts for provisioning, hardening, and maintaining a production-ready Ubuntu 26.04 VPS. 

The primary objective of this infrastructure is to provide a highly secure, automated, and self-hosted Platform as a Service (PaaS) environment using **Coolify** as a wrapper over Docker, backed by robust monitoring, disaster recovery policies, and edge security.

---

## 🏗️ Architecture & Rationale (Why this stack?)

Every component in this stack was selected to balance developer experience (DX), security, and performance:

* **Ubuntu 26.04 LTS:** Provides a stable, long-term supported foundation with a modern kernel optimized for containerized workloads.
* **Coolify & Docker:** Instead of managing raw Docker Compose files, Coolify acts as our self-hosted PaaS. It dramatically speeds up deployment cycles, handles automated SSL generation, and provides a Vercel-like experience while keeping data strictly on our own servers.
* **Zero-Trust Identity (Authentik):** Internal tools and dashboards are never exposed directly to the internet. They are shielded behind an Identity Provider enforcing hardware-backed MFA (YubiKey/FIDO2) via Traefik Forward-Auth.
* **Traefik:** Acts as an edge router and reverse proxy. It dynamically discovers Docker containers and handles routing without requiring manual configuration restarts.
* **Cloudflare Edge:** Offloads malicious traffic (WAF), blocks AI crawlers, enforces strict SSL, and hides the origin server's IP address, mitigating direct DDoS attacks before they reach our VPS.
* **Cloudflare R2:** Provides S3-compatible, cost-effective, and highly durable off-site storage for our automated database and configuration backups with zero egress fees.

---

## ⚙️ Tech Stack

* **OS & Core Security:** Ubuntu Server 26.04 LTS, UFW (Uncomplicated Firewall), Fail2Ban, SSH Key-Based Auth
- **PaaS & Containers:** Coolify, Docker Engine, Traefik (Reverse Proxy)
- **Identity & Access Management:** Authentik (SSO), YubiKey (FIDO2 / WebAuthn)
- **Monitoring & Observability:** Uptime Kuma (Status), Dozzle (Log Viewer), Diun (Image Updates), Ntfy (Push Alerts)
- **Edge & DNS:** Cloudflare (WAF, DNSSEC, Strict SSL, Bot Management)
- **Backups & Recovery:** Cloudflare R2 (Off-site Object Storage), Cron scheduling
- **Auditing & Integrity:** Lynis, Rkhunter (Rootkit Hunter), Debsums

---

## 📂 Repository Structure

```text
vps-linux-hardening-paas/
├── docs/       # Detailed step-by-step documentation and operational guides
├── scripts/    # Bash scripts for automation (e.g., initial setup, backups)
├── config/     # Static configuration files (e.g., sshd_config, jail.local)
└── README.md   # Project overview and indexg
```

---

### 📖 Documentation (Table of Contents)

Detailed deployment and configuration guides are located in the `/docs` directory. Please follow them in strict sequence for a new server deployment:

0. [Server Provisioning](docs/00-server-provisioning.md) - Initial server setup, user creation, and fundamental OS configuration.
1. [Security & Hardening](docs/01-security-hardening.md) - Firewall rules (UFW), Fail2Ban implementation, SSH hardening (disabling root login, password auth), and kernel network tweaks
2. [Coolify PaaS Setup](02-coolify-paas-setup.md) - Container engine installation, secure networking, Traefik reverse proxy configuration, and Coolify deployment.
3. [Monitoring & Alerts](docs/03-monitoring-alerts.md) - Real-time container log aggregation (Dozzle), uptime monitoring (Uptime Kuma), container update tracking (Diun), and critical system push notifications via a self-hosted Ntfy gateway.
4. [Backup & Disaster Recovery](04-backup-recovery-policy.md) - Multi-tiered recovery strategies, from VM-level snapshots to automated, encrypted off-site database and configuration backups utilizing Cloudflare R2.
5. [Edge Security & WAF](05-cloudflare-waf-security.md) - Cloudflare Edge configuration, WAF custom rules, rate limiting, and origin traffic filtering.
6. [Routine Maintenance & Cheatsheet](docs/06-routine-maintenance.md) - Routine maintenance, cheatsheets, essential commands for troubleshooting, system updates, and resource management.
7. [[07-authentik-sso | Authentik SSO & Zero-Trust]] -  Deployment of the Authentik Identity Provider, enforcement of YubiKey/FIDO2 MFA, and configuration of Traefik Forward-Auth to shield internal monitoring tools.
8. [[08-security-audits| Security Audits & Incident Analysis]]Automated security scoring (Lynis), rootkit detection (Rkhunter), binary integrity verification (Debsums), and live resource profiling.
