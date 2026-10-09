
# TryHackMe - TakeOver CTF Writeup
**Author:** Rully Miftahur R  

---

## 📌 Overview
This **TryHackMe** challenge focuses on **subdomain takeover**.
The objective is to perform reconnaissance against the target domain, discover available subdomains, and investigate a potential takeover to obtain the **flag**.

---

## 🧭 Target Information

| Item | Value |
| --- | --- |
| Target IP | 10.49.153.93 |
| Domain | futurevera.thm |
| Tools | ping, nmap, gobuster |
| Attack Type | Subdomain Enumeration / Takeover |

---

## 1️⃣ Connectivity Test

The first step was to confirm that the target was reachable from the attacker machine.

```bash
ping 10.49.153.93
```

A response to the ping confirms that the target is reachable.

---

## 2️⃣ Reconnaissance (Port Scanning)

Next, I used **Nmap** to identify the target's open ports and services.

```bash
nmap -sV 10.49.153.93
```

Parameter used:

| Parameter | Function |
|-----------|--------|
| -sV | detects the versions of services running on ports |

This identifies the services active on the server.

---

## 3️⃣ Domain Discovery

The challenge description supplied the following domain:

```
https://futurevera.thm
```

The domain could not initially be accessed because the local system could not resolve it.

---

## 4️⃣ Configuring /etc/hosts

To access the domain, add an IP-to-hostname mapping to:

```
/etc/hosts
```

Example configuration:

```
10.49.153.93 futurevera.thm
```

After saving the mapping, the domain can be accessed in the browser.

---

## 5️⃣ Subdomain Enumeration

Once the main domain was accessible, the next step was to discover possible **subdomains**.

I used **Gobuster** for this step.

```bash
gobuster vhost -k -u https://futurevera.thm -w /usr/share/wordlists/dirb/common.txt
```

Parameter explanation:

| Parameter | Function |
|----------|--------|
| vhost | performs virtual host enumeration |
| -k | skips TLS certificate verification |
| -u | target URL |
| -w | wordlist to use |

---

## 6️⃣ Enumeration Results

The scan discovered the following subdomains:

- `support.futurevera.thm`
- `blog.futurevera.thm`

The **blog.futurevera.thm** subdomain returned:

```
421 Misdirected Request
```

This HTTP status indicates that the request was sent to a server unable to respond for the requested domain.

---

## 7️⃣ Adding Subdomains to the Hosts File

Because the subdomains could not initially be accessed in the browser, add their mappings to:

```
/etc/hosts
```

Example:

```
10.49.153.93 support.futurevera.thm
10.49.153.93 blog.futurevera.thm
```

After this configuration, the subdomains can be accessed in the browser.

---

## 8️⃣ Certificate Analysis

While accessing:

```
support.futurevera.thm
```

I inspected the **SSL certificate** and found additional information about the **DNS used by the domain**.

This information provided an important clue for continuing the investigation.

---

## 9️⃣ Subdomain Takeover

After identifying the DNS configuration, I updated the mappings in `/etc/hosts`.

With the correct configuration, the domain became fully accessible.

At this stage, I found the **flag** required to complete the challenge.

---

## 🏁 Flag

```
FLAG_FOUND_HERE
```

---

## 🧰 Tools Used

- Nmap
- Gobuster
- Linux CLI
- Browser (for certificate inspection)

---

## 📚 Key Learning

Lessons learned from this challenge:

- Performing **target reconnaissance**
- **Subdomain enumeration** techniques
- Understanding **HTTP response codes**
- **SSL certificate** analysis
- The concept of **subdomain takeover**

---

## 📖 Conclusion

This challenge demonstrates the importance of correct DNS and subdomain configuration.
Subdomain misconfigurations can enable **subdomain takeover**, which attackers may use for phishing, malware hosting, or redirecting domain traffic.

