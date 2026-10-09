# Valentine — Hack The Box Writeup

**Platform:** Hack The Box  
**Machine:** Valentine  
**Date:** 29 September 2026  
**Environment:** Kali Linux in UTM; target Linux host on the HTB VPN  
**Evidence status:** User access and attachment to a tmux session are shown. The supplied screenshots do not show `id` inside tmux or a root flag, so root access is not claimed here.

## Overview

I found three exposed services: SSH on port 22, HTTP on port 80, and HTTPS on port 443. The HTTPS service was vulnerable to Heartbleed (CVE-2014-0160), which disclosed a Base64-encoded phrase from server memory. A web-accessible file under `/dev/hype_key` contained an encrypted RSA private key represented as hexadecimal text. After decoding the key and using the leaked phrase as its passphrase, I authenticated over SSH as `hype`. I then found a tmux socket accessible to the `hype` group and attached to its existing session.

This activity took place on the HTB Valentine lab machine. Its assigned IP changed during the exercise: early screenshots and a saved memory dump use `10.129.232.136`, while the later Nmap scan and SSH login use `10.129.48.67`. Substitute the **current** HTB-assigned IP when repeating the commands.

## 1. Reconnaissance

The service scan of `10.129.48.67` showed three open TCP ports:

| Port | Service | Evidence |
| --- | --- | --- |
| 22 | SSH | OpenSSH 5.9p1 Debian 5ubuntu1.10 |
| 80 | HTTP | Apache httpd 2.2.22 on Ubuntu |
| 443 | HTTPS | Apache httpd 2.2.22 on Ubuntu; certificate common name `valentine.htb` |

The same output showed 997 other scanned TCP ports closed. The complete command for this initial service scan is not visible in the screenshot, so I have not reconstructed its flags.

![Nmap service detection showing ports 22, 80, and 443](images/01-nmap-services.png)
*Figure 1. Initial service enumeration and TLS certificate details.*

I then tested the HTTPS port with Nmap's Heartbleed script:

```bash
nmap --script=ssl-heartbleed -p 443 10.129.48.67
```

The script reported `State: VULNERABLE` on port 443. This established a reason to inspect the data returned by the TLS heartbeat service.

![Nmap ssl-heartbleed result marked vulnerable](images/02-heartbleed-nmap.png)
*Figure 2. Nmap confirmed Heartbleed on HTTPS.*

## 2. Heartbleed memory disclosure

I used Metasploit's Heartbleed scanner to collect a memory sample:

```text
use auxiliary/scanner/ssl/openssl_heartbleed
set RHOSTS 10.129.48.67
run
```

The module reported a heartbeat response with a leak of **65,535 bytes** and saved the response as a `.bin` loot file. That is a memory disclosure, not proof that every byte in the dump contains a secret.

![Metasploit Heartbleed response and saved loot path](images/03-heartbleed-metasploit.png)
*Figure 3. A Heartbleed response was saved for local analysis.*

I inspected printable strings in a dump saved from the earlier target assignment, `10.129.232.136`:

```bash
strings -a -t x /home/rully/.msf4/loot/20260928123706_default_10.129.232.136_openssl.heartble_246323.bin
```

Here, `-a` scans the entire binary file and `-t x` prints hexadecimal offsets. Among HTTP header fragments, the output included a form body:

```text
Referer: https://127.0.0.1/decode.php
Content-Type: application/x-www-form-urlencoded
Content-Length: 42
$text=aGVhcnRibGVlZGJlbGlldmV0aGVoeXBlCg==
```

The `$text` value is Base64. Decoding it yielded `heartbleedbelievethehype` followed by a newline:

```bash
printf '%s' 'aGVhcnRibGVlZGJlbGlldmV0aGVoeXBlCg==' | base64 -d
```

The visible `Referer` identifies a local `/decode.php` page in the leaked HTTP material; it does not by itself identify the destination of the request. The phrase became relevant when testing the encrypted key below.

![Printable strings from the Heartbleed dump and the encoded text field](images/04-memory-base64.png)
*Figure 4. The memory sample exposed an encoded form value.*

![Additional HTTP request fragments in the same Heartbleed dump](images/12-memory-http-fragments.png)
*Figure 4a. A wider view of the dump also contains HTTP request fragments; these are context, not additional credentials.*

## 3. Recovering and checking the RSA key

Browsing to `http://10.129.48.67/dev/hype_key` displayed pairs of hexadecimal digits. The opening bytes, `2d 2d 2d 2d 2d 42 45 47 49 4e`, decode to `-----BEGIN`; the resulting file starts with `-----BEGIN RSA PRIVATE KEY-----`. Its header also contains `Proc-Type: 4,ENCRYPTED` and `DEK-Info: AES-128-CBC`, indicating that the private key requires a passphrase.

![Hexadecimal representation of the encrypted RSA key](images/05-hype-key-hex.png)
*Figure 5. The web endpoint exposed an encrypted RSA key as hexadecimal text.*

I converted the hexadecimal response into a local PEM file, restricted its permissions, and checked whether the leaked phrase would open it:

```bash
curl -fsS http://10.129.48.67/dev/hype_key | xxd -r -p > hype_key.pem
chmod 600 hype_key.pem
head -n 3 hype_key.pem
openssl pkey -in hype_key.pem -check -noout
```

`xxd -r -p` reverses a plain hexadecimal dump. `chmod 600` limits access to the key file to its owner. OpenSSL prompted for a passphrase and printed `Key is valid` after I entered `heartbleedbelievethehype`. This verifies the key/passphrase pairing; it does not alone verify SSH account access.

![Decoded PEM header, OpenSSL validation, and initial SSH failure](images/06-key-check-ssh-error.png)
*Figure 6. OpenSSL validated the encrypted key; an initial SSH attempt then failed during signature negotiation.*

## 4. SSH access as `hype`

The first SSH attempt with the key reached `sign_and_send_pubkey: no mutual signature supported` and fell back to an account password prompt. The key passphrase and the account password are different prompts; entering the leaked phrase at the account password prompt did not resolve the signature error.

I retried with the legacy `ssh-rsa` public-key signature enabled **for this single connection**:

```bash
ssh -i hype_key.pem -o IdentitiesOnly=yes \
  -o PubkeyAcceptedAlgorithms=+ssh-rsa \
  hype@10.129.48.67
```

`IdentitiesOnly=yes` tells SSH to offer the specified identity, while `PubkeyAcceptedAlgorithms=+ssh-rsa` permits the signature method required by this older SSH server. After the key passphrase prompt, the screenshot shows the Ubuntu welcome message and a `hype@Valentine` shell. `pwd` returned `/home/hype`.

![Successful SSH login as hype with the legacy RSA signature option](images/07-ssh-hype.png)
*Figure 7. The corrected SSH command established the `hype` foothold.*

The directory listing showed `/home/hype/user.txt`, but the screenshots do not show the flag's contents. No flag value is reproduced here.

## 5. tmux session discovery

I inspected the `hype` user's shell history and tmux configuration. The history contained commands referencing a custom tmux socket at `/.devs/dev_sess`:

```text
tmux -L dev_sess
tmux a -t dev_sess
tmux -S /.devs/dev_sess
```

The `.tmux.conf` file also had a `run-shell` entry pointing to a tmux-resurrect script. These observations motivated checking the custom socket; they do not themselves establish elevated privileges.

![Shell history showing tmux and the custom socket path](images/08-tmux-history.png)
*Figure 8. Shell history identified tmux and `/.devs/dev_sess`.*

I checked the socket and listed sessions through it:

```bash
ls -l /.devs/dev_sess
tmux -S /.devs/dev_sess ls
```

The socket was `srw-rw----`, owned by `root` with group `hype`; the leading `s` indicates a Unix-domain socket. The group permissions allowed `hype` to connect. tmux listed session `0` with one window.

![Socket ownership and a live tmux session](images/09-tmux-socket.png)
*Figure 9. The root-owned socket was accessible through the `hype` group, and session `0` existed.*

I attached to that session and then detached:

```bash
tmux -S /.devs/dev_sess attach -t 0
```

To detach from tmux while keeping the session running, press `Ctrl+b` and then `d`. The screenshot shows `[detached]` and a subsequent `tmux ... ls` still listing session `0`. It does **not** show `whoami`, `id`, or a root flag from inside the tmux session, so its effective user remains unverified in this record.

![tmux attach, detach, and session still listed](images/10-tmux-attach-detach.png)
*Figure 10. The existing tmux session was accessible and remained available after detaching.*

## 6. Troubleshooting observed during the lab

| Symptom | Evidence and resolution |
| --- | --- |
| `sign_and_send_pubkey: no mutual signature supported` | The key passed the OpenSSL check, but the first SSH connection could not agree on a signature. Retrying with `-o PubkeyAcceptedAlgorithms=+ssh-rsa` led to a `hype` login. |
| `failed to connect to server: Connection refused` on the tmux socket | A later `tmux -S /.devs/dev_sess ls` failed despite the previously listed session. The screenshots do not establish what stopped the server. On the later target instance, the session was again listed and successfully attached/detached. Avoid treating a remaining socket file as proof of a live tmux server. |
| `ssh: Could not resolve hostname pubkeyacceptedalgorithms=+ssh-rsa` | One command line omitted the `-o` before the option, so SSH interpreted the option text as a hostname. Supplying `-o PubkeyAcceptedAlgorithms=+ssh-rsa` fixed the command syntax. |

![tmux socket connection refused on the earlier instance](images/11-tmux-connection-refused.png)
*Figure 11. A socket pathname alone did not guarantee the tmux server was still running.*

## Verification and limits

| Milestone | Status in supplied evidence |
| --- | --- |
| Ports 22, 80, and 443 identified | Verified |
| Heartbleed reported and memory leak saved | Verified |
| Encoded phrase recovered from memory | Verified |
| Encrypted RSA key decoded and validated | Verified |
| SSH login as `hype` | Verified |
| tmux session listed, attached, and detached | Verified |
| Identity inside tmux / root access | **Not shown** |
| User and root flag contents | **Not shown** |

The decisive chain was the Heartbleed memory disclosure, the Base64 phrase, the exposed encrypted private key, and SSH's legacy signature compatibility option. The tmux socket is a promising privilege-escalation lead because of its ownership and group permissions, but the supplied evidence stops before confirming the user identity inside that session.

## References

- [OpenSSL vulnerability entry for CVE-2014-0160](https://www.openssl-library.org/news/vulnerabilities-1.0.1/)
- [OpenSSH legacy algorithm options](https://www.openssh.com/legacy.html)
