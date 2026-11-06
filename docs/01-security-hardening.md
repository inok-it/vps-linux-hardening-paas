> [!NOTE] 
> **Understanding File Naming Conventions (`00-` vs `99-`):**
> Throughout this guide, you will notice configuration files prefixed with numbers, such as `00-hardening.conf` or `99-security.conf`. Linux services read drop-in directories (like `/etc/ssh/sshd_config.d/` or `/etc/sysctl.d/`) in strict **lexicographical (numerical) order**.
> 
>  **`00-` (Loads First):** For the SSH daemon, the *first* parsed directive wins. If a setting is defined in a `00-` file, SSH locks it in and ignores any conflicting settings in files loaded later. We use `00-` to guarantee our hardening rules take absolute priority.
>  
>  **`99-` (Loads Last):** For kernel parameters (`sysctl`), the *last* parsed directive overwrites previous ones. We use `99-` to ensure our custom security parameters overwrite any default Ubuntu network configurations (which are typically named things like `10-default.conf`).

## 1.1 Legal Warning Banner

##### **Objective:** 
Display a legal warning message before the login prompt to establish that unauthorized access is strictly prohibited.

##### **Action:** 
Run the following commands to write the warning message to both the local console and remote SSH network login files:

```bash
echo "Authorized uses only. All activity may be monitored and reported." | sudo tee /etc/issue
echo "Authorized uses only. All activity may be monitored and reported." | sudo tee /etc/issue.net
```

##### **Rationale:** 

While a text banner does not technically block automated brute-force attacks, it is a critical security compliance requirement. Displaying a clear, legally binding notification that the system is restricted and monitored removes the legal defense of "accidental" or "unaware" access, providing a foundation for prosecuting malicious actors.

## 1.2 SSH Port Modification
> [!IMPORTANT]
> 
> The SSH port is being changed to **60022**. You must remember this custom port for the upcoming UFW (Firewall) and Fail2Ban configuration steps. Failing to open this port in the firewall before logging out will lock you out of the server.
> 
>   

##### **Objective:** 
Move the SSH daemon to a non-standard port to reduce automated attack noise.

##### **Action:**

###### **Step 1: Reconfigure Systemd SSH Socket**
Recent Ubuntu releases use socket activation for SSH. You must change the port at the systemd level first.

Open the socket override editor:

```Bash
sudo systemctl edit ssh.socket
```

Paste the following lines exactly as shown (the empty `ListenStream=` is required to clear the default port 22 binding) between the commented boundaries:

```ini
[Socket]
ListenStream=
ListenStream=0.0.0.0:60022
ListenStream=[::]:60022

### Edits below this comment will be discarded
```

Save the file and apply the socket changes:

```Bash
sudo systemctl daemon-reload && sudo systemctl restart ssh.socket
```

###### **Step 2: Create the Initial Drop-in Configuration**

> [!NOTE]
> The full SSH hardening configuration with disabling passwords, root logins, and tunneling is located in the `config/sshd_config/00-hardening.conf` file of this repository, which we will apply later).

Create a drop-in file to explicitly set the new port:

```Bash
sudo nano /etc/ssh/sshd_config.d/00-hardening.conf
```

Add the following configuration:

```ini
# /etc/ssh/sshd_config.d/00-hardening.conf
Port 60022
```

###### **Step 3: Comment Out the Default Port**

To prevent conflicts, ensure the main SSH configuration isn't trying to bind to port 22.

Open the main config file:

```bash
sudo nano /etc/ssh/sshd_config
```

Find the `#Port 22` line and ensure it remains commented out (or comment it out if it isn't).

```ini
#Port 22
```

  
###### **Step 4: Syntax Verification and Service Restart**

Always verify the SSH configuration syntax before restarting the daemon to avoid locking yourself out.

```Bash
sudo sshd -t
sudo systemctl restart ssh
sudo systemctl daemon-reload
```

Verify that the server is now actively listening on the new port:

```Bash
sudo ss -tlpn | grep ssh
```

> [!IMPORTANT]
> **Open a NEW terminal window** (do not close your current session yet) and test the connection:

```Bash
ssh -p 60022 -i ~/.ssh/your_key inok@<your_server_ip>
```

##### **Rationale:**
**Port Change (60022):** 
Moving away from port 22 does not stop targeted attacks, but it eliminates 99% of automated botnet "background noise" and script-kiddie brute-force attempts. This keeps our authentication logs clean and reduces CPU overhead from Fail2Ban processing endless failed attempts.

**Socket Activation:** 
Ubuntu uses `systemd` sockets to spawn SSH instances dynamically. Modifying only `sshd_config` without updating the socket causes binding failures.
## 1.3 SSH Connection Hardening

##### **Objective:** 
Apply strict cryptographic authentication, rate-limiting, session timeouts, and disable tunneling to thoroughly secure the SSH daemon.

##### **Action:** 
We will create a central drop-in configuration file to apply all hardening rules.

> [!NOTE]
>  A backup of this exact configuration is available in the repository under `config/ssh/00-hardening.conf` for reference)

Open the hardening configuration file:

```bash
sudo nano /etc/ssh/sshd_config.d/00-hardening.conf
```

Paste the following comprehensive configuration:

```ini
# /etc/ssh/sshd_config.d/00-hardening.conf

# 1. Base Configuration & Legal
Port 60022
Banner /etc/issue.net
LogLevel VERBOSE

# 2. Access Control
AllowGroups sudo
PermitRootLogin no

# 3. Authentication Types (Keys Only)
PasswordAuthentication no
PubkeyAuthentication yes
KbdInteractiveAuthentication no
UsePAM yes

# 4. Connection Rate Limiting (Anti-DoS)
MaxAuthTries 3
MaxSessions 2
LoginGraceTime 30
MaxStartups 10:30:60

# 5. Idle Session Management
ClientAliveInterval 300
ClientAliveCountMax 2
ChannelTimeout session=30m
TCPKeepAlive no

# 6. Tunneling & Forwarding Restrictions
X11Forwarding no
AllowTcpForwarding no
AllowAgentForwarding no
```

Save the file and verify the syntax before restarting the SSH service (ensure you have an active session open to verify you aren't locked out):

```bash
sudo sshd -t
sudo systemctl restart ssh
sudo systemctl daemon-reload
sudo ss -tlpn | grep ssh
```

>[!WARNING] 
>**Crucial Verification Step:** 
>Do **not** close your current terminal session yet. Open a **new** terminal window and verify that you can successfully authenticate using the new port and your SSH key:

```bash
ssh -p 60022 -i ~/.ssh/your_key inok@<your_server_ip>
```

If the login fails, return to your original (still active) terminal to check the logs or revert the changes.

```bash
sudo tail -f /var/log/auth.log
```

> [!NOTE] 
> **VS Code Remote SSH Compatibility:** 
> If you plan to manage this server using Visual Studio Code's Remote - SSH extension, you **must** change `AllowTcpForwarding` to `yes`. VS Code relies on TCP forwarding to establish its remote server backend. If you use SSH Agent forwarding to pass your local keys to git inside the server, you must also set `AllowAgentForwarding yes`.

##### **Rationale (Grouped by Function):**

**Access Control (`AllowGroups sudo`, `PermitRootLogin no`):** 
Completely blocks any user not in the `sudo` group from even attempting an SSH connection. Disabling direct `root` login forces attackers to guess both a valid username and provide a cryptographic key.

**Authentication Restrictions:** 
Disabling `PasswordAuthentication` and `KbdInteractiveAuthentication` ensures that only cryptographic SSH keys are accepted. We keep `UsePAM yes` because modern Linux systems rely on Pluggable Authentication Modules for session setup, environment variables, and user limits.

**Rate Limiting (`MaxStartups`, `LoginGraceTime`):** 
Protects the SSH daemon from resource exhaustion (DoS) attacks. `MaxStartups 10:30:60` means the server will accept 10 unauthenticated connections, start randomly dropping connections if the queue exceeds that, and reject all new connections if there are 60 unauthenticated attempts pending. `LoginGraceTime 30` drops any connection that doesn't authenticate within 30 seconds.

**Session Timeouts (`ClientAlive*`, `ChannelTimeout`):**
Automatically terminates inactive sessions after 10 minutes (300 seconds * 2 count). This prevents abandoned SSH terminals from being hijacked if a developer forgets to log out.

**Forwarding Restrictions:** 
Disabling `X11`, `TCP`, and `Agent` forwarding ensures that if a user's SSH session is somehow hijacked, the attacker cannot use that connection as a VPN tunnel to access internal Docker networks or lateral systems.

**Logging (`LogLevel VERBOSE`):** 
Standard logging only records successful/failed logins. `VERBOSE` logs the exact SSH protocol and key fingerprint used, which is critical for `fail2ban` to properly parse and ban malicious IPs.

## 1.4 SSH Cryptographic Protocol Hardening

##### **Objective:** 
Audit the SSH daemon for weak legacy protocols and restrict it to use only modern, highly secure cryptographic algorithms, including Post-Quantum Cryptography (PQC) key exchanges.

##### **Action:** 
First, install `ssh-audit`, a tool used to scan the SSH server for vulnerabilities and weak algorithms. Run an initial scan to establish a baseline:

```bash
sudo apt install -y ssh-audit
ssh-audit -p 60022 localhost
```

> [!NOTE]
> You will likely see several algorithms highlighted in yellow or red, indicating weak or legacy ciphers enabled by default.

Next, append the strict cryptographic parameters to our existing hardening configuration file:

```bash
sudo nano /etc/ssh/sshd_config.d/00-hardening.conf
```

```ini
#/etc/ssh/sshd_config.d/00-hardening.conf

KexAlgorithms mlkem768x25519-sha256,sntrup761x25519-sha512@openssh.com,curve25519-sha256

Ciphers chacha20-poly1305@openssh.com,aes256-gcm@openssh.com,aes128-gcm@openssh.com

MACs hmac-sha2-512-etm@openssh.com,hmac-sha2-256-etm@openssh.com
 
HostKeyAlgorithms ssh-ed25519,rsa-sha2-512,rsa-sha2-256
```

Save the file, verify the syntax, and restart the SSH service:

```bash
sudo sshd -t
sudo systemctl restart ssh
```

Finally, run the audit tool again to verify the hardening was successful:

```bash
ssh-audit -p 60022 localhost
```

> [!NOTE]
> The output should now be much shorter, with all remaining algorithms colored green, indicating a highly secure configuration.

> [!IMPORTANT]
> If you are using an extremely old SSH client (e.g., an outdated CI/CD pipeline runner), this strict configuration might reject the connection. Ensure your local machine and automation tools are using modern SSH clients.

##### **Rationale:**
**Audit First (`ssh-audit`):** 
Running an audit before and after provides measurable proof that the hardening steps successfully eliminated vulnerable protocols.

**Key Exchange (KexAlgorithms):** 
We explicitly disabled weak NIST curves. The inclusion of `mlkem768x25519-sha256` and `sntrup761x25519-sha512` adds hybrid Post-Quantum Cryptography. This protects current SSH session traffic from being recorded by attackers and decrypted later when quantum computers become viable.

**Ciphers & MACs:** 
We stripped out all vulnerable CBC (Cipher Block Chaining) ciphers and legacy SHA1 algorithms, enforcing only fast, authenticated encryption like ChaCha20-Poly1305 and AES-GCM.

**Host Keys:** 
We restricted host keys to modern `ed25519` and strong RSA (minimum SHA-2), completely removing weak ECDSA and DSA keys.

## 1.5 Uncomplicated Firewall (UFW) Configuration

##### **Objective:** 
Implement a strict "Deny by Default" network policy to block all unauthorized inbound traffic, explicitly whitelisting only essential web traffic and our custom SSH port.
##### **Action:** 
Run the following commands to configure the default policies, open the required ports, and activate the firewall:

```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow 60022/tcp
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
sudo ufw enable
sudo ufw reload
sudo ufw status
```

> [!WARNING]
**Double-check your SSH port:** 
Ensure the port opened here `60022` exactly matches the port you configured in the `sshd_config.d/00-hardening.conf` file. If you enable UFW without allowing your custom SSH port, your active session will eventually drop, and you will be permanently locked out of the server.

>[!NOTE] 
>**Docker & UFW Bypass Architecture:** 
>Be aware that the Docker daemon manipulates `iptables` directly. This means Docker **bypasses UFW rules completely**. In our Coolify architecture, Traefik handles all routing securely through ports 80 and 443. However, if you ever manually spin up a container and publish a port to the host (e.g., `-p 3306:3306` for a database), it will be exposed to the public internet, completely ignoring the `ufw default deny` rule. Always route web traffic through Coolify/Traefik and never publish internal service ports directly to the host.

##### **Rationale:**
**Deny by Default:** 
The foundational principle of network security. Instead of trying to block bad traffic, we block _everything_ and only allow explicitly trusted protocols.

**Port 60022 (TCP):** 
Allows our administrative SSH connections on the custom port configured in the previous step.

**Ports 80 & 443 (TCP):** 
Required for Coolify and the Traefik reverse proxy to serve web applications and automatically provision Let's Encrypt SSL/TLS certificates.

## 1.6 Fail2Ban Configuration (Brute-Force Protection)

##### **Objective:** 
Deploy Fail2Ban to monitor authentication logs and automatically ban IP addresses that exhibit malicious behavior (e.g., repeated failed SSH logins), while safely whitelisting internal Docker networks and your trusted administrative IPs.

##### **Action:** 
We will create a local override file (`jail.local`). 

> [!Note]
>  A backup of this configuration is available in the repository under `config/fail2ban/jail.local` for reference.

Open a new configuration file:

```bash
sudo nano /etc/fail2ban/jail.local
```

Paste the following configuration. Make sure to replace `<YOUR_IP_ADDRESS>` with your actual static IP or office subnet if you have one (e.g., `203.0.113.50` or `203.0.113.0/24`). If you only use dynamic IPs, you can remove that placeholder.

```ini
# /etc/fail2ban/jail.local
[DEFAULT]
# Whitelist localhost and internal Docker/Coolify networks from being banned
ignoreip = 127.0.0.1/8 ::1 172.16.0.0/12 10.0.0.0/8 <YOUR_IP_ADDRESS>

[sshd]
enabled = true
port = 60022
filter = sshd
maxretry = 3
bantime = 1h
findtime = 10m
backend = systemd
```

Save the file, restart the service to apply the configuration, and verify that the `sshd` jail is actively monitoring the new port:

```bash
sudo systemctl restart fail2ban sudo fail2ban-client status sshd
```


##### **Rationale:**

**The `.local` Override:** 
We create a `jail.local` file instead of editing the default `jail.conf`. Package manager updates (via `apt upgrade`) will frequently overwrite `jail.conf`, which would silently wipe out our custom port and whitelists.
 
**Trusted IP Whitelisting (`ignoreip`):**                                                                       
**Docker Networks:** 
Coolify and Docker heavily rely on internal subnets (`172.16.0.0/12` and `10.0.0.0/8`) for container routing. Without this exclusion, 
internal routing could trigger a ban, breaking your hosted applications.                   
**Admin IP:** 
Adding your personal static IP or VPN subnet prevents you from accidentally locking yourself out if you type your SSH passphrase incorrectly multiple times.

**Port Definition (`60022`):** Fail2Ban defaults to monitoring port 22. Because we changed the SSH port, Fail2Ban would effectively do nothing unless explicitly told to block traffic on `60022`.
 
**Systemd Backend:** Modern Ubuntu distributions manage SSH logs via `systemd-journald`. Specifying `backend = systemd` ensures Fail2Ban hooks directly into the journal rather than attempting to parse legacy flat files.

## 1.7 Kernel Module Hardening (Disabling Rare Network Protocols)

##### **Objective:** 
Reduce the kernel's attack surface by disabling obscure or legacy network protocols that are not required for a modern PaaS environment but have historically been sources of critical vulnerabilities.

##### **Action:** 
We will create a configuration file instructing the kernel's module loader (`modprobe`) to explicitly ignore these protocols. 

> [!Note] 
> A backup of this configuration is available in the repository under 
> `config/modprobe/blacklist-rare-protocols.conf` for reference.

Open a new configuration file:

```bash
sudo nano /etc/modprobe.d/blacklist-rare-protocols.conf
```

Paste the following configuration:

```ini
# /etc/modprobe.d/blacklist-rare-protocols.conf
install dccp /bin/true
install sctp /bin/true
install rds /bin/true
install tipc /bin/true
```

Save the file. To ensure these modules cannot be loaded into the running kernel, you can test it with the following command (it should output `/bin/true`, meaning the system is successfully tricked into doing nothing when trying to load them):

```bash
modprobe -n -v dccp
```

##### **Rationale:**
**The `/bin/true` Trick:** Simply blacklisting a module doesn't stop an attacker (or another program) from forcing it to load. Mapping the module's installation to `/bin/true` (a command that simply returns a "success" signal and exits) tricks the kernel into thinking the module successfully loaded, completely neutralizing it. This is a strict requirement in CIS (Center for Internet Security) benchmarks.
 
**Irrelevant Protocols:** DCCP, SCTP, RDS, and TIPC are specialized networking protocols used in telecom or clustering environments. Docker, Traefik, and Coolify communicate exclusively over standard TCP and UDP.
 
**Vulnerability Mitigation:** Because these protocols are rarely used, their kernel code is heavily audited. They have historically been the source of numerous Local Privilege Escalation (LPE) exploits (e.g., allowing a compromised Docker container to break out into the host OS). Disabling them removes this attack vector entirely.

## 1.8 Disabling Core Dumps (Memory Dump Protection)

##### **Objective:** 
Prevent the operating system from writing the contents of RAM to the disk if an application crashes, ensuring sensitive data (such as database passwords, TLS encryption keys, or user session tokens) is not inadvertently leaked into local files.

##### **Action:** 
We will append a rule to the system security limits file to strictly restrict core dump file creation.

Open the limits configuration file:

```bash
sudo nano /etc/security/limits.conf
```

Scroll to the bottom and add the following line right before the `# End of file` marker:

```ini
* hard core 0
```

Save the file. This rule will automatically apply to any new user sessions or services spawned after the change.

##### **Rationale:**

**Data Leak Prevention:** When a process crashes (like a Docker container, database backend, or Node.js app), Linux can take a "snapshot" of its memory and save it as a core dump file for debugging. In a production PaaS environment, this RAM snapshot likely contains plain text environment variables (`.env` files) and database credentials. Disabling core dumps neutralizes a significant post-exploitation data leak vector.

**Disk Space Protection (Anti-DoS):** Core dumps of large databases or memory-heavy applications can be massive (several gigabytes). If a service enters a crash loop, generating continuous core dumps will quickly consume all available VPS storage, leading to a complete server outage.

## 1.9 Restricting Development Tools (Compiler Hardening)

##### **Objective:** 
Prevent unauthorized or compromised users from compiling Local Privilege Escalation (LPE) exploits or malicious payloads directly on the production host.

##### **Action:** 
Run the following command to restrict execution of the GNU Assembler (`as`) so that only the `root` administrator can use it:

```bash
sudo chmod 700 /usr/bin/as
```

##### **Rationale:**

**Exploit Mitigation:** When attackers compromise a low-privileged service (e.g., escaping a poorly configured container), their next goal is to gain full `root` access. Because pre-compiled exploits often fail due to kernel or architecture mismatches, attackers frequently attempt to download exploit source code (usually in C or Assembly) and compile it directly on the target server.

**Defense in Depth:** By stripping execute permissions (`chmod 700`) from core development tools like the assembler, we break automated exploit scripts and force attackers to cross-compile their malware off-site, significantly raising the barrier to entry.

**Zero Impact on PaaS:** In a Coolify/Docker environment, applications are built and run inside isolated containers which bring their own dependencies. The underlying Ubuntu host OS simply runs the Docker daemon and does not need unprivileged users to have access to host-level compilers.

## 1.10 Kernel Network & Sysctl Hardening

##### **Objective:** 
Secure the kernel's TCP/IP stack against common network-level attacks (such as IP spoofing, SYN floods, and MITM routing bypasses) and further restrict memory leaks from privileged processes.

##### **Action:** 
Create and open a new kernel configuration file:

> [!Note]
>  A backup of this configuration is available in the repository under `config/sysctl/99-security.conf`

```bash
sudo nano /etc/sysctl.d/99-security.conf
```

Paste the following configuration:

```ini
# /etc/sysctl.d/99-security.conf

# 1. IP Spoofing Protection
net.ipv4.conf.all.rp_filter = 1
net.ipv4.conf.default.rp_filter = 1

# 2. Smurf Attack / ICMP DoS Protection
net.ipv4.icmp_echo_ignore_broadcasts = 1

# 3. Prevent Man-in-the-Middle (MITM) Routing Bypasses
net.ipv4.conf.all.accept_source_route = 0
net.ipv6.conf.all.accept_source_route = 0

# 4. TCP SYN Flood Protection
net.ipv4.tcp_syncookies = 1
net.ipv4.tcp_max_syn_backlog = 2048

# 5. SUID Core Dump Restriction
fs.suid_dumpable = 0
```

Save the file and apply the new kernel parameters immediately without rebooting:

```bash
sudo sysctl -p /etc/sysctl.d/99-security.conf
```

##### **Rationale:**

**The `.d` Directory:** Using `/etc/sysctl.d/` keeps our custom rules cleanly separated from the OS defaults, making automation and audits much easier.

**RP Filter (`rp_filter`):** Enables Reverse Path Filtering. The kernel will verify that a packet's source IP actually matches the network interface it arrived on. This drops spoofed packets from attackers trying to impersonate internal Docker networks or trusted IPs.

**ICMP Broadcasts:** Ignoring ICMP echo requests sent to broadcast addresses prevents the server from being used as an amplifier in "Smurf" DDoS attacks.

**Source Routing (`accept_source_route`):** Source routing allows a sender to dictate the exact network path a packet should take, overriding the server's routing table. Attackers use this to bypass firewalls or intercept traffic. Disabling it forces all traffic to adhere to our strict routing rules.

**SYN Cookies & Backlog:** During a SYN Flood DoS attack, an attacker spams connection requests without completing the TCP handshake, exhausting the server's connection queue. Increasing the `syn_backlog` gives Traefik more breathing room, while `tcp_syncookies` allows the kernel to cryptographically verify connections without consuming memory for incomplete handshakes.

**SUID Dumpable:** Complements our previous core dump hardening. It specifically instructs the kernel _never_ to write memory dumps for programs running with elevated privileges (SUID), closing a critical Local Privilege Escalation (LPE) vector.

## 1.11 Securing Shared Memory (`/dev/shm`)

##### **Objective:** 
Restrict the shared memory partition to prevent the execution of malicious scripts and binaries. This mitigates "fileless" malware that attempts to run exclusively in RAM to evade disk-based detection.

##### **Action:** 
We will modify the file systems table (`/etc/fstab`) to apply strict security flags to the `/dev/shm` mount point.

Open the file systems table:

```bash
sudo nano /etc/fstab
```

Scroll to the bottom of the file and append the following line:

```ini
tmpfs /dev/shm tmpfs defaults,noexec,nosuid,nodev 0 0
```

Save the file. To apply these changes immediately without rebooting the server, reload the systemd daemon (so it parses the new fstab entries) and remount the directory:

```bash
sudo systemctl daemon-reload
sudo mount -o remount /dev/shm
```

You can verify that the flags were successfully applied by checking the active mounts:

```bash
findmnt -n -o OPTIONS /dev/shm
```

##### **Rationale:**

**The `/dev/shm` Attack Vector:** Shared memory is essentially a RAM disk available to all users. Because it writes to memory instead of the physical SSD, attackers frequently use it to download, compile, and execute malware payloads (like cryptominers or exploit code) without leaving physical traces on the drive.

**`noexec` (No Execution):** The most critical flag. It explicitly instructs the kernel to deny execution of any binary or script residing in this directory, instantly neutralizing memory-based malware drops.

**`nosuid` (No Set-User-ID):** Prevents attackers from placing an executable here with the SUID bit set, which is a common technique for escalating privileges to `root`.

**`nodev` (No Devices):** Prevents the system from interpreting block or character special devices in this directory, blocking potential hardware abstraction exploits.