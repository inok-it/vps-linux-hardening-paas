# 🛡️ VPS Infrastructure & PaaS Deployment

This repository contains Infrastructure as Code (Ansible playbooks), configuration templates, automation scripts, and step-by-step operational documentation for provisioning, hardening, and maintaining a production-ready **Ubuntu 26.04 LTS** VPS.

The primary objective of this project is to deliver a repeatable, security-hardened, and self-hosted Platform as a Service (PaaS) environment using **Coolify** over Docker, backed by Zero-Trust access control, observability, disaster recovery policies, and edge security.

---

## 🏗️ Architecture & Rationale (Why this stack?)

Every component in this stack was selected to balance developer experience (DX), operational resilience, and security:

* **Infrastructure as Code (Ansible):** Eliminates manual configuration drift and human error. Multi-stage playbooks handle initial bootstrapping, OS provisioning, and system hardening using Jinja2 templates with pre-deployment syntax validation (`sshd -t`) and encrypted secrets via **Ansible Vault**.
* **Ubuntu 26.04 LTS:** Provides a stable, long-term supported foundation with a modern Linux kernel optimized for containerized workloads and native OpenSSH Post-Quantum Cryptography support.
* **Coolify & Docker:** Instead of managing scattered Docker Compose files manually, Coolify acts as a self-hosted PaaS. It accelerates deployments, automates SSL certificate lifecycle, and provides a Vercel-like workflow while keeping data strictly on sovereign infrastructure.
* **Zero-Trust Identity (Authentik):** Internal dashboards and administrative tools are never exposed directly to the public internet. Access is gated behind an Identity Provider enforcing hardware-backed MFA (**YubiKey / FIDO2**) via Traefik Forward-Auth.
* **Traefik:** Acts as an edge router and reverse proxy, dynamically discovering Docker workloads and routing traffic without manual service restarts.
* **Cloudflare Edge:** Offloads malicious traffic (WAF), blocks unwanted crawlers, enforces strict TLS, and masks the origin server's IP address to mitigate direct DDoS attacks.
* **Cloudflare R2:** Provides S3-compatible, durable off-site object storage for automated, encrypted database and configuration backups with zero egress fees.

---

## ⚙️ Tech Stack

* **Automation & IaC:** Ansible (`ansible-core`, Jinja2 Templates, Ansible Vault), `ansible-lint` (`production` profile validated)
* **OS & Core Security:** Ubuntu Server 26.04 LTS, UFW, Fail2Ban, OpenSSH Hardening (`systemd` socket port override, key-only auth, PQC key exchange verified via `ssh-audit`), `sysctl` kernel hardening
* **PaaS & Containers:** Coolify, Docker Engine, Traefik (Reverse Proxy)
* **Identity & Access Management:** Authentik (SSO / Forward-Auth), YubiKey (FIDO2 / WebAuthn)
* **Monitoring & Observability:** Uptime Kuma (Status), Dozzle (Live Container Logs), Diun (Image Update Alerts), Ntfy (Self-Hosted Push Gateway)
* **Edge & DNS:** Cloudflare (WAF, DNSSEC, Strict SSL, Bot Management)
* **Backups & Recovery:** Cloudflare R2 (Off-site Object Storage), Cron & Bash automation
* **Auditing & Integrity:** Lynis, Rkhunter (Rootkit Hunter), Debsums

---

## 📂 Repository Structure

```text
vps-hardening-paas-setup/
├── ansible/                                # Infrastructure as Code (Ansible)
│   ├── ansible.cfg                         # Local Ansible behavior & SSH settings
│   ├── hosts.yml                           # Inventory definition
│   ├── group_vars/
│   │   ├── all.yml                         # Global defaults
│   │   └── servers_group.yml               # Target server group variables
│   ├── host_vars/
│   │   └── prod_vps/
│   │       ├── vars.yml                    # Host-specific parameters
│   │       └── vault.yml                   # Encrypted host secrets (Ansible Vault)
│   ├── templates/
│   │   └── sshd_hardening.conf.j2          # Validated OpenSSH & PQC drop-in template
│   ├── bootstrap.yml                       # Stage 1: Initial user & SSH key setup (Port 22)
│   ├── playbook_system_provisioning.yml    # Stage 2: Base packages & system setup (Port 22)
│   └── playbook_system_hardening.yml       # Stage 3: SSH, socket, UFW & OS hardening (Port 60022)
├── docs/                                   # Detailed step-by-step operational guides
├── scripts/                                # Auxiliary Bash scripts (backups, maintenance)
└── README.md                               # Project overview and documentation index
```

---

## 🚀 Automated Deployment (Ansible Quick Start)

Playbooks are structured sequentially to safely handle the transition from a fresh OS image on standard port `22` to a hardened instance running on a custom SSH socket port.

1. **Validate playbooks against production linting rules:**
   ```bash
   cd ansible
   ansible-lint playbook_system_hardening.yml
   ```

2. **Configure inventory and encrypted secrets:**
   Update `hosts.yml` and `host_vars/<host>/vars.yml` with your target server details, then encrypt sensitive variables using Ansible Vault:
   ```bash
   ansible-vault create host_vars/prod_vps/vault.yml
   ```

3. **Execute playbooks in order:**
   ```bash
   # 1. Bootstrap administrative user and SSH keys (Port 22)
   ansible-playbook bootstrap.yml --ask-vault-pass

   # 2. Provision base packages and system utilities (Port 22)
   ansible-playbook playbook_system_provisioning.yml --ask-vault-pass

   # 3. Apply OS, UFW, systemd socket, and OpenSSH PQC hardening (switches to custom SSH port)
   ansible-playbook playbook_system_hardening.yml --ask-vault-pass
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
7. [Authentik SSO & Zero-Trust](docs/07-authentik-sso.md) -  Deployment of the Authentik Identity Provider, enforcement of YubiKey/FIDO2 MFA, and configuration of Traefik Forward-Auth to shield internal monitoring tools.
8. [Security Audits & Incident Analysis](08-security-audits.md) - Automated security scoring (Lynis), rootkit detection (Rkhunter), binary integrity verification (Debsums), and live resource profiling.
