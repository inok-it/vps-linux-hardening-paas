## 0.1 OS Release Upgrade Pinning (LTS Pinning)
##### **Objective:** 
Prevent accidental operating system upgrades to non-LTS versions.
##### **Action:**
Open the release upgrade configuration:

```Bash
sudo nano /etc/update-manager/release-upgrades
```

Set `Prompt` to `never`:

```ini
Prompt=never
```

##### **Rationale:**

Ubuntu 26.04 LTS ensures stable dependencies for Docker and Coolify. Upgrading to interim releases (e.g., 26.10) introduces bleeding-edge packages that can break container networking. Hardcoding this prevents accidental system breakage via the `do-release-upgrade` command.

## 0.2 System Updates, Essential Packages, and Timezone Setup

##### **Objective:** 
Apply the latest security patches, install core system utilities, and configure the server's timezone to ensure log consistency.

##### **Action:** 
Run the following commands to update the system, install the required packages, set the timezone, and clean up leftover dependencies:

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y curl wget unattended-upgrades fail2ban
sudo timedatectl set-timezone Europe/Warsaw
sudo apt autoremove -y
```

##### **Rationale:**

**Updates & Cleanup (`upgrade`, `autoremove`):** 
Ensures we are starting with a secure, patch-compliant baseline and removes unnecessary bloat.

**Essential Packages (`curl`, `wget`):** 
Required by Docker, Coolify, and various installation scripts.

**Security (`unattended-upgrades`, `fail2ban`):** 
`unattended-upgrades` automates critical kernel and security patches. `fail2ban` provides immediate protection against SSH brute-force attacks.

**Timezone (`Europe/Warsaw`):** 
Synchronizing the host timezone is crucial. Without it, correlating Traefik access logs, Coolify deployment logs, and system alerts (via ntfy) during an incident becomes a debugging nightmare.

## 0.3 Provisioning the Administrative User

##### **Objective:** 
Create a new, dedicated administrative user (`inok`) and grant `sudo` privileges.

##### **Action:** 
Run the following commands:

```bash
sudo adduser --comment "" inok
sudo usermod -aG sudo inok
```

##### **Rationale:** 

Cloud providers typically deploy instances with default usernames (like `root`, `ubuntu`, or `debian`). These predictable names are primary targets for automated brute-force attacks. Creating a custom administrative user immediately reduces the server's attack surface.

## 0.4 SSH Key Migration and Default Account Lockout

##### **Objective:** 
Transfer SSH key access to the new administrative user and permanently neutralize the default `ubuntu` account.

##### **Action:**
First, copy the SSH keys from the default user to your new user and set the strict permissions required by the SSH daemon:

```bash
sudo mkdir -p /home/inok/.ssh
sudo cp /home/ubuntu/.ssh/authorized_keys /home/inok/.ssh/authorized_keys
sudo chown -R inok:inok /home/inok/.ssh
sudo chmod 700 /home/inok/.ssh
sudo chmod 600 /home/inok/.ssh/authorized_keys
```

Next, **open a second terminal window** and verify you can log in as the new user:

```bash
ssh -p 22 -i ~/.ssh/your_ssh_key inok@<your_server_ip>
```

Once logged in successfully via the new terminal, permanently lock and terminate the default `ubuntu` user:

```bash
sudo passwd -l ubuntu
sudo usermod -e 1 ubuntu
sudo usermod -s /usr/sbin/nologin ubuntu
sudo pkill -KILL -u ubuntu
```

> [!NOTE]
> The `pkill` command will forcibly disconnect your original terminal session

##### **Rationale:**

**Key Migration & Permissions:** 
SSH strictly requires `700` permissions on the `.ssh` directory and `600` on the `authorized_keys` file. If these are incorrect, SSH key authentication will silently fail.

**Account Lockout:** 
We don't just delete the `ubuntu` user immediately, as some lingering provider scripts might rely on its UID/GID initially. Instead, we lock the password (`passwd -l`), expire the account (`usermod -e 1`), strip its ability to spawn a shell (`/usr/sbin/nologin`), and terminate active sessions. This effectively creates a dead,

## 0.5 Generating and Transferring SSH Keys (If Not Provisioned)

##### **Objective:** 
Generate a modern, secure SSH key pair—favoring hardware-backed FIDO2 authentication—and authorize it on the server for passwordless login.
##### **Action:** 
Run **one** of the following commands on your local machine depending on your hardware setup, then transfer the public key to the server.
###### **Option A: YubiKey 5 Series (Resident Key - Recommended)** 
Stores the key directly on the hardware token, allowing you to use it across different local machines.

```bash
ssh-keygen -t ed25519-sk -O resident -O application=ssh:vps-main -C "Main key for VPS server"
```

###### **Option B: Standard YubiKey (Non-Resident)** 
Binds the key to the hardware token, but requires the private key stub file to remain on your local machine.

```bash
ssh-keygen -t ed25519-sk -C "Main key for VPS server"
```

###### **Option C: Standard SSH Key (No Hardware Token)**
Standard, highly secure software key generation.

```bash
ssh-keygen -t ed25519 -C "inok@local-computer"
```

###### Once the key is generated, transfer the public key to the server:

```bash
ssh-copy-id -i ~/.ssh/id_ed25519_sk.pub inok@<your_server_ip>
```

> [!NOTE]
> Replace `id_ed25519_sk.pub` with the actual name of your generated public key, e.g., `vps-main.pub`

##### **Rationale:**

**Key Algorithm (`ed25519`):** 
This elliptic curve algorithm is the modern standard for SSH. It is significantly faster, more secure, and generates much shorter keys compared to legacy RSA.

**Hardware Backing (`-sk`):** 
Using `ed25519-sk` binds the SSH session to a physical FIDO2/U2F token (like a YubiKey). Even if your local machine is compromised and the private key stub is stolen, an attacker cannot authenticate without physically touching the YubiKey.

**Resident Keys:** 
Passing `-O resident` stores the private key securely inside the YubiKey's persistent memory. This allows you to walk up to a completely new laptop and load your SSH identity without manually transferring private key files.
## **0.6 Configuring Swap Space (Swapfile)**

##### **Objective:**
Create a 4 GB swap file to prevent the system from terminating critical processes during sudden memory spikes (OOM Killer), secure its permissions, and ensure it mounts automatically on boot.

##### **Action:** 
Run the following commands to allocate, format, secure, and enable the swap file:

```bash
sudo fallocate -l 4G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile
echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab
```

##### **Rationale:**

**OOM Prevention:** 
Self-hosted PaaS environments running Coolify, databases, and container builds can experience sudden, intense RAM spikes. Without swap space, the Linux Out-Of-Memory (OOM) Killer will aggressively terminate high-memory processes (often taking down the database or Traefik proxy) to save the OS. Swap provides an emergency memory buffer, prioritizing system stability and up time over pure performance.

**Security (`chmod 600`):** 
The swap file acts as an extension of physical RAM and can temporarily hold sensitive data, including plain text passwords, session tokens, or TLS keys. Restricting permissions to `600` ensures only the `root` user can read or write to this file, preventing privilege escalation or data leaks.

## 0.7 Automating Security Patches (Unattended Upgrades)

^fde462
##### **Objective:** 
Enable the automatic installation of critical security updates to minimize the vulnerability window (time-to-patch) and configure a predictable execution schedule (e.g., 6:00 AM) to avoid disrupting peak traffic.

##### **Action:** 
First, enable the unattended-upgrades service interactively (select "Yes" when prompted):

```bash
sudo dpkg-reconfigure --priority=low unattended-upgrades
```

Next, adjust the update schedule. By default, Ubuntu runs these updates at somewhat randomized times. To bind them to a specific low-traffic window, open the systemd timer editor:

```bash
sudo systemctl edit apt-daily-upgrade.timer
```

Paste the following configuration into the editor:

> [!NOTE]
> The first empty `OnCalendar=` line is required to clear the default system schedule before setting our new one

```ini
[Timer]
OnCalendar=
OnCalendar=*-*-* 06:00:00
RandomizedDelaySec=10m
Persistent=true

### Edits below this comment will be discarded
```

Save the file, reload the systemd daemon to apply the changes, and verify your new timer schedule:

```bash
sudo systemctl daemon-reload
sudo systemctl restart apt-daily-upgrade.timer
systemctl list-timers apt-daily-upgrade.timer
```

##### **Rationale:**

**Time-to-Patch:**
Relying on manual updates often leads to delays, leaving the server exposed to publicly known vulnerabilities (1-day exploits). Automating security patches ensures the host OS remains secure without requiring constant human oversight.

**Predictable Scheduling:**
The default randomized timer might trigger an update and daemon restart during peak business hours. Pinning the execution to a known low-traffic window prevents performance degradation for hosted Coolify applications.

**Randomized Delay (`10m`):** 
Adding a small jitter prevents a "thundering herd" problem—where multiple servers attempt to pull updates from the apt mirrors at the exact same millisecond.