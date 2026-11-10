# Wazuh-Linux-Package-Installation-Investigation
## Overview
### 1. Introduction

Package management is an important part of Linux system administration. Software packages are installed, upgraded, removed, and maintained through package-management utilities. These operations can introduce new executables, libraries, services, configuration files, and dependencies into a system.

From a cybersecurity perspective, package-management activity is important because an attacker with sufficient privileges may install unauthorized tools, introduce persistence mechanisms, or modify existing software.

A SOC analyst should therefore be able to identify package-management activity, determine which account initiated it, understand what operation was performed, and validate whether the package state actually changed.

On Rocky Linux, two important utilities used for package management are DNF and RPM.

### 2. Understanding DNF

DNF (Dandified YUM) is the high-level package manager used by Rocky Linux and other RPM-based distributions.

DNF manages software installation and removal while also resolving dependencies and communicating with configured software repositories.

Common DNF operations include:

```bash
dnf install package_name
dnf remove package_name
dnf update
dnf upgrade
dnf history
dnf history info transaction_id
```

DNF history is particularly useful during investigations because it provides information about previous package-management transactions.

A transaction can include:

- Transaction ID.
- Command line used.
- Date and time.
- User information.
- Return code.
- Packages installed, upgraded, or removed.

For example:

```bash
dnf history
dnf history info 29
```

The transaction details can help an analyst understand whether a package-management operation completed successfully and which packages were affected.

However, historical transactions must be correlated with the incident timeline. An old installation should not be attributed to a current investigation.

### 3. Understanding RPM

RPM (RPM Package Manager) is responsible for managing packages in RPM-based Linux distributions.

It maintains package information in the RPM database and supports package installation, removal, verification, and querying.

Useful RPM commands include:

```bash
rpm -qa
rpm -q package_name
rpm -qi package_name
rpm -ql package_name
rpm -qf /path/to/file
rpm -V package_name
```

These commands serve different investigative purposes:

- `rpm -qa` lists installed packages.
- `rpm -q` checks whether a package is installed.
- `rpm -qi` displays package metadata.
- `rpm -ql` lists files associated with a package.
- `rpm -qf` identifies which package owns a particular file.
- `rpm -V` verifies installed package files against recorded package metadata.

RPM queries are useful for establishing the current package state.

An important distinction is that querying a package does not modify it. For example, `rpm -q tree` only checks package information; it does not install or upgrade the package.

### 4. DNF History and Package-Management Logs

Package-management investigations should use more than the current installed-package inventory.

DNF maintains transaction history and log information that can provide additional context.

Common evidence locations include:

| Location | Investigative value |
|---|---|
| `/var/log/dnf.log` | DNF operations, repository activity, and package-manager messages |
| `/var/log/dnf.rpm.log` | RPM-related activity recorded by DNF, when available |
| `/var/log/hawkey.log` | Repository and dependency-resolution activity, when available |
| `/var/lib/dnf/history.sqlite` | DNF transaction-history database |

These sources should be examined together.

For example, an analyst may find a package listed in the RPM database but still need to determine when it was installed and which transaction introduced it.

DNF history can provide transaction context, while RPM queries establish the current package state.

Log availability and retention may vary between systems.

### 5. Understanding Auditd

Auditd is the Linux auditing service used to record security-relevant system activity.

It can capture information about process execution, system calls, file access, identity-related activity, and audit configuration changes, depending on the rules configured.

Auditd is useful in package-management investigations because it can monitor execution of package-management utilities.

For this lab, the following temporary rules were used:

```bash
auditctl -w /usr/bin/dnf -p x -k lab133_package_mgmt
auditctl -w /usr/bin/rpm -p x -k lab133_package_mgmt
```

The rule components mean:

- `-w`: Adds a watch on a specified path.
- `/usr/bin/dnf` or `/usr/bin/rpm`: The executable path being monitored.
- `-p x`: Monitors execution.
- `-k lab133_package_mgmt`: Assigns a searchable audit key.

The key makes it easier to locate related records:

```bash
ausearch -k lab133_package_mgmt -i
```

The active rules can be reviewed with:

```bash
auditctl -l
```

Auditd can record the executable, command arguments, user identity, process information, and execution result.

However, an execution event alone does not prove that a package was installed successfully.

For example, executing `rpm -q tree` generates RPM execution activity, but it only queries the package database.

### 6. Understanding Audit Event Records

Linux Auditd records contain event types that provide different parts of an activity record.

Important event types include:

| Event type | Purpose |
|---|---|
| `SYSCALL` | Records system-call information, including process and execution details |
| `EXECVE` | Records command-line arguments passed to an executed program |
| `PROCTITLE` | Provides the process command line in encoded form |
| `PATH` | Records relevant filesystem paths |
| `CWD` | Records the current working directory |
| `CONFIG_CHANGE` | Records changes to Auditd configuration or rules |

Multiple records may share the same audit event ID and timestamp. They should be correlated as parts of one event rather than interpreted as unrelated activities.

For package-management investigations, the command arguments are particularly useful because they help distinguish installation, removal, upgrade, and query operations.

### 7. Wazuh Integration with Auditd

Wazuh can collect Linux Auditd records and process them through its analysis engine.

In this lab, the Wazuh configuration included:

```xml
<localfile>
  <log_format>audit</log_format>
  <location>/var/log/audit/audit.log</location>
</localfile>
```

This configuration identifies the Auditd log file and specifies its format for collection.

The relevant components are:

1. Auditd generates records.
2. Records are written to `/var/log/audit/audit.log`.
3. Wazuh Logcollector reads the configured log source.
4. Wazuh decodes and analyzes the collected event.
5. Matching rules generate alerts that can be reviewed in Wazuh Discover.

A configured log source should not automatically be considered proof of successful ingestion. Analysts should validate actual events in the SIEM.

### 8. Understanding Wazuh Audit Alerts

Wazuh rules classify and describe events based on their decoded content and matching conditions.

For example, this lab produced an alert with:

```text
Rule ID: 80705
Description: Auditd: Configuration changed.
Level: 3
Groups: audit, audit_configuration
```

The underlying Auditd record was a `CONFIG_CHANGE` event showing the addition of the Lab 133 RPM execution watch rule.

This is an important distinction.

The alert confirmed that Wazuh processed an Auditd configuration-change event. It did not indicate that a package was installed.

When investigating a Wazuh alert, analysts should review:

- Rule ID and description.
- Event type.
- Event timestamp and ID.
- User identity.
- Executable and command arguments.
- Raw event content.
- Source log location.

If a decoded field is null, the raw event should be checked before concluding that the information is absent.

### 9. Correlating Package-Management Evidence

A reliable investigation should correlate several evidence sources instead of depending on one log or alert.

| Evidence source | Question answered |
|---|---|
| RPM inventory | Is the package currently installed? |
| RPM metadata | What is the package version and installation information? |
| RPM file listing | Which files belong to the package? |
| DNF history | Which package-management transactions were recorded? |
| DNF logs | What package-manager activity was logged? |
| Auditd records | Which package-management commands were executed? |
| Wazuh alerts | Which collected events matched detection rules? |

Consider a scenario in which an administrator runs a package installation command.

The investigation should establish:

1. Which account initiated the command?
2. Which package was requested?
3. Was the package already installed?
4. Did DNF record a transaction?
5. What was the transaction return code?
6. Which package changes were recorded?
7. Does the RPM database reflect the expected package state?
8. Did Auditd record the relevant execution?
9. Did Wazuh collect and process the event?

This approach helps distinguish an attempted operation from a completed package modification.

### 10. Important Investigative Distinctions

A package-management investigation should avoid several common assumptions.

#### Command execution is not installation success

A process execution event proves that a command was invoked. It does not independently prove that the requested operation completed.

#### Package presence is not proof of recent installation

A package appearing in `rpm -qa` establishes its current presence in the RPM database. It does not establish when or why it was installed.

#### DNF log activity is not always a package transaction

Repository metadata refreshes and dependency-resolution activity may appear in package-manager logs without a package being installed or removed.

#### An Auditd configuration alert is not a package alert

A `CONFIG_CHANGE` event can indicate that an audit rule was added or removed. It should not be interpreted as package installation activity.

#### Missing telemetry is not proof of missing activity

If an expected event is absent from Wazuh, investigate the Auditd rule, executable path, log source, collector status, event decoding, and detection rules before drawing a conclusion.

### 11. Investigation Workflow

A practical Linux package-management investigation can follow this sequence:

1. Identify the operating system and package-manager versions.
2. Collect the installed-package inventory.
3. Review DNF transaction history.
4. Examine relevant package-management logs.
5. Identify DNF and RPM executable paths.
6. Validate Auditd service and rule status.
7. Configure temporary execution monitoring where appropriate.
8. Perform controlled package-management activity.
9. Review Auditd records and command arguments.
10. Correlate transaction results with RPM package state.
11. Validate Wazuh collection and alert processing.
12. Remove temporary rules and verify cleanup.

### 12. Conclusion

Linux package-management investigations require correlation between package-manager records, package database information, process execution telemetry, and SIEM alerts.

DNF provides transaction context, RPM provides package-state and metadata information, Auditd records configured system activity, and Wazuh provides centralized event collection and analysis.

The most important principle is to distinguish what was attempted, what was recorded, and what was actually completed.

A defensible investigation follows the evidence rather than assuming that every package-management command represents a successful installation.


This lab explores how Linux package-management activity can be investigated using DNF, RPM, Auditd, and Wazuh on Rocky Linux 9.8.

The investigation focuses on identifying installed packages, reviewing package-manager history, examining RPM metadata, monitoring execution of package-management utilities, and validating whether Auditd telemetry is collected by Wazuh.

The exercise also demonstrates an important investigation principle: execution of a package-management command does not automatically establish that a package was installed or modified.

## Lab Objectives

### Primary Objectives

The primary objective of this exercise is to develop an investigation workflow for Linux package-management activity and understand how host-level evidence can be used to validate software installation or modification.

By completing this lab, the analyst should be able to:

1. **Identify the package-management environment**
   - Determine the Linux distribution and version.
   - Identify the installed DNF and RPM versions.
   - Locate the package-management executables.

2. **Establish the existing package state**
   - Collect the installed-package inventory.
   - Query individual package metadata.
   - Identify files associated with a package.
   - Determine which package owns a particular executable or file.

3. **Investigate package-management history**
   - Review DNF transaction history.
   - Examine individual transaction details.
   - Identify the command line, recorded user, timestamp, return code, and affected packages.
   - Separate historical transactions from activity performed during the investigation.

4. **Examine package-management logs**
   - Inspect available DNF log files.
   - Identify repository and package-manager activity.
   - Recognize the difference between metadata operations and package transactions.

5. **Monitor package-manager execution**
   - Validate Auditd service status.
   - Identify existing audit rules.
   - Configure temporary execution watches for DNF and RPM.
   - Understand the meaning of the execution permission and audit key.

6. **Analyze Auditd records**
   - Search records using an audit key.
   - Interpret relevant event types.
   - Examine executable names and command arguments.
   - Correlate records sharing an audit event ID.
   - Distinguish package queries from package modifications.

7. **Validate Wazuh collection**
   - Inspect the configured Auditd log source.
   - Verify Wazuh manager and log collector status.
   - Review Wazuh alert details.
   - Compare decoded fields with the original raw event.
   - Determine what the alert actually represents.

8. **Apply evidence-based conclusions**
   - Distinguish attempted operations from successful transactions.
   - Correlate DNF history with RPM package state.
   - Avoid attributing unrelated historical activity to the current investigation.
   - Record confirmed, unconfirmed, and unavailable findings separately.

9. **Perform controlled cleanup**
   - Remove only the temporary audit rules created for this exercise.
   - Preserve unrelated rules.
   - Verify the final Auditd rule state.

### Expected Learning Outcome

At the end of the exercise, the analyst should be able to explain how DNF, RPM, Auditd, and Wazuh contribute different types of evidence to a Linux package-management investigation.

The analyst should also be able to explain why an executable event or SIEM alert alone is insufficient to confirm a successful package installation.

## Lab Scenario

### Scenario

A Linux server is being reviewed as part of a routine security investigation. The security team wants to understand how software package-management activity can be identified and validated on the host.

Package installation and modification are important investigation areas because unauthorized software may introduce new tools, dependencies, executables, or services. A SOC analyst must be able to distinguish legitimate administrative activity from potentially unauthorized changes.

The system under investigation is a Rocky Linux 9.8 virtual machine. It uses DNF for high-level package management and RPM for package database operations.

The analyst is asked to establish the current package state, review historical package transactions, and determine how package-management command execution can be monitored through Linux Auditd.

### Investigation Situation

The initial review identifies an existing software package named `tree`.

The analyst checks the package using RPM and attempts to install it through DNF. However, DNF reports that the package is already installed and that no action is required.

This creates an important investigative distinction: a package-management command was executed, but the requested installation did not result in a new package installation.

To examine command execution telemetry, the analyst configures temporary Auditd execution watches for DNF and RPM using the key `lab133_package_mgmt`.

The analyst then reviews the generated records to understand which commands were executed and what arguments were supplied.

### Investigation Tasks

The analyst is expected to:

- Identify the Linux operating system and package-manager versions.
- Establish the existing installed-package inventory.
- Review DNF history and relevant transaction details.
- Examine package-management logs.
- Validate the installed state and metadata of the selected package.
- Configure and verify temporary Auditd execution rules.
- Search the Auditd log for package-management activity.
- Identify the difference between package queries and installation operations.
- Validate whether Wazuh is collecting the Auditd log.
- Inspect any matching Wazuh alert and determine what the underlying event represents.
- Remove the temporary audit rules without affecting unrelated monitoring rules.

### Evidence Available

The investigation uses the following sources:

| Evidence | Purpose |
|---|---|
| RPM package inventory | Establishes the current installed-package state |
| DNF history | Provides historical package transaction details |
| DNF logs | Provides package-manager and repository activity |
| RPM metadata queries | Provides package details and file ownership |
| Auditd records | Provides process execution and audit configuration events |
| Wazuh configuration | Establishes the configured Auditd log source |
| Wazuh Discover | Provides visibility into collected and analyzed events |

### Expected Analyst Approach

The analyst should correlate the available evidence rather than relying on a single command or alert.

A package being present in the RPM database does not establish that it was installed during the current investigation. Likewise, an RPM execution event may represent a query rather than a package modification.

The analyst must validate transaction history, command arguments, package metadata, and raw audit records before determining what happened.

Wazuh alerts should also be interpreted according to their actual event type and underlying log content. An Auditd configuration-change alert should not be treated as a package-installation alert.

### Investigation Outcome

The exercise confirms that the `tree` package was already installed and that the DNF installation request resulted in no package changes.

Auditd recorded RPM query execution under the temporary Lab 133 monitoring key. Wazuh also processed an Auditd configuration-change event associated with adding the RPM execution watch rule.

The exercise did not confirm a new package installation or a Wazuh alert specifically identifying a successful package installation.

The scenario demonstrates the importance of validating the difference between command execution, package-manager transaction results, actual package state, and SIEM alert classification.

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

