# CTF Write-ups

A collection of Capture The Flag write-ups and security training notes by **Rully Miftahur Rozaq**, organized by platform and machine. Each write-up documents the approach, observations, and available evidence from the lab session.

**23 write-ups:** 17 Hack The Box · 5 TryHackMe · 1 OverTheWire.

## Hack The Box

| Write-up | Focus |
| --- | --- |
| [Arctic](hackthebox/arctic/README.md) | ColdFusion, Windows privilege escalation |
| [Bank](hackthebox/bank/README.md) | Exposed account data, file upload, SUID |
| [Blocky](hackthebox/blocky/README.md) | Java plugin credentials, password reuse, sudo |
| [Blue](hackthebox/blue/README.md) | SMB, MS17-010 / EternalBlue |
| [Cap](hackthebox/cap/README.md) | IDOR, packet captures, Linux capabilities |
| [Devel](hackthebox/devel/README.md) | Anonymous FTP, IIS, Windows privilege escalation |
| [Grandpa](hackthebox/grandpa/README.md) | IIS WebDAV, Windows privilege escalation |
| [Granny](hackthebox/granny/README.md) | WebDAV enumeration, Windows privilege escalation |
| [Lame](hackthebox/lame/README.md) | Samba usermap_script |
| [Legacy](hackthebox/legacy/README.md) | SMB, MS08-067 |
| [Mirai](hackthebox/mirai/README.md) | Default credentials, sudo, deleted-file recovery |
| [Nexus](hackthebox/nexus/README.md) | Exposed configuration, file upload, Git path traversal |
| [Optimum](hackthebox/optimum/README.md) | HTTP File Server, Windows privilege escalation |
| [Orion](hackthebox/orion/README.md) | Craft CMS, password cracking, Telnet |
| [Reactor](hackthebox/reactor/README.md) | Next.js, SQLite credentials, Node.js Inspector |
| [Unified](hackthebox/unified/README.md) | UniFi, Log4j, MongoDB |
| [Valentine](hackthebox/valentine/README.md) | Heartbleed, SSH, tmux |

## TryHackMe

| Write-up | Focus |
| --- | --- |
| [Basic Pentesting](tryhackme/basic-pentesting/README.md) | Enumeration and access |
| [Neighbour](tryhackme/neighbour/README.md) | IDOR |
| [Nmap](tryhackme/nmap/README.md) | Network scanning and NSE |
| [RootMe](tryhackme/rootme/README.md) | File upload and SUID |
| [TakeOver](tryhackme/takeover/README.md) | Subdomain takeover |

## OverTheWire

| Write-up | Focus |
| --- | --- |
| [Bandit — Levels 1–10](overthewire/bandit/levels-01-10/README.md) | Linux command line and file inspection |

## Repository Structure

```text
hackthebox/<machine>/README.md
tryhackme/<room>/README.md
overthewire/bandit/levels-01-10/README.md
docs/MIGRATION.md
templates/WRITEUP.md
```

Images are stored in an `images/`, `assets/`, `screenshots/`, or `evidence/` folder alongside the corresponding write-up. Ten images referenced in the Reactor report were missing from the source repository; their descriptions and filenames are recorded in that report.

## Adding a Write-up

Use the [write-up template](templates/WRITEUP.md) and follow the [contribution guide](CONTRIBUTING.md).

## Consolidation History

This repository is the central home for all CTF write-ups. Source repository mappings, original commits, and file moves are documented in the [migration notes](docs/MIGRATION.md).
