
## 5.0 Edge Security & DNS: Cloudflare WAF and Bot Management

This module outlines the configuration of Cloudflare as our edge network provider. Cloudflare acts as a massive shield in front of our OVH server, intercepting all traffic, blocking malicious bots, enforcing HTTPS, and mitigating DDoS attacks before they ever reach our infrastructure.

## 5.1 Domain Onboarding and DNS Setup

##### **Objective:** 
Delegate your domain's DNS resolution to Cloudflare so it can proxy and protect your web traffic.

##### Action:
###### **Step 1: Add Site to Cloudflare**
1. Log into your Cloudflare dashboard and click **Add a Site** (or **Add a Domain**).
2. Enter your domain name (e.g., `inogi.pl`) and select the Free plan.
3. Cloudflare will automatically scan your existing DNS records. Verify they are correct and click **Continue**.

###### **Step 2: Delegate Nameservers**
1. Cloudflare will provide two custom nameservers (e.g., `matias.ns.cloudflare.com`, `ximena.ns.cloudflare.com`).
2. Log into the panel of your domain registrar (the company where you bought the domain, e.g., OVH, Seohost).
3. Locate the **DNS / Nameservers** section for your domain.
4. Replace the existing nameservers (e.g., `dns200.anycast.me`) with the two provided by Cloudflare.
5. Return to Cloudflare and click **I updated my nameservers**.
###### **Step 3: Initial DNS Security Settings**

1. In Cloudflare, navigate to the **DNS** ➔ **Settings** menu.
2. Locate **DNSSEC** and click **Enable**.

##### **Rationale:**
**The Power of the Proxy:**
By changing the nameservers, we route all global traffic through Cloudflare's infrastructure first. This allows Cloudflare to inspect packets and drop malicious payloads before they ever consume bandwidth on our OVH server.

**DNSSEC (DNS Security Extensions):** 
This acts like a cryptographic signature for your domain. It prevents "DNS cache poisoning" or spoofing, ensuring that when a client types `inogi.pl`, their ISP guarantees they are routed to your actual server, not a malicious phishing clone.

## 5.2 SSL/TLS Configuration (Encryption in Transit)

##### **Objective:** 
Enforce strict encryption at the edge level to protect data from Man-In-The-Middle (MITM) attacks and ensure all users connect via secure protocols.
##### **Action:**
Navigate to **SSL/TLS** ➔ **Edge Certificates** in the Cloudflare sidebar and configure the following:
1. **Always Use HTTPS:** Turn **ON**.
2. **Minimum TLS Version:** Set to **TLS 1.2**.
3. **HTTP Strict Transport Security (HSTS):** Click **Enable HSTS** and configure carefully:
    - **Max-Age Header:** `6 months` 
    - **Include subdomains:** `On` 
    - **Preload:** `Off` _(Only enable this after months of stable HTTPS, as it is very difficult to undo)._
    - **No-Sniff Header:** `On`

##### Rationale:
**Edge-Level Redirection:** 
Enforcing `Always Use HTTPS` at the edge means unencrypted `http://` requests are rejected by Cloudflare instantly. Your server doesn't have to waste CPU cycles handling redirects.

**HSTS Prevention:** 
HSTS prevents "SSL Stripping" attacks, where a compromised public Wi-Fi network attempts to downgrade a user's connection to HTTP. It instructs the browser to hardcode the HTTPS requirement for your domain.

**Deprecating TLS 1.0/1.1:**
Legacy TLS versions suffer from known cryptographic vulnerabilities. Modern security compliance frameworks strictly mandate dropping support for anything below TLS 1.2.

## 5.3 Bot Management and AI Protection

##### **Objective:** 
Stop automated scrapers, vulnerability scanners, and AI training bots from consuming your server resources or scraping your intellectual property.

##### **Action:**

###### **Step 1: Enable Bot Fight Mode**
1. Navigate to **Security** ➔ **Settings**.
2. Under the **Bot traffic** section, find **Bot Fight Mode**
3. Turn it **ON**.
###### **Step 2: AI Crawl Control**
1. Navigate to **Security** ➔ **AI Crawl Control** (This might be located under a specific menu or quick actions depending on your dashboard layout).
2. Review the list of AI crawlers (like GPTBot, CCBot).
3. For any bots that violate your content policies, select **Block** in the Action column 
###### **Step 3: Browser Integrity Check**
1. Navigate to **Security** ➔ **Settings**.
2. Turn **Browser Integrity Check** **ON**.

##### **Rationale:**
**Resource Conservation:** 
Malicious bots and scrapers can account for up to 40% of internet traffic. By turning on Bot Fight Mode, Cloudflare uses JS challenges to block them, keeping your server's RAM and CPU entirely free for legitimate, paying clients.

**Protecting Intellectual Property:**
AI crawlers aggressively scrape website content to train Large Language Models without compensation or attribution. Blocking them at the edge is far more effective than updating a `robots.txt` file, which bad actors simply ignore.

**Browser Integrity Check:** 
This acts as a lightweight filter that instantly drops traffic from amateur scripts (like basic Python `requests` or `curl` commands) that fail to send proper HTTP headers or User-Agents typical of real web browsers.
## 5.4 Custom Security Rules (WAF / Rate Limiting)

##### **Objective:** 
Protect critical administrative endpoints (like the Coolify dashboard or WordPress login pages) from brute-force password guessing attacks.

##### **Action:**

###### **Step 1: Create a Rate Limiting Rule**

1. Navigate to **Security** ➔ **Security Rules** (formerly WAF)
2. Click **Create rule** and select **Rate limiting rules** 
3. Name the rule (e.g., "Protect Coolify Login").
4. Define the **If...** conditions:
    - `URI Path` `equals` `/` _(Or specify the exact path of your login form, e.g., `/login`)_
    - **AND** `Method` `equals` `POST` _(Targets form submissions/login attempts)._
5. Define the **Rate limit**: (e.g., `5` requests per `1 minute`).
6. Set the **Action** to **Block** (or **Managed Challenge** to force a Turnstile CAPTCHA) 
7. Click **Deploy**.

##### **Rationale:**
**Edge over Origin:** 
Doing all of this at the Cloudflare Edge means malicious traffic is dropped _before_ it reaches your OVH server. This saves your VPS CPU and bandwidth for legitimate users.

**Rate Limiting Specifics:** 
By targeting `POST` requests, we only limit login attempts, not standard page loads (`GET` requests), ensuring legitimate users browsing the site are not accidentally blocked.