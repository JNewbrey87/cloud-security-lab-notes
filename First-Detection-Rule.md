# Lab 1: Wazuh Agent Deployment & First Detection Rule

**Project:** P1 — Wazuh SIEM Home Lab  
**Phase:** 1 | **Platform:** Hyper-V on Windows 11 Pro  
**MITRE ATT&CK:** T1110 — Brute Force (Credential Access)  
**Cert Alignment:** CySA+ CS0-004, SC-200  
**Date Completed:** May 2026

---

## Scenario

A SIEM manager with no agents is just a very expensive log file. Lab 1 is where the home lab stops being a VM setup and becomes an actual security tool: a Wazuh agent is deployed onto a Windows endpoint, a stable agent-manager communication path is established, and a custom detection rule is written to catch real Windows security events.

The detection target for this lab is Windows Event ID 4625, failed logon attempts, mapped to MITRE T1110 (Brute Force). This is a foundational detection present in virtually every SOC environment. The objective is not simply to see an alert fire; it is to understand the full detection pipeline from audit policy configuration through Windows Security log collection, agent forwarding, and manager-side rule evaluation.

---

## Architecture

```
[Windows 11 Host]                    [Ubuntu 22.04 VM]
  [WazuhSvc - Agent]     ──────────►  [Wazuh Manager v4.14.5-1]
                                        /var/ossec/
  vEthernet (Internal)   ◄──────────   eth1 (static IP, internal switch)
  isolated 192.xx.xx.0/24 subnet
```

> Real IPs, hostnames, and usernames are omitted from this README per standard home lab portfolio security practice. Substitute values appropriate to your own environment.

**Network Architecture Decision: Internal Switch over Default Switch**

Hyper-V's Default Switch is NAT-based with DHCP assignment. Across reboots, the host and VMs receive addresses in different subnets, which breaks host-to-VM connectivity entirely. An isolated internal switch with static IPs resolves this; there is no DHCP, no NAT, and no subnet drift between sessions. All Wazuh agent-manager traffic is routed through it.

| Network | Use Case | Wazuh Traffic? |
|---------|----------|----------------|
| Default Switch (NAT, DHCP) | Internet access for apt/downloads | No; unstable across reboots |
| Internal Switch (static) | Agent-manager communication | Yes |

---

## What I Built

### 1. Enhanced Session Mode

Installed `xrdp` and the Ubuntu Hyper-V integration packages on the Ubuntu VM to enable clipboard passthrough between host and VM. On Ubuntu 22.04, the correct packages are `linux-tools-virtual` and `linux-cloud-tools-virtual`; the package name `hyperv-daemons` exists only on Fedora and RHEL-based distributions and is not available in Ubuntu repositories.

```bash
# Ubuntu VM
sudo apt install -y xrdp linux-tools-virtual linux-cloud-tools-virtual
sudo systemctl enable xrdp
sudo adduser xrdp ssl-cert

# Host (elevated PowerShell)
Set-VM -VMName "<ubuntu-vm-name>" -EnhancedSessionTransportType HvSocket
```

After rebooting the Ubuntu VM, close the Hyper-V connection window and reconnect. A display resolution prompt confirms Enhanced Session Mode is active.

### 2. Host Added to Internal Switch

The Windows 11 host was added to the internal virtual switch via Hyper-V Manager by enabling the management operating system adapter option, then assigned a static IP from the same subnet as the Wazuh manager.

```powershell
New-NetIPAddress -InterfaceAlias "vEthernet (<switch-name>)" -IPAddress 192.xx.xx.xx -PrefixLength 24
```

Connectivity to the manager should be verified with a ping test before proceeding. If this step fails, nothing downstream will work.

### 3. Wazuh Agent Installation

The version-matched MSI was downloaded and installed silently with the manager IP embedded at install time. The agent version must match the manager version exactly; mismatched versions result in silent enrollment failures.

```powershell
Invoke-WebRequest -Uri "https://packages.wazuh.com/4.x/windows/wazuh-agent-4.14.5-1.msi" `
  -OutFile "$env:TEMP\wazuh-agent.msi"

msiexec /i "$env:TEMP\wazuh-agent.msi" `
  WAZUH_MANAGER="<manager-ip>" `
  WAZUH_AGENT_NAME="<agent-name>" /q
```

The agent was enrolled via `agent-auth.exe`, which performs the one-time registration handshake with the authd service on port 1515.

```powershell
& "C:\Program Files (x86)\ossec-agent\agent-auth.exe" -m <manager-ip>
# Expected output: INFO: Valid key created. Finishing.
```

Agent registration was confirmed on the manager side.

```bash
sudo /var/ossec/bin/agent_control -l
# ID: 001, Name: <agent-name>, IP: any, Active
```

### 4. Windows Audit Policy Configuration

Event ID 4625 is only generated if Windows is configured to audit failed logon attempts. This is not enabled by default on all systems, and its absence is a common reason analysts assume Wazuh is malfunctioning when the agent is actually forwarding events correctly.

```powershell
auditpol /get /subcategory:"Logon"            # verify current setting
auditpol /set /subcategory:"Logon" /failure:enable
```

### 5. Windows Firewall Rules

The internal switch adapter is classified as a Public or Unidentified network profile in Windows. The `Set-NetConnectionProfile` cmdlet cannot modify the profile on Hyper-V virtual adapters. The workaround is to add outbound firewall rules scoped to all profiles rather than attempting to change the network category.

```powershell
New-NetFirewallRule -DisplayName "Wazuh 1514 UDP" -Direction Outbound -Protocol UDP -RemotePort 1514 -Action Allow -Profile Any
New-NetFirewallRule -DisplayName "Wazuh 1515 TCP" -Direction Outbound -Protocol TCP -RemotePort 1515 -Action Allow -Profile Any
```

### 6. Custom Detection Rule

The first custom detection rule was written in `/var/ossec/etc/rules/local_rules.xml`. It targets Event ID 4625 directly on the eventID field using a regex anchor, which is more reliable than inheriting from the `authentication_failed` parent group, as the latter requires a chain of built-in rules to evaluate first.

```xml
<group name="local,windows,authentication_failed,">
  <rule id="100002" level="10">
    <if_group>windows</if_group>
    <field name="win.system.eventID">^4625$</field>
    <description>Windows failed logon detected (Event 4625)</description>
    <mitre>
      <id>T1110</id>
    </mitre>
  </rule>
</group>
```

Key decisions in the rule design:

- **Rule ID 100002**: The default `local_rules.xml` template already contains an example rule using ID 100001. Duplicate IDs cause one rule to be silently dropped; custom rules should start at 100002 or higher.
- **Parent condition**: `<if_group>windows</if_group>` provides a broader, more stable parent match than `authentication_failed`.
- **Alert level 10**: High priority; warrants immediate analyst attention.
- **MITRE T1110**: Brute Force technique, Credential Access tactic.

```bash
sudo systemctl restart wazuh-manager
```

---

## What I Detected

Failed logon attempts were generated on the Windows endpoint to trigger the detection rule.

```powershell
runas /user:nonexistentuser cmd
# Enter any incorrect password when prompted; repeat 2-3 times
```

The alert was confirmed in `/var/ossec/logs/alerts/alerts.log` within seconds.

```
Rule: 100002 (level 10) -> 'Windows failed logon detected (Event 4625)'

win.system.eventID: 4625
win.system.severityValue: AUDIT_FAILURE
win.eventdata.targetUserName: nonexistentuser
win.eventdata.status: 0xc000006d       # bad username or wrong password
win.eventdata.logonType: 2             # interactive logon attempt
win.eventdata.authenticationPackageName: Negotiate
```

The full pipeline was confirmed: Windows Security log, Wazuh agent, internal switch, manager, custom rule evaluation, alert generation.

---

## What I Learned

**The Default Switch creates a network topology problem, not a Wazuh problem.** The error message when the agent cannot reach the manager on port 1515 looks like a Wazuh configuration or authentication failure. In this case, the root cause was that the Hyper-V Default Switch had assigned the host and the Ubuntu VM to different subnets after a reboot, breaking basic IP connectivity before Wazuh was ever involved. An isolated internal switch with static IPs is the correct solution for lab environments where stable agent-manager communication is required.

**Audit policy is a prerequisite, not an optional step.** If `auditpol` is not configured for failure auditing, Event ID 4625 is never generated and never forwarded. The agent can be functioning correctly and the alerts log will remain empty. Verifying audit policy should be part of any Wazuh Windows agent checklist.

**Avoid `apt full-upgrade` on Hyper-V Kali images.** A full upgrade replaces display and graphics components that are incompatible with the Hyper-V framebuffer. The VM continues running but presents a black screen. The safe approach is `apt upgrade` only, which skips the problematic package replacements.

**Duplicate rule IDs in `local_rules.xml` cause silent failures.** The default template file includes an example rule using ID 100001. Adding a second rule with the same ID results in one being silently discarded; the analyst gets no error and no alert. Replacing the full file contents rather than appending, and using IDs starting at 100002, avoids this entirely.

**`hyperv-daemons` is not available on Ubuntu.** The Fedora and RHEL package by that name does not have a direct Ubuntu equivalent by the same name. The equivalent functionality on Ubuntu 22.04 is provided by `linux-tools-virtual` and `linux-cloud-tools-virtual`.

---

## Lab Environment

| Component | Details |
|-----------|---------|
| Host | Windows 11 Pro, Hyper-V enabled |
| SIEM Manager | Ubuntu 22.04 LTS, Wazuh v4.14.5-1 (manager only; indexer and dashboard deferred to hardware upgrade) |
| Wazuh Agent | Windows 11 host, v4.14.5-1, ID 001 |
| Network | Isolated internal Hyper-V switch, static IPs |

---

## Files

| File | Description |
|------|-------------|
| `local_rules.xml` | Custom Wazuh detection rules (rule 100002) |
| `agent_install.ps1` | PowerShell script: agent download, silent install, enrollment |
| `network_setup.ps1` | PowerShell script: internal switch adapter and static IP assignment |

---

## Up Next

- **Lab 1 Extension:** SSH brute force detection using the Kali attacker VM; Hydra against the Ubuntu manager SSH service; custom Wazuh detection rule.
- **Lab 2:** Microsoft Sentinel and KQL Detection Engineering; Azure free tenant, Log Analytics Workspace, data connector configuration.

---

*Part of the [azure-security-homelab](https://github.com/JNewbrey87/azure-security-homelab) portfolio, building toward Cloud Security Analyst and SC-200/CySA+/AZ-500 certifications.*
