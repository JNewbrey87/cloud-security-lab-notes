# Lab 1d: Detecting a Two-Stage Kill Chain with Rules 100004 and 100005

**Project:** P1, Wazuh SIEM Home Lab  
**Phase:** 1d | **Platform:** VirtualBox on Windows 11 Pro  
**MITRE ATT&CK:** [T1110 — Brute Force](https://attack.mitre.org/techniques/T1110/), [T1021.002 — Remote Services: SMB/Windows Admin Shares](https://attack.mitre.org/techniques/T1021/002/)  
**Cert Alignment:** CySA+ CS0-004, Domain 1 (Security Operations); SC-200  
**Date:** June 2026  

---

## Why I Did This One

My earlier labs each caught one thing: a failed Windows logon, or an SSH brute force burst. Each one saw a single stage of an attack. For this one, I wanted to chain two stages together and see whether my detections could tell the story end to end: a brute force attempt on the jump box, followed by someone using those credentials to try to reach the domain controller.

The real question I was after is whether Wazuh could see both halves of a lateral movement attempt. One half is the outbound credential use from the jump box, and the other is the failed network logon arriving at the DC. Catching both means neither side of the exchange is invisible, and that takes two different rules, two different Windows events, and an agent on the domain controller itself.

## Architecture

```
[LAB-WSJP-25: Tier 1 Jump Box]          [LAB-WSDC-22: Tier 0 DC]        [LAB-UBTU: Wazuh Manager]
  Windows Server 2025                     Windows Server 2022              Ubuntu 25.04
  Wazuh agent 002                         Wazuh agent 003                  Wazuh Manager v4.14.5
  RDP + SMB share                         AD DS + DNS, lab.local           local_rules.xml
  (intentional attack surface)            console-only, no RDP             (rules 100002-100005)

  Stage 1: failed logons        → Event 4625 → Rule 100002 (T1110, level 10)   [jump box]
  Stage 2: outbound cred use    → Event 4648 → Rule 100004 (T1021.002, level 12) [jump box]
  Stage 2: inbound failed logon → Event 4625 type 3 → Rule 100005 (T1021.002, level 14) [DC]
```

All hosts sit on an isolated, host-only VirtualBox network (192.168.10.0/24) with no outside exposure. The DC is Tier 0 and has no RDP, so any network logon attempt reaching it is significant on its own. That's why rule 100005 sits at level 14, higher than anything on the jump box.

## MITRE ATT&CK Coverage

| Technique | ID | Rule | Event ID | Host |
|-----------|-----|------|----------|------|
| [Brute Force](https://attack.mitre.org/techniques/T1110/) | T1110 | 100002 | 4625 | LAB-WSJP-25 |
| [Remote Services: SMB/Windows Admin Shares](https://attack.mitre.org/techniques/T1021/002/) | T1021.002 | 100004 | 4648 | LAB-WSJP-25 |
| [Remote Services: SMB/Windows Admin Shares](https://attack.mitre.org/techniques/T1021/002/) | T1021.002 | 100005 | 4625 (type 3) | LAB-WSDC-22 |

## What I Built

### 1. Rule 100004: Outbound Credential Use (Event 4648)

Windows writes Event 4648 when a process authenticates to a remote host using credentials that were supplied explicitly, which is the footprint left on the source machine by something like `net use` or `Invoke-Command` reaching for a domain resource. This rule fires on the jump box and captures the outbound side of the attempt.

```xml
<rule id="100004" level="12">
  <if_group>windows</if_group>
  <field name="win.system.eventID">^4648$</field>
  <description>Custom: Explicit credential use toward remote host detected - possible lateral movement (Event 4648)</description>
  <mitre>
    <id>T1021.002</id>
  </mitre>
</rule>
```

The choices behind it:
- **Level 12.** Explicit credential use toward a remote host isn't routine user activity.
- **Event 4648, not 4625.** This matters: 4625 is a failure record on the target, while 4648 is an attempt record on the source. Having both means neither side is invisible.
- **`<if_group>windows</if_group>`** scopes the match to the Windows eventchannel so other log sources don't create noise.

### 2. Agent 003 on the Domain Controller

The DC needs its own agent, or lateral movement is only visible from the jump box's point of view. Since the DC is console-only with no RDP, I couldn't just RDP in and run an installer. Instead I downloaded the agent MSI to the jump box's SMB share, then copied it from that share to the DC at the console and installed it there, pointed at the manager.

```powershell
# At the DC console, pulling the installer from the jump box share
Copy-Item "\\192.168.10.60\LabShare\wazuh-agent-4.14.5-1.msi" -Destination "C:\Temp\"
msiexec.exe /i "C:\Temp\wazuh-agent-4.14.5-1.msi" /q `
  WAZUH_MANAGER='192.168.10.10' WAZUH_AGENT_NAME='LAB-WSDC-22'
NET START WazuhSvc
```

Then I confirmed agent 003 registered as Active on the manager. One thing to know: the agent name is set permanently at enrollment, so I matched it to the VM name at install time.

### 3. Rule 100005: The Arrival Side on the DC (Event 4625, Logon Type 3)

With the DC reporting, a second rule catches the other half. Event 4625 with logon type 3 is specifically a failed network logon, as opposed to an interactive one. On a Tier 0 DC that has no business receiving interactive logons from the lab, a failed network logon is a high-confidence lateral movement signal.

```xml
<rule id="100005" level="14">
  <if_group>windows</if_group>
  <field name="win.system.eventID">^4625$</field>
  <field name="win.eventdata.logonType">^3$</field>
  <description>Custom: Failed network logon on Tier 0 DC - possible lateral movement arrival (Event 4625, logon type 3)</description>
  <mitre>
    <id>T1021.002</id>
  </mitre>
</rule>
```

The choices behind it:
- **Level 14**, higher than rule 100004 on the jump box. Anything reaching the Tier 0 DC is treated as more serious than activity that originates on a Tier 1 host.
- **The `logonType` filter is the important part.** Without it, this rule would also fire on failed interactive logons (type 2) and overlap with rule 100002, creating duplicate high-severity alerts for events that aren't lateral movement. The filter is what makes it precise.

## What I Detected

I ran the attack in two stages from the jump box, with live monitoring set up on the manager first. One note on the monitoring: when you pipe `tail -f` into `grep`, you have to add `--line-buffered`, or grep holds its output and the window looks empty even while alerts are writing.

```bash
sudo tail -f /var/ossec/logs/alerts/alerts.log | grep --line-buffered -E "Rule: 10000[245]"
```

**Stage 1** was a handful of failed logon attempts against a made-up account on the jump box, which produced Event 4625 each time and fired rule 100002.

**Stage 2** was a series of attempts to reach the DC's administrative SMB share using fabricated credentials, targeting the DC by IP address. Each attempt wrote Event 4648 on the jump box (the outbound credential use) and a failed network logon on the DC. Targeting by IP forces an NTLM fallback rather than Kerberos, which is what lands as a logon-type-3 4625 on the DC.

Rule 100004 fired three times on the jump box for the outbound attempts:

```
Rule: 100004 (level 12) -> 'Custom: Explicit credential use toward remote host detected - possible lateral movement (Event 4648)'
```

### Confirmed Alert Counts (End of Session)

| Rule | Event ID | Host | Hits | Technique |
|------|----------|------|------|-----------|
| 100002 | 4625 | LAB-WSJP-25 | 3 | T1110 |
| 100004 | 4648 | LAB-WSJP-25 | 3 | T1021.002 |
| 100002 | 4625 | LAB-WSDC-22 | 1 | T1021.002 |

In the Wazuh Threat Hunting dashboard, filtered to level 12 and up, the kill chain was visible as a sequence: failed logons on the jump box, then explicit credential use from the same host, then a failed network logon arriving at the DC. In this session, the DC-side failure was caught by rule 100002, my general failed-logon rule. Rule 100005 was deployed and the DC-side events were confirmed flowing to the manager, but it hadn't been validated live yet. I did the full live-run validation of rule 100005, including the NTLM-fallback detail, in a follow-up session on June 27.

## What I Took Away

**Events 4648 and 4625 are two sides of one attempt.** Rule 100004 catches the outbound credential use on the jump box, and rule 100005 catches the failed logon that lands on the DC. One shows intent, the other shows impact. Either is useful alone, but together they correlate the source and the destination without me having to line up logs by hand.

**Logon type is what separates lateral movement from a user fat-fingering a password.** Event 4625 fires for every failed authentication, interactive or network. Without the `logonType` filter, every mistyped interactive password on the DC would throw a level-14 alert. That one filter is the difference between a rule that's actionable and one that cries wolf.

**`--line-buffered` is not optional with `tail -f | grep`.** A silent tail window looks exactly like a broken detection. It cost me time chasing a pipeline problem that didn't exist before I realized grep was just buffering.

**The pipeline has real latency under light load.** Even with a healthy agent, some events took several minutes to show up in `alerts.log`. For validation work that matters: a quiet tail window isn't proof the rule failed, it might just mean the event hasn't arrived yet. Counting across the whole log after the run is more reliable than watching a live tail.

**Routing the installer through the SMB share is the right call for a console-only host.** Getting the agent onto a DC with no RDP meant relaying the MSI through the jump box's share. That's also realistic, since pushing software to restricted hosts in a real environment follows a similar path through a managed share or deployment server.

## Lab Environment

| Component | Details |
|-----------|---------|
| Host | Windows 11 Pro on the ASUS laptop, VirtualBox 7 |
| SIEM Manager | Ubuntu 25.04, Wazuh v4.14.5 (LAB-UBTU, 192.168.10.10) |
| Domain Controller (Tier 0) | Windows Server 2022, AD DS + DNS, `lab.local`, console-only, Wazuh agent 003 (LAB-WSDC-22, 192.168.10.50) |
| Jump Box (Tier 1) | Windows Server 2025, domain-joined, RDP + SMB share, Wazuh agent 002 (LAB-WSJP-25, 192.168.10.60) |
| Network | Isolated host-only VirtualBox network (192.168.10.0/24), no external exposure |

## What Came Next

- **[P2-1a](00-LAB-P2-1a.md):** Azure Arc onboarding for all three machines and an MCSB baseline assessment through Azure Policy.
- **[P2-1b](00-LAB-P2-1b.md):** rule 100003 rebuilt for Ubuntu 25.04, plus Wazuh Active Response blocking the attacker automatically.
- All four custom rules from this series are collected in [`detection-rules/local_rules.xml`](detection-rules/local_rules.xml).

---

*Part of the [cloud-security-lab-notes](https://github.com/JNewbrey87/cloud-security-lab-notes) portfolio, building toward Cloud Security Analyst and the CySA+ / SC-200 / SC-500 certifications.*
