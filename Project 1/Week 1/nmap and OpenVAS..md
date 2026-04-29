# Network Scanning with Nmap & OpenVAS — Beginner's Guide

> Learn how to scan networks for open ports and security vulnerabilities using two essential tools: Nmap for quick scanning and OpenVAS for deeper vulnerability checks.

---

##  Important Reminder

> Only scan systems **you own** or have **explicit permission** to test. Scanning other people's networks without permission is **illegal**.

---

## Part 1 — Nmap (Network Mapper)

Nmap helps you discover what devices and services are running on a network by scanning for open ports.

---

### Step 1: Install Nmap

Nmap is pre-installed on Kali Linux. If you need to install it manually:

**Kali / Ubuntu:**
```bash
sudo apt install nmap -y
```

**Windows:**
- Download the installer from **nmap.org** and follow the setup wizard.

---

### Step 2: Run Your First Scan

Replace the IP address with your target machine's IP (use your lab VM like Metasploitable).

```bash
# Basic scan — find open ports
nmap 192.168.32.134

# Service scan — shows what software is running on each port
nmap -sV 192.168.32.134

# Full scan — OS detection + services + scripts
nmap -A 192.168.32.134
```

>  **Tip:** Use your Metasploitable VM's IP as the target — it's designed for safe practice.

---

### Step 3: Common Nmap Commands to Know

| Command | What It Does |
|---|---|
| `nmap -sn 192.168.32.0/24` | Ping scan — finds all live devices on the network |
| `nmap -p 22,80,443 192.168.32.134` | Scan only specific ports |
| `nmap -O 192.168.32.134` | Try to detect the operating system |
| `nmap -sV 192.168.32.134` | Detect service versions on open ports |
| `nmap -A 192.168.32.134` | Full aggressive scan (OS + version + scripts) |
| `nmap -oN results.txt 192.168.32.134` | Save results to a text file |

---
<img src="https://github.com/Endlesscyber/Zaalima-Learnings/blob/f242ff122a62cc3d234ae80c7413f9eaecfa4fec/Project%201/Images/nmap%20aggresice%20scan.png">

### Step 4: Reading Scan Results

After a scan, you'll see output like this:

```
PORT     STATE    SERVICE    VERSION
22/tcp   open     ssh        OpenSSH 4.7
80/tcp   open     http       Apache 2.2.8
3306/tcp open     mysql      MySQL 5.0.51
8180/tcp filtered http
```
<img src="https://github.com/Endlesscyber/Zaalima-Learnings/blob/f242ff122a62cc3d234ae80c7413f9eaecfa4fec/Project%201/Images/nmap%20service%20and%20version.png">


Here's what the terms mean:

- **open** — the port is reachable and a service is listening
- **closed** — the port is reachable but nothing is running on it
- **filtered** — a firewall is blocking the port
- **Service** — the name of the software running on that port
- **Version** — the exact version of that software

>  **Tip:** Open ports are potential entry points. More open ports = a larger attack surface.

---

## Part 2 — OpenVAS (Vulnerability Scanner)

OpenVAS goes deeper than Nmap. It checks your target against thousands of known vulnerabilities and security weaknesses.

---

### Step 1: Install OpenVAS on Kali

```bash
# Install OpenVAS (also called GVM)
sudo apt install openvas -y

# Run the setup (downloads vulnerability definitions — takes 10–30 mins)
sudo gvm-setup

# Start the service
sudo gvm-start
```

>  **Tip:** During setup, your admin password will be shown in the terminal — write it down!

---

### Step 2: Access the Web Dashboard

OpenVAS runs as a web application. Open it in your browser:

- Go to: **https://127.0.0.1:9392**
- Accept the browser security warning (it uses a self-signed certificate — this is normal)
- Log in with username `admin` and the password shown during setup

---
<img src="https://github.com/Endlesscyber/Zaalima-Learnings/blob/f865e1a4724f7c67e4f5635685ac2fb6ea612765/Project%201/Images/openvas%20(2).png">

### Step 3: Run Your First Vulnerability Scan

- Click **Scans → Tasks → New Task**
- Give the task a name (e.g., `Metasploitable scan`)
- Under **Scan Targets**, enter your target machine's IP address
- Leave all other settings as default and click **Save**
- Click the **play button** next to the task to start the scan

> **Tip:** A full scan can take 20–60 minutes depending on the number of services found.

---

### Step 4: Reading the Scan Results

When the scan finishes, click on it to open the report. Vulnerabilities are colour-coded by severity:

<img src="https://github.com/Endlesscyber/Zaalima-Learnings/blob/f865e1a4724f7c67e4f5635685ac2fb6ea612765/Project%201/Images/metasploit%20scan.png">

| Severity | Colour | Meaning |
|---|---|---|
| Critical / High | Red | Serious risk — fix immediately |
| Medium | Orange | Moderate risk — should be addressed |
| Low | Yellow/Green | Minor issue — low priority |
| Info | Grey | No risk — informational only |

- Click any vulnerability to see a **description**, **CVSS score** (a number showing how serious it is), and a **suggested fix**.
- A high CVSS score (7–10) means the vulnerability is serious.

<img src="https://github.com/Endlesscyber/Zaalima-Learnings/blob/f865e1a4724f7c67e4f5635685ac2fb6ea612765/Project%201/Images/cvss.png">
---

## Quick Reference — Nmap vs OpenVAS

| Feature | Nmap | OpenVAS |
|---|---|---|
| Speed | Very fast (seconds) | Slow (20–60 minutes) |
| Purpose | Port & service discovery | Vulnerability detection |
| Output | Terminal / text file | Web-based visual report |
| Skill level | Beginner-friendly | Slightly more complex |
| Best for | Quick network mapping | Deep security assessment |

---

## You're Ready to Scan

Use **Nmap** for fast, lightweight port scanning and **OpenVAS** for thorough vulnerability assessments. Together they give you a complete picture of what's running and what's at risk on your lab network.

---

## Next Steps

- Practice scanning your **Metasploitable 2** VM with both tools
- Look up the CVE numbers from OpenVAS results on **cve.mitre.org**
- Try using Metasploit to exploit vulnerabilities that Nmap and OpenVAS discover
- Learn more at:
  - [Nmap Official Docs](https://nmap.org/docs.html)
  - [Greenbone / OpenVAS Docs](https://docs.greenbone.net/)
  - [Nmap Cheat Sheet — StationX](https://www.stationx.net/nmap-cheat-sheet/)

---

##  Legal & Ethical Reminder

> Always practise on systems you own or have written permission to test — such as your own virtual lab. Unauthorised scanning is illegal under laws like the **Computer Fraud and Abuse Act (CFAA)** and equivalents worldwide.

---

*Scan smart. Stay ethical. *
