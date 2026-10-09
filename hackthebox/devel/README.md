# Penetration Testing Report — HTB "Devel" Machine

**Target IP:** 10.129.18.40
**Attacker IP:** 10.10.14.7
**Platform:** Hack The Box (Starting Point / Machines)
**Target OS:** Microsoft Windows 7 Enterprise, Build 7600 (no hotfixes installed)
**Report Date:** August 3, 2026

---

## 1. Executive Summary

This report documents the full compromise of the Hack The Box machine **Devel**, from initial reconnaissance through remote code execution to full **SYSTEM**-level privilege escalation. Both the user flag (`babis` desktop) and the root/SYSTEM flag were successfully retrieved.

The engagement relied on two core weaknesses:

1. **Anonymous FTP access** enabled on the target, with the FTP root directory mapped directly to the IIS web root — allowing arbitrary file upload that could be executed by the web server.
2. **An unpatched, legacy Windows 7 (Build 7600) operating system**, which was vulnerable to numerous well-known local privilege escalation exploits.

Several practical obstacles were encountered during exploitation — mostly related to payload architecture mismatches, listener/port conflicts, and unstable Meterpreter sessions — all of which were diagnosed and resolved during the engagement. These are documented in detail in Section 4.

---

## 2. Methodology / Attack Chain

### 2.1 Initial Access — Anonymous FTP
- Connected to the target's FTP service (`ftp 10.129.18.40`) using anonymous credentials.
- FTP login succeeded with `anonymous` / blank password, confirming **anonymous access was enabled**.
- Directory listing (`ls`) revealed IIS default files (`iisstart.htm`, `welcome.png`, `aspnet_client`), indicating the FTP root was the same directory as the **IIS web root**.

### 2.2 Foothold — Web Shell Upload
- An initial PHP reverse shell (`shell.php`) was uploaded via FTP but could not be executed by the web server, since IIS on this host does not have a PHP handler configured — only ASP/ASP.NET (`.aspx`) is supported.
- A proper ASP.NET reverse shell payload was generated using `msfvenom` in `.aspx` format and uploaded via FTP in **binary transfer mode**.
- A Metasploit `multi/handler` listener was configured to match the payload, and the `.aspx` file was accessed via the browser (`http://10.129.18.40/shell.aspx`), triggering the payload and returning an initial Meterpreter session.
- Initial session context: **`IIS APPPOOL\Web`**.

### 2.3 Privilege Discovery
- `getprivs` on the initial session confirmed the process token held **`SeImpersonatePrivilege`**, a strong indicator that token-impersonation ("Potato"-family) or kernel-level privilege escalation techniques would be viable.
- `systeminfo` confirmed the target was **Windows 7 Enterprise, Build 7600**, with **no hotfixes installed (`Hotfix(s): N/A`)** — indicating a large attack surface of known local exploits.

### 2.4 Privilege Escalation
- `post/multi/recon/local_exploit_suggester` was run against the session, returning over a dozen plausible local privilege escalation modules for this OS build (e.g. `ms10_015_kitrap0d`, `ms10_092_schelevator`, `ms16_075_reflection_juicy`, and others).
- After several unsuccessful attempts (see Section 4), **`exploit/windows/local/ms16_075_reflection_juicy`** was run against a fresh, stable Meterpreter session and successfully returned a new session running as:
  ```
  NT AUTHORITY\SYSTEM
  ```

### 2.5 Flag Retrieval
- **User flag:** retrieved from `C:\Users\babis\Desktop\user.txt` after privilege escalation to SYSTEM (initial attempts as `IIS APPPOOL\Web` returned "Access is denied").
- **Root/SYSTEM flag:** retrieved from the Administrator's desktop path following successful escalation.
- Both flags were submitted and confirmed as owned on the HTB platform.

---

## 3. Tools Used

| Tool | Purpose |
|---|---|
| `ftp` (CLI client) | Anonymous access, directory listing, file upload |
| `msfvenom` | Payload generation (PHP → discarded; ASPX x64 → failed; ASPX x86 → successful) |
| `msfconsole` (`exploit/multi/handler`) | Reverse shell listener / Meterpreter handler |
| `netcat` (`nc`) | Attempted alternative listener (not compatible with Meterpreter staging) |
| `post/multi/recon/local_exploit_suggester` | Automated enumeration of local privilege escalation candidates |
| `exploit/windows/local/ms16_075_reflection_juicy` | Successful privilege escalation exploit to SYSTEM |
| Firefox (Kali) | Manual triggering of uploaded web shell via HTTP requests |

---

## 4. Challenges Encountered & Troubleshooting

This section documents every significant obstacle faced during the engagement, along with the root cause and resolution, since diagnosing these was a substantial part of the overall exercise.

### 4.1 Attempting to Access the Web Shell via the FTP Port
- **Symptom:** Browsing to `http://10.129.18.40:21/shell.php` failed to load anything.
- **Root Cause:** Port 21 is the FTP control port, not an HTTP port. Sending an HTTP request to an FTP listener is protocol-mismatched and will never succeed.
- **Resolution:** Identified (from the FTP directory contents) that the FTP root matched the IIS web root, and accessed the file over HTTP on the correct port (80) instead: `http://10.129.18.40/shell.php`.

### 4.2 PHP File Returning a 404 Error
- **Symptom:** `http://10.129.18.40/shell.php` returned an IIS-style **"404 – File or directory not found"** page, despite the file existing on disk.
- **Root Cause:** The IIS installation on this host has no PHP handler/module registered. IIS did not recognize `.php` as an executable script type, so it was treated as an unknown/missing resource.
- **Resolution:** Switched to a payload format natively supported by IIS: **ASP.NET (`.aspx`)**.

### 4.3 Misconception: Renaming `.php` to `.aspx`
- **Symptom:** A proposal was made to simply rename the existing `shell.php` file to `shell.aspx` rather than regenerating the payload.
- **Root Cause (clarified before implementation):** File extension alone does not change the underlying code. A file containing PHP syntax, renamed to `.aspx`, would be handed to the ASP.NET engine and fail to compile/execute correctly.
- **Resolution:** Generated a **genuine ASP.NET payload** using `msfvenom -f aspx`, rather than renaming the existing PHP file.

### 4.4 Port Conflict Between `netcat` and Metasploit Handler
- **Symptom:**
  ```
  [-] Handler failed to bind to 10.10.14.7:443
  [-] Exploit failed [bad-config]: Rex::BindFailed The address is already in use or unavailable
  ```
- **Root Cause:** A `netcat` listener (`nc -lvnp 443`) was already bound to port 443 in a separate terminal at the same time the Metasploit handler attempted to bind to the same port. Only one process can listen on a given TCP port at a time.
- **Resolution:** Identified the conflicting process (`ss -tulpn`) and ensured only one listener occupied the target port before retrying.

### 4.5 Meterpreter Payload Incompatible with Netcat
- **Symptom:** No shell was returned in the `netcat` listener after the `.aspx` page was accessed.
- **Root Cause:** The payload used (`windows/x64/meterpreter/reverse_tcp`) uses a staged binary protocol that only `exploit/multi/handler` in Metasploit understands. Raw `netcat` cannot negotiate this handshake and will appear to receive nothing usable.
- **Resolution:** Standardized on using `exploit/multi/handler` with a matching payload type as the sole listener for all Meterpreter-based payloads.

### 4.6 x64 Payload Failing to Execute Silently
- **Symptom:** After uploading and accessing `shell.aspx` (built with `windows/x64/meterpreter/reverse_tcp`), the browser tab hung/loaded indefinitely but **no session was ever received**, with no visible server-side error.
- **Root Cause:** The IIS worker process (`w3wp.exe`) on this legacy host runs as a **32-bit process**, even though the OS itself may report x64 characteristics. A 64-bit payload cannot be loaded/executed inside a 32-bit worker process, and this failure occurs silently from the attacker's perspective.
- **Resolution:** Regenerated the payload with `-a x86` and the `windows/meterpreter/reverse_tcp` (non-x64) payload type. This resolved the issue immediately and a session was established.

### 4.7 `getsystem` Timing Out
- **Symptom:**
  ```
  [-] Send timed out. Timeout currently 15 seconds
  ```
  after running `getsystem` on the initial Meterpreter session.
- **Root Cause:** Likely network/session latency over the lab VPN connection rather than a fundamental failure; the underlying session remained responsive to other commands (`getuid` succeeded immediately afterward).
- **Resolution:** Did not rely on `getsystem` further; proceeded directly to `post/multi/recon/local_exploit_suggester` for a more deterministic path to privilege escalation.

### 4.8 Repeated Privilege Escalation Failures Due to a Dead Session
- **Symptom:** Multiple local exploit modules (`ms10_015_kitrap0d`, `ms10_092_schelevator`, `ms16_075_reflection_juicy`) failed with:
  ```
  [-] Msf::OptionValidateError The following options failed to validate: SESSION.
  ```
  despite `set SESSION <n>` appearing to succeed.
- **Root Cause:** The original Meterpreter session (Session 1) had silently died:
  ```
  [*] 10.129.18.40 - Meterpreter session 1 closed. Reason: Died
  ```
  All subsequent exploit attempts referenced a session number that no longer existed, causing validation to fail regardless of the exploit module chosen.
- **Resolution:** Re-established a **fresh** Meterpreter session by re-triggering the uploaded `.aspx` payload against an actively listening handler, confirmed the new session number with `sessions -l`, and used **that** session ID for all subsequent exploit attempts.

### 4.9 Command Typo During Troubleshooting
- **Symptom:** `session -1` returned `Unknown command: session. Did you mean sessions?`
- **Root Cause:** Simple typo — the correct Metasploit console command is `sessions` (plural), not `session`.
- **Resolution:** Corrected the command and used `sessions -l` to list active sessions accurately going forward.

### 4.10 `ms10_015_kitrap0d` Failing to Return a Session
- **Symptom:** Two attempts against a validated, live session completed without error but reported:
  ```
  [*] Exploit completed, but no session was created.
  ```
  A second attempt on a different port timed out entirely.
- **Root Cause:** This exploit is known to be comparatively unreliable/timing-sensitive in some lab environments; it is not guaranteed to succeed even against a nominally vulnerable target.
- **Resolution:** Abandoned this module in favor of alternative candidates suggested by `local_exploit_suggester`, ultimately succeeding with `ms16_075_reflection_juicy`.

### 4.11 Insufficient Privileges to Read the User Flag
- **Symptom:** `type C:\Users\babis\desktop\user.txt` returned `Access is denied` while operating in the `IIS APPPOOL\Web` context.
- **Root Cause:** The web application pool identity does not have file-read permissions on another local user's desktop folder by default.
- **Resolution:** This was expected behavior and confirmed the need for privilege escalation; the flag was successfully read once a SYSTEM-level session was obtained.

---

## 5. Key Lessons Learned

1. **Always verify the correct service/port for a given protocol** — attempting HTTP requests against an FTP port (or vice versa) will never succeed regardless of upload success.
2. **File extensions determine server-side execution behavior**, not file content — renaming a payload's extension without regenerating it in the target language/framework will not produce a working exploit.
3. **Never run two listeners on the same port simultaneously**, and always match the listener type (`netcat` vs. Metasploit handler) to the payload's protocol (raw shell vs. staged Meterpreter).
4. **Payload architecture (x86/x64) must match the target process**, not just the OS — many legacy IIS installations run 32-bit worker processes regardless of the host OS architecture, and mismatches fail silently.
5. **Always verify session liveness with `sessions -l` before reusing a session ID** — attempting to escalate privileges on a dead session produces confusing validation errors that can be mistaken for exploit-specific failures.
6. **Automated tooling (`local_exploit_suggester`) significantly accelerates privilege escalation** on legacy, unpatched systems by narrowing down dozens of theoretically applicable exploits to a practical shortlist.

---

## 6. Flags Captured

| Flag | Status |
|---|---|
| User flag (`babis` desktop) | Captured |
| Root/SYSTEM flag | Captured |

---

*End of report.*
