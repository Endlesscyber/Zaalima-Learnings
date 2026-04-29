# Installing Kali Linux & Vulnerable Machines — Beginner's Guide

> Set up a complete ethical hacking lab with Kali Linux, Metasploitable 2, and OWASP Juice Shop inside a virtual environment. No prior experience needed.

---

## ⚠️ Before You Start

- Make sure **VMware** or **VirtualBox** is already installed on your computer.
- All vulnerable VMs must run on a **Host-Only network** — never expose them to the internet.
- This lab is for **educational and ethical purposes only**.

---

## Part 1 — Kali Linux (Attacker Machine)

Kali Linux is the main machine you will use to practice security testing. It comes with hundreds of pre-installed tools.

### Step 1: Download Kali Linux

- Go to **kali.org/get-kali** and click **Virtual Machines**
 <img src="https://github.com/Endlesscyber/Zaalima-Learnings/blob/03aca9d05fad6fc02f9cdcbc0dd1f26f4a6290a2/Project%201/Images/kali-download.png" >

- Download the **VMware** or **VirtualBox** version (whichever you have installed)
<img src="https://github.com/Endlesscyber/Zaalima-Learnings/blob/03aca9d05fad6fc02f9cdcbc0dd1f26f4a6290a2/Project%201/Images/open%20iso.png">

- Extract the downloaded `.7z` file using 7-Zip or a similar tool

<img src="https://github.com/Endlesscyber/Zaalima-Learnings/blob/03aca9d05fad6fc02f9cdcbc0dd1f26f4a6290a2/Project%201/Images/Open%20kali.png">
> 💡 **Tip:** The pre-built VM saves you from doing a full OS installation — it's ready to boot straight away.

---

### Step 2: Import into your hypervisor

**VMware:**
- Go to **File → Open** and select the `.vmx` file

**VirtualBox:**
- Go to **File → Import Appliance** and select the `.ova` file
- Click **Import** and wait for it to finish

---

### Step 3: Boot and log in

- Click **Power On** (VMware) or **Start** (VirtualBox)
- Log in with the default credentials:
  - **Username:** `kali`
  - **Password:** `kali`
- Open a terminal and update the system:

```bash
sudo apt update && sudo apt upgrade -y
```

> 💡 **Tip:** Change the default password immediately by running: `passwd`

---

## Part 2 — Metasploitable 2 (Vulnerable Linux Target)

Metasploitable 2 is an intentionally vulnerable Linux machine — great for practising network and service-based attacks.

### Step 1: Download Metasploitable 2

- Search for **"Metasploitable 2 download SourceForge"** and go to the official page
- Click **Download** — you'll receive a `.zip` file
- Extract it — inside you'll find a folder containing a `.vmdk` disk file
<img src="https://github.com/Endlesscyber/Zaalima-Learnings/blob/358917808c823da424a3984043859bbe053ffdcb/Project%201/Images/Screenshot%202026-04-29%20141440.png"> 
---

### Step 2: Create a new VM using the disk file

- Create a new virtual machine and select **Linux / Ubuntu (32-bit)** as the OS type
- When asked about storage, choose **"Use an existing virtual disk"**
- Select the `.vmdk` file you just extracted
- Allocate **512 MB – 1 GB** of RAM

---

### Step 3: Set network to Host-Only and boot

- Go to the VM **Settings → Network**
- Set the adapter to **Host-Only**
- Power on the VM and log in:
  - **Username:** `msfadmin`
  - **Password:** `msfadmin`
- From Kali, run the following to confirm it's reachable:

```bash
nmap -sV <metasploitable-ip>
```

You should see several open ports (FTP, SSH, HTTP, etc.).

> ⚠️ **Warning:** Never set Metasploitable's network adapter to **Bridged** — this would expose it to your real network.

---

## Part 3 — OWASP Juice Shop (Vulnerable Web App)

OWASP Juice Shop is a modern, deliberately vulnerable web application for practising web security attacks like SQL injection, XSS, and more.

### Step 1: Install Docker on Kali

Open a terminal in Kali and run:

```bash
sudo apt install docker.io -y
sudo systemctl start docker
```

---

### Step 2: Pull and run Juice Shop

```bash
sudo docker pull bkimminich/juice-shop
sudo docker run -d -p 3000:3000 bkimminich/juice-shop
```

Then open the Kali browser and visit:

```
http://localhost:3000
```
<img src="https://github.com/Endlesscyber/Zaalima-Learnings/blob/734624fe60544c8c45459abde373296e67168591/Project%201/Images/Screenshot%202026-04-29%20142248.png">

> 💡 **Tip:** If the page loads, Juice Shop is running. Your first challenge — find the hidden **Score Board**!

---

### Step 3: Verify everything is working

Do a quick check to confirm all three systems are up:

| System | How to verify |
|---|---|
| Kali Linux | Desktop loads, terminal works |
| Metasploitable 2 | Kali can ping it; `nmap` shows open ports |
| OWASP Juice Shop | `http://localhost:3000` loads in the Kali browser |

---

## ✅ Your Lab is Ready!

You now have a complete, isolated ethical hacking lab:

- **Kali Linux** — your main attack machine with all the tools you need
- **Metasploitable 2** — a vulnerable target for practising network exploits
- **OWASP Juice Shop** — a vulnerable web app for practising web security challenges

Everything runs safely inside virtual machines — nothing you do will affect your real computer.

---

## 📚 What to Try Next

- Use **Metasploit** on Kali to scan and exploit Metasploitable 2
- Explore Juice Shop challenges at `http://localhost:3000` (find the Score Board first!)
- Learn more at:
  - [Kali Linux Docs](https://www.kali.org/docs/)
  - [Metasploit Unleashed (free)](https://www.metasploitunleashed.com/)
  - [Juice Shop Companion Guide](https://pwning.owasp-juice.shop/)

---

## 🔒 Legal & Ethical Reminder

> Only practice on systems **you own** or have **explicit written permission** to test. Unauthorized hacking is illegal under laws such as the Computer Fraud and Abuse Act (CFAA) and equivalents worldwide.

---

*Happy ethical hacking!
