>> >> IP Plan - Virtual Lab (LAN-INSIDE 192.168.40.0/24)

| Device | IP |
|---|---|
| pfSense-FW (gateway) | 192.168.40.1 |
| Ubuntu-Wazuh | 192.168.40.100 |
| Win11-Endpoint | 192.168.40.101 |
| WinSrv-Lab | 192.168.40.102 |

>> >> Ping tests
- Win11 -> pfSense: OK
- Win11 -> Ubuntu: OK
- Win11 -> WinSrv: OK (after allowing ICMP in Windows firewall)
- Win11 -> 8.8.8.8: OK
