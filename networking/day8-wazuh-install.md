# Day 8 – Wazuh SIEM Installation (Manager, Indexer, Dashboard)

## Objective
Install the Wazuh all-in-one stack on the Ubuntu VM and confirm I can log in to the dashboard. This is the SIEM for the rest of the lab.

## Environment
| Item | Value |
|---|---|
| Host | Ubuntu-Wazuh VM, 192.168.0.60 |
| OS | Ubuntu 26.04.1 LTS (not on Wazuh's supported list – see Issues) |
| Wazuh version | 4.14.10 |
| Components | Wazuh manager, indexer, dashboard, Filebeat (all-in-one) |
| Dashboard | https://192.168.0.60 (port 443, self-signed certificate) |

## Steps
1. Downloaded the official installer:
   ```bash
   curl -sO https://packages.wazuh.com/4.14/wazuh-install.sh
   ```
2. Ran the all-in-one install, ignoring the OS check and overwriting a failed earlier install:
   ```bash
   sudo bash wazuh-install.sh -a -i -o
   ```
3. Saved the generated `admin` password to a password manager (not committed to this repo).
4. Opened `https://192.168.0.60` in a browser, accepted the self-signed certificate warning, and logged in as `admin`.
5. Checked **Server Management → Status**: the manager is running and the core services are green.
6. Checked services from the shell:
   ```bash
   sudo systemctl status wazuh-manager wazuh-indexer wazuh-dashboard --no-pager
   ```

## Evidence
**Services running**

![Wazuh services running](../screenshots/day8-wazuh-services-running.png)

**Dashboard login**

![Wazuh dashboard login](../screenshots/day8-wazuh-dashboard-login.png)

**Server Management → Status**

![Wazuh status page](../screenshots/day8-wazuh-status.png)

## Issues and how I fixed them
| Problem | Cause | Fix |
|---|---|---|
| Installer error: ports 1515 and 55000 in use | Leftover Wazuh processes from an earlier attempt | Checked with `sudo ss -tulpn \| grep -E ':(1515\|55000)\b'`; ports were free after cleanup |
| "Wazuh manager already installed" | Broken package record from the failed run | Re-ran with `-o` (overwrite) |
| `wazuh-keystore: No such file or directory`, then `wazuh-manager could not be started` | Corrupted dpkg state: package registered but files missing | Purged all Wazuh packages with `dpkg --purge --force-all`, removed `/var/ossec`, then reinstalled |
| Warning: system not in recommended list | Ubuntu 26.04 is newer than the supported 16.04–22.04 | Used the `-i` flag; installation completed, noted as unsupported |
| Dashboard login rejected at first | Password mis-copied from the terminal (similar-looking characters) | Copied the password again from the installer file / reset it |

## What I learned
- Port checks (`ss -tulpn`) and dpkg state matter as much as the installer itself.
- A clean purge is the fix for a half-installed package; reinstalling on top does not work.
- Running an unsupported OS is possible but should be documented as a risk.
- The red items on the Status page (agentlessd, csyslogd, maild, reportd, integratord, clusterd) are optional modules disabled by default.

## Next
Day 9: deploy the Wazuh agent on Windows and confirm it appears in the dashboard.
