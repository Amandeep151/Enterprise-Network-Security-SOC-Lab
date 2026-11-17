Day 13 — Firewall Rules on pfSense

Goal: I wanted to write and test three firewall rules that enforce segmentation. The first lets IT reach the servers, the second denies Guest access to the internal networks, and the third lets the Security side reach the systems it needs.

Prereq: Days 6–7 done (pfSense reachable at https://192.168.40.1).

> Catch: VMs on the same subnet never pass through pfSense, so rules on one LAN can't block anything. Give pfSense two extra interfaces (IT and Guest). Note "separate interfaces" in `firewall/firewall-rules.md`.

 Part A — Add zones
First, in VirtualBox, I took snapshots of pfSense-FW and Win11-Endpoint so I could roll back if I broke something, and then powered off pfSense-FW.

I added two more internal network adapters to the pfSense VM. Adapter 3 is IT-NET and Adapter 4 is GUEST-NET. Then I started pfSense again.

In pfSense I went to Interfaces, then Assignments, and added the two new interfaces, which showed up as OPT1 and OPT2. I matched them to the right adapters using their MAC addresses. I enabled OPT1, renamed it IT, and gave it the static IPv4 address 192.168.10.1/24. I did the same with OPT2, renamed it GUEST, and gave it 192.168.60.1/24. Then I saved and applied the changes.

Finally, under Services and DHCP Server, I turned on DHCP for both new networks. IT hands out 192.168.10.100 to 192.168.10.199 and GUEST hands out 192.168.60.100 to 192.168.60.199.

 Part B — Rather than typing IP addresses into every rule, I created aliases under Firewall, then Aliases, then IP. This keeps the rules readable.
| Name | Type | Values |
|---|---|---|
| IT_HOSTS | Network | 192.168.10.0/24 |
| GUEST_HOSTS | Network | 192.168.60.0/24 |
| SERVER_HOSTS | Host | 192.168.40.100 (Wazuh), 192.168.40.102 (WinSrv) |
| WAZUH_MGR | Host | 192.168.40.100 |
| INTERNAL_NETS | Network | 192.168.10.0/24, 192.168.40.0/24 |

If the Wazuh agent uses another manager IP (e.g. 192.168.0.60), add it to SERVER_HOSTS too.

Part C — Rules I ticked the Log option on every rule so I could see them in the firewall logs later, and then clicked Apply Changes.
For Rule 1, I changed the Win11 VM’s Adapter 1 to IT-NET and ran ipconfig /renew. It picked up an address in 192.168.10.x as expected. Then I ran Test-NetConnection 192.168.40.102 -Port 445 and it returned True, so IT can reach the server. As a bonus check I ran Test-NetConnection 8.8.8.8 -Port 443, which failed. That is the implicit deny at work, because I never created a rule letting IT out to the internet.

For Rule 2, I changed Adapter 1 to GUEST-NET and renewed, and the VM got an address in 192.168.60.x. I ran the same port 445 test and this time it returned False, so Guest is blocked from the server.

For Rule 3, I pinged from the Ubuntu VM with ping -c 3 192.168.10.1. I then checked that the hit counter on Rule 3 went up and that a green pass entry showed in Status, System Logs, Firewall.

For the log check, I found the red block entry for the Guest test. It showed the interface as GUEST, the source as 192.168.60.x and the destination as 192.168.40.102 on port 445.

Drag Rule 3 above "Default allow LAN to any" (first match wins).

 Part D — Tests
- Rule 1: Win11 Adapter 1 → IT-NET; `ipconfig /renew` (expect 192.168.10.x). `Test-NetConnection 192.168.40.102 -Port 445` → True. Bonus: `Test-NetConnection 8.8.8.8 -Port 443` → fails (implicit deny).
- **Rule 2:** Win11 Adapter 1 → GUEST-NET; renew (192.168.60.x). Same port 445 test → False.
- **Rule 3:** Ubuntu: `ping -c 3 192.168.10.1`. Check hit counter on Rule 3 and a green pass in Status → System Logs → Firewall.
- **Log:** Find the red block entry (Interface GUEST, src 192.168.60.x, dst 192.168.40.102:445).

Part E — Wrap up
- I put Win11 Adapter 1 back on LAN-INSIDE and ran ipconfig /renew, and it came back with 192.168.40.101.
- Diagnostics → Backup & Restore → Download configuration as XML (needed for Day 14).
- Screenshots: `day13-rule-it-servers.png`, `day13-rule-guest-deny.png`, `day13-rule-security-allow.png`, `day13-firewall-log-block.png`.


