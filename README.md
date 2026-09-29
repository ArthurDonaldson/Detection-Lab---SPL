# T1053.005 — Scheduled Task: schtasks.exe creation

## Goal

Detect persistence or execution established via the Windows Task Scheduler through the `schtasks.exe` command-line utility, per [MITRE ATT&CK T1053.005](https://attack.mitre.org/techniques/T1053/005/).

## Data source

Sysmon Event ID 1 (process creation), forwarded via Splunk Universal Forwarder to `index=main`.

## Detection logic

```spl
index=main source="WinEventLog:Microsoft-Windows-Sysmon/Operational" EventCode=1
    Image="*\\schtasks.exe" (CommandLine="*/create*" OR CommandLine="*-create*")
| table _time, host, User, ParentImage, ParentCommandLine, CommandLine
```

- `Image="*\\schtasks.exe"` — scoped to the specific binary rather than a bare `schtasks` string match, to reduce accidental matches on unrelated command lines that happen to contain the word.
- `CommandLine="*/create*" OR CommandLine="*-create*"` — `schtasks.exe` accepts the create flag with either a slash or a dash; an earlier version of this rule only checked for `/create` and would have missed the dash form.
- `ParentImage` / `ParentCommandLine` included in the output because the parent process is often the more interesting signal — e.g. a task created from a macro-enabled Office document's child process versus an interactive admin session.

## Validation

Ran with [Atomic Red Team](https://github.com/redcanaryco/atomic-red-team) T1053.005 tests, on the VM with the network adapter set to host-only (no internet route) to keep simulation traffic off any real network.

```powershell
Import-Module "C:\AtomicRedTeam\invoke-atomicredteam\Invoke-AtomicRedTeam.psd1" -Force
Invoke-AtomicTest T1053.005 -ShowDetailsBrief
Invoke-AtomicTest T1053.005 -TestNumbers <n>
```

See the [coverage matrix](README.md#coverage-matrix-t1053005) in the README for per-test results.

Cleaned up after each run:

```powershell
Invoke-AtomicTest T1053.005 -TestNumbers <n> -Cleanup
```

## Known gaps

- **PowerShell `Register-ScheduledTask`** never launches `schtasks.exe`, so this rule does not see it. This is the same technique achieved a different way, and it's the most likely real-world evasion of this specific rule. Planned as a separate detection (either against the Event ID 1 command line for `powershell.exe` + `Register-ScheduledTask`, or against PowerShell Script Block Logging / Event ID 4104 if enabled).
- **COM-based task creation** (e.g. via the Task Scheduler COM API directly, without touching `schtasks.exe` or the `ScheduledTasks` PowerShell module) would also evade this rule and isn't detected by any rule in this repo yet.
- A file-creation detection on Sysmon Event ID 11 for new files under `C:\Windows\System32\Tasks\` is planned as a lower-level backstop that any creation method has to touch, regardless of which tool was used.

## False positives

[Fill in after running the lab under normal use for a few days — note here anything that fired and how the rule was tuned, e.g. software installers or update mechanisms that call schtasks.exe legitimately.]

## Sigma conversion

[Add once converted with `sigma-cli` — track the .yml file alongside this write-up.]
