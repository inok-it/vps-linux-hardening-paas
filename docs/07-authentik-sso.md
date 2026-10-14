## 7.0 Authentik SSO & Zero-Trust Architecture

This module covers the deployment of Authentik, an open-source Identity Provider. By integrating Authentik with our Traefik reverse proxy (Forward-Auth), we will mandate strict Multi-Factor Authentication (MFA) before anyone can even view the login screens of our internal monitoring tools (like Dozzle or Uptime Kuma).

## 7.1 Authentik Deployment & Initialization (Coolify)

##### **Objective:** 
Deploy the Authentik stack via Coolify, bind it to a secure domain, and initialize the immutable root administrator account.
##### **Action:**

###### **Step 1: Deploy via Coolify**
1. In the Coolify dashboard, navigate to **Projects**.
2. Select your designated project (e.g., "Monitoring" or "Infrastructure").
3. Click **+ New Resource**, search for **Authentik**, and click **Deploy**.

###### **Step 2: Network Configuration**

1. Navigate to the **Domains** tab for the new Authentik resource.
2. Enter your chosen authentication domain (e.g., `[https://auth.your_domain.pl](https://auth.your_domain.pl)`).

> [!WARNING]
>  Ensure the underlying port field is blank (remove `9000`).

3.  Save the configuration. Coolify/Traefik will now provision the SSL certificate.

###### **Step 3: Initial Setup Wizard (Root Account)**
Unlike standard applications, Authentik secures its initial setup behind a specific hidden path.
1. Open your browser and navigate precisely to: `[https://auth.your_domain.pl/if/flow/initial-setup/](https://auth.your_domain.pl/if/flow/initial-setup/)`
2. Follow the setup wizard to create the root administrator account.
3. Generate a highly complex password and save it immediately in your password manager
## 7.2 Mandatory MFA: Hardware Security Keys (FIDO2/WebAuthn)

##### **Objective:** 
Lock down the newly created root administrator account by enforcing hardware-backed Multi-Factor Authentication (YubiKey). This prevents credential stuffing or phishing attacks from compromising the core Identity Provider.
##### **Action:**
###### **Step 1: Access MFA Settings**
1. Log into your new Authentik dashboard at `[https://auth.your_domain.pl](https://auth.your_domain.pl)`.
2. Click your profile icon in the top right corner and select **User interface** (or **My Settings**, depending on the UI layout).
3. Navigate to the **MFA Settings** section.
###### **Step 2: Register the Security Key**
1. Click **Register WebAuthn device**.
2. Assign a clear, identifiable name to the key (e.g., `YubiKey_Primary`).
3. Click **Register**.
4. When your browser prompts you, insert your hardware security key (YubiKey) into the USB port and physically touch the sensor.
5. The device is now permanently cryptografically bound to your account.

##### **Rationale:**
**Removing Port 9000:** 
Authentik runs natively on port 9000. If you leave this port in the Coolify domain configuration, Traefik might attempt to route external traffic to `[https://auth.your_domain.pl:9000](https://auth.your_domain.pl:9000)` or expose the port publicly, bypassing standard HTTPS routing. Clearing the field forces Coolify to map it cleanly to standard web ports.

**WebAuthn vs TOTP:** 
While Authentik supports authenticator apps (TOTP - 6-digit codes), we strictly mandate WebAuthn (FIDO2) for the administrator. TOTP codes can be phished (stolen via fake login pages), whereas hardware keys cryptographically verify the actual domain (`auth.your_domain.pl`) before releasing the token, making them immune to phishing.

## 7.3 Traefik Forward-Auth Integration: Securing Dozzle

##### **Objective:** 
Protect the public Dozzle log viewer behind Authentik SSO using Traefik's Forward-Auth middleware. Dozzle will trust Authentik to handle authentication and will read the injected HTTP headers to identify the user.

##### **Action:**

###### Step 1: Authentik - Create Provider & Application
First, we must define Dozzle within our Identity Provider.

1. Log into your Authentik dashboard as the administrator.
2. Navigate to **Applications** ➔ **Providers** and click **Create**.
3. Select **Proxy Provider** and configure the following:
    - **Name:** `dozzle-auth-provider`
    - **Authorization flow:** `default-provider-authorization-implicit-consent`
    - **Type:** `Forward auth (single application)`
    - **External Host:** `[https://logs.your_domain.pl](https://logs.your_domain.pl)` _(Must match your Dozzle domain exactly)_
4. Click **Finish**.
5. Navigate to **Applications** ➔ **Applications** and click **Create with Wizard** (or **Create** ➔ **New Application with existing provider**).
6. Configure the application:
    - **Name:** `Logi - Dozzle`
    - **Provider:** Select `dozzle-auth-provider`
    - _(Optional: Set a UI icon and UI URL for the App Launcher)_
7. Click **Create**.

###### Step 2: Authentik - Bind to the Embedded Outpost
An "Outpost" is the physical listener that Traefik talks to. We must tell the default outpost to manage our new Dozzle application.
1. Navigate to **Applications** ➔ **Outposts**.
2. Click the edit icon (pencil) next to the **authentik Embedded Outpost**.
3. In the **Applications** list, click on `Logi - Dozzle` to highlight it, then click the right-arrow (`>`) to move it into the **Selected applications** column.
4. Click **Update**.
###### Step 3: Find the Internal Authentik Server Name
Traefik needs to know exactly where to route the authentication requests inside the internal Docker network.
1. In your Coolify dashboard, navigate to your Authentik resource.
2. Look at the running containers list or check the terminal (`docker ps | grep authentik-server`).
3. Note the exact container name of the Authentik server (it will look something like `authentik-server-yiadkiyamgvbg`).


> [!Warning]
> In the next step, you MUST replace `<YOUR_AUTHENTIK_CONTAINER_NAME>` with the exact string you found in Step 3. If this name is incorrect, Traefik will return a "502 Bad Gateway" error because it cannot reach the authentication server.

###### Step 4: Coolify - Configure Dozzle Docker Compose
Now we instruct Traefik to intercept Dozzle traffic and pass it to Authentik.
1. In Coolify, navigate to your **Dozzle** resource.
2. Open the **Docker Compose** configuration editor.
3. Inject the `environment` and `labels` sections exactly as shown below, updating the Traefik middleware address:

YAML

```
services:
  dozzle:
    # ... (keep existing image/volumes configuration) ...
    environment:
      - DOZZLE_AUTH_PROVIDER=forward-proxy
      - DOZZLE_AUTH_HEADER_USER=X-authentik-username
      - DOZZLE_AUTH_HEADER_EMAIL=X-authentik-email
      - DOZZLE_AUTH_HEADER_NAME=X-authentik-name
      - DOZZLE_AUTH_LOGOUT_URL=https://logs.your_domain.pl/outpost.goauthentik.io/sign_out
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.dozzle.rule=Host(`logs.your_domain.pl`)"
      - "traefik.http.routers.dozzle.entrypoints=https"
      - "traefik.http.routers.dozzle.tls.certresolver=letsencrypt"
      - "traefik.http.services.dozzle.loadbalancer.server.port=8080"
      
      # Forward-Auth Middleware configuration
      - "traefik.http.middlewares.dozzle-auth.forwardauth.address=http://<YOUR_AUTHENTIK_CONTAINER_NAME>:9000/outpost.goauthentik.io/auth/traefik"
      - "traefik.http.middlewares.dozzle-auth.forwardauth.trustForwardHeader=true"
      - "traefik.http.middlewares.dozzle-auth.forwardauth.authResponseHeaders=X-authentik-username,X-authentik-email,X-authentik-name"
      
      # Attach the middleware to the router
      - "traefik.http.routers.dozzle.middlewares=dozzle-auth"
```

4. **Save** and **Restart** the Dozzle container.

###### **Step 5: Verification**
1. Open an incognito/private browser window.
2. Navigate to `[https://logs.your_domain.pl](https://logs.your_domain.pl)`.
3. You should be immediately intercepted and redirected to the Authentik SSO login screen.
4. After logging in with your YubiKey-secured admin account, you will be seamlessly redirected back to Dozzle, now fully authenticated.

##### **Rationale:**
**The Intercept:** 
The label `traefik.http.routers.dozzle.middlewares=dozzle-auth` is the magic switch. It tells Traefik: "Before serving `logs.inogi.pl`, run this traffic through the `dozzle-auth` middleware."

**Header Injection:** 
The `authResponseHeaders` label tells Traefik to take the user's identity data returned by Authentik and inject it as HTTP headers (`X-authentik-username`, etc.). Dozzle's `environment` variables tell Dozzle to look for these specific headers to establish the user session natively within the app.
## 7.4 Traefik Forward-Auth Integration: Securing Uptime Kuma

##### **Objective:** 
Protect the Uptime Kuma status and dashboard page with Authentik. Because Coolify automatically generates dynamic Traefik router names for certain one-click deployments, we must extract the exact router name before applying our middleware.

##### **Action:**
###### Step 1: Disable Native Uptime Kuma Authentication
Before we lock the front door with Authentik, we should disable Uptime Kuma's internal login screen to avoid forcing users to log in twice.

1. Access your Uptime Kuma instance natively (`[https://status.inogi.pl](https://status.inogi.pl)`) and log in.
2. Navigate to **Settings** ➔ **Security**.
3. Locate the option to **Disable Authentication** (or similar, depending on the version) and save.

###### Step 2: Authentik - Create Provider & Application

1. Log into your Authentik dashboard.
2. Navigate to **Applications** ➔ **Providers** and click **Create**.
3. Select **Proxy Provider** and configure:
    - **Name:** `uptime-kuma-auth`
    - **Authorization flow:** `default-provider-authorization-implicit-consent`
    - **Type:** `Forward auth (single application)`
    - **External Host:** `[https://status.inogi.pl](https://status.inogi.pl)`
4. Click **Finish**.
5. Navigate to **Applications** ➔ **Applications** and click **Create with Wizard** (or **Create** ➔ **New Application with existing provider**).
6. Configure the application:
    - **Name:** `Uptime Kuma`
    - **Provider:** Select `uptime-kuma-auth`
7. Click **Create**.

###### Step 3: Authentik - Bind to the Embedded Outpost
1. Navigate to **Applications** ➔ **Outposts**.
2. Click the edit icon (pencil) next to the **authentik Embedded Outpost**.
3. In the **Applications** list, select `Uptime Kuma` and click `>` to move it to the **Selected applications** column.
4. Click **Update**.
###### Step 4: Coolify - Extract the Dynamic Router Name
Coolify generates unique Traefik router identifiers for certain resources to prevent network collisions. You must find yours.

1. In the Coolify dashboard, navigate to your **Uptime Kuma** resource.
2. Click on **Edit Compose File** (or access the advanced configuration).
3. Click the **Preview Generated Compose** button.
4. Scroll down to the `labels:` section. Find the line that looks like `traefik.http.routers.http-0-xxxxxxxxxxxx.rule=...`.
5. Copy that exact router name (e.g., `http-0-m5bvtyvjmiuvtnmhx380ik8w-uptime-kuma-6596`).

> [!WARNING]
> **Dynamic Router Names**
> Do not copy the router name from this documentation blindly. You must use the exact router name generated by your specific Coolify instance, otherwise Traefik will completely ignore your middleware and leave the application unprotected. Ensure you also replace the Authentik container ID in the address line.

###### Step 5: Coolify - Apply Middleware Labels
1. Return to the normal Compose editor for Uptime Kuma.
2. Add the following lines to the `labels:` section. Replace `<YOUR_AUTHENTIK_CONTAINER>` and `<YOUR_DYNAMIC_ROUTER_NAME>` with your specific values.
```YAML
labels:
	# Forward-Auth Middleware Definition
	- "traefik.http.middlewares.uptimekuma-auth.forwardauth.address=http://<YOUR_AUTHENTIK_CONTAINER>:9000/outpost.goauthentik.io/auth/traefik"
	- "traefik.http.middlewares.uptimekuma-auth.forwardauth.trustForwardHeader=true"
	- "traefik.http.middlewares.uptimekuma-auth.forwardauth.authResponseHeaders=X-authentik-username,X-authentik-email,X-authentik-name"
      
      # Attach the middleware to your specific generated router
	- "traefik.http.routers.<YOUR_DYNAMIC_ROUTER_NAME>.middlewares=uptimekuma-auth"
```
3. **Save** and **Restart** the Uptime Kuma container.
4. Open an incognito window and visit `[https://status.your_domain.pl](https://status.your_domain.pl)`. You should instantly see the Authentik login prompt.

##### **Rationale:**
**The Architecture of Forward Auth:**
Unlike reverse proxies (which handle all traffic themselves), Forward Auth instructs Traefik to act as a bouncer. Traefik asks Authentik "is this session valid?". If Authentik says yes, Traefik lets the user through to Uptime Kuma. This means Uptime Kuma doesn't even need to know Authentik exists.

**Disabling Native Auth:**
By disabling Kuma's internal auth, we fully delegate identity management to Authentik. This ensures our YubiKey (FIDO2) policy is strictly enforced and cannot be bypassed.