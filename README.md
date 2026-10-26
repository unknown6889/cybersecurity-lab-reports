# 🛡️ Red Team & Cybersecurity Engineering Technical Reports

Welcome to my security lab repository. This repository contains detailed, step-by-step penetration testing writeups, attack path documentations, and post-exploitation analyses conducted across active Active Directory domains, vulnerable web services, and Linux targets.

---

## 📌 Featured Technical Writeups

### 1. 🏰 [Full Active Directory Domain Compromise (maquette.local)](./active-directory/full-ad-domain-compromise.md)
* **Target:** Windows Server 2022 Domain Controller (`maquette.local`)[cite: 1]
* **Techniques & Vector:** AS-REP Roasting[cite: 2], Kerberoasting[cite: 2], BloodHound Graph Analysis[cite: 3], Abuse of Native `Account Operators` Rights[cite: 3], DCSync[cite: 4], Pass-the-Hash[cite: 4].
* **Outcome:** Escalate from initial unauthenticated access to full **Domain Admin / SYSTEM** shell[cite: 1, 4].

### 2. 🌐 [Drupalgeddon2 (CVE-2018-7600) Web RCE to SUID Root Escalation](./web-exploitation/drupalgeddon2-rce-suid.md)
* **Target:** Drupal CMS on Proxmox DMZ (`DC-1`)[cite: 8]
* **Techniques & Vector:** Unauthenticated Web Remote Code Execution via Form API payload injection[cite: 8, 12], Metasploit/Python PoC[cite: 9, 10], GTFOBins SUID `/usr/bin/find` escalation[cite: 11, 12].
* **Outcome:** Escalate from web user `www-data` to interactive **Root** shell[cite: 10, 11].

### 3. 🔑 [GPP cpassword Decryption & SYSVOL Security Analysis](./active-directory-attacks/gpp-cpassword-exploitation.md)
* **Target:** SYSVOL Share on Domain Controller[cite: 5]
* **Techniques & Vector:** Authenticated SYSVOL SMB enumeration[cite: 5], GPP XML parsing[cite: 5], offline AES key decryption using `gpp-decrypt`[cite: 6].
* **Mitigation:** MS14-025 compliance[cite: 5, 6], Windows LAPS implementation, and gMSA service accounts[cite: 7].

### 4. ⚡ [Kioptrix Level 2: SQLi to Linux Kernel Privilege Escalation](./web-exploitation/kioptrix-sqli-command-injection.md)
* **Target:** CentOS Linux 2.6.9[cite: 13]
* **Techniques & Vector:** Authentication Bypass via SQL Injection[cite: 13], OS Command Injection via Ping Utility[cite: 13], Python Bind Shell[cite: 14], Local Kernel Exploit Compilation (`9545.c`)[cite: 14].
* **Outcome:** Full Root Access[cite: 13, 14].

---

## 🛠️ Tools & Technologies Used
* **Offensive & Reconnaissance:** Nmap[cite: 1, 9, 13], Metasploit[cite: 9, 10], Burp Suite, Impacket (GetNPUsers, GetUserSPNs, secretsdump, psexec)[cite: 2, 4], NetExec (`nxc`)[cite: 2, 5, 6], BloodHound & Python Collector[cite: 3].
* **Systems & Environment:** Active Directory (Windows Server 2022)[cite: 1], Red-Hat / CentOS Linux[cite: 13], Proxmox VE[cite: 8, 13], Docker, VMware ESXi.
* **Languages & Scripting:** Python, Bash, SQL, C (Exploit Compilation)[cite: 10, 14].

---
*Author: **Oussama DIMOQRATI** — Cybersecurity Engineering Student (CESI Reims)*
