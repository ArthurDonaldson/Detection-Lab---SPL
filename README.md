# Detection Lab

A home detection engineering lab: Windows endpoint telemetry (Sysmon) forwarded to Splunk, with MITRE ATT&CK techniques simulated using Atomic Red Team and detections written, tested, and tuned against the resulting data.

The goal is not just to write rules, but to document what each rule misses and what it flags by mistake.

## Lab architecture

```
+-----------------------------+        TCP 9997        +--------------------------+
| Windows 11 VM (VMware)      |  ------------------->  | Host laptop              |
| - Sysmon (SwiftOnSecurity   |   host-only network    | - Splunk Enterprise      |
|   config)                   |     192.168.22.0/24    | - Indexer + search head  |
| - Splunk Universal Forwarder|                        |                          |
| - Atomic Red Team           |                        |                          |
+-----------------------------+                        +--------------------------+
```

- **Hypervisor:** VMware Workstation Pro
- **Endpoint:** Windows 11 Enterprise (evaluation), 4 GB RAM, 2 vCPU
- **Telemetry:** Sysmon with SwiftOnSecurity's config, collected through the Windows Event Log input
- **SIEM:** Splunk Enterprise on the host (receiving on TCP 9997)
- **Simulation:** Atomic Red Team, run with the VM on a host-only network
- **Domain controller:** [ADD WHEN BUILT, OR DELETE THIS LINE]

Build notes and troubleshooting log: [lab-setup.md](lab-setup.md)

## Detections

| Technique | Detection | Data source | Status |
|---|---|---|---|
| T1053.005 Scheduled Task | [schtasks.exe task creation](T1053.005-scheduled-task.md) | Sysmon Event ID 1 | Tested, gaps documented |
| T1053.005 Scheduled Task | PowerShell `Register-ScheduledTask` | [Event ID 1 / script block logging] | Planned |
| T1053.005 Scheduled Task | New file in `C:\Windows\System32\Tasks\` | Sysmon Event ID 11 | Planned |

## Coverage matrix: T1053.005

Each Atomic Red Team test for the technique, and which detections fired. Blank or "no" cells are documented gaps.

| Atomic test | Description | schtasks detection | PowerShell detection | File-create detection |
|---|---|---|---|---|
| [#] | [from `-ShowDetailsBrief`] | [fired / no] | [fired / no / not built] | [fired / no / not built] |
| [#] | | | | |
| [#] | | | | |

**Coverage:** [N] of [M] tests detected by at least one rule.

## Method

For each technique:

1. Read the Atomic Red Team test and predict what telemetry it should produce.
2. Run the test in the isolated VM and find the events in Splunk.
3. Write a detection scoped to one behavior.
4. Run every atomic test for the technique and record what fires.
5. Leave the lab running under normal use, record false positives, and tune.
6. Clean up (`Invoke-AtomicTest <technique> -Cleanup`) and document.

## Safety notes

Simulations run only on a host-only network with no internet route. No data from any employer or third party is included in this repository.

## References

- [MITRE ATT&CK T1053.005](https://attack.mitre.org/techniques/T1053/005/)
- [Atomic Red Team](https://github.com/redcanaryco/atomic-red-team)
- [SwiftOnSecurity sysmon-config](https://github.com/SwiftOnSecurity/sysmon-config)
- [SigmaHQ rules](https://github.com/SigmaHQ/sigma)
