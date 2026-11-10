# Day 9 – Wazuh Agent on Windows (Win11-Endpoint)

## Objective
Install the Wazuh agent on the Windows 11 VM so it reports to the Wazuh SIEM, and confirm the endpoint shows as **Active** in the dashboard.

## Environment
| Item | Value |
|---|---|
| Wazuh server | Ubuntu-Wazuh, 192.168.0.60 |
| Wazuh version | 4.14.10 |
| Endpoint | Win11-Endpoint (Windows 11) |
| Agent ports | 1514 (events), 1515 (enrolment) |

## Steps
1. Checked the endpoint can reach the Wazuh server (PowerShell on Win11-Endpoint):
   ```powershell
   Test-NetConnection 192.168.0.60 -Port 1515
   Test-NetConnection 192.168.0.60 -Port 1514
   ```
   Both returned `TcpTestSucceeded : True`.
2. Opened the dashboard: **☰ menu → Server Management → Endpoints Summary → Deploy new agent** (`https://192.168.0.60/app/endpoints-summary`). Selected **Windows (MSI)**, server address `192.168.0.60`, agent name `Win11-Endpoint`.
3. Ran the install command in PowerShell as Administrator. The manual equivalent is:
   ```powershell
   Invoke-WebRequest -Uri https://packages.wazuh.com/4.x/windows/wazuh-agent-4.14.10-1.msi -OutFile $env:TEMP\wazuh-agent.msi
   msiexec.exe /i $env:TEMP\wazuh-agent.msi /q WAZUH_MANAGER="192.168.0.60" WAZUH_AGENT_NAME="Win11-Endpoint"
   ```
4. Started the agent service and checked it:
   ```powershell
   NET START WazuhSvc
   Get-Service -Name WazuhSvc
   ```
5. In the dashboard, confirmed `Win11-Endpoint` shows **Active** and reports inventory data (OS, IP).

## Evidence
**Agent Active in the dashboard**

![Agent active](../screenshots/day9-agent-active.png)

**WazuhSvc running on the endpoint**

![Get-Service running](../screenshots/day9-get-service-running.png)

## Log forwarding test
Locked and unlocked the screen on Win11-Endpoint to create a logon event (Event ID 4624), then searched Threat Hunting in Wazuh with:
```
agent.name: "Win11-Endpoint" and data.win.system.eventID: 4624
```

## Notes
- If the agent never turns Active, check the 1514/1515 connectivity test and the firewall first.
- Wazuh VM must be running, and needs a few minutes after a reboot before the dashboard and manager are ready.
- A wrong **PIN** at the lock screen may not create Event ID 4625; use the account password, or `runas /user:fakeuser cmd`, to generate failed-logon events.

## Next
Day 10: learn Windows security Event IDs 4624, 4625 and 4688 in Event Viewer.
