
## 6.1 SSH Operations

This section contains daily operational commands for managing server access. All commands reflect the strict security policies enforced during the initial server provisioning.
### 6.1.1 Standard Server Access

##### **Context:** 
Regular daily login from your primary workstation. 

##### **Reference:** 
[[01-security-hardening#^4814f5 | SSH Port Modification]] 
We strictly use custom port `60022` and cryptographic keys. Password authentication is disabled.

```bash
ssh -p 60022 -i ~/.ssh/<your_key> inok@<server_ip>
```

### 6.1.2 Hardware Security Keys (YubiKey / FIDO2) Lifecycle

##### **Context:** 
Managing physical security keys. FIDO2 `ed25519-sk` keys generate cryptographic proofs directly on the hardware token.

##### A. Generating a New Hardware Key (Resident Key)

Use this when setting up a new YubiKey. The `-O resident` flag ensures the private key stub is stored securely on the hardware token itself, allowing you to move between computers.

```bash
ssh-keygen -t ed25519-sk -O resident -O application=ssh:vps-main -C 'vps-main'
```


##### B. Pushing the Key to the Server

To authorize a newly generated key on the server:

```bash
ssh-copy-id -p 60022 -i ~/.ssh/id_ed25519_sk_rk.pub inok@<server_ip>
```

> [!NOTE]
> Because our server disables password logins, you must execute this command from a machine that already has an authorized key, or manually append the `.pub` string to the server's `~/.ssh/authorized_keys` file via an existing session.

##### C. Logging In from a New/Secondary Workstation

If you move to a new laptop, you don't need to generate a new key. You can extract the public stub directly from your YubiKey.

###### **Prerequisites (Linux Workstations Only):** 
To interface with the USB hardware token without root privileges, ensure the FIDO2 middleware and UDEV rules are loaded on your local machine (e.g., Arch/Manjaro/CachyOS):

```bash
sudo pacman -Syu libfido2
sudo udevadm control --reload-rules && sudo udevadm trigger
```

###### **Extraction & Login:** 
Plug in your YubiKey and extract the Resident Key (`-K`):

```bash
cd ~/.ssh/
ssh-keygen -K
```

> [!WARNING] 
> **Multiple Keys Note:** 
> The `ssh-keygen -K` command will download **all** resident keys currently stored on your FIDO2 token, not just the one for this server. If you use this token for GitHub or other servers, those key stubs will also be downloaded to your current directory.

Once extracted, identify the correct key file (usually `id_ed25519_sk_rk` or suffixed with the application name) and authenticate:

```bash
ssh -p 60022 -i ~/.ssh/id_ed25519_sk_rk inok@<server_ip>
```

### 6.1.3 Secure File Transfers

##### **Context:** 
Copying local files to the server, or downloading backups to your workstation. 

##### **Reference:**
Note that `scp` requires a **capital `-P`** to specify the custom port and the `-i` flag to specify the SSH key, as password authentication is globally disabled..

###### **Upload a local file to the server:**

```bash
scp -P 60022 -i ~/.ssh/your_key ./local_file.txt inok@<server_ip>:/home/inok/
```

###### **Download a remote file to your local machine:**

```bash
scp -P 60022 -i ~/.ssh/your_key inok@<server_ip>:/home/inok/remote_backup.tar.gz ./local_directory/
```

## 6.2 Operating System Updates & Maintenance

##### **Context:** 
Routine OS maintenance, applying non-critical updates, and auditing the automated security patch system. 

##### **Reference:** 
[[00-server-provisioning#^fde462 | Automating Security Patches]]
Recall that critical security updates are installed automatically every day at 6:00 AM via `unattended-upgrades`.

### 6.2.1 Manual System Updates and Cleanup

To check for and apply standard system updates (feature updates, non-critical bugs) that are not covered by the automatic security patcher:

###### **1. Refresh repositories and list available updates:**

```bash
sudo apt update 
apt list --upgradable
```

###### **2. Apply the updates:**

```bash
sudo apt upgrade
```

###### **3. Clean up orphaned packages:** 
After updating (especially after kernel updates), remove old dependencies to free up disk space:

```bash
sudo apt autoremove
```

### 6.2.2 Audit & Update Logs

If a service suddenly breaks or the server reboots unexpectedly, checking the update logs is the first step in debugging.

###### **1. View manual update history:**
Shows what packages were installed or upgraded manually via `apt`:

```bash
less /var/log/apt/history.log
```

###### **2. Verify Automated Security Patches:**
Check the logs of the `unattended-upgrades` daemon to confirm that the scheduled 6:00 AM security patches are running successfully:

```bash
sudo tail -n 50 /var/log/unattended-upgrades/unattended-upgrades.log
```

### 6.2.3 Fail2Ban (Intrusion Prevention) Operations

##### **Context:** 
Monitoring automated bans and managing blocked IP addresses.

##### **Reference:** 
[[01-security-hardening#^b575ff | Fail2Ban Configuration]] 
Remember that our custom `jail.local` is set to ban IPs for 1 hour after 3 failed attempts, while explicitly whitelisting internal Docker networks to keep Coolify applications running.

###### **1. Check global status and list active jails:

```bash
sudo fail2ban-client status
```

###### **2. Check which specific IPs are currently banned for SSH:

```bash
sudo fail2ban-client status sshd
```

###### **3. Unban a specific IP address:** 
If a valid user (or yourself) gets accidentally locked out due to a misconfigured SSH client:

```bash
sudo fail2ban-client set sshd unbanip <IP_ADDRESS>
```

###### **4. Watch Fail2Ban activity in real-time:** 
Monitor active bans and unbans as they happen:

```bash
sudo tail -f /var/log/fail2ban.log
```


### 6.2.4 SSH Authentication Auditing

##### **Context:** 
Investigating who is logging in, or attempting to log in, to the server.

##### **Reference:** 
[[01-security-hardening#^34045e | SSH Connection Hardening]]
We set `LogLevel VERBOSE` in our SSH config. When you view these logs, you will see not just the login attempt, but the exact cryptographic key fingerprint used.

###### **1. View the last 20 SSH daemon events:**
Shows both successful and failed attempts seamlessly:

```bash
sudo journalctl -u ssh --no-pager -n 20
```

###### **2. Watch authentication logs in real-time (live stream):**
Highly useful when testing a new connection or observing an active brute-force scan:

```bash
sudo journalctl -u ssh -f
```

###### **3. Find the last 10 SUCCESSFUL logins:

```bash
sudo grep "Accepted" /var/log/auth.log | tail -n 10
```

###### **4. Find the last 10 FAILED or rejected attempts:**

```bash
sudo grep -iE "failed|invalid|disconnected" /var/log/auth.log | tail -n 10
```

## 6.3 Docker & Coolify Container Management

##### **Context:**
Daily operations for the underlying container engine. While Coolify manages deployments via its UI, interacting directly with Docker is often required for debugging, reading states, or recovering from server-wide network changes.

##### **Reference:**
[[01-security-hardening#^70640f | Uncomplicated Firewall (UFW)]] 
Remember that Docker bypasses UFW. If you modify or spin up containers manually outside of Coolify, they might bypass your firewall rules.

### **6.3.1 Listing Containers

List currently running containers:

```bash
docker ps
```

List ALL containers (including stopped or crashed ones):

```bash
docker ps -a
```

By default, `docker ps` wraps text and becomes unreadable on standard terminal windows. Use this formatted table to strictly show what matters:

```bash
docker ps --format "table {{.Names}}\t{{.Status}}\t{{.Ports}}"
```

### 6.3.2 Searching for Specific Containers

Coolify often appends unique random strings to deployed application containers (e.g., `dozzle-abc123xyz`). You must find the exact name before restarting it.

Using standard `grep` (Running containers only):

```bash
docker ps | grep dozzle
```

Using standard `grep` (Including stopped containers):

```bash
docker ps -a | grep coolify
```

Native Docker Filtering:

```bash
docker ps -a -f "name=dozzle"
```


### 6.3.3 Restarting Services

Restart a single deployed application:

```bash
docker restart <exact_container_name>
```

###### **Restart the main Coolify Control Panel:**
The Coolify dashboard is a multi-container stack. It is safest to restart it using its native `docker-compose` configuration directory:

```bash
cd /data/coolify/source 
docker compose restart
```

> [!WARNING]
> Executing the command below will restart every single running container on the server simultaneously. This will cause brief downtime for ALL client websites and databases. Use this `nuclear option` only after major host firewall changes or daemon crashes.

Restart ALL running containers:

```bash
docker restart $(docker ps -q)
```