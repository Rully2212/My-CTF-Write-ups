# Nmap Write Up TryHackMe

**By Rully Miftahur R**

---

## Overview
This lab focuses on advanced network scanning techniques using Nmap (Network Mapper). The exploration covers various TCP scan types, OS fingerprinting, and the utilization of the Nmap Scripting Engine (NSE) to detect vulnerabilities or misconfigured services.

---

## Tools & Environment
- **OS**: Kali Linux (Running via UTM Virtual Machine on Apple Silicon)  
- **Connectivity**: OpenVPN (Tunneling into TryHackMe's internal network)  
- **Main Tools**: Nmap, Wireshark  

---

## Methodology & Walkthrough

### 1. Host Discovery & Firewall Bypass
Initial reconnaissance revealed that the target (Windows Server) does not respond to ICMP (Ping) requests.

**Command:**
```bash
nmap -Pn [TARGET_IP]
```

**Analysis:**
The `-Pn` flag is crucial here to skip host discovery and proceed directly to port scanning on targets that protect themselves by blocking ping probes.

---

### 2. Exploring Scan Types (Xmas Scan)
To understand TCP protocol behavior, an Xmas Scan was performed on the first 999 ports.

**Command:**
```bash
sudo nmap -Pn -sX -p 1-999 [TARGET_IP]
```

**Result:**
Ports appeared as `open|filtered` with the reason **No Response**.

**Insight:**
This proves the target uses a TCP/IP stack (Windows) that does not strictly follow RFC 793 standards, as it fails to send a `RST` response for open ports when receiving Xmas packets.

---

### 3. Service Enumeration (TCP SYN Scan)
A fast scan of the first 5000 ports was conducted to map the attack surface.

**Command:**
```bash
sudo nmap -Pn -sS -p 1-5000 [TARGET_IP]
```

**Findings:**
Identified 5 open ports hosting various standard Windows services.

---

### 4. Vulnerability Assessment with NSE
Utilized Nmap's automated scripting to check for specific vulnerabilities in the FTP service (Port 21).

**Command:**
```bash
sudo nmap -Pn -p 21 --script ftp-anon [TARGET_IP]
```

**Objective:**
To verify if the **Anonymous Login** feature is enabled on the FTP server.

---

## Key Takeaways
- Learned the different responses between Windows and Linux operating systems regarding various TCP flags (FIN, PSH, URG).
- Understood the importance of choosing the correct Nmap switches based on target conditions (e.g., using `-Pn` for ICMP-blocking hosts).
- Leveraged NSE for initial automation during the enumeration phase before moving to manual exploitation.
