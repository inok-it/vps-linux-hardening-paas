## 8.0 - Security Audits & Incident Analysis

This module establishes a routine auditing framework. Even with a Zero-Trust architecture and Edge WAF, a secure server requires active monitoring for misconfigurations, stealthy rootkits, and anomalies in resource consumption.
## 8.1 Automated Security Auditing (Lynis)

##### **Objective:** 
Install and run Lynis, an industry-standard security auditing tool. We will bypass the default Ubuntu repositories to ensure we are scanning with the latest vulnerability definitions directly from the vendor.

##### **Action:**

###### **Step 1: Add the Official Repository & Install**
Execute the following commands to securely import the GPG key and install the latest community edition of Lynis:

```Bash
# 1. Download and dearmor the GPG key
wget -O - https://packages.cisofy.com/keys/cisofy-software-public.key | sudo gpg --dearmor -o /usr/share/keyrings/cisofy-software-public.gpg

# 2. Add the repository to APT sources
echo "deb [signed-by=/usr/share/keyrings/cisofy-software-public.gpg] https://packages.cisofy.com/community/lynis/deb/ stable main" | sudo tee /etc/apt/sources.list.d/cisofy-lynis.list

# 3. Update index and install
sudo apt update
sudo apt install lynis -y
```
###### **Step 2: Run the Initial Audit**
```Bash
sudo lynis audit system
```

> [!Note]
Review the output carefully. Lynis will highlight warnings in red and suggestions in yellow. Note the Hardening Index score at the bottom.

## 8.2 Rootkit & Malware Detection (Rkhunter & Debsums)

##### **Objective:** 
Deploy tools to detect kernel-level rootkits, backdoors, and unauthorized modifications to core system binaries.

##### **Action:**
###### **Step 1: Install and Initialize Rkhunter**

```Bash
sudo apt install rkhunter debsums -y

# Update the malware definitions database
sudo rkhunter --update

# Create the foundational baseline of your system
sudo rkhunter --propupd
```


> [!NOTE]
Whenever you run sudo apt upgrade (which updates packages like the Linux kernel, SSH, or coreutils), the file checksums will legally change. You MUST run `sudo rkhunter --propupd` immediately after upgrading the system to update the baseline. Failing to do this will trigger massive false-positive alarms during the next scan.

  

###### **Step 2: Perform a Clean Verification Scan**

```Bash
# 1. Check for rootkits (ignore verbose "OK" lines)
sudo rkhunter --check --rwo
# 2. Verify APT binary integrity (prints only modified/failing files)
sudo debsums -s
```

##### **Rationale:**
**The `--rwo` Flag:** Stands for _Report Warnings Only_. Rkhunter outputs hundreds of lines during a scan. This flag filters the noise, showing only actionable threats.

**Debsums:** A highly effective, lightweight supply-chain defense. It compares the cryptographic hashes of installed executables against the official Ubuntu repository manifests. If an attacker silently replaces `/bin/ls` or `/usr/sbin/sshd` with a malicious version, `debsums -s` will instantly catch it.


## 8.3 Live Resource & Network Profiling

##### **Objective:** 
Identify unauthorized background processes or resource exhaustion (e.g., a cryptominer or data exfiltration script) using live telemetry.
##### **Action:**
Ensure the necessary profiling tools are installed:
```Bash
sudo apt install htop iotop nethogs -y
```

Run these commands during your audit window to profile system behavior:
1. **CPU/RAM Profiling:** `htop`
    _(Look for orphaned processes consuming 100% CPU)._
2. **Disk I/O Profiling:** `sudo iotop`
    _(Identifies if a specific container or process is thrashing the SSD, potentially causing system-wide latency)._
3. **Network Exfiltration:** `sudo nethogs`
    _(Groups bandwidth by process PID rather than protocol. Immediately reveals if an unexpected binary is transmitting large amounts of outbound data)._


## 8.4 Incident Analysis (Fail2Ban Logs)

##### **Objective:**
Analyze edge-case brute-force attempts that bypassed Cloudflare (e.g., direct-to-IP SSH attacks) to identify coordinated attack campaigns.

##### **Action:**
###### **Step 1: Check Jail Status**
Verify that the SSH jail is actively monitoring and blocking IPs:

```Bash
sudo fail2ban-client status sshd
```
###### **Step 2: Review Recent Bans**
Extract the last 20 banned IP addresses from the system log:

```Bash
grep "Ban" /var/log/fail2ban.log | tail -n 20
```

##### **Rationale:**
If you notice a sudden, massive spike in banned IPs from a specific subnet, you can proactively blacklist that entire CIDR range at the Cloudflare WAF level (from [`05-cloudflare-waf-security.md`](docs/05-cloudflare-waf-security.md)) or via UFW, stopping the traffic before it even reaches the Fail2Ban processing daemon.
