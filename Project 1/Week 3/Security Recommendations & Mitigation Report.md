#  Security Recommendations & Mitigation Report

**Tool:** OpenVAS | **Target:** 127.0.0.1 (Metasploitable) | **Date:** May 15, 2026
**Total CVEs:** 36 | **Scope:** 12 unique vulnerability classes | **Host(s):** 1

---

##  Table of Contents

1. [Risk Prioritization Matrix](#1-risk-prioritization-matrix)
2. [Tier 1 — Critical: Immediate Action Required](#2-tier-1--critical-immediate-action-required)
3. [Tier 2 — High: Remediate Within 1 Week](#3-tier-2--high-remediate-within-1-week)
4. [Hardening & Long-Term Strategy](#4-hardening--long-term-strategy)
5. [Compliance Mapping](#5-compliance-mapping)
6. [Verification Checklist](#6-verification-checklist)

---

## 1. Risk Prioritization Matrix

Vulnerabilities are ranked by CVSS score, exploitability, and presence of public exploits or confirmed backdoors.

| Priority | CVE(s) | Vulnerability | CVSS | Public Exploit | Backdoor |
|----------|--------|--------------|------|---------------|---------|
| 🔴 **P1** | CVE-2008-5304/5305 | TWiki XSS / Command Execution | 10.0 | ✅ Yes | ❌ |
| 🔴 **P1** | CVE-1999-0618 | rexec service running | 10.0 | ✅ Yes | ❌ |
| 🔴 **P1** | CVE-2011-2523 | vsftpd 2.3.4 Backdoor | 9.8 | ✅ Yes | ✅ **CONFIRMED** |
| 🔴 **P1** | CVE-2012-1823/2311/2336/2335 | PHP CGI RCE | 9.8 | ✅ Yes | ❌ |
| 🔴 **P1** | CVE-2020-1938 | Apache Tomcat Ghostcat AJP RCE | 9.8 | ✅ Yes | ❌ |
| 🔴 **P2** | CVE-2004-2687 | DistCC RCE | 9.3 | ✅ Yes | ❌ |
| 🟠 **P2** | CVE-2010-2075 | UnrealIRCd 3.2.8.1 Backdoor | 7.5 | ✅ Yes | ✅ **CONFIRMED** |
| 🟠 **P3** | CVE-2016-7144 | UnrealIRCd Auth Spoofing | 8.1 | ✅ Yes | ❌ |
| 🟠 **P3** | CVE-1999-0651 | rsh Cleartext Login | 7.5 | ✅ Yes | ❌ |
| 🟠 **P3** | CVE-2011-3556 | Java RMI Insecure Default RCE | 7.5 | ✅ Yes | ❌ |
| 🟠 **P3** | CVE-1999-0501 et al. | FTP Default Credentials | 7.5 | ✅ Yes | ❌ |
| 🟠 **P3** | CVE-1999-0651 | rlogin service running | 7.5 | ✅ Yes | ❌ |

> ⚠️ **2 confirmed supply-chain backdoors** (vsftpd 2.3.4, UnrealIRCd 3.2.8.1) — treat the host as **compromised** until fully audited.

---

## 2. Tier 1 — Critical: Immediate Action Required

> **SLA: Remediate within 24 hours.** These vulnerabilities have CVSS ≥ 9.3, active public exploits, and/or confirmed backdoors.

---

### 🔴 VULN-01 — vsftpd 2.3.4 Backdoor

**CVE:** `CVE-2011-2523` | **CVSS:** 9.8 (Critical) | **EPSS:** 94.26%

**Why it's critical:**
The vsftpd 2.3.4 archive was tampered with between June 30 and July 3, 2011. Sending a username containing `:)` triggers a root shell on port 6200/tcp — no authentication required. A Metasploit module (`exploit/unix/ftp/vsftpd_234_backdoor`) automates exploitation in seconds.

**Step-by-step mitigation:**

```bash
# 1. Stop the vsftpd service immediately
sudo systemctl stop vsftpd
sudo systemctl disable vsftpd

# 2. Verify the backdoored version
vsftpd --version   # should show 2.3.4

# 3. Remove the package
sudo apt-get remove --purge vsftpd   # Debian/Ubuntu
sudo yum remove vsftpd               # RHEL/CentOS

# 4. Block port 6200 at the firewall (in case backdoor was already triggered)
sudo iptables -A INPUT -p tcp --dport 6200 -j DROP
sudo iptables -A OUTPUT -p tcp --sport 6200 -j DROP

# 5. Check for active backdoor shells
netstat -tulnp | grep 6200
lsof -i :6200

# 6. Scan for indicators of compromise
sudo chkrootkit
sudo rkhunter --check
```

**Replacement options:**

| Option | Notes |
|--------|-------|
| vsftpd ≥ 3.0.5 | Latest clean release from verified mirror |
| OpenSSH SFTP | Preferred — encrypted, no separate install |
| ProFTPD | Feature-rich alternative FTP server |

> **Forensics note:** If port 6200 was open at any point, assume full root compromise. Preserve disk image before remediation and investigate for persistence mechanisms (cron jobs, SSH keys, new user accounts).

---

### 🔴 VULN-02 — UnrealIRCd 3.2.8.1 Backdoor

**CVE:** `CVE-2010-2075` | **CVSS:** 7.5 (High) | **Exploit:** Metasploit `exploit/unix/irc/unreal_ircd_3281_backdoor`

**Why it's critical:**
A trojan horse was embedded in the `DEBUG3_DOLOG_SYSTEM` macro of UnrealIRCd 3.2.8.1, distributed from November 2009 to June 2010. It allows unauthenticated remote command execution — any attacker can run arbitrary OS commands with daemon privileges.

**Step-by-step mitigation:**

```bash
# 1. Stop and remove UnrealIRCd immediately
sudo systemctl stop unrealircd
sudo systemctl disable unrealircd

# 2. Verify the compromised build (check MD5 hash)
md5sum Unreal3.2.8.1.tar.gz
# Compromised hash: 752e46f2d873c1679fa99de3f52a274d — DELETE if matched

# 3. Remove all UnrealIRCd files
sudo rm -rf /usr/local/unrealircd
sudo rm -rf /etc/unrealircd

# 4. Scan for backdoor artifacts
sudo rkhunter --check
sudo chkrootkit
sudo find / -name "*.so" -newer /bin/bash 2>/dev/null  # unusual shared libs

# 5. Install clean, verified version
# Download from https://www.unrealircd.org/ with PGP signature verification
gpg --verify unrealircd-6.x.x.tar.gz.asc unrealircd-6.x.x.tar.gz
```

**Recommended replacement:** UnrealIRCd 6.x (current stable, actively maintained).

---

### 🔴 VULN-03 — Apache Tomcat Ghostcat (AJP RCE)

**CVE:** `CVE-2020-1938` | **CVSS:** 9.8 (Critical) | **Affected:** Tomcat 6.x / 7.x / 8.x / 9.x

**Why it's critical:**
The AJP connector is enabled by default on port 8009 and bound to `0.0.0.0`. An unauthenticated attacker can read arbitrary files from the web application directory (including `WEB-INF/web.xml` containing credentials). If file upload is enabled, full RCE is achievable. Over 1 million internet-facing Tomcat servers were vulnerable at disclosure.

**Step-by-step mitigation:**

**Option A — Upgrade (preferred):**

```bash
# Upgrade to patched versions:
# Tomcat 9.x → 9.0.31+
# Tomcat 8.x → 8.5.51+
# Tomcat 7.x → 7.0.100+
```

**Option B — Disable AJP connector (if not used):**

```xml
<!-- In $CATALINA_HOME/conf/server.xml, comment out or remove: -->
<!-- <Connector port="8009" protocol="AJP/1.3" redirectPort="8443" /> -->
```

```bash
# Restart Tomcat after change
sudo systemctl restart tomcat
```

**Option C — Secure AJP with requiredSecret (if AJP is needed):**

```xml
<Connector port="8009" protocol="AJP/1.3" redirectPort="8443"
           address="127.0.0.1"
           requiredSecret="STRONG_RANDOM_SECRET_HERE" />
```

**Firewall rule (always apply regardless of option chosen):**

```bash
# Block external access to AJP port
sudo iptables -A INPUT -p tcp --dport 8009 ! -s 127.0.0.1 -j DROP
```

**Detection:** Monitor for unusual connections to port 8009 and access to `/WEB-INF/` directory in Tomcat access logs.

---

### 🔴 VULN-04 — TWiki XSS / Command Execution

**CVE:** `CVE-2008-5304`, `CVE-2008-5305` | **CVSS:** 10.0 (Critical)

**Why it's critical:**
TWiki < 4.2.4 allows both reflected XSS and arbitrary command execution. Maximum CVSS score indicates no privileges or interaction required for exploitation.

**Step-by-step mitigation:**

```bash
# 1. Check current TWiki version
cat /var/www/twiki/lib/TWiki.pm | grep VERSION

# 2. Upgrade TWiki to 4.2.4 or later (or migrate to Foswiki — the actively maintained fork)
# Download from https://foswiki.org/

# 3. If immediate upgrade is not possible, restrict access
sudo a2enmod headers
# Add to Apache config:
# Header always set Content-Security-Policy "default-src 'self'"

# 4. Apply input sanitization at WAF level
# Block requests containing shell metacharacters in TWiki CGI params
```

**Long-term:** Consider migrating to Foswiki (the community fork of TWiki), which is actively maintained.

---

### 🔴 VULN-05 — PHP CGI Argument Injection RCE

**CVEs:** `CVE-2012-1823`, `CVE-2012-2311`, `CVE-2012-2336`, `CVE-2012-2335` | **CVSS:** 9.8

**Why it's critical:**
PHP in CGI mode passes user-supplied query strings directly as arguments to the PHP interpreter. An attacker can inject flags like `-d allow_url_include=1 -d auto_prepend_file=php://input` to execute arbitrary code.

**Step-by-step mitigation:**

```bash
# 1. Upgrade PHP
sudo apt-get update && sudo apt-get upgrade php   # Debian/Ubuntu
# Target: PHP 5.3.13+, 5.4.3+, or preferably PHP 8.x

# 2. If upgrade is not possible immediately — disable PHP CGI mode
# In Apache, replace mod_cgi with mod_php or PHP-FPM
sudo a2dismod cgi
sudo a2enmod php8.x  # or configure php-fpm

# 3. Apply the official workaround (add to .htaccess or Apache config)
RewriteEngine on
RewriteCond %{QUERY_STRING} ^[^=]*$
RewriteCond %{QUERY_STRING} [^&=\%]{13} [OR]
RewriteCond %{QUERY_STRING} \-[dDeEerRsSz]
RewriteRule .* - [F]

# 4. Verify fix
curl "http://localhost/index.php?-s"  # Should return 403, not PHP source
```

---

### 🔴 VULN-06 — DistCC Remote Code Execution

**CVE:** `CVE-2004-2687` | **CVSS:** 9.3 (Critical)

**Why it's critical:**
DistCC is a distributed compiler daemon. When exposed to untrusted networks it allows any remote host to submit and execute compiler commands — effectively arbitrary command execution — with the privileges of the DistCC process (often root or a build user).

**Step-by-step mitigation:**

```bash
# 1. Stop DistCC
sudo systemctl stop distccd
sudo systemctl disable distccd

# 2. If DistCC is required for builds, restrict to trusted hosts only
# Edit /etc/default/distcc or /etc/distcc/hosts
ALLOWEDNETS="192.168.1.0/24"   # Replace with your trusted build network

# 3. Firewall rule to block external access
sudo iptables -A INPUT -p tcp --dport 3632 ! -s 192.168.1.0/24 -j DROP

# 4. Run distccd as a non-privileged user
sudo useradd -r -s /bin/false distcc-user
# Configure daemon to run as distcc-user in init/systemd unit
```

---

### 🔴 VULN-07 — rexec / rsh / rlogin — Cleartext Legacy Services

**CVEs:** `CVE-1999-0618` (rexec), `CVE-1999-0651` (rsh/rlogin) | **CVSS:** 10.0 / 7.5

**Why it's critical:**
The BSD r-services (rexec, rsh, rlogin) transmit credentials and session data in cleartext. They rely on IP-based trust via `.rhosts` and `hosts.equiv`, which is trivially spoofable. CIS Benchmarks mandate their removal.

**Step-by-step mitigation (CIS-aligned):**

```bash
# Disable via systemctl (modern systems)
sudo systemctl disable rsh.socket rlogin.socket rexec.socket
sudo systemctl stop rsh.socket rlogin.socket rexec.socket

# Disable via xinetd (older systems)
# Edit /etc/xinetd.d/rsh, rlogin, rexec — set: disable = yes
# Then restart xinetd:
sudo systemctl restart xinetd

# Disable via inetd (legacy)
# Comment out lines starting with shell, login, exec in /etc/inetd.conf
sudo systemctl restart inetd

# Remove trust files
sudo rm -f /etc/hosts.equiv
sudo find / -name ".rhosts" -delete 2>/dev/null

# Remove the server package entirely
sudo apt-get remove --purge rsh-server rsh-client   # Debian/Ubuntu
sudo yum remove rsh-server rsh                       # RHEL/CentOS

# Block ports at firewall
sudo iptables -A INPUT -p tcp --dport 512 -j DROP   # rexec
sudo iptables -A INPUT -p tcp --dport 513 -j DROP   # rlogin
sudo iptables -A INPUT -p tcp --dport 514 -j DROP   # rsh
```

**Replace with SSH:**

```bash
sudo apt-get install openssh-server
sudo systemctl enable --now sshd
# Harden SSH config in /etc/ssh/sshd_config:
# PermitRootLogin no
# PasswordAuthentication no  (use key-based auth)
# Protocol 2
```

---

## 3. Tier 2 — High: Remediate Within 1 Week

---

### 🟠 VULN-08 — UnrealIRCd Authentication Spoofing

**CVE:** `CVE-2016-7144` | **CVSS:** 8.1 (High)

**Mitigation:**

```bash
# Upgrade UnrealIRCd to 6.x (current stable)
# https://www.unrealircd.org/downloads

# If IRC is not business-critical, disable the service
sudo systemctl stop unrealircd && sudo systemctl disable unrealircd

# Restrict IRC ports via firewall if service must remain
sudo iptables -A INPUT -p tcp --dport 6667 -s TRUSTED_IP_RANGE -j ACCEPT
sudo iptables -A INPUT -p tcp --dport 6667 -j DROP
sudo iptables -A INPUT -p tcp --dport 6697 -s TRUSTED_IP_RANGE -j ACCEPT
sudo iptables -A INPUT -p tcp --dport 6697 -j DROP
```

**Recommendation:** Assess whether an IRC server is genuinely needed. If not, permanently decommission the service.

---

### 🟠 VULN-09 — Java RMI Insecure Default Configuration RCE

**CVE:** `CVE-2011-3556` | **CVSS:** 7.5 (High)

**Mitigation:**

```bash
# 1. Identify the RMI registry and servers
netstat -tulnp | grep -E '1099|1098'

# 2. If RMI is not required, disable it
# Remove or comment out RMI references from application startup scripts

# 3. Apply Java Security Manager (if RMI must remain)
# Add to JVM startup args:
-Djava.security.manager -Djava.security.policy=/path/to/strict.policy

# 4. Firewall: block RMI default port from external access
sudo iptables -A INPUT -p tcp --dport 1099 ! -s 127.0.0.1 -j DROP

# 5. Upgrade to Java 17+ (LTS) which has improved RMI security defaults
sudo apt-get install openjdk-17-jdk
```

---

### 🟠 VULN-10 — FTP Brute Force / Default Credentials

**CVEs:** `CVE-1999-0501`, `CVE-1999-0502` et al. (16 CVEs) | **CVSS:** 7.5 (High)

**Mitigation:**

```bash
# 1. Change all FTP credentials immediately
sudo passwd ftpuser  # Change FTP user password

# 2. Implement fail2ban to block brute force
sudo apt-get install fail2ban
# Add to /etc/fail2ban/jail.local:
# [vsftpd]
# enabled = true
# port = ftp
# filter = vsftpd
# maxretry = 3
# bantime = 3600

# 3. Restrict FTP to known IPs
# In /etc/vsftpd.conf or /etc/hosts.allow:
echo "vsftpd: 192.168.1.0/24" >> /etc/hosts.allow
echo "vsftpd: ALL" >> /etc/hosts.deny

# 4. Preferred: migrate to SFTP (SSH File Transfer Protocol)
# Enable SFTP subsystem in /etc/ssh/sshd_config:
# Subsystem sftp /usr/lib/openssh/sftp-server

# 5. Disable anonymous FTP login
# In /etc/vsftpd.conf:
# anonymous_enable=NO
```

**Preferred long-term:** Decommission FTP entirely and migrate file transfers to SFTP or FTPS with certificate pinning.

---

## 4. Hardening & Long-Term Strategy

### 4.1 Network Segmentation

```
Internet
    │
    ▼
[Edge Firewall] ──── Block: 512, 513, 514, 6200, 8009, 1099, 3632
    │
    ▼
[DMZ] ── Only expose ports: 22 (SSH), 80, 443
    │
    ▼
[Internal Network] ── Application servers, databases
```

- Place all public-facing services in a DMZ, separated from internal systems by a stateful firewall.
- Use allowlist-only firewall rules (default-deny).
- Deploy an IDS/IPS (e.g., Snort, Suricata) with rules for Ghostcat, vsftpd backdoor, and DistCC RCE.

### 4.2 Patch Management

| Frequency | Action |
|-----------|--------|
| Daily | Review CISA Known Exploited Vulnerabilities (KEV) catalog |
| Weekly | Apply OS and package security updates |
| Monthly | Run OpenVAS / Nessus full scan |
| Quarterly | Third-party penetration test |

```bash
# Automate security updates (Ubuntu/Debian)
sudo apt-get install unattended-upgrades
sudo dpkg-reconfigure unattended-upgrades
```

### 4.3 Principle of Least Privilege

- All services should run as dedicated non-root users.
- Remove SUID/SGID bits from unnecessary binaries: `find / -perm /6000 -type f 2>/dev/null`
- Audit sudoers: `sudo visudo` — remove wildcard entries.
- Use AppArmor or SELinux profiles to confine service processes.

### 4.4 Logging & Monitoring

```bash
# Enable centralized logging
sudo apt-get install auditd
sudo systemctl enable --now auditd

# Key events to audit
auditctl -w /etc/passwd -p wa -k passwd_changes
auditctl -w /etc/shadow -p wa -k shadow_changes
auditctl -w /var/log/auth.log -p wa -k auth_log

# Monitor for backdoor shell on port 6200 (ongoing)
sudo iptables -A INPUT -p tcp --dport 6200 -j LOG --log-prefix "BACKDOOR_ATTEMPT: "
```

### 4.5 Incident Response Checklist

If compromise is suspected (given the two confirmed backdoors):

- [ ] Isolate the host from the network immediately
- [ ] Preserve a forensic disk image before any changes
- [ ] Collect running process list: `ps aux`, `netstat -tulnp`
- [ ] Check for new user accounts: `cat /etc/passwd | grep -v nologin`
- [ ] Review recently modified files: `find / -mtime -7 -type f 2>/dev/null`
- [ ] Check cron jobs: `crontab -l && cat /etc/cron*`
- [ ] Check SSH authorized_keys for unauthorized entries
- [ ] Review `/var/log/auth.log` and `/var/log/syslog`
- [ ] Reset all credentials system-wide after cleaning

---

## 5. Compliance Mapping

| Vulnerability Class | NIST 800-53 | CIS Benchmark | OWASP |
|--------------------|------------|---------------|-------|
| Backdoors (vsftpd, UnrealIRCd) | SI-3, IR-4 | CIS Control 10 | A08:2021 |
| Legacy cleartext services (rexec/rsh/rlogin) | IA-5, SC-8 | CIS Control 4, CIS Linux 2.1.3 | A02:2021 |
| Unpatched software (PHP, Tomcat, TWiki) | SI-2, RA-5 | CIS Control 7 | A06:2021 |
| Default credentials (FTP) | IA-5(1) | CIS Control 5 | A07:2021 |
| Exposed services (DistCC, Java RMI) | CM-7, AC-17 | CIS Control 4 | A05:2021 |
| Missing encryption (rsh, FTP) | SC-8, SC-28 | CIS Control 3 | A02:2021 |

---

## 6. Verification Checklist

Run these checks after remediation to confirm each vulnerability is resolved.

```bash
# ── BACKDOORS ─────────────────────────────────────────────
# Confirm vsftpd backdoor removed
nc -zv localhost 6200          # Should FAIL (connection refused)
vsftpd --version 2>&1          # Should NOT return 2.3.4

# Confirm UnrealIRCd backdoor removed
pgrep -x unrealircd            # Should return nothing
nc -zv localhost 6667          # Should FAIL

# ── LEGACY SERVICES ───────────────────────────────────────
systemctl is-active rsh.socket rlogin.socket rexec.socket
# All should return: inactive

nc -zv localhost 512           # rexec — should FAIL
nc -zv localhost 513           # rlogin — should FAIL
nc -zv localhost 514           # rsh — should FAIL

# ── TOMCAT AJP ────────────────────────────────────────────
nc -zv localhost 8009          # Should FAIL if AJP disabled

# ── PHP CGI ───────────────────────────────────────────────
curl -s "http://localhost/index.php?-s" | head -5
# Should return 403 Forbidden, not PHP source code

# ── DISTCC ────────────────────────────────────────────────
nc -zv localhost 3632          # Should FAIL

# ── FTP CREDENTIALS ───────────────────────────────────────
ftp localhost
# Attempt login with known default credentials — should FAIL

# ── JAVA RMI ──────────────────────────────────────────────
nc -zv localhost 1099          # Should FAIL if disabled

# ── FULL RESCAN ───────────────────────────────────────────
# Run a new OpenVAS scan and verify 0 Critical findings remain
```

---

## Summary

| Action | Owner | Deadline | Status |
|--------|-------|----------|--------|
| Remove vsftpd 2.3.4 backdoor | Sysadmin | **NOW** | ☐ |
| Remove UnrealIRCd 3.2.8.1 backdoor | Sysadmin | **NOW** | ☐ |
| Disable rexec / rsh / rlogin | Sysadmin | **NOW** | ☐ |
| Patch/disable Tomcat AJP (Ghostcat) | Sysadmin | **NOW** | ☐ |
| Upgrade PHP (CGI RCE) | Dev/Ops | 24 hours | ☐ |
| Restrict DistCC to trusted hosts | Sysadmin | 24 hours | ☐ |
| Upgrade TWiki to 4.2.4+ | Dev/Ops | 48 hours | ☐ |
| Upgrade UnrealIRCd to 6.x | Sysadmin | 1 week | ☐ |
| Migrate FTP → SFTP | Dev/Ops | 1 week | ☐ |
| Disable / firewall Java RMI | Sysadmin | 1 week | ☐ |
| Deploy fail2ban & IDS | Security | 1 week | ☐ |
| Full OpenVAS rescan post-remediation | Security | After above | ☐ |
| Conduct forensic audit (backdoor hosts) | Security | Immediate | ☐ |

---

*Report compiled from OpenVAS findings + CVE research | May 15, 2026*
*Sources: NVD, CISA KEV, Tenable, CIS Benchmarks, Black Duck, SentinelOne*
*Classification: Confidential — Authorized personnel only*
