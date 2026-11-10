# Wazuh Linux Package Installation Investigation

## Overview

This lab explores how Linux package-management activity can be investigated using DNF, RPM, Auditd, and Wazuh on Rocky Linux 9.8.

The investigation focuses on identifying installed packages, reviewing package-manager history, examining RPM metadata, monitoring execution of package-management utilities, and validating whether Auditd telemetry is collected by Wazuh.

The exercise also demonstrates an important investigation principle: execution of a package-management command does not automatically establish that a package was installed or modified.

## Objectives

- Understand the roles of DNF and RPM in Linux package management.
- Review installed package inventory and package metadata.
- Examine DNF transaction history and package-management logs.
- Configure Auditd to monitor execution of DNF and RPM.
- Validate generated audit records using `ausearch`.
- Verify Wazuh collection of `/var/log/audit/audit.log`.
- Correlate host-level evidence with Wazuh alerts.
- Validate temporary audit-rule cleanup.

## Lab Environment

| Component | Details |
|---|---|
| Operating System | Rocky Linux 9.8 (Blue Onyx) |
| Kernel | 5.14.0-687.31.1.el9_8.x86_64 |
| Virtualization | VMware |
| DNF | 4.14.0 |
| RPM | 4.16.1.3 |
| Auditd | 3.1.5 |
| Wazuh | 4.13.1 |
| Audit Log | `/var/log/audit/audit.log` |
| Wazuh Configuration | `/var/ossec/etc/ossec.conf` |

## Investigation Scope

The investigation covered the following areas:

1. Linux operating-system and package-manager identification.
2. Installed package inventory.
3. DNF transaction history.
4. DNF package-management logs.
5. RPM package metadata and file ownership.
6. Auditd service and audit-rule status.
7. Execution monitoring for `/usr/bin/dnf` and `/usr/bin/rpm`.
8. Audit event validation using the `lab133_package_mgmt` key.
9. Wazuh audit-log collection and alert validation.
10. Removal of temporary Lab 133 audit rules.

## Auditd Monitoring

The following temporary rules were used:

```bash
auditctl -w /usr/bin/dnf -p x -k lab133_package_mgmt
auditctl -w /usr/bin/rpm -p x -k lab133_package_mgmt
```

These rules monitored execution of the specified package-management paths.

The `x` permission monitors execution. It does not independently prove that a package installation, upgrade, or removal completed successfully.

Audit events were reviewed using:

```bash
ausearch -k lab133_package_mgmt -i
```

## RPM Package Validation

The `tree` package was used for package inventory and metadata validation.

```bash
rpm -q tree
rpm -qi tree
rpm -ql tree
rpm -qf /usr/bin/tree
tree --version
```

The package was already installed:

```text
tree-1.8.0-10.el9.x86_64
```

The attempted command:

```bash
dnf install tree -y
```

returned:

```text
Package tree-1.8.0-10.el9.x86_64 is already installed.
Dependencies resolved.
Nothing to do.
Complete!
```

Therefore, this command did not produce a new package installation.

## Wazuh Validation

The Wazuh configuration contained an Auditd log collection entry:

```xml
<localfile>
  <log_format>audit</log_format>
  <location>/var/log/audit/audit.log</location>
</localfile>
```

The Wazuh manager and log collector were running during validation.

A Wazuh alert was observed with these details:

| Field | Observed value |
|---|---|
| Rule ID | 80705 |
| Rule Description | Auditd: Configuration changed. |
| Rule Level | 3 |
| Audit Event Type | CONFIG_CHANGE |
| Audit Event ID | 221 |
| Audit Key Field | null |
| Audit Log Location | `/var/log/audit/audit.log` |
| Agent ID | 000 |
| Timestamp | Nov 10, 2026 @ 08:53:09.152 |

The raw event recorded the addition of the `lab133_package_mgmt` RPM execution watch rule.

This confirms collection and alert processing of an Auditd configuration-change event. It does not demonstrate a Wazuh alert for a successful package installation.

## Key Findings

- The host was running Rocky Linux 9.8 with DNF 4.14.0 and RPM 4.16.1.3.
- Auditd was active and enabled.
- Audit status showed `lost 0` during the recorded validation.
- The `tree` package was already installed before the attempted installation.
- DNF returned `Nothing to do` for the `tree` installation request.
- RPM package queries generated execution audit records under `lab133_package_mgmt`.
- The Wazuh manager was active and its log collector process was running.
- Wazuh generated rule 80705 for an Auditd configuration-change event.
- The Wazuh event's `data.audit.key` field was null, while the raw `full_log` contained the key in the configuration-change record.
- Both temporary Lab 133 audit rules were removed and the final matching-rule check returned no output.

## Investigation Limitations

- A successful new package installation was not demonstrated.
- The available Wazuh alert represents an Auditd rule-configuration change, not a package installation.
- The RPM execution records demonstrate package-query activity, not a package modification.
- The historical DNF transactions shown in the output predate this lab and should not be attributed to the current exercise.
- A Wazuh alert for package installation activity was not confirmed.

## Conclusion

This lab demonstrated how package-management investigation can combine DNF history, RPM metadata, Auditd execution telemetry, and Wazuh event collection.

The evidence confirmed RPM query execution and Wazuh processing of an Auditd configuration-change event. However, because `tree` was already installed, no new installation was completed during the exercise.

The main takeaway is to correlate package-manager history, package state, audit records, and SIEM telemetry before concluding that a package was installed or modified.
