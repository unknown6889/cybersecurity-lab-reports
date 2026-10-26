# 🛡️ Cybersecurity Research & Technical Lab Reports

Welcome to my cybersecurity research and technical lab portfolio. This repository documents technical penetration testing reports, security audits, and attack chain analyses. Covered topics include Active Directory exploitation, web application security, container security (Docker), and network pivoting.

---

## 📌 Featured Technical Writeups & Lab Reports

### 🏰 Active Directory & Internal Infrastructure Security
* **[Full Active Directory Domain Compromise](./active-directory/full-ad-domain-compromise.md)**  
  *Techniques:* AS-REP Roasting, Kerberoasting, BloodHound Graph Analysis, Native `Account Operators` Rights Abuse, DCSync, Pass-the-Hash.  
  *Outcome:* Full domain compromise of `maquette.local` resulting in a `SYSTEM` / `Domain Admin` shell.

* **[Group Policy Preferences (GPP) `cpassword` Analysis](./active-directory-attacks/gpp-cpassword.md)**  
  *Techniques:* Authenticated SMB enumeration of SYSVOL, GPP XML parsing, offline AES key decryption via `gpp-decrypt`.  
  *Remediation:* MS14-025 compliance, deployment of Windows LAPS, and gMSA accounts.

* **[TryHackMe: Ra — AD & Protocol Exploitation](./ctfs/ra.md)**  
  *Techniques:* Anonymous SMB share enumeration, XMPP client (Spark) analysis, NTLMv2 hash capture via Responder (HTML injection), Windows escalation auditing[cite: 9].

---

### 🌐 Web Exploitation, DevSecOps & Cloud/Container Security
* **[Web RCE via Drupalgeddon2 (CVE-2018-7600) to SUID Root Escalation](./web-exploitation/drupalgeddon2-rce.md)**  
  *Techniques:* Form API payload injection (CVE-2018-7600), Python/Metasploit PoCs, privilege escalation via `/usr/bin/find` SUID binary (GTFOBins).  
  *Outcome:* Escalation from a `www-data` web shell to an interactive **Root** shell.

* **[Kioptrix Level 2: SQL Injection to Kernel Escalation](./web-exploitation/kioptrix-level-2.md)**  
  *Techniques:* Web Authentication Bypass via SQLi, OS Command Injection (Ping utility), Interactive Python Bind Shell, local kernel exploit compilation (`9545.c`).

* **[TryHackMe: Dogcat — LFI, Log Poisoning & Container Escape](./ctfs/dogcat.md)**  
  *Techniques:* LFI filter bypass, Apache access log poisoning, `sudo env` abuse (GTFOBins), Docker container escape via shared volume mount (`backup.sh`)[cite: 7].

* **[TryHackMe: Borderlands — Secret Leakage & Network Pivoting](./ctfs/borderlands.md)**  
  *Techniques:* Exposed `.git` directory enumeration, secret extraction from commit history, OS shell via SQLMap, network pivoting (SOCKS proxy / Meterpreter) to internal vsFTPd service[cite: 8].

* **[Hack The Box: Layover — Multi-Stage Enterprise Attack Chain](./ctfs/layover.md)**  
  *Techniques:* RDP jumpbox pivoting[cite: 10], open Wi-Fi HTTP credential capture[cite: 10], Craft CMS authenticated RCE[cite: 10], Yii2 application key decryption[cite: 10], CUPS local privilege escalation (CVE-2026-34990)[cite: 10].  
  *Résultat:* Compromission complète multi-réseau de l'environnement jumpbox vers le serveur cible et élévation de privilèges root.

---

## 🛠️ Security Methodology & Tooling

* **Reconnaissance & Enumeration:** Nmap, NetExec (`nxc`), BloodHound & Python Ingestion Engine, Responder, Impacket Suite.
* **Exploitation & Frameworks:** Metasploit, Burp Suite, Hashcat, SQLMap, GTFOBins, gpp-decrypt.
* **Environments & Virtualization:** Windows Server AD, Red-Hat / CentOS Linux, Docker Containers, Proxmox VE, VMware ESXi.
* **Languages & Scripting:** Python, Bash, SQL, C (Exploit Compilation), PowerShell.

---

## 🔒 Responsible Disclosure & Ethics

All experiments, security assessments, and technical writeups contained in this repository were conducted strictly within isolated lab environments or authorized CTF platforms.

---
**Author:** Oussama DIMOQRATI  
*Cybersecurity Engineering Student | CESI Graduate School of Engineering*  
