# Troubleshooting Notes

## 1. DNF Reported That the Package Was Already Installed

### Observation

The command:

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

### Cause

The `tree` package was already present in the RPM database.

### Resolution

The existing installation was treated as baseline evidence. The package was not reinstalled or removed merely to generate a transaction.

Validation commands:

```bash
rpm -q tree
rpm -qi tree
rpm -ql tree
```

### Lesson

Always verify the current package state before attempting an installation. A successful DNF command does not necessarily mean a package was installed.

---

## 2. No New Tree Installation Appeared in DNF History

### Observation

The DNF history did not show a new `tree` installation associated with the exercise.

### Cause

DNF reported that the package was already installed and had nothing to do.

### Resolution

The existing package state and historical DNF transactions were documented separately.

### Lesson

Do not attribute older package transactions to the current investigation. Correlate the transaction date, command line, package list, and return code.

---

## 3. Auditd Displayed an Old-Style Watch Warning

### Observation

When the execution watches were added, Auditd displayed:

```text
Old style watch rules are slower
```

### Cause

The `-w` syntax is the legacy Auditd watch-rule format.

### Resolution

The rules were retained for this controlled exercise because they appeared in the active rule list:

```bash
auditctl -l
```

### Lesson

A warning does not automatically mean that a rule failed. Validate whether the rule was loaded and whether expected events were recorded.

---

## 4. Initial Audit Rule Deletion Failed

### Observation

The following commands failed:

```bash
auditctl -W /usr/bin/dnf -k lab133_package_mgmt
auditctl -W /usr/bin/rpm -k lab133_package_mgmt
```

Auditd returned:

```text
Error sending delete rule data request (No such file or directory)
There was an error while processing parameters
```

### Cause

The deletion arguments did not match the active watch-rule definitions.

The installed rules included the execution permission:

```text
-w /usr/bin/dnf -p x -k lab133_package_mgmt
-w /usr/bin/rpm -p x -k lab133_package_mgmt
```

### Resolution

The exact rule parameters were included during deletion.

DNF rule:

```bash
auditctl -W /usr/bin/dnf -p x -k lab133_package_mgmt
```

RPM rule:

```bash
auditctl -W /usr/bin/rpm -p x -k lab133_package_mgmt
```

After removing the DNF rule, the rule list was checked before removing the RPM rule.

Final verification:

```bash
auditctl -l | grep -E 'lab133_package_mgmt'
```

The final check returned no matching output.

### Lesson

Audit rules should be removed using parameters that match their installed definitions. Avoid using `auditctl -D` because it removes all active audit rules, including rules unrelated to the current lab.

---

## 5. Wazuh Displayed a Null Audit Key Field

### Observation

The Wazuh alert contained:

```text
data.audit.key: null
```

However, the raw event in `full_log` contained:

```text
type=CONFIG_CHANGE
op=add_rule
key="lab133_package_mgmt"
```

### Explanation

The displayed event was a `CONFIG_CHANGE` record. Its decoded fields did not expose the key in `data.audit.key`, even though the raw log contained the key.

### Resolution

The raw `full_log`, event type, event ID, rule description, and source log location were reviewed together.

### Lesson

A null decoded field does not always mean the source log lacks the value. Inspect the raw event before concluding that information is missing.

---

## 6. Wazuh Alert Was About Audit Configuration, Not Package Installation

### Observation

The observed alert was:

```text
Rule ID: 80705
Description: Auditd: Configuration changed.
Level: 3
Groups: audit, audit_configuration
```

### Explanation

The raw record showed the addition of the `lab133_package_mgmt` audit rule.

This was an Auditd configuration-change event, not a package installation event.

### Resolution

The alert was documented according to its actual event type and raw log content.

### Lesson

Do not interpret an alert based only on its presence or its relationship to a lab key. Validate the underlying event type and operation.

---

## 7. RPM Query Events Were Not Package Installation Events

### Observation

Audit records included RPM commands such as:

```text
rpm -q tree
rpm -ql tree
rpm -qf /usr/bin/tree
```

### Explanation

These commands queried package information, listed package files, and identified file ownership.

They did not install, upgrade, or remove a package.

### Resolution

The commands were classified as package-query activity.

### Lesson

Executable monitoring records process execution. The command arguments and corroborating package-manager evidence are needed to determine what operation actually occurred.

---

## 8. DNF and RPM Executable Paths Were Different

### Observation

Path resolution showed:

```text
/usr/bin/dnf
/usr/bin/dnf-3
```

The RPM path resolved to:

```text
/usr/bin/rpm
```

### Explanation

The DNF command path resolved to `/usr/bin/dnf-3`, while RPM resolved to its own executable path.

### Resolution

Both the command path and resolved executable path were recorded during investigation.

### Lesson

Validate executable paths before designing Auditd monitoring. Path resolution can affect how executable watches should be interpreted and tested.

---

## 9. Wazuh Collection Must Be Validated Separately

### Observation

The Wazuh configuration contained an Auditd localfile entry, and the Wazuh manager and log collector were running.

### Resolution

The following checks were used:

```bash
grep -n -A 8 -B 3 "/var/log/audit/audit.log" /var/ossec/etc/ossec.conf
systemctl status wazuh-manager --no-pager
pgrep -af wazuh-logcollector
```

The Wazuh Discover event provided evidence that an Auditd configuration-change record reached the alerting pipeline.

### Lesson

A configured log source and a running collector are useful validation points, but the actual event must also be inspected in Wazuh.

---

