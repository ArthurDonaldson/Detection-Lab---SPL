# Lab setup and troubleshooting log

## Environment

- Host: personal Windows laptop, 16 GB RAM, WSL2 installed
- Hypervisor: VMware Workstation Pro (free for personal use since Nov 2024), chosen over VirtualBox because it coexists with Hyper-V/WSL2 rather than falling back to slow emulation
- Guest: Windows 11 Enterprise evaluation (90-day, Microsoft Evaluation Center), 4 GB RAM, 2 vCPU, 60 GB disk, virtual TPM + Secure Boot enabled (required for Windows 11 install)
- Splunk Enterprise on the host (60-day full trial, drops to Splunk Free after — Free does not support alerting, see note below)
- Splunk Universal Forwarder on the VM

## Build order

1. Installed VMware Workstation Pro, created a host-only network (VMnet1) for the lab, kept a NAT adapter for initial OS setup and downloads.
2. Built the Windows 11 VM on NAT, since Windows Setup and tool downloads need internet access.
3. Installed VMware Tools, took a snapshot.
4. Installed Splunk Enterprise on the host, enabled the receiving port (Settings → Forwarding and receiving → Configure receiving → 9997), added a Windows Firewall inbound rule for TCP 9997.
5. Installed Sysmon with SwiftOnSecurity's config on the VM.
6. Installed the Splunk Universal Forwarder on the VM, pointed at the host's VMnet1 address on port 9997.

## Issue 1: Sysmon never actually installed

**Symptom:** Forwarder connected fine, `index=main` and even `index=_internal` under a bare `index=*` search showed nothing.

**Wrong turns first:** spent a long time checking Splunk-side config (inputs.conf syntax, receiving port, license, index existence, forwarder logs) before checking the most upstream link in the chain — whether Sysmon was producing any events at all.

**Root cause:** the install command (`Sysmon64.exe -accepteula -i sysmonconfig-export.xml`) was run from a non-elevated Command Prompt and failed silently. Nothing in the terminal made this obvious at the time.

**Fix:** re-ran from an actual elevated prompt (title bar confirmed "Administrator: Command Prompt"). Installed cleanly, Event Viewer under `Applications and Services Logs → Microsoft → Windows → Sysmon → Operational` started populating immediately.

## Issue 2: forwarder could not read the Sysmon channel (error code 5)

**Symptom:** Sysmon was confirmed running and logging in Event Viewer, but the events still never reached Splunk.

**Diagnosis:** live-tailed the forwarder's own log —

```powershell
Get-Content "C:\Program Files\SplunkUniversalForwarder\var\log\splunk\splunkd.log" -Tail 20 -Wait
```

— and forced a fresh subscription attempt by restarting the service (`net stop SplunkForwarder` / `net start SplunkForwarder`) while watching. Found:

```
ERROR ExecProcessor - message from "...splunk-winevtlog.exe" splunk-winevtlog -
WinEventLogChannel::init: Init failed, unable to subscribe to Windows Event Log
channel 'Microsoft-Windows-Sysmon/Operational': errorCode=5
```

Error 5 is Windows' `ERROR_ACCESS_DENIED`.

**Root cause:** the Universal Forwarder service was running as the virtual service account `NT SERVICE\SplunkForwarder`, not as Local System. The Sysmon channel's security descriptor (checked with `wevtutil get-log Microsoft-Windows-Sysmon/Operational`) explicitly grants Local System (`SY`), Administrators (`BA`), and a couple of other built-in SIDs — but not the per-service virtual account.

**Fix:** changed the SplunkForwarder service's logon account to Local System (`services.msc` → SplunkForwarder → Log On tab → Local System account), restarted the service. The subscription error was gone on the next restart and events began arriving.

**Lesson:** a channel's ACL and the account a service actually runs under are two independent things worth checking separately — matching one without checking the other still fails silently (a connection can be fully live with no data moving).

## Issue 3: forwarder stopped connecting after switching NAT → host-only

**Symptom:** everything above was working. After switching the VM's network adapter from NAT to host-only (to isolate attack simulation traffic from the network), the forwarder stopped delivering data. Restarting the forwarder service and Splunk itself did nothing.

**Diagnosis path:**
- Confirmed IP addressing was correct on both sides: VM at `192.168.22.128/24` (via `ipconfig`), host's VMnet1 at `192.168.22.1/24` — same subnet, `outputs.conf` on the VM already pointed at the right address.
- Ran `Test-NetConnection -ComputerName 192.168.22.1 -Port 9997` from the VM: both the ping and the TCP test timed out (not refused — timed out, which points to something silently dropping the traffic rather than actively rejecting it).
- Confirmed both VMware virtual adapters (VMnet1, VMnet8) were `Up` on the host.

**Root cause (inferred, not fully confirmed):** the original Windows Firewall rule for port 9997 was scoped to the Domain and Private network profiles only. Switching the VM's adapter mode likely caused Windows to reclassify the VMnet1 host-only network, probably to Public, which that rule did not cover. (Confirming this precisely would need `Get-NetConnectionProfile` on the host, checked against the VMnet1 interface alias — noted for a future pass.)

**Fix:** replaced the profile-scoped rule with one scoped by source address instead, which works regardless of profile classification:

```powershell
New-NetFirewallRule -DisplayName "Splunk 9997 VMnet1" -Direction Inbound -Protocol TCP `
    -LocalPort 9997 -RemoteAddress 192.168.22.0/24 -Profile Any -Action Allow
```

Scoping by `-RemoteAddress` to the lab's own /24 keeps this safe to leave enabled even while the host is on an untrusted network (e.g. school wifi), since nothing outside that private subnet can reach the rule.

**Lesson:** `TIME_WAIT` vs `LISTENING` vs a timeout in `netstat`/`Test-NetConnection` output distinguishes "nothing is listening," "something rejected the connection," and "something is silently dropping it" — worth learning to read precisely rather than guessing at causes to try.

## Splunk licensing note

Splunk Enterprise installs as a full 60-day trial and then auto-downgrades to Splunk Free. Free does **not** support alerting, which this project depends on for every detection. Trial started mid-September 2026; needs a developer license or similar applied before it lapses in mid-November, or all saved searches stop firing on schedule (they still run manually).

## Open items / next steps

- [ ] Confirm the Public-profile explanation for Issue 3 with `Get-NetConnectionProfile`
- [ ] Apply for Splunk developer license before the trial ends (~mid-November 2026)
- [ ] Build the domain controller, join the workstation, enable Advanced Audit Policy + PowerShell script block logging
- [ ] Convert working SPL detections to Sigma with `sigma-cli`
