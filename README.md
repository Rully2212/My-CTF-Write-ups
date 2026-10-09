# CTF Write-ups

Kumpulan write-up Capture The Flag dan latihan keamanan oleh **Rully Miftahur Rozaq**, disusun berdasarkan platform dan nama mesin. Setiap write-up berisi langkah pengerjaan, hasil pengamatan, dan bukti yang tersedia dari sesi lab.

**23 write-up:** 17 Hack The Box · 5 TryHackMe · 1 OverTheWire.

## Hack The Box

| Write-up | Fokus |
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

| Write-up | Fokus |
| --- | --- |
| [Basic Pentesting](tryhackme/basic-pentesting/README.md) | Enumeration and access |
| [Neighbour](tryhackme/neighbour/README.md) | IDOR |
| [Nmap](tryhackme/nmap/README.md) | Network scanning and NSE |
| [RootMe](tryhackme/rootme/README.md) | File upload and SUID |
| [TakeOver](tryhackme/takeover/README.md) | Subdomain takeover |

## OverTheWire

| Write-up | Fokus |
| --- | --- |
| [Bandit — Levels 1–10](overthewire/bandit/levels-01-10/README.md) | Linux command line and file inspection |

## Struktur repo

```text
hackthebox/<mesin>/README.md
tryhackme/<room>/README.md
overthewire/bandit/levels-01-10/README.md
docs/MIGRATION.md
templates/WRITEUP.md
```

Gambar disimpan di folder `images/`, `assets/`, `screenshots/`, atau `evidence/` di sebelah write-up terkait. Sepuluh gambar yang dirujuk laporan Reactor belum tersedia dalam repo asal; penandanya dicatat di laporan tersebut.

## Menambah write-up

Gunakan [template write-up](templates/WRITEUP.md) dan ikuti [panduan kontribusi](CONTRIBUTING.md).

## Riwayat penggabungan

Repo ini menjadi tempat utama untuk seluruh write-up CTF. Pemetaan repo asal, commit sumber, dan perubahan lokasi berkas tersedia di [catatan migrasi](docs/MIGRATION.md).
