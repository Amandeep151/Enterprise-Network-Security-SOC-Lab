# Day 13 — Firewall Rules on pfSense

**Goal:** Write and test three rules enforcing segmentation: IT → Servers ALLOW, Guest → Internal DENY, Security → required systems ALLOW.

**Prereq:** Days 6–7 done (pfSense reachable at https://192.168.40.1).

> **Catch:** VMs on the same subnet never pass through pfSense, so rules on one LAN can't block anything. Give pfSense two extra interfaces (IT and Guest). Note "separate interfaces" in `firewall/firewall-rules.md`.

## Part A — Add zones
1. **[VirtualBox]** Snapshot pfSense-FW and Win11-Endpoint, then power off pfSense-FW.
2. Add adapters (Internal Network): **Adapter 3 = IT-NET**, **Adapter 4 = GUEST-NET**. Start pfSense.
3. **[pfSense]** Interfaces → Assignments → Add twice (OPT1, OPT2; match by MAC).
4. OPT1: Enable, name **IT**, Static IPv4 **192.168.10.1/24**. OPT2: **GUEST**, **192.168.60.1/24**. Save + Apply.
5. Services → DHCP Server: enable on IT (192.168.10.100–199) and GUEST (192.168.60.100–199).

## Part B — Aliases (Firewall → Aliases → IP)
| Name | Type | Values |
|---|---|---|
| IT_HOSTS | Network | 192.168.10.0/24 |
| GUEST_HOSTS | Network | 192.168.60.0/24 |
| SERVER_HOSTS | Host | 192.168.40.100 (Wazuh), 192.168.40.102 (WinSrv) |
| WAZUH_MGR | Host | 192.168.40.100 |
| INTERNAL_NETS | Network | 192.168.10.0/24, 192.168.40.0/24 |

If the Wazuh agent uses another manager IP (e.g. 192.168.0.60), add it to SERVER_HOSTS too.

## Part C — Rules (tick **Log** on each, then Apply Changes)
| # | Tab | Action | Source | Destination | Description |
|---|---|---|---|---|---|
| 1 | IT | Pass | IT_HOSTS | SERVER_HOSTS | Allow IT to reach Servers |
| 2 | GUEST | Block | GUEST_HOSTS | INTERNAL_NETS | Deny Guest to internal networks |
| 2b (opt) | GUEST | Pass | GUEST_HOSTS | any | Guest internet (keep **below** the Block) |
| 3 | LAN | Pass | WAZUH_MGR | INTERNAL_NETS | Allow Security zone to required systems |

Drag Rule 3 **above** "Default allow LAN to any" (first match wins).

## Part D — Tests
- **Rule 1:** Win11 Adapter 1 → IT-NET; `ipconfig /renew` (expect 192.168.10.x). `Test-NetConnection 192.168.40.102 -Port 445` → **True**. Bonus: `Test-NetConnection 8.8.8.8 -Port 443` → fails (implicit deny).
- **Rule 2:** Win11 Adapter 1 → GUEST-NET; renew (192.168.60.x). Same port 445 test → **False**.
- **Rule 3:** Ubuntu: `ping -c 3 192.168.10.1`. Check hit counter on Rule 3 and a green pass in Status → System Logs → Firewall.
- **Log:** Find the red block entry (Interface GUEST, src 192.168.60.x, dst 192.168.40.102:445).

## Part E — Wrap up
- Set Win11 Adapter 1 back to LAN-INSIDE; `ipconfig /renew` → 192.168.40.101.
- Diagnostics → Backup & Restore → **Download configuration as XML** (needed for Day 14).
- Screenshots: `day13-rule-it-servers.png`, `day13-rule-guest-deny.png`, `day13-rule-security-allow.png`, `day13-firewall-log-block.png`.

## Troubleshooting
Rule does nothing? Usually you forgot **Apply Changes**, or a broader block sits above it.
