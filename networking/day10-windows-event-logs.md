# Day 10 – Windows Security Event Logs (4624, 4625, 4688)

## Objective
Read the three most important Windows security events by hand in Event Viewer (`eventvwr.msc`), so I understand what Wazuh is summarising. Focus: **who, when, from where, what**.

## The events

| Event ID | Meaning | Key fields |
|---|---|---|
| **4624** | Successful logon | Account Name (who), Logon Type (how), Source Network Address (from where), timestamp (when) |
| **4625** | Failed logon | Same fields, plus Failure Reason (why it failed) |
| **4688** | A new process was created | New Process Name (what ran), Creator Process Name (parent), Account Name / Logon ID (who) |

**Logon types:** 2 = interactive (console), 3 = network, 5 = service, 10 = RDP.

## What I found on Win11-Endpoint

### 4624 – Successful logon
- **Logon Type:** 5 (service logon)
- **Account Name:** _<fill in>_
- **Source Network Address:** `-` (none, local service logon)
- **When:** 10/11/2026 16:46:17

### 4625 – Failed logon
- **Account Name:** aman
- **Logon Type:** 2 (interactive)
- **Source Network Address:** 127.0.0.1 (local machine)
- **Failure Reason:** Unknown username or bad password
- **When:** 10/11/2026 15:58:32

### 4688 – Process created
- **New Process Name:** `C:\Windows\System32\lsass.exe`
- **Creator Process Name:** `C:\Windows\System32\wininit.exe`
- **Logon ID:** `0x3e7` (the SYSTEM account)
- This is normal: `wininit.exe` starts `lsass.exe` at boot. An `lsass.exe` started by anything else would be suspicious.

## Setup note
4688 is not logged by default. I enabled it in `secpol.msc` → Advanced Audit Policy Configuration → Detailed Tracking → **Audit Process Creation** → Success. Optionally enable command-line logging under Administrative Templates → System → Audit Process Creation → "Include command line in process creation events".

## Evidence
![4624](../screenshots/day10-event-4624.png)
![4625](../screenshots/day10-event-4625.png)
![4688](../screenshots/day10-event-4688.png)

## Takeaways
- Logon Type matters: repeated failed **Type 3** (network) looks very different from failed **Type 10** (RDP).
- Many 4625s from one Source Network Address can indicate brute force.
- 4688 shows what ran and what launched it, which is key for spotting malicious activity.

## Next
Day 11: generate controlled failed-login events and investigate them in Wazuh.
