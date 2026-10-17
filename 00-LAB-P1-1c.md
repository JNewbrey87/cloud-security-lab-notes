# Lab 1c: Tiered AD Build and Failed Logon Detection on a Domain Host

**Project:** P1, Wazuh SIEM Home Lab  
**Phase:** 1c | **Platform:** VirtualBox on Windows 11 Pro  
**MITRE ATT&CK:** [T1110 — Brute Force](https://attack.mitre.org/techniques/T1110/) (primary); [T1070.006 — Indicator Removal: Timestomp](https://attack.mitre.org/techniques/T1070/006/), [T1562 — Impair Defenses](https://attack.mitre.org/techniques/T1562/) (bonus finding)  
**Cert Alignment:** CySA+ CS0-004, Domain 1 (Security Operations); SC-200  
**Date:** June 2026  

---

## Why I Did This One

This was my first lab on new hardware. Between Lab 1b and this one, I moved the whole homelab to an ASUS laptop running VirtualBox, partly because of the Hyper-V issues I kept running into in the first two labs.

My first two labs also ran on flat, non-domain machines, and real environments run on Active Directory. So I built a small tiered AD setup: a Tier 0 domain controller and a Tier 1 jump box, following the same admin-tiering ideas used in production. Then I put a Wazuh agent on the domain-joined jump box and checked that my failed-logon rule from Lab 1 still worked there, fired every time, and kept working through an agent restart.

## Architecture

```
[Domain Controller: Tier 0]           [Jump Box: Tier 1]               [SIEM Manager]
  Windows Server 2022                   Windows Server 2025               Ubuntu 25.04
  AD DS + DNS, lab.local forest         Domain-joined to lab.local         Wazuh Manager v4.14.5
  Console-only, no RDP                  RDP (lab subnet only)              local_rules.xml
                                         SMB share                          alerts.log
                                         Wazuh Agent v4.14.5 ────────────► manager:1514/tcp
```

Everything sits on an isolated, host-only network with no outside exposure.

- **Tier 0, the domain controller:** the most trusted machine in the domain. Console access only, no remote management, since every other tier ultimately trusts it.
- **Tier 1, the jump box:** domain-joined, with RDP limited to the lab subnet and an SMB share. This is the machine I use for detection testing in the labs that follow.

## What I Built

### 1. The Domain Forest

I stood up a new single-domain forest, `lab.local`, on a fresh Windows Server 2022 VM:

```powershell
Install-ADDSForest -DomainName "lab.local"
Get-ADDomain   # confirms the forest and domain are healthy
```

I set a DNS forwarder to a public resolver for outside name resolution, and I left the DC console-only with no RDP. That took a few deliberate configuration choices, not just leaving a setting off.

### 2. The Jump Box

A second VM running Windows Server 2025 joined `lab.local` as the Tier 1 jump box. I enabled RDP scoped to the lab subnet and created an SMB share for the labs that follow. One lab shortcut worth naming: the lab VMs share a local admin credential. That's acceptable on an isolated network with no outside exposure, and it's not something I'd carry into anything real.

### 3. The Wazuh Agent

I deployed a Wazuh agent (v4.14.5) on the jump box with the dashboard's agent-install wizard, then checked three things:

- The agent showed **Active** on the manager.
- `ossec.conf` had the Security event channel enabled (`<location>Security</location>`, `<log_format>eventchannel</log_format>`), and nothing in the default exclusions filtered out Event 4625.
- The agent log showed a clean startup with no connection errors.

## What I Detected

### Rule 100002 on a Domain Host

```xml
<rule id="100002" level="10">
  <if_group>windows</if_group>
  <field name="win.system.eventID">^4625$</field>
  <description>Custom: Windows failed logon attempt detected (Event 4625)</description>
  <mitre>
    <id>T1110</id>
  </mitre>
</rule>
```

To generate test events, I repeatedly tried to start a process as an account that doesn't exist on the jump box, which writes Event 4625 to the Security log each time. Here's one of the alerts from `alerts.log`:

```
2026 Jun 14 16:58:13 (jump-box) any->EventChannel
Rule: 100002 (level 10) -> 'Custom: Windows failed logon attempt detected (Event 4625)'
win.system.eventID: 4625
win.system.channel: Security
win.system.computer: jump-box.lab.local
win.system.severityValue: AUDIT_FAILURE
win.eventdata.targetUserName: testuser
win.eventdata.targetUserSid: S-1-0-0
win.eventdata.logonType: 2
win.eventdata.failureReason: %%2313
win.eventdata.subStatus: 0xc0000064
win.eventdata.ipAddress: ::1
```

The `targetUserName` matching my test account is what tied this alert to my test, rather than to some older or unrelated event.

### 17 Firings, Including 3 After a Restart

At the end of the session I counted everything the jump box had generated:

```bash
sudo awk '/<jump-box> any->EventChannel/{getline; print}' /var/ossec/logs/alerts/alerts.log | sort | uniq -c | sort -rn
```

```
17 Rule: 100002 (level 10) -> 'Custom: Windows failed logon attempt detected (Event 4625)'
 2 Rule: 60642 (level 3)  -> 'Software protection service scheduled successfully.'
 1 Rule: 60132 (level 5)  -> 'System time changed'
```

Rule 100002 fired 17 times, once for every failed attempt. Three of those came after I manually restarted the Wazuh agent, which showed the pipeline picked right back up with no re-registration.

### A Bonus Finding: Rule 60132

At one point I changed the clock on the jump box by hand, and Wazuh's built-in rule 60132 (level 5) caught it. Clock changes map to MITRE T1070.006 (Timestomp) and T1562 (Impair Defenses), since messing with system time is a known way to muddy log timelines during an investigation. I wasn't testing for it, but it was a nice reminder that the default ruleset covers more than what I built myself.

## What I Took Away

**Tier 0 isolation is something you build on purpose.** Keeping the DC console-only took deliberate choices. It made it clear to me why tiered admin models treat the DC as its own trust class: every way into Tier 0 is a way an attacker would want too.

**One field can prove the alert is yours.** The `targetUserName` in the 4625 event is what connected this alert to my test. Without something like that, an alert from my test and an unrelated alert look the same in the log.

**Live tailing and an aggregate count answer different questions.** Early on, watching `alerts.log` with `tail -f | grep` showed nothing, and it looked like the rule might be broken. Counting across the whole log told the real story: 17 firings. The quiet tail was a buffering and timing issue, not a detection problem.

**Testing a restart was worth it.** I didn't expect the agent restart to break anything, but checking turned "it should recover" into "it did recover, three alerts later."

**Built-in rules can surprise you.** Rule 60132 wasn't the point of the lab, but it mapped cleanly to real ATT&CK techniques. Unplanned alerts are worth a look before writing them off as noise.

## Lab Environment

| Component | Details |
|-----------|---------|
| Host | Windows 11 Pro on the ASUS laptop, VirtualBox |
| SIEM Manager | Ubuntu 25.04, Wazuh v4.14.5 |
| Domain Controller (Tier 0) | Windows Server 2022, AD DS + DNS, `lab.local` forest, console-only |
| Jump Box (Tier 1) | Windows Server 2025, domain-joined, RDP + SMB share, Wazuh agent v4.14.5 |
| Network | Isolated host-only network, no external exposure |

## What Came Next

- **[Lab 1d](00-LAB-P1-1d.md):** a two-stage kill chain, from failed logons on the jump box to a lateral movement attempt toward the DC, detected by rules 100004 and 100005.
- **[P2-1a](00-LAB-P2-1a.md):** Azure Arc onboarding and a cloud security baseline for these same machines.

---

*Part of the [cloud-security-lab-notes](https://github.com/JNewbrey87/cloud-security-lab-notes) portfolio, building toward Cloud Security Analyst and the CySA+ / SC-200 / SC-500 certifications.*
