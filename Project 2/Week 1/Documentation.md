# Web Application Security Testing Guide

> A practical guide covering OWASP Top 10 vulnerabilities, test environment setup, and Burp Suite usage for reconnaissance and spidering.

---

## Table of Contents

1. [OWASP Top 10 Vulnerabilities](#owasp-top-10-vulnerabilities)
2. [Setting Up the Test Environment](#setting-up-the-test-environment)
   - [DVWA (Damn Vulnerable Web Application)](#dvwa-damn-vulnerable-web-application)
   - [Juice Shop](#juice-shop-owasp)
3. [Using Burp Suite for Reconnaissance and Spidering](#using-burp-suite-for-reconnaissance-and-spidering)
4. [Practice Workflow](#practice-workflow)
5. [Resources & References](#resources--references)

---

## OWASP Top 10 Vulnerabilities

The **OWASP Top 10** is a standard awareness document representing the most critical security risks to web applications (latest edition: 2021).

---

### A01 — Broken Access Control

**What it is:** Users can act outside their intended permissions — accessing other users' data, admin pages, or performing unauthorized actions.

**Examples:**
- Accessing `/admin` without authentication
- Modifying `user_id=123` to `user_id=124` in a request to view another user's data (IDOR)
- Bypassing `hidden` HTML fields

**How to test:**
- Attempt to access restricted URLs directly
- Modify object references (IDs, filenames) in requests
- Test horizontal and vertical privilege escalation

---

### A02 — Cryptographic Failures

**What it is:** Sensitive data (passwords, credit cards, PII) is exposed due to weak or missing encryption.

**Examples:**
- Passwords stored as plain MD5 hashes
- HTTP used instead of HTTPS
- Weak cipher suites (e.g., DES, RC4)

**How to test:**
- Check if sensitive data is transmitted over HTTP
- Inspect cookies for `Secure` and `HttpOnly` flags
- Review password storage mechanisms

---

### A03 — Injection

**What it is:** Untrusted data is sent to an interpreter (SQL, OS, LDAP), causing unintended commands to execute.

**Types:**
- **SQL Injection:** `' OR '1'='1`
- **Command Injection:** `; ls -la`
- **LDAP Injection, XPath Injection**

**How to test:**
- Submit special characters (`'`, `"`, `;`, `--`) in input fields
- Use automated tools: `sqlmap`, Burp Suite Scanner
- Test all input vectors: forms, headers, cookies, URL parameters

---

### A04 — Insecure Design

**What it is:** Architectural flaws and missing security controls at the design level — not just implementation bugs.

**Examples:**
- No rate limiting on login pages (allows brute force)
- Weak password recovery flows
- Missing anti-CSRF logic by design

**How to test:**
- Review application workflows for logical flaws
- Test account recovery mechanisms
- Attempt high-volume requests to detect missing rate limiting

---

### A05 — Security Misconfiguration

**What it is:** Improperly configured servers, frameworks, or cloud environments expose attack surfaces.

**Examples:**
- Default credentials (`admin/admin`)
- Verbose error messages revealing stack traces
- Directory listing enabled
- Unnecessary features/services enabled

**How to test:**
- Check for default credentials
- Review HTTP response headers (`Server`, `X-Powered-By`)
- Enumerate directories with tools like `gobuster` or `dirb`

---

### A06 — Vulnerable and Outdated Components

**What it is:** Using libraries, frameworks, or software with known vulnerabilities.

**Examples:**
- Running jQuery 1.x with known XSS vectors
- Outdated Apache/Nginx versions
- Unpatched CMS plugins

**How to test:**
- Identify versions using `Wappalyzer`, response headers, or source code
- Cross-reference versions against CVE databases (NVD, Snyk)
- Use tools like `retire.js` for JavaScript libraries

---

### A07 — Identification and Authentication Failures

**What it is:** Weaknesses in authentication or session management allow attackers to assume other identities.

**Examples:**
- Weak passwords permitted
- Session tokens that don't expire
- Predictable session IDs

**How to test:**
- Attempt common credentials (`admin/admin`, `user/password`)
- Inspect session cookies for randomness and expiry
- Test session fixation by reusing old tokens after logout

---

### A08 — Software and Data Integrity Failures

**What it is:** Code and infrastructure that don't protect against integrity violations — e.g., insecure CI/CD pipelines, unsigned updates.

**Examples:**
- Auto-updating plugins without signature verification
- Insecure deserialization of user-supplied objects
- Tampering with JWT tokens (changing `alg` to `none`)

**How to test:**
- Inspect JWT tokens (decode at jwt.io) and test `alg: none` attack
- Test deserialization endpoints with crafted payloads
- Review update mechanisms for signature verification

---

### A09 — Security Logging and Monitoring Failures

**What it is:** Insufficient logging means attacks go undetected and unresponsive.

**Examples:**
- Failed logins not logged
- No alerting on brute-force attempts
- Logs not protected from tampering

**How to test:**
- Trigger suspicious events and verify if they are logged
- Check if log files are accessible via the web
- Attempt log injection via input fields

---

### A10 — Server-Side Request Forgery (SSRF)

**What it is:** The server is tricked into making requests to internal resources on behalf of the attacker.

**Examples:**
- Fetching `http://localhost/admin` via a URL parameter
- Accessing cloud metadata at `http://169.254.169.254/`
- Port scanning internal networks through the server

**How to test:**
- Submit internal URLs in parameters that accept URLs (`url=`, `redirect=`, `fetch=`)
- Use Burp Collaborator to detect out-of-band SSRF
- Test with `file://`, `dict://`, `gopher://` protocols

---

## Setting Up the Test Environment

> ⚠️ **Legal Disclaimer:** Only perform security testing on systems you own or have explicit written permission to test. Never target live production systems.

---

### DVWA (Damn Vulnerable Web Application)

**DVWA** is a PHP/MySQL web application intentionally made vulnerable for security training.

#### Option 1 — Docker (Recommended)

```bash
# Pull and run DVWA
docker pull vulnerables/web-dvwa
docker run -d -p 80:80 vulnerables/web-dvwa

# Access in browser
http://localhost/
```

**Default credentials:** `admin / password`

#### Option 2 — Manual Installation (Linux)

```bash
# Install dependencies
sudo apt update
sudo apt install apache2 php php-mysqli mariadb-server git -y

# Clone DVWA
cd /var/www/html
sudo git clone https://github.com/digininja/DVWA.git dvwa

# Configure database
sudo mysql -u root -e "CREATE DATABASE dvwa;"
sudo mysql -u root -e "CREATE USER 'dvwa'@'localhost' IDENTIFIED BY 'p@ssw0rd';"
sudo mysql -u root -e "GRANT ALL ON dvwa.* TO 'dvwa'@'localhost';"

# Set config
cd dvwa
cp config/config.inc.php.dist config/config.inc.php
# Edit config.inc.php — set db_user, db_password, db_database
sudo chmod -R 777 hackable/uploads/
sudo service apache2 restart
```

#### Initial DVWA Setup

1. Navigate to `http://localhost/dvwa/setup.php`
2. Click **Create / Reset Database**
3. Log in with `admin / password`
4. Set **Security Level** to `Low` to begin (Settings → DVWA Security)

#### DVWA Security Levels

| Level | Description |
|-------|-------------|
| Low | No security controls — ideal for learning |
| Medium | Some defenses — learn to bypass them |
| High | Strong controls — advanced bypass techniques |
| Impossible | Secure reference implementation |

---

### Juice Shop (OWASP)

**OWASP Juice Shop** is a modern vulnerable Node.js application covering all OWASP Top 10 categories with 100+ challenges.

#### Option 1 — Docker (Recommended)

```bash
# Pull and run Juice Shop
docker pull bkimminich/juice-shop
docker run -d -p 3000:3000 bkimminich/juice-shop

# Access in browser
http://localhost:3000
```

#### Option 2 — Node.js

```bash
# Prerequisites: Node.js v18+
git clone https://github.com/juice-shop/juice-shop.git
cd juice-shop
npm install
npm start

# Access at http://localhost:3000
```

#### Option 3 — Heroku / TryHackMe / HackTheBox

Juice Shop is available as a hosted instance on platforms like TryHackMe for browser-based practice without local setup.

#### Juice Shop Key Features

- Built-in **score board** at `/#/score-board` to track challenge completion
- Covers: XSS, SQL injection, broken authentication, IDOR, XXE, SSRF, and more
- Hidden challenges discovered through exploration

---

## Using Burp Suite for Reconnaissance and Spidering

**Burp Suite** is the industry-standard web application security testing platform by PortSwigger.

**Editions:**
- **Community** — Free, manual testing tools
- **Professional** — Paid, includes automated scanner and advanced tools

---

### Initial Setup

#### 1. Configure Browser Proxy

```
Proxy Host: 127.0.0.1
Proxy Port: 8080
```

For Firefox: Settings → Network Settings → Manual Proxy Configuration
For Chrome: Use the FoxyProxy extension

#### 2. Install Burp CA Certificate

```
1. With Burp running, visit http://burpsuite in your browser
2. Click "CA Certificate" to download
3. Import into browser:
   - Firefox: Settings → Certificates → Import
   - Chrome: Settings → Privacy → Manage Certificates → Import
```

#### 3. Configure Burp Target Scope

```
Target → Scope → Add
Enter: http://localhost/ (or your target URL)
Check "Use advanced scope control"
```

---

### Reconnaissance with Burp Suite

#### Passive Reconnaissance (Proxy Intercept)

Browse the target application normally while Burp captures all traffic.

```
Proxy → Intercept → "Intercept is ON"
Browse the target app
Review captured requests in: Proxy → HTTP History
```

**What to look for:**
- Input parameters (GET/POST fields, cookies, headers)
- Authentication endpoints
- API endpoints and their structure
- Hidden parameters in requests
- Session token patterns

#### Target Site Map

```
Target → Site Map
```

- Shows a hierarchical tree of all discovered URLs
- Right-click any item → "Add to scope" to focus testing
- Filter by file type, response code, or content type

#### Analyzing Requests

In **HTTP History**, look for:

| Item | Significance |
|------|-------------|
| `?id=`, `?user=` | Potential injection/IDOR points |
| `Authorization: Bearer` | JWT tokens to decode |
| `Set-Cookie` headers | Session management details |
| JSON/XML bodies | Deserialization targets |
| Redirects (3xx) | Potential open redirects |

---

### Spidering / Crawling with Burp Suite

#### Burp Suite Pro — Automated Crawler

```
Dashboard → New Scan → Crawl and Audit
Enter target URL
Configure: Crawler settings → Max depth, Max requests
Start scan
```

#### Burp Suite Community — Manual Crawling

Use the built-in **Spider** (older versions) or manual browsing:

```
1. Browse all application pages manually
2. Submit all forms
3. Click all links
4. Log in and browse authenticated areas
5. All traffic appears in Target → Site Map
```

#### Forced Browsing / Content Discovery

```
Target → right-click URL → Engagement Tools → Discover Content
```

Or use the **Intruder** with a wordlist to find hidden paths:

```
Send a request to Intruder
Set payload position on the path: GET /§FUZZ§ HTTP/1.1
Load wordlist: /usr/share/wordlists/dirb/common.txt
Start attack → filter by response status 200/301
```

---

### Essential Burp Suite Tools

#### Repeater — Manual Request Manipulation

```
HTTP History → Right-click request → "Send to Repeater"
Modify parameters, headers, body
Click "Send" and observe response
```

Use for: Testing injection payloads, parameter tampering, authentication bypass.

#### Intruder — Automated Payload Testing

```
HTTP History → Right-click request → "Send to Intruder"
Positions tab → Select attack type (Sniper, Cluster Bomb, etc.)
Payloads tab → Load wordlist or define payload list
Start Attack
```

**Attack Types:**

| Type | Use Case |
|------|----------|
| Sniper | Single position, iterate one payload list |
| Battering Ram | Same payload in all positions simultaneously |
| Pitchfork | Multiple positions, multiple lists (paired) |
| Cluster Bomb | Multiple positions, all combinations |

#### Decoder — Encode/Decode Data

```
Decoder tab
Paste encoded value (Base64, URL, HTML, hex)
Select decode/encode type
```

#### Comparer — Diff Responses

```
Send two responses to Comparer
Click "Words" or "Bytes" to highlight differences
```

Useful for detecting subtle differences in authentication responses.

---

### Burp Suite Workflow for DVWA / Juice Shop

```
1. Start Burp Suite → Proxy listening on 8080
2. Configure browser proxy
3. Navigate to DVWA or Juice Shop
4. Browse all features (login, forms, search, upload)
5. Review Target → Site Map for full application structure
6. Identify input vectors in HTTP History
7. Send interesting requests to Repeater
8. Test OWASP vulnerabilities manually:
   - SQLi: ' OR 1=1--
   - XSS: <script>alert(1)</script>
   - IDOR: change user IDs in parameters
9. Document findings with screenshots
```

---

## Practice Workflow

| Week | Focus | Tools |
|------|-------|-------|
| 1 | Understand OWASP Top 10 theory | Documentation, PortSwigger Web Academy |
| 2 | Set up DVWA, practice SQLi and XSS | DVWA, Burp Repeater |
| 3 | Juice Shop challenges (Injection, Auth) | Juice Shop, Burp Intruder |
| 4 | Full recon + spidering workflow | Burp Suite full workflow |
| 5+ | Advanced topics: SSRF, XXE, Deserialization | DVWA Impossible, custom labs |

---

## Resources & References

### Official Documentation
- [OWASP Top 10 (2021)](https://owasp.org/Top10/)
- [DVWA GitHub Repository](https://github.com/digininja/DVWA)
- [OWASP Juice Shop](https://owasp.org/www-project-juice-shop/)
- [PortSwigger Burp Suite Docs](https://portswigger.net/burp/documentation)

### Free Learning Platforms
- [PortSwigger Web Security Academy](https://portswigger.net/web-security) — Free labs for every OWASP category
- [TryHackMe](https://tryhackme.com) — Guided rooms for OWASP and Burp Suite
- [HackTheBox](https://hackthebox.com) — More advanced CTF-style labs

### Useful Tools Alongside Burp
- `sqlmap` — Automated SQL injection
- `gobuster` / `dirb` — Directory brute-forcing
- `nikto` — Web server scanner
- `wappalyzer` — Technology fingerprinting
- `jwt.io` — JWT token decoder

---

> **Remember:** Always practice in isolated, intentionally vulnerable environments. Unauthorized testing is illegal and unethical.
