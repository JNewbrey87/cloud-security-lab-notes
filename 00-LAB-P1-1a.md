# Lab 1: Wazuh Agent Deployment and My First Detection Rule

**Project:** P1, Wazuh SIEM Home Lab  
**Phase:** 1a | **Platform:** Hyper-V on Windows 11 Pro (May 2026; the lab later moved to VirtualBox on new hardware)  
**MITRE ATT&CK:** T1110 — Brute Force (Credential Access)  
**Cert Alignment:** CySA+ CS0-004, Domain 1 (Security Operations); SC-200  
**Date Completed:** May 2026  

---

## Why I Did This One

This was the first real lab in my homelab, and the point where it went from a few VMs to an actual security tool. Windows servers are part of my day job, and event logs aren't new to me. What I hadn't done was build the detection side myself: get an agent shipping Windows security events to a SIEM I set up, then write my own rule that fires on something real.

I picked Windows Event ID 4625, failed logons, because it's about as basic as detections get, and I wanted to understand the whole pipeline end to end before trying anything fancier: audit policy, the Security log, the agent forwarding events, and the manager evaluating them against my rule.

---

## Architecture

```
[Windows 11 Host]                    [Ubuntu 22.04 VM]
  [WazuhSvc - Agent]     ──────────►  [Wazuh Manager v4.14.5-1]
                                        /var/ossec/
  vEthernet (LabInternal) ◄────────   static IP, internal switch
  192.168.10.0/24
```

**Why an internal switch instead of the Default Switch.** This was the main networking fix from my early setup. Hyper-V's Default Switch is NAT with DHCP, and across reboots my host and VMs kept landing in different subnets, which broke host-to-VM connectivity completely. Moving everything Wazuh-related onto an isolated internal switch with static IPs fixed it: no DHCP, no NAT, no subnet drift between sessions.

| Network | Used for | Wazuh traffic? |
|---------|----------|----------------|
| Default Switch (NAT, DHCP) | Internet access for apt and downloads | No; unstable across reboots |
| Internal switch (static) | Agent-to-manager communication | Yes |

---

## What I Built

### 1. The Wazuh Manager

The all-in-one Wazuh installer wasn't viable on the hardware I had at the time; it didn't have the disk or RAM for the full stack. So I installed just the manager from the Wazuh apt repository and deferred the dashboard and indexer until I upgraded hardware. That meant everything in this lab happened at the command line.

### 2. Enhanced Session Mode on the Ubuntu VM

I wanted clipboard passthrough between my host and the Ubuntu VM, so I installed xrdp and the Hyper-V integration packages. On Ubuntu 22.04 those packages are `linux-tools-virtual` and `linux-cloud-tools-virtual`:

```bash
# Ubuntu VM
sudo apt install -y xrdp linux-tools-virtual linux-cloud-tools-virtual

# Host (elevated PowerShell)
Set-VM -VMName "<ubuntu-vm-name>" -EnhancedSessionTransportType HvSocket
```

### 3. Putting the Host on the Internal Switch

I added the Windows 11 host to the internal switch in Hyper-V Manager and gave it a static IP on the same subnet as the manager:

```powershell
New-NetIPAddress -InterfaceAlias "vEthernet (LabInternal)" -IPAddress 192.168.10.1 -PrefixLength 24
```

I pinged the manager before moving on. If that doesn't work, nothing after it will.

### 4. Installing and Registering the Agent

I installed the agent on the Windows 11 host itself, pointed at the manager's internal-switch address. The agent version has to match the manager version exactly, or enrollment fails without much of an explanation.

```powershell
msiexec.exe /i wazuh-agent-4.14.5-1.msi /q WAZUH_MANAGER="192.168.10.10"

cd "C:\Program Files (x86)\ossec-agent"
.\agent-auth.exe -m 192.168.10.10
NET START WazuhSvc
```

Then I confirmed it showed up on the manager side:

```bash
sudo /var/ossec/bin/agent_control -l
# ID: 001, Active
```

### 5. Turning On Failure Auditing

Event 4625 only gets written if Windows is set to audit failed logons. If it isn't, the agent can be working perfectly and you'll still never see a single alert, which looks a lot like Wazuh being broken.

```powershell
auditpol /set /subcategory:"Logon" /failure:enable
auditpol /get /subcategory:"Logon"   # confirm it applied
```

### 6. Firewall Rules for the Agent Ports

I added outbound rules for the agent's two ports, 1514 (events) and 1515 (enrollment), scoped to all network profiles:

```powershell
New-NetFirewallRule -DisplayName "Wazuh Agent - Events (UDP 1514)" `
  -Direction Outbound -Protocol UDP -LocalPort 1514 -Action Allow -Profile Any

New-NetFirewallRule -DisplayName "Wazuh Agent - Auth (TCP 1515)" `
  -Direction Outbound -Protocol TCP -LocalPort 1515 -Action Allow -Profile Any
```

Looking back, I'd write these with `-RemotePort` instead of `-LocalPort`. 1514 and 1515 are the ports on the manager's side, and the agent's own outbound port is random. Windows Firewall also allows outbound traffic by default, so these rules were more insurance than a requirement.

### 7. My First Custom Rule

I wrote the rule in `/var/ossec/etc/rules/local_rules.xml` on the manager:

```xml
<group name="local,">
  <rule id="100002" level="10">
    <if_group>windows</if_group>
    <field name="win.system.eventID">^4625$</field>
    <description>Custom: Windows failed logon attempt detected (Event 4625)</description>
    <mitre>
      <id>T1110</id>
    </mitre>
  </rule>
</group>
```

A few choices I made:

- **Rule ID 100002.** Custom rules live in the 100000+ range so they don't collide with Wazuh's built-in rules. The default `local_rules.xml` already ships with an example rule at 100001, and a duplicate ID gets silently dropped, so I started at 100002.
- **Matching on the event ID directly.** `<if_group>windows</if_group>` plus an anchored match on `eventID` was simpler and more reliable for me than depending on Wazuh's built-in authentication-failure rule chain.
- **Level 10.** High priority, worth an analyst looking right away.

```bash
sudo systemctl restart wazuh-manager
```

---

## What I Detected

I generated a failed logon on purpose with a user that doesn't exist:

```powershell
runas /user:fakeuser123 cmd
# enter any password; it fails and writes Event 4625
```

Then I watched the alerts log on the manager:

```bash
tail -f /var/ossec/logs/alerts/alerts.log | grep "100002"
```

Rule 100002 fired on the very first try, with the MITRE T1110 tag attached. That confirmed the full pipeline: Windows Security log, agent, internal switch, manager, my rule, alert.

---

## What I Learned

**The networking problem looked like a Wazuh problem.** When the agent couldn't reach the manager, the errors made it look like a Wazuh configuration or authentication issue. The actual cause was the Default Switch putting my host and VM on different subnets after a reboot. Wazuh never even got a chance to fail. Basic connectivity is worth checking before blaming the application.

**Audit policy comes first.** Without failure auditing turned on, Event 4625 never exists, so there's nothing to forward. I'd put it at the top of any Windows Wazuh agent checklist.

**Duplicate rule IDs fail silently.** A second rule with an ID that's already in use doesn't throw an error. One of them just disappears. Starting at 100002 avoids the example rule in the default file.

**Don't run `apt full-upgrade` on a Hyper-V Kali VM.** I didn't need Kali for this lab, but I learned this one the hard way during the same session: a full upgrade replaced display components the Hyper-V framebuffer didn't like, and I got a black screen. Twice. `apt upgrade` only.

---

## Lab Environment

| Component | Details |
|-----------|---------|
| Host | Windows 11 Pro, Hyper-V (May 2026 hardware) |
| SIEM Manager | Ubuntu 22.04 LTS, Wazuh v4.14.5-1 (manager only; dashboard and indexer added after the hardware upgrade) |
| Wazuh Agent | Windows 11 host, v4.14.5-1, agent ID 001 |
| Network | Isolated internal Hyper-V switch, static IPs |

---

## What Came Next

- **[Lab 1b](00-LAB-P1-1b.md):** SSH brute force detection, with a Kali attacker VM, Hydra, and a frequency-based rule.
- **[Lab 1c](00-LAB-P1-1c.md):** a tiered Active Directory lab (lab.local forest, Tier 0 DC, Tier 1 jump box), with rule 100002 running on a domain-joined host.
- **[Lab 1d](00-LAB-P1-1d.md):** a two-stage kill chain, detecting lateral movement with rules 100004 and 100005.
- **[P2-1a](00-LAB-P2-1a.md):** Azure Arc onboarding and a cloud security baseline for the same lab machines.
- All of the custom rules from this series are collected in [`detection-rules/local_rules.xml`](detection-rules/local_rules.xml).

---

*Part of [cloud-security-lab-notes](https://github.com/JNewbrey87/cloud-security-lab-notes) | Joshua E. Newbrey | [LinkedIn](https://www.linkedin.com/in/jnewbrey87/) | [GitHub Profile](https://github.com/JNewbrey87)*
