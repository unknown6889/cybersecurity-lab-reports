# TryHackMe: Borderlands — Technical Writeup

## Executive Summary
This report details the systematic compromise of the **Borderlands** lab environment. The attack path demonstrates the risks of exposed version control metadata, moving from historical Git secret leakage to database exploitation, command execution via SQLMap, internal host discovery, and lateral pivoting across internal network boundaries.

| Metric | Details |
|---|---|
| **Target Platform** | TryHackMe — Borderlands |
| **Primary Category** | Web Exploitation & Network Pivoting |
| **Attack Vectors** | Exposed `.git` repository, SQL Injection, SQLMap OS Shell, Internal Pivoting (vsFTPd) |

---

## 1. Information Gathering & Git Exposure Analysis
Initial web reconnaissance identified an exposed `.git` directory available on the target application:

```bash
# Enumerating exposed version control assets
curl -s http://[TARGET_IP]/.git/
```

Analyzing the repository commit log revealed past modifications aimed at security cleanup:

```text
commit 2a8f...
Author: Developer <dev@borderlands.local>
    basic prototype of api gateway
    removed sensitive data
```

### Technical Vulnerability Analysis

Removing sensitive configuration files or API keys from the current working tree does **not** purge them from the Git object history. Reverting to prior commits extracted leaked database credentials and internal API keys used to map the application infrastructure.

---

## 2. Web Vulnerability Assessment & SQL Injection

Using the credentials obtained from the Git history, the application endpoints were mapped.

1. **Parameter Tampering:** The `documentid` parameter in the API gateway was found to accept unsanitized user input.
2. **Error-Based Verification:** Inputting SQL control characters triggered database error responses, confirming an active SQL injection vulnerability.
3. **Data Extraction:** Utilizing structured payloads allowed the retrieval of internal application parameters and initial system tokens.

---

## 3. Remote Code Execution via SQLMap

To determine if the database service possessed OS-level execution privileges, SQLMap was deployed to escalate the attack vector from data retrieval to system access:

```bash
sqlmap -u "http://[TARGET_IP]/api/v1/gateway?documentid=1" --batch --os-shell
```

* **Outcome:** The database process was running with elevated system permissions, allowing SQLMap to drop a lightweight web backdoor and spawn an interactive OS command shell on the host.

---

## 4. Internal Network Reconnaissance & Lateral Pivoting

From the initial system foothold, network interface enumeration revealed an internal dual-homed network configuration:

* **Internal Subnet:** `172.16.1.0/24`
* **Discovered Internal Hosts:** `172.16.1.10`, `172.16.1.128`

### Pivoting Strategy

1. **Local Proxy Setup:** Configured a local SOCKS proxy via SSH/Meterpreter to route traffic into the `172.16.1.0/24` internal segment.
2. **Internal Service Enumeration:** Scanned internal targets through the tunnel, discovering an internal service running an outdated **vsFTPd 2.3.4** instance on `172.16.1.10`.
3. **Exploitation & Pivoting:** Targeted the internal FTP service to establish secondary access across the internal network zone.

---

## 5. Security Recommendations & Mitigation

* **Git Repository Hardening:** Ensure `.git` directories are blocked from public web access via server configuration (e.g., `LocationMatch` blocks in Apache/Nginx).
* **Secret Management:** Treat any key or password exposed in a Git history as fully compromised. Enforce secret scanning tools (e.g., `gitleaks`, TruffleHog) in CI/CD pipelines and rotate compromised credentials immediately.
* **SQLi Remediation:** Enforce parameterized queries (prepared statements) across all API endpoints to completely decouple SQL logic from user input.
* **Database Principle of Least Privilege:** Run database services under low-privileged dedicated accounts without permissions to write to web directories or spawn shell processes.

---

*Author: **Oussama DIMOQRATI***
