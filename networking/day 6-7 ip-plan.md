>> Day 6-7: Virtual Lab Build (pfSense, Windows 11, Ubuntu Server, Windows Server)

>>Goal<<
Build a real virtual lab in VirtualBox to host the SOC project. Packet Tracer stays as the simulation reference, and the security work now happens in real VMs.

>> What I did <<
1. Downloaded the ISOs: pfSense, Ubuntu Server, Windows 11 evaluation, Windows Server evaluation.
2. Created the pfSense-FW VM (2 GB RAM, 20 GB disk) with two adapters:
   - Adapter 1: Bridged (WAN)
   - Adapter 2: Internal Network `LAN-INSIDE` (LAN)
3. Installed pfSense, assigned the WAN and LAN interfaces, and set the LAN IP to 192.168.40.1/24 with a DHCP range of .100 to .200.
4. Opened the pfSense web GUI from inside a LAN VM, completed the setup wizard and changed the default admin password.
5. Built Ubuntu-Wazuh (4 GB RAM, 40 GB disk, OpenSSH enabled) and updated it with `sudo apt update && sudo apt upgrade -y`.
6. Built Win11-Endpoint and WinSrv-Lab (Desktop Experience) on `LAN-INSIDE`.
7. Verified IP addresses with `ip a` and `ipconfig`, then ran ping tests between all VMs.

>> IP plan (LAN-INSIDE 192.168.40.0/24)

| Device | IP | Role |
|---|---|---|
| pfSense-FW | 192.168.40.1 | Gateway, firewall, DHCP |
| Ubuntu-Wazuh | 192.168.40.100 | Wazuh server (Day 8+) |
| Win11-Endpoint | 192.168.40.101 | Monitored endpoint |
| WinSrv-Lab | 192.168.40.102 | Monitored server |

Note: this range is for the virtual lab only and is separate from the Packet Tracer addressing.

>> Ping tests <<

| From | To | Result |

| Win11-Endpoint | pfSense 192.168.40.1 | Success |
| Win11-Endpoint | Ubuntu-Wazuh | Success |
| Win11-Endpoint | 8.8.8.8 | Success |
| Win11-Endpoint | WinSrv-Lab | Failed at first, success after fix |

>> Problem and fix: Windows Server did not reply to ping
- Symptom: the other pings worked but WinSrv-Lab showed 100% packet loss.
- Cause: the Windows Server firewall blocks inbound ICMP echo requests by default. The VM had an IP and network access, so the network itself was fine.
- Fix: ran in PowerShell (Admin):
  `New-NetFirewallRule -DisplayName "Allow ICMPv4 In" -Protocol ICMPv4 -IcmpType 8 -Direction Inbound -Action Allow`
  *Result: ping from Win11 to WinSrv-Lab succeeded.

>> Commands used <<

sudo apt update && sudo apt upgrade -y
ip a
ipconfig
ping 192.168.40.1


