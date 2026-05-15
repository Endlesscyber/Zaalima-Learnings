# 🔐 Vulnerability Assessment Report

**Tool:** OpenVAS | **Target:** 127.0.0.1 (Metasploitable) | **Date:** May 15, 2026 | **Scanned by:** admin

---
<img src="https://github.com/Endlesscyber/Zaalima-Learnings/blob/66c307dd7b162fbf7a64b73502a58a8a3258c1a7/Project%201/Images/vul%201.png">
<img src="https://github.com/Endlesscyber/Zaalima-Learnings/blob/66c307dd7b162fbf7a64b73502a58a8a3258c1a7/Project%201/Images/vul%202.png">
## 📊 Summary

| Severity | Count | Occurrences |
|----------|-------|-------------|
| 🔴 Critical | 6 | 8 |
| 🟠 High | 6 | 8 |
| **Total** | **12** | **16** |

> ⚠️ **2 confirmed backdoors** detected — immediate action required.

---

## 🔴 Critical Findings

---

### 1. TWiki < 4.2.4 Multiple XSS / Command Execution Vulnerabilities

| Field | Details |
|-------|---------|
| **CVE(s)** | `CVE-2008-5304` `CVE-2008-5305` |
| **CVSS Score** | 10.0 (Critical) |
| **Hosts Affected** | 1 |
| **Occurrences** | 1 |

**Description:**
TWiki versions prior to 4.2.4 contain multiple cross-site scripting (XSS) vulnerabilities and command execution flaws. An attacker can inject malicious scripts or execute arbitrary commands on the server.

**Recommendation:**
> Upgrade TWiki to version 4.2.4 or later. Sanitize all user inputs and apply content security policies.

---

### 2. The rexec Service Is Running

| Field | Details |
|-------|---------|
| **CVE(s)** | `CVE-1999-0618` |
| **CVSS Score** | 10.0 (Critical) |
| **Hosts Affected** | 1 |
| **Occurrences** | 1 |

**Description:**
The rexec (remote execution) service is running, which allows remote command execution without strong authentication. This service transmits credentials in plaintext and poses a critical security risk.

**Recommendation:**
> Disable the rexec service immediately. Use SSH for remote command execution. Block port 512/tcp at the firewall.

---

### 3. PHP < 5.3.13 / 5.4.x < 5.4.3 Multiple Vulnerabilities

| Field | Details |
|-------|---------|
| **CVE(s)** | `CVE-2012-1823` `CVE-2012-2311` `CVE-2012-2336` `CVE-2012-2335` |
| **CVSS Score** | 9.8 (Critical) |
| **Hosts Affected** | 1 |
| **Occurrences** | 1 |

**Description:**
PHP versions prior to 5.3.13 and 5.4.x prior to 5.4.3 are vulnerable to multiple critical flaws including remote code execution via CGI argument injection. These vulnerabilities allow a remote attacker to execute arbitrary code.

**Recommendation:**
> Upgrade PHP to version 5.3.13+ or 5.4.3+. Disable PHP CGI or apply vendor patches. Configure PHP securely per hardening guidelines.

---

### 4. Apache Tomcat AJP RCE Vulnerability — Ghostcat

| Field | Details |
|-------|---------|
| **CVE(s)** | `CVE-2020-1938` |
| **CVSS Score** | 9.8 (Critical) |
| **Hosts Affected** | 1 |
| **Occurrences** | 1 |

**Description:**
The Apache Tomcat Ghostcat vulnerability allows unauthenticated attackers to read files from the server and potentially execute code via the AJP connector. This is a critical flaw in the Apache JServ Protocol (AJP) on port 8009.

**Recommendation:**
> Upgrade Apache Tomcat to a patched version. Disable or restrict access to the AJP connector (default port 8009). Apply network-level controls to limit AJP exposure.

---

### 5. ⚠️ vsftpd 2.3.4 Compromised Source — Backdoor

| Field | Details |
|-------|---------|
| **CVE(s)** | `CVE-2011-2523` |
| **CVSS Score** | 9.8 (Critical) |
| **Hosts Affected** | 1 |
| **Occurrences** | 2 |

**Description:**
A backdoor was introduced in the vsftpd 2.3.4 source package. When a username containing `:)` is provided during FTP authentication, a backdoor shell is opened on TCP port 6200, granting **remote root access**.

**Recommendation:**
> Remove the compromised vsftpd 2.3.4 package immediately. Upgrade to a clean, verified version of vsftpd. Audit any system that ran this version for signs of compromise.

---

### 6. DistCC Remote Code Execution

| Field | Details |
|-------|---------|
| **CVE(s)** | `CVE-2004-2687` |
| **CVSS Score** | 9.3 (Critical) |
| **Hosts Affected** | 1 |
| **Occurrences** | 1 |

**Description:**
DistCC, a distributed C/C++ compiler, is exposed to the network and allows arbitrary command execution. An attacker can exploit this to run arbitrary code with the privileges of the DistCC daemon.

**Recommendation:**
> Restrict DistCC access to trusted hosts only using firewall rules or the `--allow` option. Disable DistCC if not required. Run the service in a restricted environment.

---

## 🟠 High Findings

---

### 7. UnrealIRCd Authentication Spoofing Vulnerability

| Field | Details |
|-------|---------|
| **CVE(s)** | `CVE-2016-7144` |
| **CVSS Score** | 8.1 (High) |
| **Hosts Affected** | 1 |
| **Occurrences** | 1 |

**Description:**
UnrealIRCd contains an authentication spoofing vulnerability that allows a remote attacker to bypass authentication mechanisms and gain unauthorized access to the IRC server.

**Recommendation:**
> Upgrade UnrealIRCd to the latest patched version. Review and enforce authentication configurations. Monitor IRC server logs for suspicious activity.

---

### 8. rsh — Unencrypted Cleartext Login

| Field | Details |
|-------|---------|
| **CVE(s)** | `CVE-1999-0651` |
| **CVSS Score** | 7.5 (High) |
| **Hosts Affected** | 1 |
| **Occurrences** | 1 |

**Description:**
The rsh (remote shell) service is running and transmits authentication credentials and session data in cleartext. This exposes the system to credential interception and man-in-the-middle attacks.

**Recommendation:**
> Disable rsh service immediately. Migrate to SSH for all remote shell access. Block rsh ports (514/tcp) at the network perimeter.

---

### 9. FTP Brute Force Logins With Default Credentials

| Field | Details |
|-------|---------|
| **CVE(s)** | `CVE-1999-0501` `CVE-1999-0502` `CVE-1999-0507` `CVE-1999-0508` `CVE-2001-1594` `CVE-2013-7404` `CVE-2014-9198` `CVE-2015-7261` `CVE-2016-8731` `CVE-2017-8218` `CVE-2018-9068` `CVE-2018-17771` `CVE-2018-19063` `CVE-2018-19064` `CVE-2018-25147` `CVE-2020-36915` |
| **CVSS Score** | 7.5 (High) |
| **Hosts Affected** | 1 |
| **Occurrences** | 2 |

**Description:**
The FTP service allows login with default or weak credentials. Multiple CVEs are associated with various FTP server implementations accepting guessable usernames and passwords, exposing the server to brute force and credential stuffing attacks.

**Recommendation:**
> Change all default FTP credentials immediately. Implement account lockout policies. Consider replacing FTP with SFTP or FTPS. Restrict FTP access to known IP addresses.

---

### 10. ⚠️ UnrealIRCd 3.2.8.1 — Backdoor

| Field | Details |
|-------|---------|
| **CVE(s)** | `CVE-2010-2075` |
| **CVSS Score** | 7.5 (High) |
| **Hosts Affected** | 1 |
| **Occurrences** | 1 |

**Description:**
A backdoor was discovered in UnrealIRCd 3.2.8.1 distributed between November 2009 and June 2010. The backdoor allows a remote attacker to execute arbitrary commands with the privileges of the UnrealIRCd process.

**Recommendation:**
> Immediately remove and replace the compromised version of UnrealIRCd. Upgrade to a verified clean version. Conduct a full system audit for indicators of compromise.

---

### 11. Java RMI Server Insecure Default Configuration RCE

| Field | Details |
|-------|---------|
| **CVE(s)** | `CVE-2011-3556` |
| **CVSS Score** | 7.5 (High) |
| **Hosts Affected** | 1 |
| **Occurrences** | 1 |

**Description:**
The Java RMI server is running with an insecure default configuration that allows unauthenticated remote code execution. Attackers can exploit this to execute arbitrary code on the target system.

**Recommendation:**
> Restrict Java RMI access using firewall rules. Apply Java security manager policies. Upgrade to a secure Java version. Disable RMI if not in use.

---

### 12. The rlogin Service Is Running

| Field | Details |
|-------|---------|
| **CVE(s)** | `CVE-1999-0651` |
| **CVSS Score** | 7.5 (High) |
| **Hosts Affected** | 1 |
| **Occurrences** | 1 |

**Description:**
The rlogin (remote login) service is active. This legacy service sends authentication data in cleartext and relies on IP-based trust relationships, making it susceptible to spoofing and interception.

**Recommendation:**
> Disable the rlogin service. Replace with SSH-based remote access. Block port 513/tcp at the firewall.

---

## 🛠️ Prioritized Remediation Plan

### 🔴 Immediate — Within 24 Hours

- [ ] Remove **vsftpd 2.3.4** (confirmed backdoor — opens root shell on port 6200)
- [ ] Remove **UnrealIRCd 3.2.8.1** (confirmed backdoor — arbitrary command execution)
- [ ] Disable **rexec**, **rsh**, and **rlogin** — replace all with SSH
- [ ] Upgrade **PHP** to eliminate CGI remote code execution
- [ ] Disable or restrict **Apache Tomcat AJP connector** (Ghostcat)

### 🟠 Short-Term — Within 1 Week

- [ ] Upgrade **UnrealIRCd** to a verified clean version
- [ ] Replace **FTP** with SFTP/FTPS and enforce strong credential policies
- [ ] Disable **Java RMI** or apply strict firewall rules and security manager policies
- [ ] Restrict **DistCC** to trusted compile hosts only
- [ ] Upgrade **TWiki** to 4.2.4+

### 🟡 Ongoing Hardening

- [ ] Implement a regular patch management cycle
- [ ] Schedule periodic vulnerability scans with OpenVAS
- [ ] Apply the principle of least privilege across all services
- [ ] Perform network segmentation to reduce the attack surface
- [ ] Enable logging and alerting for all critical services

---

## 📋 CVE Reference Index

| CVE | Vulnerability | Score |
|-----|--------------|-------|
| CVE-2008-5304 | TWiki XSS / Command Execution | 10.0 |
| CVE-2008-5305 | TWiki XSS / Command Execution | 10.0 |
| CVE-1999-0618 | rexec service running | 10.0 |
| CVE-2012-1823 | PHP CGI argument injection | 9.8 |
| CVE-2012-2311 | PHP multiple vulnerabilities | 9.8 |
| CVE-2012-2336 | PHP multiple vulnerabilities | 9.8 |
| CVE-2012-2335 | PHP multiple vulnerabilities | 9.8 |
| CVE-2020-1938 | Apache Tomcat Ghostcat AJP RCE | 9.8 |
| CVE-2011-2523 | vsftpd 2.3.4 backdoor | 9.8 |
| CVE-2004-2687 | DistCC RCE | 9.3 |
| CVE-2016-7144 | UnrealIRCd auth spoofing | 8.1 |
| CVE-1999-0651 | rsh/rlogin cleartext login | 7.5 |
| CVE-2010-2075 | UnrealIRCd 3.2.8.1 backdoor | 7.5 |
| CVE-2011-3556 | Java RMI insecure default RCE | 7.5 |
| CVE-1999-0501 | FTP default credentials | 7.5 |
| CVE-2020-36915 | FTP default credentials | 7.5 |

---

*Report generated from OpenVAS scan · Kali Linux · May 15, 2026*
*Classification: Confidential — For authorized personnel only*
