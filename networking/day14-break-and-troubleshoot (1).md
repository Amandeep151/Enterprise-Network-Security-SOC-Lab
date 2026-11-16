# Day 14 — Break and Troubleshoot the Full Virtual Lab

**Goal:** I deliberately broke five things in my virtual lab (gateway, DNS, link, zone and a firewall rule), diagnosed each one, fixed it and proved the fix by re-testing. This closes out Project 1.

**Order I worked in:** Baseline → Fault B → Fault C → Fault E → Fault A → Fault D → Final check → Write-ups → Repo and README

## Lab values

| Item | Value |
|---|---|
| LAN-INSIDE | 192.168.40.0/24 (pfSense LAN 192.168.40.1) |
| IT-NET | 192.168.10.0/24 (pfSense 192.168.10.1) |
| GUEST-NET | 192.168.60.0/24 (pfSense 192.168.60.1) |
| Win11-Endpoint | 192.168.40.101 on LAN-INSIDE |
| Ubuntu-Wazuh | 192.168.40.100 |
| WinSrv-Lab | 192.168.40.10 |

**Screenshot convention:** `before` shows the broken state with the symptom and evidence. `after` shows the fixed state with the proof. All screenshots are saved in `screenshots/`.

---

## Baseline (before breaking anything)

1. **[pfSense GUI]** I downloaded a configuration backup from Diagnostics → Backup & Restore → Download configuration as XML.
2. **[VirtualBox]** I took a snapshot of all four VMs (pfSense-FW, Ubuntu-Wazuh, Win11-Endpoint, WinSrv-Lab), each named `day14-clean-baseline`. I'm keeping these for Project 2.
3. I confirmed Win11 was on `LAN-INSIDE` with 192.168.40.101.
4. I recorded how Win11's address was configured (automatic or static) using `ipconfig /all` ("DHCP Enabled"), so I could restore it afterwards.
5. **[Wazuh dashboard]** I confirmed Win11 showed as **Active**.
6. 📸 `day14-baseline-wazuh-active.png` — Wazuh dashboard with Win11 Active.

### WinSrv preparation for the zone and firewall faults (A and D)
Before testing from the IT zone, I made three changes so WinSrv could actually answer traffic from another subnet:
- **pfSense:** I set alias `SERVER_HOSTS` to `192.168.40.100` and `192.168.40.10` (the doc's example IP `.102` did not match my real WinSrv address) and applied the change.
- **WinSrv routes:** because WinSrv had no default gateway on its LAN adapter, I added persistent return routes via pfSense in an Administrator PowerShell:
  ```
  route add 192.168.10.0 mask 255.255.255.0 192.168.40.1 -p
  route add 192.168.60.0 mask 255.255.255.0 192.168.40.1 -p
  ```
- **WinSrv Windows Firewall:** I widened the scope of the "File and Printer Sharing (SMB-In)" rule to include `192.168.10.0/24` (and `192.168.60.0/24`), because by default it only accepts the local subnet.

---

## Fault B — Wrong default gateway

Win11 was on LAN-INSIDE.

1. **Break:** In the adapter's IPv4 properties I chose "Use the following IP address" and filled in every field, so the only thing wrong was the gateway:
   - IP address: `192.168.40.101`
   - Subnet mask: `255.255.255.0`
   - Default gateway: `192.168.40.250` (the fault)
   - Preferred DNS: `192.168.40.1`
   - Alternate DNS: `1.1.1.1`
2. **Symptom:** Pings to `192.168.40.1` and `192.168.40.100` worked because they are on the same subnet, but `ping 8.8.8.8` failed.
3. **Diagnosis:** `ipconfig /all` showed the gateway as .250, and `route print` showed the default route (0.0.0.0) pointing at .250.
4. 📸 `day14-faultB-before.png` — `ipconfig /all` (gateway .250), successful ping to .1, failed ping to 8.8.8.8 (plus `route print`).
5. **Fix:** I set the gateway back to `192.168.40.1`.
6. **Verification:** `ping 8.8.8.8` succeeded again.
7. 📸 `day14-faultB-after.png` — `ipconfig /all` with the correct gateway and a successful `ping 8.8.8.8`.
8. Wazuh stayed Active because the manager is on the same subnet and doesn't need the gateway.

**Problem I hit:** at first, pings to .1 and .100 also failed. The cause was the network connection and not the fault itself, so I checked that Win11's Adapter 1 was on `LAN-INSIDE`, the cable was connected and pfSense and Ubuntu were running.

---

## Fault C — DNS failure

Win11 was still on LAN-INSIDE.

1. **Break:** I kept the IP (`192.168.40.101`), mask and gateway (`192.168.40.1`) correct and changed only the DNS: preferred `192.168.99.99`, alternate blank. I then ran `ipconfig /flushdns`.
2. **Symptom:** `nslookup github.com` timed out and browsing by name failed, but `ping 8.8.8.8` still worked.
3. **Diagnosis:** `ipconfig /all` showed DNS as .99.99, and `nslookup github.com` failed.
4. 📸 `day14-faultC-before.png` — `ipconfig /all` (DNS .99.99), failed `nslookup github.com`, successful `ping 8.8.8.8`.
5. **Fix:** I set preferred DNS to `192.168.40.1` and alternate to `1.1.1.1`, then ran `ipconfig /flushdns`.
6. **Verification:** `nslookup github.com` returned an address.
7. 📸 `day14-faultC-after.png` — successful `nslookup github.com` and `ipconfig /all` with the correct DNS.

**Takeaway:** being able to ping an IP but not a name is the quickest way to tell a DNS problem from a routing problem.

---

## Fault E — LAN link down

Win11 was on LAN-INSIDE with its address, gateway and DNS correct again.

1. **Break:** In the running pfSense VM window I went to Devices → Network and unticked **Connect Network Adapter** for the LAN adapter (Adapter 2). The VM's Settings screen can't change this while it's running.
2. **Symptom:** From Win11, pings to `192.168.40.1` and `8.8.8.8` failed. A ping to `192.168.40.100` (Wazuh) still worked, because VM-to-VM traffic on the same segment doesn't go through pfSense.
3. **Diagnosis:** The ping results above, and pfSense Status → Interfaces showed the LAN interface as no carrier / down.
4. 📸 `day14-faultE-before.png` — Win11 with the failed pings to .1 and 8.8.8.8 alongside the successful ping to .100 (and pfSense showing LAN down).
5. **Fix:** I ticked **Connect Network Adapter** again.
6. **Verification:** Pings to `192.168.40.1` and `8.8.8.8` worked again and pfSense showed LAN up.
7. 📸 `day14-faultE-after.png` — pings succeeding and the pfSense interface up.

---

## Fault A — Wrong zone

Win11 was moved between the IT and Guest networks.

1. **Preparation:** I set Win11's adapter to obtain both the IP address and DNS automatically, since it needs DHCP from pfSense in the IT and Guest zones.
2. **Working state:** I set Win11 Adapter 1 to Internal Network `IT-NET` and ran `ipconfig /renew`. I got 192.168.10.x with gateway 192.168.10.1, and `Test-NetConnection 192.168.40.10 -Port 445` returned **True**.
   - It first failed. The causes were the wrong server IP in the alias, no return route on WinSrv and the SMB firewall scope (see the WinSrv preparation above).
3. **Break:** I changed Adapter 1 to `GUEST-NET` and ran `ipconfig /renew`, which gave me 192.168.60.x.
4. **Symptom:** `Test-NetConnection 192.168.40.10 -Port 445` returned **False**.
5. **Diagnosis:**
   - `ipconfig /all` showed the 192.168.60.x subnet.
   - In pfSense (from another VM, since Win11 on the Guest network can't reach the GUI) I checked Status → System Logs → Firewall for a block on the GUEST interface (source 192.168.60.x, destination 192.168.40.10:445).
   - Firewall → Aliases showed 192.168.60.0/24 is not in IT_HOSTS.
6. 📸 `day14-faultA-before.png` — `ipconfig` (192.168.60.x) and the failed Test-NetConnection.
7. 📸 `day14-faultA-pfsense-log.png` — the pfSense firewall log block entry.
8. **Fix:** I set Adapter 1 back to `IT-NET` and ran `ipconfig /renew`.
9. **Verification:** `ipconfig` showed 192.168.10.x and the port 445 test returned **True**.
10. 📸 `day14-faultA-after.png` — `ipconfig` (192.168.10.x) and Test-NetConnection **True**.

**Wazuh note:** in the Guest zone the agent probably loses its connection because the Guest rule blocks it. In the IT zone it reconnects, because SERVER_HOSTS includes the Wazuh manager.

**Problem I hit:** the block didn't show up in the pfSense log at first. I checked that the log wasn't filtered or stale (and reloaded after retesting), that "Log packets handled by this rule" was ticked on the GUEST Block rule, that the Block sat above any pass rule, and that default-deny logging was enabled under Status → System Logs → Settings.

---

## Fault D — Firewall rule disabled

Win11 stayed on `IT-NET` with a 192.168.10.x address.

1. I first confirmed `Test-NetConnection 192.168.40.10 -Port 445` returned **True**.
2. **Break:** In pfSense I went to Firewall → Rules → IT, edited Rule 1 (Allow IT to reach Servers), ticked **Disable this rule**, saved and clicked **Apply Changes**.
3. **Symptom:** The port 445 test returned **False**, even though the IP and gateway were correct. If the test still passes, a cached state is the reason, so I reset states under Diagnostics → States → Reset States.
4. **Diagnosis:** Status → System Logs → Firewall showed a block from the default deny, and Firewall → Rules → IT showed Rule 1 greyed out.
5. 📸 `day14-faultD-before.png` — Win11 with Test-NetConnection **False**.
6. 📸 `day14-faultD-pfsense.png` — the greyed-out rule and the block log entry.
7. **Fix:** I edited the rule, unticked **Disable this rule**, saved and clicked **Apply Changes**.
8. **Verification:** The port 445 test returned **True** again.
9. 📸 `day14-faultD-after.png` — Test-NetConnection **True** and the rule shown enabled.
10. **Restore:** I set Adapter 1 back to `LAN-INSIDE`, ran `ipconfig /renew`, and confirmed 192.168.40.101 (re-entering the static values if it was originally static).

---

## Final verification

- I pinged between all four VMs.
- `nslookup github.com` resolved.
- The Day 13 rules still behaved as designed: IT → Servers passed, Guest → Internal was blocked and the Rule 3 hit counter increased.
- Wazuh showed Win11 as **Active**.
- 📸 `day14-final-verification.png` — Wazuh dashboard showing Active, plus a successful ping and nslookup.

---

## Write-ups

I documented each fault with the same template: Symptom / Diagnostic commands / Root cause / Fix / Verification.

| Fault | File |
|---|---|
| A (zone), B (gateway), E (link down) | `networking/vlan-troubleshooting.md` |
| C (DNS) | `networking/dns-troubleshooting.md` (new file) |
| D (firewall rule) | `firewall/firewall-rules.md` |

Each write-up links to its before and after screenshots in `screenshots/`.

---

## Repo and README

1. I checked the folder structure: `README.md`, `architecture/`, `packet-tracer/`, `networking/`, `firewall/`, `wazuh/`, `wireshark/`, `screenshots/`.
2. I committed and pushed everything, then checked GitHub to confirm it was all there:
   ```
   git status
   git add .
   git commit -m "Day 14: troubleshooting write-ups, Project 1 complete"
   git push
   ```
3. I updated `README.md` with a project summary, the architecture diagram, a skills list (pfSense firewalling, segmentation, Wazuh SIEM, Wireshark, network troubleshooting) and links to my key write-ups.
4. I read the repo through as a hiring manager would.

---

## Outcome

- All five faults produced a clear, explainable symptom, and each fix was verified by re-testing.
- Wazuh showed Win11 as Active before and after each fault. In the Guest zone it dropped, which I recorded.
- Project 1 is pushed to GitHub, and I kept the `day14-clean-baseline` snapshots for Project 2.
