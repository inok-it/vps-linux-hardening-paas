## 2.1 Installation and Initial Configuration

##### **Objective:**
Install the Coolify PaaS (Platform as a Service) architecture using the official deployment script, and securely initialize the Root Administrator account via a temporary firewall exception.

##### **Action:**
###### **Step 1: Temporarily Open the Setup Port (UFW)** 
Coolify requires port 8000 for the initial browser-based setup before its internal reverse proxy (Traefik) is fully configured.

```bash
sudo ufw allow 8000/tcp
sudo ufw reload
```

###### **Step 2: Execute the Official Installation Script** 
Run the automated installer. This script will automatically install the Docker Engine (if not present), pull the required Coolify container images (API, UI, Database, Redis), and set up the internal Docker networks.

```bash
curl -fsSL https://cdn.coollabs.io/coolify/install.sh | sudo bash
```

###### **Step 3: Initial Browser Setup**
1. Open your local web browser and navigate to: `http://<YOUR_SERVER_IP>:8000`
2. Create the first main account (**Root Admin**) and click **"Create Account"**.
3. On the onboarding screen, click **"Choose Server Type"** and select **"This Machine"** and wait..
4. **⚠️ Expected Error:** Coolify will attempt to validate the local server connection using the default SSH port `22`. Because we blocked port 22 in our firewall and moved SSH to `60022`, this connection will fail and throw an error on the screen.
5. Do not panic. Scroll down below the machine selection window. The UI will reveal connection settings that you can now edit.
6. Change the **SSH Port** to `60022`.
7. Change the **SSH User** to `root`.
8. Click validate/save. Coolify will now successfully connect to itself.


##### **Rationale:**

**The Installation Script:** 
Using the official `curl | bash` script is the supported deployment method. It ensures file permissions and Docker socket mounts are set exactly as intended.

**The Port 22 Error:** 
Coolify manages servers (even the one it is installed on) via SSH. By explicitly documenting the failure on port 22, we prevent future administrators from thinking the installation is broken. We simply override Coolify's default assumption to match our Defense-in-Depth setup.

**Temporary Port Exposure:** 
We only expose port 8000 temporarily to establish the initial credentials. Going forward, Coolify will route its own dashboard traffic securely through Traefik, maintaining our strict `Deny-by-Default` firewall policy.

## 2.2 DNS and Local Server Configuration
##### **Objective:** 
Bind your domain to the Coolify instance so the internal Traefik proxy can automatically provision Let's Encrypt SSL/TLS certificates and route external traffic to your deployed applications.

##### **Action:**
###### **Step 1: DNS Records Configuration (Domain Registrar / Cloudflare)** 
Log into your DNS provider and point your domain to the server's IP address. You need two `A` records:

| Type  | Name    | IPv4 Address  | Description                                                                               |
| ----- | ------- | ------------- | ----------------------------------------------------------------------------------------- |
| **A** | `panel` | `[SERVER_IP]` | Directs traffic to the Coolify Dashboard (e.g., `panel.domain.pl`)                        |
| **A** | `*`     | `[SERVER_IP]` | Wildcard record. Directs all dynamically created subdomains for client apps to the server |

###### **Step 2: Coolify UI - Server Settings** 
1. Navigate to **Servers** ➔ **localhost** (or the name of your server).
2. Set the panel domain (e.g., `[https://panel.domain.pl](https://panel.domain.pl)`).
3. Set the wildcard domain (e.g., `[https://domain.pl](https://domain.pl)`).

###### **Step 3: Coolify UI - General Settings**
1. Navigate to **Settings** ➔ **General**.
2. Set the main **URL** to your dashboard address (e.g., `[https://panel.inogi.pl](https://panel.inogi.pl)`).
3. Change the **Timezone** to `Europe/Warsaw` to ensure deployment logs and CRON jobs run on local time.
4. Save settings. Coolify/Traefik will now attempt to provision an SSL certificate for your panel.

> [!NOTE]
> Wait a moment, then open a new browser tab and navigate to `https://panel.domain.pl`. You should see the login screen securely served over HTTPS)


##### **Rationale:**

**Wildcard DNS (`*`):** 
Coolify acts as a dynamic router. When you deploy a new client site (e.g., `client-shop`), Coolify automatically assigns it a URL like `[https://client-shop.domain.pl](https://client-shop.domain.pl)`. The wildcard DNS record ensures you don't have to manually edit DNS settings for every new deployment.



## 2.3 Panel Hardening & Securing the Perimeter

##### **Objective:** 
Lock down the Coolify dashboard against unauthorized access and telemetry, and finally close the temporary setup port.

##### **Action:**
###### **Step 1: Disable Unnecessary Features**
1. Navigate to **Settings** ➔ **Advanced**.
2. Enable the following security and privacy toggles:
	1. **Disable Registration:** 
		Prevents random internet users from creating accounts on your private PaaS.
	2. **Require 2-step confirmation:** 
		Adds a confirmation prompt for destructive actions (like deleting databases).
	3. **Disable Telemetry:** 
	   Stops sending usage metrics to external servers.
	4. **Disable Sponsorship reminder:** 
		Cleans up the UI.
###### **Step 2: Enable Two-Factor Authentication (2FA)**
1. Navigate to **Profile** ➔ **Two-factor Authentication**.
2. Enable 2FA, scan the QR code with your authenticator app (e.g., Bitwarden, Proton Pass, or Authy), and verify the token.
3. _Crucial:_ Save the backup recovery codes in your password manager!

##### **Rationale:**

**Disable Registration:** 
Coolify is designed for teams, but also allows open registration by default. On a public-facing server, leaving this enabled is a massive security risk, essentially offering free compute resources to crypto-miners or attackers.

## 2.4 Docker Network Isolation & Final UFW Cleanup

##### **Objective:** 
Bind the Coolify UI explicitly to the local loopback interface (`127.0.0.1`) to prevent Docker from bypassing the firewall, and finally remove the temporary public access rule from UFW.

##### **Action:**

###### **Step 1: Verify Environment Variables**
Ensure the base port is correctly defined in Coolify's environment file.

```bash
nano /data/coolify/source/.env
```

Check that the `APP_PORT` is set (add it or modify it if necessary):

```ini
APP_PORT=8000
```

> [!Warning]
> While you have this file open, securely copy its entire contents and save it in your password manager as a secure note (e.g., Bitwarden or Proton Pass). This file contains the master cryptographic keys (`APP_KEY`, `APP_ID`) and internal database passwords. If this server fails, you cannot restore a Coolify backup without matching the exact keys from this file.

###### **Step 2: Create the Docker Compose Override** 
We will create an override file. This is the safest way to modify Coolify's deployment without directly editing the core files, which would be overwritten during an update.

```bash
nano /data/coolify/source/docker-compose.override.yml
```

Paste the following YAML configuration. The `!reset` tag clears the default exposed ports, allowing us to enforce the `127.0.0.1` binding:

```yml
services:
  coolify:
    ports: !reset
      - '127.0.0.1:8000:80'
```

###### **Step 3: Apply Changes and Restart Coolify Containers** 
Navigate to the source directory and forcefully rebuild the stack using all associated compose files so the override takes effect.

```bash
cd /data/coolify/source
docker compose --env-file .env -f docker-compose.yml -f docker-compose.prod.yml -f docker-compose.override.yml up -d
```

###### **Step 4: Final UFW Cleanup** 
Now that the panel is securely bound to localhost and safely hidden behind the internal proxy (Traefik) responding on standard HTTPS ports, we must remove the temporary rule to block direct external access.

```bash
sudo ufw delete allow 8000/tcp
sudo ufw reload
sudo ufw status
```

##### **Rationale:**

**Order of Operations (Firewall):** 
We explicitly delay closing port 8000 until SSL is provisioned. Traefik requires DNS propagation to validate the Let's Encrypt certificate. If we closed port 8000 too early and the SSL challenge failed, we would be locked out of the UI.
 
**The Docker iptables Bypass:** 
By default, when Docker exposes a port, it adds permissive rules to `iptables` that evaluate _before_ UFW's deny rules. If a container is bound to `0.0.0.0` (all interfaces), closing the port in UFW will **not** block public traffic. Binding strictly to `127.0.0.1` is a mandatory architectural security measure.

**`docker-compose.override.yml`:** 
Docker Compose natively supports reading an `.override.yml` file. This allows administrators to customize deployments (like port bindings, resource limits, or volumes) while keeping the main `docker-compose.yml` clean for safe automated updates from the upstream developers.

## 2.5 Redis Memory Management

##### **Objective:** 
Resolve the standard Redis `overcommit_memory` kernel warning. This ensures Redis can safely perform background memory snapshots (RDB saves) without crashing during high-load memory spikes.

##### **Action:**
###### **Step 1: Apply the Setting to the Current Session** 
Tell the running kernel to allow memory overcommit immediately, without requiring a server reboot:

```bash
sudo sysctl vm.overcommit_memory=1
```

###### **Step 2: Make the Setting Persistent** 
To ensure this configuration survives server reboots, we will create a dedicated drop-in file. Using the `99-` prefix guarantees this rule loads last and is not overwritten by default system updates.

```bash
echo 'vm.overcommit_memory = 1' | sudo tee /etc/sysctl.d/99-redis.conf
```

###### **Step 3: Restart the Redis Container**
1. Find the exact ID or name of the running Redis container:

```Bash
docker ps | grep redis
```

2. Restart the container using the identified name or ID:

```Bash
docker restart <container_ID_or_name>
```

3. Verify the restart was successful:

```Bash
docker ps | grep redis
```

##### **Rationale:**

**How Redis Saves Data:** 
Redis operates entirely in RAM. To save data to the disk, it "forks" its own process. The Linux kernel normally checks if there is enough free RAM to completely duplicate the process. If you have a 2GB Redis instance and only 1GB of free RAM left on the server, the kernel will block the save, even though Redis doesn't actually need 2GB of _new_ RAM (it shares the memory pages).

**`overcommit_memory = 1`:** 
This setting tells the Linux kernel to always pretend there is enough memory and allow the fork. This prevents fatal background save failures, which is critical for maintaining database integrity in a PaaS environment.

## 2.6 Docker Cleanup & Panel Updates

##### **Objective:** 
Configure automated cron jobs within the Coolify dashboard to proactively manage disk space by purging unused container data, and keep the PaaS control panel securely up to date without manual intervention.

##### **Action:**

###### **Step 1: Automated Docker Cleanup** 
Over time, frequent application deployments leave behind unused Docker images, old network configurations, and build caches that can exhaust your server's storage.

1. In the Coolify dashboard, navigate to **Servers** ➔ **localhost** (or your primary server name).
2. Go to the **Settings** tab and locate the **Docker Cleanup** section.
3. Ensure the feature is enabled and set the **Frequency (Cron)** to run daily at midnight:

```Plaintext
0 0 * * *
```

###### **Step 2: Automated Coolify Updates**
Keep the PaaS control panel patched with the latest features and security fixes.

1. In the main left sidebar, navigate to **Settings** ➔ **Updates**.
2. Toggle **Automatic updates** to **Enabled**. 
3. Set the **Check Frequency (Cron)** to midnight:

```Plaintext
0 0 * * *
```

4. Set the **Update Frequency (Cron)** to execute one hour later, at 1:00 AM:
```Plaintext
0 1 * * *
```

##### **Rationale:**
**Preventing Disk Exhaustion:** 
Running out of disk space (`No space left on device`) is fatal for database containers (like PostgreSQL or MariaDB) and often leads to unrecoverable data corruption. The nightly Docker cleanup acts as a scheduled `docker system prune`, ensuring old deployment artifacts are safely discarded.

**Zero Downtime Updates:** 
It is important to know that updating Coolify _only_ restarts the control panel containers. It does **not** stop, restart, or affect your clients' deployed websites, applications, or databases. They remain online and securely routed via Traefik during the panel update.

**Staggered Maintenance Windows:** 
By scheduling Coolify updates at 1:00 AM, we intentionally avoid overlapping with the underlying host OS automated security patches (which we previously configured via `unattended-upgrades` to run at 6:00 AM). Staggering these tasks prevents extreme CPU/IO spikes and makes debugging much easier if an alert triggers overnight.

## 2.7 Source Control Integration: GitHub App

##### **Objective:** 
Securely connect Coolify to your GitHub account or organization using a dedicated GitHub App. This enables automated CI/CD pipelines (deploying code when you push to a branch) without relying on fragile Personal Access Tokens (PATs) or static SSH keys.
##### **Action:**

###### **Step 1: Initiate the Integration**
1. In the Coolify dashboard, navigate to **Sources** on the left menu.
2. Click the **+ Add** button and select **GitHub App**.

###### **Step 2: App Configuration**
1. **Name:** Enter a friendly identifier for this connection (e.g., `Inogi GitHub`).
2. **Organization (Optional):** If you are linking a GitHub Organization rather than a personal account, enter the exact GitHub organization name here.
###### **Step 3: GitHub Registration Wizard**

1. Click the **Register Now** button.
2. Coolify will securely redirect you to GitHub. Log in if prompted.
3. You will be asked to install the new Coolify App into your GitHub account/organization.
###### **Step 4: Grant Permissions (Least Privilege)**

During the GitHub installation wizard, you must choose which repositories Coolify can access:
- _Recommended:_ Select **Only select repositories** and manually choose the projects you intend to host.
- _Alternative:_ Select **All repositories** (only recommended if this is a strictly private, single-developer GitHub account).
###### **Step 5: Finalize and Sync**

1. Once authorized, GitHub will automatically redirect you back to the Coolify dashboard.
2. Click **Save** on the Coolify page.
3. The page will reload. Verify that your GitHub repositories are now successfully listed and ready for deployment.

##### **Rationale:**
**Why a GitHub App?** 
Older CI/CD methods relied on generating a Personal Access Token (PAT) or SSH key and pasting it into the server. If that token was compromised, the attacker gained access to your entire GitHub account. A GitHub App restricts access _only_ to the repositories you explicitly select, and its keys are managed securely via OAuth.

**Webhook Automation:** 
The GitHub App automatically configures webhooks for you. When you run `git push` on your local machine, GitHub instantly pings the Coolify App, triggering a zero-downtime rebuild of your container on the server automatically.

**Principle of Least Privilege:** 
By selecting specific repositories instead of "All repositories," you ensure that even if the PaaS environment is fully compromised, attackers cannot read or modify the source code of your other, unrelated projects.
