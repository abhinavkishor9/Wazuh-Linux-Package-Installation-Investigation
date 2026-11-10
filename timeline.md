# Timeline – Linux Package Installation Investigation

**Date:** November 10, 2026  
**Host:** Rocky Linux 9.8  
**Audit Key:** `lab133_package_mgmt`

> Times below are taken from the recorded terminal and Wazuh output. Where an exact time was not captured, the sequence is described without inventing a timestamp.

| Time (IST) | Activity | Observation |
|---|---|---|
| Initial checks | Operating-system identification | Rocky Linux 9.8 running on VMware was confirmed. |
| Initial checks | Package-manager identification | DNF 4.14.0 and RPM 4.16.1.3 were identified. |
| Initial checks | Package inventory | Existing installed packages were reviewed using RPM. |
| Initial checks | DNF history review | Existing transactions were identified, including historical Wazuh component installations from August 1, 2026. |
| Initial checks | Auditd validation | Auditd was active, audit was enabled, and the reported lost-event count was 0. No active rules were initially listed. |
| Initial checks | Executable path validation | DNF resolved to `/usr/bin/dnf-3`; RPM resolved to `/usr/bin/rpm`. |
| 08:53:08 | DNF execution watch added | Auditd recorded a configuration-change event for adding the DNF watch rule. |
| 08:53:08 | RPM execution watch added | Auditd recorded event ID 221 for adding the RPM watch rule. |
| 08:53:09 | Wazuh alert observed | Wazuh generated rule 80705, `Auditd: Configuration changed.`, for the configuration-change event. |
| During validation | Existing package checked | `tree-1.8.0-10.el9.x86_64` was found to be already installed. |
| During validation | DNF installation attempted | `dnf install tree -y` returned `Nothing to do` because the package was already installed. |
| During validation | RPM metadata reviewed | Package information, installed files, file ownership, and version were queried. |
| During validation | Audit records reviewed | RPM query executions were found under `lab133_package_mgmt`. |
| During validation | Wazuh collection checked | Auditd log collection configuration, Wazuh manager status, and log collector process were verified. |
| Cleanup | Initial rule deletion attempted | Deletion failed because the command did not include the complete installed watch-rule parameters. |
| Cleanup | DNF rule removed | The DNF rule was removed using its matching path, permission, and key. |
| Cleanup | RPM rule removed | The RPM rule was removed using its matching path, permission, and key. |
| Final verification | Lab 133 rule check | No matching `lab133_package_mgmt` rules remained. |

## Final State

- Auditd remained available for system auditing.
- Both temporary Lab 133 execution watch rules were removed.
- No new `tree` installation was performed.
- RPM query activity was recorded.
- Wazuh processed an Auditd configuration-change event.
- A successful package-installation alert was not demonstrated.
