## 3.1 Ntfy Push Notification Gateway 
### 3.1.1 Deployment & Hardening

##### **Objective:** 
Deploy a self-hosted Ntfy instance via Coolify to act as a centralized, secure push notification gateway for all infrastructure alerts (e.g., SSH logins, backup statuses, downtime).

##### **Action:**

###### **Step 1: Prepare the Host Filesystem (SSH)**
Before deploying the service, we must manually create the template directory on the host. This prevents Docker from automatically creating it with strict `root-only` permissions, ensuring we can easily edit template files later.

```bash
sudo mkdir -p /opt/coolify/ntfy_templates
```

###### **Step 2: Deploy via Coolify UI**
1. Navigate to **Projects** ➔ **Tools** in the Coolify dashboard.
2. Click **+ Add Resource**, search for **Ntfy**, and click **Deploy**.

###### **Step 3: Configure the Docker Compose Override**
In the Coolify UI for the Ntfy service, locate the Compose configuration box. Ensure the `volumes` section explicitly mounts our new templates directory. 
_(Append the template mapping to the existing volumes)_:

```yml
volumes:
  - 'ntfy-cache:/var/cache/ntfy'
  - 'ntfy-db:/var/lib/ntfy/'
  - '/opt/coolify/ntfy_templates:/etc/ntfy/templates'
```

###### **Step 4: Secure Environment Variables (.env)**
Switch to the **Environment Variables** tab. Ntfy is highly permissive by default. We must lock it down so only authenticated services (and authorized administrators) can publish or subscribe to topics.

Configure the following variables (ensure the domain matches your Traefik routing):

```env
# Base Configuration
SERVICE_URL_NTFY=https://ntfy.your_domain.pl
NTFY_BASE_URL=${SERVICE_URL_NTFY}
TZ=Europe/Warsaw

# Proxy & Upstream Routing (Crucial for iOS)
NTFY_BEHIND_PROXY=true
NTFY_UPSTREAM_BASE_URL=https://ntfy.sh
UPSTREAM_BASE_URL=https://ntfy.sh

# Security & Authentication (Zero-Trust)
NTFY_AUTH_FILE=/var/lib/ntfy/auth.db
NTFY_AUTH_DEFAULT_ACCESS=deny-all
NTFY_ENABLE_SIGNUP=false
NTFY_ENABLE_LOGIN=true
NTFY_REQUIRE_LOGIN=true

# Resource Limits & Cache Mitigation
NTFY_CACHE_DURATION=24h
NTFY_ATTACHMENT_TOTAL_SIZE_LIMIT=1G
NTFY_ATTACHMENT_FILE_SIZE_LIMIT=10M
NTFY_ATTACHMENT_EXPIRY_DURATION=24h
NTFY_VISITOR_MESSAGE_DAILY_LIMIT=100
NTFY_VISITOR_ATTACHMENT_DAILY_BANDWIDTH_LIMIT=1G
NTFY_KEEPALIVE_INTERVAL=5m
NTFY_MANAGER_INTERVAL=5m
```

##### **Rationale:**
**Zero-Trust Notifications (`deny-all`):** 
By default, anyone who guesses your Ntfy URL can subscribe to topics or publish messages. Setting `NTFY_AUTH_DEFAULT_ACCESS=deny-all` and disabling signups closes this vector. From now on, publishing an alert requires a valid access token.

**Upstream Base URL (`ntfy.sh`):**
Apple's iOS ecosystem strictly forbids background WebSocket connections to conserve battery. To receive instant push notifications on an iPhone when the Ntfy app is closed, your private server must forward a lightweight "wake-up" ping through the official `ntfy.sh` server to Apple's APNs (Apple Push Notification service). No message content is sent to `ntfy.sh`, only the wake-up trigger.

**Template Directory:** 
Mounting `/opt/coolify/ntfy_templates` prepares our server to parse complex external webhooks (like GitHub actions or Uptime Kuma downtime alerts) and translate them into beautifully formatted text notifications using Go templates.

### 3.1.2 Ntfy User & Access Management (CLI)

##### **Objective:** 
Configure internal Ntfy accounts. Because we enforced `deny-all` globally, we must explicitly create users and define their access rights to specific topics (channels).

##### **Action:**

**Pre-requisite: Find the Ntfy Container Name** 
Coolify appends random strings to container names. Find your specific Ntfy container name first:

```bash
docker ps | grep ntfy
```

###### A. Create the Main Administrator
The admin account can manage all settings and access all topics.

```bash
docker exec -it <ntfy_container> ntfy user add --role=admin inok_admin
```

###### B. Create a Service Account (Script Bot)
This account is used by bash scripts and server daemons (like Fail2Ban or Uptime Kuma) to send alerts. It receives `write-only` access to the `inok_vps` topic and authenticates via a Bearer token rather than a password.

```bash
# 1. Create the bot user
docker exec -it <ntfy_container> ntfy user add raport_bot

# 2. Grant write-only access to the specific topic
docker exec -it <ntfy_container> ntfy access raport_bot inok_vps write-only

# 3. Generate a permanent API token for scripts
docker exec -it <ntfy_container> ntfy token add raport_bot
```

> [!NOTE]
> Copy the output string starting with `tk_...` into password manager

###### C. Create a Client/Personal Account
This account is for the mobile app to receive alerts.

```bash
# 1. Create the user
docker exec -it <ntfy_container> ntfy user add inok_user

# 2. Grant read-only access (Recommended for mobile consumers)
docker exec -it <ntfy_container> ntfy access inok_user inok_vps read-only
```

###### D. Token Lifecycle (Reset & Revocation)
If a script token is ever compromised, you must rotate it:

```bash
# List all active tokens for the bot
docker exec -it <ntfy_container> ntfy token list raport_bot

# Revoke the compromised token
docker exec -it <ntfy_container> ntfy token del raport_bot <token_id>

# Issue a new one
docker exec -it <ntfy_container> ntfy token add raport_bot
```


### 3.1.3 Client Application Setup

##### **Objective:** 
Connect your mobile device securely to the private Ntfy gateway.
##### Action:
1. Download the **Ntfy** app from the iOS App Store or Google Play Store.
2. Tap the **+** icon to subscribe to a new topic.
3. Enter your full private server URL and topic: `[https://ntfy.your_domain.pl/inok_vps](https://ntfy.your_domain.pl/inok_vps)`
4. Because of our strict ACLs, the app will prompt for authentication. Enter the credentials for `inok_user`.
5. _Success:_ The app will subscribe, and you are ready to receive push notifications.

### 3.1.4 Verification & Security Testing

##### **Objective:** 
Verify that the ACLs (Access Control Lists) correctly block unauthenticated traffic and properly parse tokens.

##### Action:
Run these tests from your local machine (or any external terminal):
###### **Test 1: The Global Block (Should Fail)** 
Attempt to send a message without authentication. You should receive a `403 Forbidden` or `Unauthorized` error. 

```bash
curl -d "Unauthorized test" https://ntfy.your_domain.pl/inok_vps
```

###### **Test 2: Basic Authentication (User/Password)** 
Test the admin or user credentials (useful for Uptime Kuma or webhooks that support Basic Auth).

```bash
curl -u "inok_admin:YOUR_PASSWORD" \
     -d "Basic auth successful! Server is responding." \
     https://ntfy.your_domain.pl/inok_vps
```

###### **Test 3: Bearer Token Authentication (Script Bot)** 
This is the format you will use in your bash scripts and cron jobs.

```bash
curl -H "Authorization: Bearer tk_YOUR_SECRET_TOKEN" \
     -d "Token auth successful! Bot is operational." \
     https://ntfy.your_domain.pl/inok_vps
```

##### **Rationale:**
**Tokens over Passwords for Scripts:** 
Using `tk_...` tokens in automated scripts is vastly safer than storing raw passwords in plain text on the server. Tokens can be instantly revoked (`token del`) and rotated without changing the underlying account password.

**Write-Only / Read-Only Separation:** 
If a hacker extracts the bot token from a backup script, they can only _send_ alerts (which is annoying, but harmless). They cannot _read_ other alerts (which might contain sensitive server data) because the bot is strictly `write-only`. Conversely, if your phone is stolen, the attacker can read alerts but cannot use the app to spam the server.

## 3.2 Diun: Automated Docker Image Update Alerts

##### **Objective:** 
Deploy Diun (Docker Image Update Notifier) to automatically scan running containers and push notifications to Ntfy whenever a new upstream image (e.g., a security patch or feature update) is released on Docker Hub or other registries.

##### Action:
###### **Step 1: Deploy Diun via Coolify**
1. In the Coolify dashboard, navigate to **Projects** ➔ **Tools**.
2. Click **+ Add Resource**.
3. Search for **Diun** in the application list and click **Deploy**.

###### **Step 2: Configure Environment Variables (Ntfy Integration)** 
Navigate to the **Environment Variables** tab for the Diun resource. Paste the following configuration to integrate Diun with our secure Ntfy gateway.

> [!NOTE]
> Replace `<YOUR_BOT_TOKEN>` with the `tk_...` token generated for the `raport_bot`

```yml
environment:
# rest of env's...
# Ntfy Endpoint & Authentication
	- 'DIUN_NOTIF_NTFY_ENDPOINT=https://ntfy.your_domain.pl'
	- DIUN_NOTIF_NTFY_TOKEN= tk_token
	- DIUN_NOTIF_NTFY_TOPIC=inogi_vps
	  
# Notification Metadata
	- DIUN_NOTIF_NTFY_PRIORITY=3
	- DIUN_NOTIF_NTFY_TAGS=whale
	- 'DIUN_NOTIF_NTFY_ICON=https://raw.githubusercontent.com/crazy-max/diun/master/.res/diun.png'
	- 'DIUN_NOTIF_NTFY_CLICK=https://panel.your_domain.pl'
	- DIUN_NOTIF_NTFY_TIMEOUT=10s
	  
# Inline Templating (Required by Diun Architecture)
	- 'DIUN_NOTIF_NTFY_TEMPLATETITLE={{ .Entry.Image }} {{ if (eq .Entry.Status "new") }}is available{{ else }}has been updated{{ end }}'

	- 'DIUN_NOTIF_NTFY_TEMPLATEBODY=Docker tag {{ if .Entry.Image.HubLink }}[**{{ .Entry.Image }}**]({{ .Entry.Image.HubLink }}){{ else }}**{{ .Entry.Image }}**{{ end }} which you subscribed to through {{ .Entry.Provider }} provider {{ if (eq .Entry.Status "new") }}is available{{ else }}has been updated{{ end }} on {{ .Entry.Image.Domain }} registry (triggered by {{ .Meta.Hostname }} host).'
```

###### **Step 3: Verification & Testing**
To confirm Diun has successfully authenticated with Ntfy and is actively monitoring your images, trigger a manual test notification from the server's CLI.

 1. Find the Diun container name:
 
```bash
docker ps | grep diun
```

2. Execute the test command inside the container:

```bash
docker exec -it <diun_container_name> diun notif test
```

3. **Check your phone:**
	You should immediately receive a test push notification formatted via our Go templates, complete with the Docker (whale) icon and a clickable link to your Coolify panel.
##### **Rationale:**
**Inline Templating Constraint:** 
Unlike other services that allow mounting external `.tmpl` files via Docker volumes, Diun strictly requires Go templates to be passed directly through environment variables (`TEMPLATEBODY` and `TEMPLATETITLE`). The provided syntax parses Diun's raw JSON payloads into human-readable Markdown messages.

**RBAC Authentication:** 
By using the `DIUN_NOTIF_NTFY_TOKEN`, Diun authenticates as our restricted `raport_bot`. It only has `write-only` permission to the `inok_vps` topic, ensuring that even if Diun is compromised, the attacker cannot read other alerts on the system.

**Actionable Alerts:**
The `NTFY_CLICK` variable turns the notification block into a clickable shortcut. When you receive an update alert on your phone, tapping it takes you directly to the Coolify dashboard (`panel.inogi.pl`) so you can initiate the container rebuild.

## 3.3 Coolify System Alerts: Ntfy Webhook Integration

##### **Objective:** 
Configure the Coolify control panel to push deployment statuses, server health warnings, and application error logs directly to your mobile device. This utilizes a custom Go template to format Coolify's raw JSON payloads into human-readable alerts, using URL-based authentication.

##### Action:
###### Step 1: Create the Translation Template on the Server
Coolify sends webhooks as complex JSON objects. We must define a template in Ntfy to translate these objects into a clean title and message.

1. Navigate to the templates directory we created earlier:

```bash
cd /opt/coolify/ntfy_templates
```

2. Create and edit the template file:

```bash
nano coolify.yml
```

3. Paste the following configuration:

```yaml
title: "{{if .success}}✅{{else}}⚠{{end}} Coolify: {{if .project_name}}{{.project_name}} - {{end}}{{.event}}"  
message: |  
  {{if .message}}{{.message}}{{end}}{{if .status}}  
  📊 Status: {{.status}}{{end}}{{if .project_name}}  
  📁 Project: {{.project_name}}{{end}}{{if .environment_name}}  
  🌱 Environment: {{.environment_name}}{{end}}{{if .application_name}}  
  📦 Application: {{.application_name}}{{end}}{{if .database_name}}  
  🗄 Database: {{.database_name}}{{end}}{{if .server_name}}  
  🖥 Server: {{.server_name}}{{end}}{{if .container_name}}  
  🐳 Container: {{.container_name}}{{end}}{{if .fqdn}}  
  🌐 Domain: {{.fqdn}}{{end}}{{if .task_name}}  
  ⏱ Task: {{.task_name}}{{end}}{{if .disk_usage}}  
  💾 Disk usage: {{.disk_usage}}% (Threshold: {{.threshold}}%){{end}}{{if .total_updates}}  
  🔄 Updates: {{.total_updates}} (Critical: {{.critical_packages_count}}){{end}}{{if .affected_servers_count}}  
  ⚠ Outdated Traefik: {{.affected_servers_count}} servers {{end}}{{if .error_message}}  
  ❌ Error: {{.error_message}}{{end}}{{if .error_output}}  
  📜 Error log: {{.error_output}}{{end}}{{if .deployment_url}}  
  🚀 Deployment: {{.deployment_url}}{{end}}{{if .url}}  
  🔗 Link: {{.url}}{{end}}
```


###### **Step 2: Generate the Base64 Authentication Token**
Because Coolify's standard Webhook UI does not natively support injecting custom `Authorization: Bearer` headers, we must pass the credentials inside the URL. Ntfy requires URL tokens to be Base64 encoded without padding.

Run this command in your terminal, replacing `<YOUR_BOT_TOKEN>` with the `tk_...` string you generated.

```bash
echo -n "Bearer <YOUR_BOT_TOKEN>" | base64 | tr -d '='
```

> [!NOTE]
> Copy the resulting long string of characters. This is your `<BASE64_TOKEN>`

###### **Step 3: Configure the Webhook in Coolify**
1. Open your Coolify dashboard in the browser.
2. Navigate to **Settings** ➔ **Notifications** ➔ **Webhook**
3. Construct your final Webhook URL using the correct topic (`inok_vps`), your encoded token, and instructing Ntfy to use the `coolify` template:

```Plaintext
https://ntfy.your_domain.pl/inok_vps?auth=<BASE64_TOKEN>&template=coolify
```

4. Paste this URL into the Coolify Webhook destination field and Enable the notification channel.
###### **Step 4: Test the Integration**
1. Click the **Send Test** button in the Coolify Webhook settings.
2. **Check your phone**, you should receive a neatly formatted alert instantly, parsed by your new `coolify.yml` template, confirming the pipeline is fully operational.
##### **Rationale:**
**Template Mapping (`coolify.yml`):** 
Ntfy automatically detects files in the `/templates` directory. When we append `&template=coolify` to the URL, Ntfy intercepts the raw POST request from Coolify, matches the JSON keys (like `project_name` or `error_message`) to our template logic, and pushes out a polished message.

**Base64 URL Auth:** 
Passing `?auth=...` in the query string is highly effective for rigid external webhooks. Stripping the padding (`tr -d '='`) is a strict requirement defined by the Ntfy documentation to ensure the URL remains valid and parses correctly.

## 3.4 GitHub Security Alerts: Dependabot & Vulnerability Webhooks

##### **Objective:** 
Integrate GitHub's Dependabot and native security scanning directly into your Ntfy gateway. Instead of missing critical vulnerability emails in a cluttered inbox, you will receive immediate push notifications whenever a library in your source code is flagged for a security flaw (CVE).

##### **Action:**
###### **Step 1: Create the GitHub Template on the Server** 
First, we must create a template that parses GitHub's specific webhook payload structure into a readable notification.

1. Navigate to your Ntfy templates directory on the server:

```bash
   cd /opt/coolify/ntfy_templates
```

2. Create and edit the new template file:

```bash
nano github.yml
```

3. Paste the following configuration:
```yaml
title: "{{if .zen}}Ping{{else}}🚨 {{.repository.name}}{{end}}" 
message: | 
  {{if .zen}}Zen: {{.zen}}
  Od: {{.sender.login}}{{end}}{{if .alert}}🎬 {{.action}} 
  📦 {{.repository.full_name}}{{if .alert.security_vulnerability}}
  🧩 {{.alert.security_vulnerability.package.name}}{{end}}{{if .alert.security_advisory}} 
  🔥 {{.alert.security_advisory.severity}} 
  🐛 {{.alert.security_advisory.cve_id}}{{end}}{{if .alert.html_url}}
  🔗 {{.alert.html_url}}{{end}}{{end}}
```

###### **Step 2: Prepare the Webhook URL**
You will need the same Base64 encoded bot token generated in the previous Coolify webhook step. Your final URL will look like this:
`
```Plaintext
https://ntfy.your_domain.pl/inok_vps?auth=<BASE64_TOKEN>&template=github
```

###### **Step 3: Configure the Webhook in GitHub**
1. Navigate to your Organization (or specific Repository) settings on GitHub.
2. In the left sidebar, select **Settings** ➔ **Webhooks**.
3. Click the **Add webhook** button.
4. Fill out the payload form:
    - **Payload URL:** Paste the URL from Step 2 (containing your auth token and `&template=github`).
    - **Content type:** Select `application/json` _(CRITICAL: The template will fail if left as `application/x-www-form-urlencoded`)_.
5. Under **"Which events would you like to trigger this webhook?"**:
    - Select **Let me select individual events**.
    - Uncheck the default "Pushes" option.
    - Check **ONLY** these two options:
        - ☑ **Repository vulnerability alerts**
        - ☑ **Dependabot alerts**
###### **Step 4: Save and Verify**
1. Click **Add webhook** at the bottom of the page.
2. GitHub will immediately send a test payload (a "ping" event containing a "zen" quote).
3. **Check your phone:** You should receive a notification titled "Ping" with a zen quote from GitHub. You can also verify the HTTP `200 OK` delivery status at the bottom of the webhook settings page under the **Recent Deliveries** tab.

##### **Rationale:**
**Filtering Event Types:** GitHub webhooks can be incredibly noisy. If we subscribed to "Push" or "Everything", your phone would buzz every single time a developer pushes code. By explicitly filtering _only_ for Vulnerability and Dependabot alerts, we ensure this channel remains reserved strictly for actionable security incidents.

**The "Zen" Ping:** GitHub's architecture always sends an initial `ping` event to verify the endpoint is alive. Our `github.yml` template specifically looks for the `{{.zen}}` variable to elegantly handle this test payload without breaking the parsing logic for actual security alerts.

## 3.5 External Heartbeat Monitoring (Healthchecks.io & Telegram)

##### **Objective:** 
Implement a "Dead Man's Switch" for the server. Because our Ntfy gateway is hosted on the server itself, a catastrophic host failure (power loss, network partition) will take down our notification system too. Healthchecks.io will act as an external watcher, alerting us via an out-of-band channel (Telegram) if our server suddenly stops pinging it.

##### **Action:**
###### Step 1: Create the Project and Check (Healthchecks.io)
1. Create a free account at [healthchecks.io](https://healthchecks.io) and complete the initial setup.
2. Create a new "Check" (e.g., named `VPS Heartbeat`).
3. Set the **Period** to 15 minutes, and the **Grace Time** to 5 minutes.
4. Copy the unique Ping URL provided (it will look like `[https://hc-ping.com/](https://hc-ping.com/)<your-uuid>`).

###### **Step 2: Configure the Server Heartbeat (Cron)**
Log into your server via SSH. We will add a recurring task that constantly tells Healthchecks.io "I am alive."

1. Open the crontab editor for the root user (ensure you are using `sudo` so it survives user-level restrictions):
```bash
sudo crontab -e
```

2. Paste the following line at the very bottom of the file, replacing `<your-uuid>` with your actual code:

```Plaintext
*/15 * * * * curl -fsS -m 10 --retry 5 -o /dev/null https://hc-ping.com/<your-uuid>
```

3. Save and close the editor.
4. Go back to the Healthchecks.io dashboard. Wait a maximum of 15 minutes (or run the curl command manually once). The check's status should turn **Green (Up)**.

###### **Step 3: Out-of-Band Alerts via Telegram (Optional)**
To complement email alerts, we will connect Telegram as our emergency channel.

1. In the Healthchecks.io dashboard, navigate to the **Integrations** tab for your project.
2. Locate the **Telegram** integration block.
3. Open the Telegram app on your phone or desktop and search for the official bot: `@HealthchecksBot`.
4. Start a chat with the bot (`/start`). It will reply with a special authorization link.
5. Click the link provided by the bot. This will instantly securely bind your Telegram account to your Healthchecks.io project.
6. Return to the Healthchecks.io dashboard and click **Test** on the Telegram integration to ensure the message successfully arrives on your phone.
##### Rationale
**The Out-of-Band Concept:**
By intentionally using Telegram instead of Ntfy for this specific alert, we avoid a single point of failure. If the entire server rack in the datacenter loses power, Healthchecks.io will notice the missing pings and message you on Telegram, completely bypassing your dead server.

**Robust Cron Command:** 
The curl command provided is highly resilient. `-fsS` silences progress bars but shows errors, `-m 10` enforces a 10-second timeout so the cron job doesn't hang indefinitely if the network is slow, and `--retry 5` ensures that a minor network blip doesn't cause a false alarm.

**15-Minute Frequency:** 
`*/15 * * * *` means the ping executes every 15 minutes. Combined with a 5-minute grace period, you will be notified via Telegram exactly 20 minutes after your server suffers a hard crash.

## 3.6 Proactive Monitoring & Log Streaming (Uptime Kuma & Dozzle)

##### **Objective:** 
Deploy a dual-tool monitoring stack. **Uptime Kuma** will actively ping our services and websites to alert us immediately if they go down, while **Dozzle** will provide a lightweight, real-time web interface to stream internal Docker logs for rapid troubleshooting without SSH.

##### **Action:**

###### **Step 1: Deploy via Coolify**
1. In the Coolify dashboard, navigate to **Projects** ➔ **Tools**.
2. Click **+ New Resource** again, search for **Dozzle**, and click **Deploy**.
3. Click **+ New Resource**, search for **UptimeKuma**, and click **Deploy**.
4. Before starting the container (or immediately after), navigate to the **Domains** tab for both resources and set the public URL to: `[https://uptime.your_domain.pl](https://uptime.your_domain.pl)` and `[https://logs.your_domain.pl](https://logs.your_domain.pl)`
5. Open the Dozzle URL in your browser. You can now view live, color-coded logs for every running container on the server, as Coolify automatically maps the `/var/run/docker.sock` volume during deployment.
###### **Step 2: Initial Setup & Reverse Proxy Configuration**
1. Open your browser and navigate to your new domain: `[https://uptime.your_domain.pl](https://uptime.your_domain.pl)`.
2. Complete the initial setup wizard by creating the primary administrator account. (Save these credentials immediately in your password manager, e.g., Proton Pass).
3. Log into the Uptime Kuma dashboard and navigate to **Settings** ➔ **Reverse Proxy**
4. Scroll to **HTTP Headers** and set **Trust Proxy** to **YES**.
5. Click **Save**.

 > [!Warning]
 > **Security Roadmap: Authentik SSO**
 > By default, the login pages for these tools (or lack thereof, in Dozzle's case) are exposed to the public internet. To maintain our Zero-Trust architecture, we will eventually protect these services behind a centralized Single Sign-On gateway. The deployment of **Authentik** and the Traefik Forward-Auth configuration will be fully covered in [[07-authentik-sso | 07-authentik-sso.md]].

###### **Step 3: Integrate Notifications (Ntfy)** 
We will route all downtime alerts directly to your phone using the secure gateway established earlier.

1. In the Uptime Kuma dashboard, navigate to **Settings** ➔ **Notifications**.
2. Click **Setup Notification**.
3. Configure the parameters exactly as follows:
	- **Notification Type:** `ntfy`
	- **Friendly Name:** `Ntfy Inogi VPS`
	- **ntfy topic:** `inok_vps`
	- **Server URL:** `[https://ntfy.your_domain.pl](https://ntfy.your_domain.pl)`
	- **Authentication Method:** `Access Token`
	- **Access Token:** Paste the `<tk_...>` token generated for the `raport_bot` account.
4. Click the **Test** button. You should instantly receive a push notification on your mobile device.
5. Click **Save**. 

##### **Rationale:**
**The "Trust Proxy" Setting:** 
Coolify automatically routes traffic through its internal Traefik reverse proxy. If Uptime Kuma does not trust this proxy, it will register all incoming traffic as originating from the internal Docker IP (e.g., `172.17.0.x`) rather than the real visitor's IP address. 
Enabling this ensures proper security auditing, rate-limiting, and UI redirects.

**Token Authentication:**
By explicitly using the `Access Token` method with our `raport_bot`, we ensure Uptime Kuma does not have our master password. If Kuma is ever compromised, the attacker only gains `write-only` access to the notification topic.