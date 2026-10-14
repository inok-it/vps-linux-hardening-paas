
## 4.0 - Backup & Disaster Recovery: Safeguarding the Infrastructure

This runbook outlines the multi-layered backup strategy required to ensure zero data loss and rapid recovery in the event of a catastrophic failure (e.g., hardware failure, datacenter fire, or severe ransomware attack).

## 4.1 Host-Level Backups (Automated VM Snapshots)

##### **Objective:** 
Guarantee the ability to restore the entire server to a working state (including the OS, Docker configurations, installed binaries, and data volumes) with a single click at the hosting provider level.
##### **Action:**
###### **Step 1: Configure Automated Snapshots in the Hosting Panel**
Instead of relying on manual backups, configure automated, scheduled snapshots directly within your cloud provider's dashboard (e.g., OVHcloud, Hetzner, AWS, or DigitalOcean).

1. Log into your hosting provider's control panel.
2. Navigate to your specific VPS or Compute Instance.
3. Locate the **Backups** or **Snapshots** section.
4. Enable **Automated Daily Backups**.
5. Set the backup window to a low-traffic period (e.g., between 02:00 AM and 04:00 AM).
6. Configure the retention policy strictly to **7 or 14 days**.

###### **Step 2: Documentation & Provider Guidelines**
Because UI layouts change frequently, always refer to your specific provider's official documentation for the exact click-path:
- _OVHcloud:_ [Automated Automated Backup on VPS](https://www.google.com/search?q=https://docs.ovh.com/)
- _Hetzner:_ [Using Snapshots and Backups](https://docs.hetzner.com/)

##### **Rationale:**
**Block-Level Recovery:** Snapshots capture the exact state of the hard drive at a block level. If an Ubuntu OS upgrade completely bricks the server, restoring a snapshot is vastly faster than re-provisioning a new server and manually copying data back.

**7 to 14-Day Rotation:** Why not keep them forever? Cost and relevance. A 7-to-14-day rotation strikes the perfect balance. It keeps storage costs minimal while providing a large enough window to realize something is broken (e.g., if a database silently corrupted itself on Friday, you can still restore the snapshot from Thursday on Monday morning).

**The "Not Offsite" Caveat:** Snapshots are critical, but they are stored in the _same_ datacenter as your server. If the datacenter burns down, the snapshots burn with it. This is why snapshots are only Layer 1 of a true Disaster Recovery plan.

## 4.2 Offsite Storage: Cloudflare R2 Integration (S3 Compatible)

##### **Objective:** 
Configure a highly durable, offsite object storage vault using Cloudflare R2. This layer ensures that even if our primary OVH datacenter is completely destroyed, our databases and application states remain safe, encrypted, and instantly accessible.

##### **Action:**
###### Step 1: Create the Cloudflare R2 Bucket
1. Log into your Cloudflare dashboard.
2. In the left sidebar, navigate to **Storage & databases** ➔ **R2 Object Storage**.
3. Click the **Create bucket** button.
4. Configure the bucket parameters:
    - **Name:** `inogi-vps-coolify`
    - **Specify Jurisdiction:** `EU` _(Critical for GDPR compliance)_
    - **Storage class:** `Standard`
5. Click **Create bucket**.

###### Step 2: Generate R2 API Credentials
To allow Coolify to write backups autonomously, we must generate a dedicated set of API keys.

1. Return to the main R2 Object Storage overview page.
2. On the right side of the dashboard, click **Manage R2 API Tokens**.
3. Click **Create API token** and set the following permissions:
    - **Token name:** `coolify-inogi-vps`
    - **Permissions:** `Object Read & Write` _(Required to upload and rotate old backups)_
    - **Specify bucket(s):** `Apply to specific buckets only` ➔ Select `inogi-vps-coolify` _(Principle of Least Privilege)_.
    - **TTL:** `Forever`
4. Click **Create API Token**.

> [!Warning]
> The dashboard will display your credentials exactly once. You MUST immediately copy the Access Key ID, Secret Access Key, and the Endpoint URL. If you close this tab without saving the Secret Key, it cannot be recovered, and you will have to generate a new token.



###### Step 3: Connect R2 to Coolify
Now we instruct Coolify's internal backup engine to use this Cloudflare vault.

1. Log into your Coolify dashboard.
2. In the left sidebar, navigate to the **S3 Storages** menu and click **+ Add** (or **Create**).
3. Fill out the configuration form:
    - **Name / Description:** `Cloudflare R2 Backup Vault`
    - **Endpoint:** Paste your Cloudflare Endpoint URL, but without the `https://` prefix (e.g., `<account_id>.r2.cloudflarestorage.com`).
    - **Bucket:** `inogi-vps-coolify`
    - **Region:** `auto`
    - **Access Key:** Paste the saved Access Key ID.
    - **Secret Key:** Paste the saved Secret Access Key.
4. Click **Save**.
5. Click the **Validate connection** button. A green success message should appear, confirming that Coolify can successfully write to and read from the Cloudflare R2 bucket.

##### **Rationale:**
**Why Cloudflare R2 over AWS S3?** Cloudflare R2 is 100% S3-compatible but charges **zero egress fees** (no cost for downloading your data). If you ever need to restore a massive database during an emergency, AWS S3 can generate surprise billing spikes for the bandwidth used. R2 ensures recovery costs are predictable (free).

**GDPR / RODO Compliance:** By explicitly setting the jurisdiction to `EU`, Cloudflare guarantees the data physically resides within the European Union. This fulfills strict legal requirements when hosting databases that contain European citizens' personal information.

## 4.3 Coolify Control Panel Backup (State & Encryption Keys)

##### **Objective:** 
Secure the Coolify control panel's state (which holds the configuration for all your projects, routing rules, and deployed resources). This requires a two-pronged approach: an automated backup of the PostgreSQL database to Cloudflare R2, and a one-time manual backup of the master encryption keys (`.env`).

###### **Action (Part A): Automated Database Backup**
Instruct Coolify to dump its own database nightly and ship it to our Cloudflare vault.

1. In the Coolify dashboard, navigate to **Settings** ➔ **Backup** ➔ **Configure Backup**.
2. **Backup Schedule:**
    - **Frequency (Cron):** `0 0 * * *` _(Midnight execution)_.
    - **Timeout:** `3600` _(Wait up to 1 hour for large dumps)_.
3. **S3 Storage:**
    - Select the **Cloudflare R2 Backup Vault** created in the previous step.
    - Check the box: **Keep local copy** _(Ensures ultra-fast recovery if the server is alive but the database corrupted)_.
4. **Retention Policies (Apply to both Local and S3):**
    - **Backups to keep:** `7`
    - **Days to keep:** `7`
    - **Max size:** `5 GB`
5. Click **Save Changes**.
6. Click the **Backup Now** button to force an immediate backup. Check your Cloudflare R2 dashboard to confirm the file appeared in the bucket.

###### **Action (Part B): The Critical Manual `.env` Backup** 
Coolify's automated backup does NOT include the host-level configuration file. Without this file, your database backups are cryptographically locked and useless.

1. Log into your server via SSH.
2. Read the contents of the hidden Coolify configuration file:

```bash
sudo cat /data/coolify/source/.env
```

3. Copy the entire output from the terminal (pay special attention to `APP_ID` and `APP_KEY`).
4. **Secure Storage:** Open your password manager (e.g., Proton Pass) and create a new **Secure Note** named `Coolify Master ENV`. Paste the contents there. _(Alternative: Save it as a `.txt` file, upload it manually to your `inogi-vps-coolify` Cloudflare R2 bucket, and then PERMANENTLY delete the local `.txt` file from your computer)._

##### **Rationale:**
**The Encryption Trap:** Coolify stores all your database passwords, API tokens, and webhook secrets in its internal database. To protect them, they are heavily encrypted. The master key to decrypt them is the `APP_KEY` residing in `/data/coolify/source/.env`. If you lose the `.env` file, your R2 backups are essentially crypto-shredded.

**Local + S3 Retention:** Keeping 7 days of local backups ensures that if a bad Coolify update breaks the panel, you don't even need to download anything from Cloudflare—you just restore from the local disk instantly.

**Zero-Trace Policy:** Passwords and keys should never rest on unencrypted developer laptops. Uploading directly to a password manager or R2 and immediately wiping the clipboard/temporary files prevents accidental leaks.

## 4.4 Application & Database Backups (Ntfy, Authentik, etc.)

##### **Objective:** 
Extend the automated offsite backup strategy to individual applications and their associated databases (e.g., Ntfy's SQLite storage, Authentik's PostgreSQL database). This guarantees that user configurations, alert histories, and SSO credentials survive a catastrophic failure.

##### Action:

###### **Step 1: Locate the Application Database/Storage**

1. In the Coolify dashboard, navigate to **Projects** ➔ **Tools** (or your specific client environment).
2. Select the application you wish to back up (e.g., **Ntfy**).
3. Scroll down the left menu and select **Storages** (or navigate to the **Persistent Storage** section at the bottom of the main view).
4. Locate the volume containing the critical data (e.g., `ntfy-db`).
5. Click the **Backup** (floppy disk) icon next to this volume.
###### **Step 2: Enable and Schedule the Backup**

1. In the backup configuration window, ensure **Database backup** (if applicable) is selected.
2. Toggle **Enable S3** to `True` to unlock offsite capabilities.
3. **General Settings:**
    - **Frequency (Cron):** `1 0 * * *` _(Executes at 00:01, exactly one minute after the main Coolify backup)_.
    - **S3 Storage:** Select your **Cloudflare R2 Backup Vault**.
    - Check the box: **Keep local copy**.
###### **Step 3: Apply Retention Policies** 
Set strict boundaries so old backups do not silently consume all your server's disk space or cloud quotas.

- **Local Retention:**
    - Backups to keep: `7`
    - Days to keep: `7`
    - Max size: `5 GB`
- **S3 Backups Retention:**
    - Backups to keep: `7`
    - Days to keep: `7`
    - Max size: `5 GB`
###### **Step 4: Save and Verify**

1. Click **Save Changes**.
2. Click the **Backup Now** button.
3. Check your Cloudflare R2 bucket to confirm the new `.tar.gz` or `.sql` archive has safely arrived.

> [!Note]
> Repeat this exact procedure for the Authentik database, Uptime Kuma storage, and any future client applications deployed on the server. Remember to increment the Cron minute for each new app, e.g., `2 0 * * *`, `3 0 * * *`, to prevent I/O bottlenecks).

##### **Rationale:**
**Cron Staggering:** 
By scheduling Ntfy at 00:01, Authentik at 00:02, and so on, we prevent an "I/O Storm." If 15 applications attempt to dump, compress, and upload their databases at exactly `0 0 * * *`, the server's CPU and disk queue will max out, potentially causing downtime for running applications.

**Universal Coverage:** 
Whether the application relies on a robust relational database (like Authentik's PostgreSQL) or simple flat files (like Ntfy's cache), triggering backups at the Coolify Volume/Storage level ensures all critical persistent state is captured and shipped offsite.

## 4.5 Disaster Recovery: Full Instance Restore (OVH)

##### **Objective:** 
Recover from a catastrophic OS-level failure (e.g., a botched kernel update, misconfigured firewall locking you out, or severe system corruption) by rolling back the entire Virtual Machine to the last known-good automated snapshot.
##### **Action:**
1. Log into your OVHcloud Control Panel.
2. Navigate to the **Public Cloud** section and select your project.
3. Go to **Instances** (lub **Compute**) and select your specific server.
4. Navigate to the **Backups** (Kopie zapasowe) tab.
5. Identify the most recent healthy backup (e.g., from 24 hours ago).
6. Click the action menu (`...`) next to the backup and select **Create an instance from this backup** (lub Restore, w zależności od dostępnego trybu).
7. _Warning:_ Depending on your choice, this will either overwrite your current disk or spin up a new instance with your cloned data. Ensure DNS is updated if a new IP is assigned.

##### **Reference:**
[OVHcloud: Using instance backups to create or restore an instance](https://docs.ovhcloud.com/en/guides/public-cloud/compute/create-restore-a-virtual-server-with-a-backup)


## 4.6 Disaster Recovery: Coolify Control Panel Restore (Bare Metal)

##### **Objective:** 
Rebuild the entire Coolify orchestration environment from scratch. Use this procedure _only_ if the OVH datacenter is completely destroyed, or if you are migrating the entire infrastructure to a brand new server from a different provider. Because this process is highly complex and subject to change with new releases, you must rely directly on the official documentation.
##### **Action:**
###### Step 1: **Prepare the New Server**
Provision a new server (Ubuntu 26.04 LTS) and perform the basic hardening steps from [[01-security-hardening | 01-security-hardening.md]].

###### **Step 2: Locate the Master Key** 
Ensure you have access to your `Coolify Master ENV` (saved securely in your password manager or in your Cloudflare bucket).

> [!Important]
> This file contains the APP_KEY. Without this exact .env file, your automated database backups stored in Cloudflare R2 are irreversibly encrypted and completely useless.

###### **Step 3: Execute Official Recovery Procedure** 
Do not attempt a manual installation. Follow the exact step-by-step restoration guide provided by the Coolify team to reinstall the platform, inject the `.env` file, and restore the database from your offsite backup.
##### **Reference:**
[Coolify Docs: Backup & Restore Coolify](https://coolify.io/docs/knowledge-base/how-to/backup-restore-coolify)

## 4.7 Granular Recovery: Application Database Restore

##### **Objective:** 
Roll back a specific application's state (e.g. a client's WordPress site, or Ntfy configurations) after accidental data deletion or a botched software update, without affecting any other services on the server.
##### **Action:**
1. In the Coolify dashboard, navigate to the specific project (e.g., **Projects** ➔ **Tools**).
2. Locate the database attached to the application you wish to restore (e.g., the PostgreSQL database linked to Authentik or NTFY).
3. Click on the database resource to open its settings.
4. Navigate to the **Backups** tab on the left menu.
5. Locate the **Import backup** (or **Restore**) section.
6. A list of available restore points will appear (comprising both Local copies and Cloudflare R2 offsite copies). Select the desired timestamp.
7. Confirm the restoration.
##### **Rationale:**
**The Three-Tiered Recovery:** 
Now we have procedures for three completely different disaster scenarios:
1. _The server OS died, but the hardware is fine_ ➔ OVH Snapshot (Fastest).
2. _The datacenter burned down / Hardware died_ ➔ Bare Metal Coolify Restore + S3 DB (Most robust).
3. _Someone deleted an important table in a single app_ ➔ Granular App Restore (Least disruptive).