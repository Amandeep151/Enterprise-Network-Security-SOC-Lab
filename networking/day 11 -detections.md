# Wazuh Detections & Investigations

## Day 11: Controlled Failed-Logon Investigation (Event ID 4625 / 4624)

**Environment:** Windows 11 VM (`Win11-VM`) monitored by a Wazuh 4.14.10 agent, reporting to a Wazuh server (`ubuntu-wazuh`, 192.168.0.60). The test was fully authorised, run on my own lab VM.

### Objective
Deliberately generate failed logons on the lab endpoint, then investigate the same activity in two places: Windows Event Viewer (source of truth) and the Wazuh dashboard (SIEM view), and confirm they correlate.

### Test performed
1. Locked the screen (Win + L).
2. Entered a wrong password **5 times** at the lock screen.
3. Logged in with the correct password.
4. Reviewed the Security log in Event Viewer, then searched Wazuh Threat Hunting for the same events.

### Investigation Summary

| Field | Finding |
|---|---|
| **Source** | Local console logon (Source Network Address `127.0.0.1`, i.e. the machine itself) |
| **Host** | `Win11-VM` (Wazuh `agent.name`; Windows machine account `AMANWIN$` as the event Subject) |
| **Username targeted** | `aman` (`data.win.eventdata.targetUserName`) |
| **Logon type** | `2` (Interactive: local keyboard/screen logon) |
| **Failure reason** | Unknown user name or bad password. Status `0xc000006d`, SubStatus `0xc000006a` (`%%2313`) |
| **Number of attempts** | 5 failed logons (Event ID 4625) |
| **Time window** | Failures around 10:19; success at 10:20 (Event Viewer, VM local time) |
| **Outcome** | A successful logon (Event ID 4624) followed the failures, using the same local account |

### Event Viewer findings
- **4625 (An account failed to log on):** logged at 10:19 for each wrong-password attempt.
  - Subject account: `AMANWIN$` (the machine / SYSTEM context that processes the lock-screen logon)
  - Logon Type: `2`
  - Source Network Address: `127.0.0.1`
  - Failure Reason: Unknown user name or bad password
- **4624 (An account was successfully logged on):** logged at 10:20, after the final failed attempt.

### Wazuh findings
- Filtered by `agent.name: Win11-VM` (last 24 hours): the dashboard showed **5 authentication failures** and **2 authentication successes**.
- The failed logons appear as `data.win.system.eventID: 4625`, normally matched by Wazuh rule **60122** (Logon failure: unknown user or bad password). Repeated failures can also be grouped by rule **60204** (Multiple Windows logon failures), which is how a SIEM correlation rule turns many raw events into one alert.
- Sample alert details:
  - Log timestamp: `Nov 12, 2026 @ 12:05:46.048` (dashboard display time)
  - System time: `2026-11-13T11:10:37.5999425Z` (event time inside the log)

### Detection logic (for later reuse)
```
data.win.system.eventID: 4625
agent.name: Win11-VM
```
Trigger an investigation when **5 or more** 4625 events occur for the same `targetUserName` within a short window, especially if followed by a 4624 for that account (possible successful password guessing).

### Lessons learned
- 4625 events identify the **target account** (`targetUserName`), while the Subject fields show the process or system context handling the logon. Don't confuse the two.
- Logon type 2 with source `127.0.0.1` indicates a physical or console attempt, not a remote attack.
- A Wazuh agent must be installed and running on the endpoint to see its events. Events generated before the agent was connected are not backfilled.
- Time zones differ between Event Viewer, the Wazuh dashboard display, and the raw log timestamp (UTC). Always compare events using the same timezone.

### Setup note: agent deployment
The agent was installed on the VM from an admin PowerShell session, then started with `NET START WazuhSvc`:
```powershell
Invoke-WebRequest -Uri https://packages.wazuh.com/4.x/windows/wazuh-agent-4.14.10-1.msi -OutFile $env:tmp\wazuh-agent.msi
Start-Process msiexec.exe -Wait -ArgumentList "/i $env:tmp\wazuh-agent.msi /q WAZUH_MANAGER=192.168.0.60 WAZUH_AGENT_NAME=Win11-VM"
NET START WazuhSvc
```

### Evidence
- `screenshots/day11-4625-cluster.png` (Event Viewer: 5 consecutive 4625 events followed by 4624)
- `screenshots/day11-wazuh-alert-view.png` (Wazuh Threat Hunting, filtered to `Win11-VM`)
