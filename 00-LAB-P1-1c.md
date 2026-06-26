# Tiered AD Lab — Domain Forest Build + Windows Failed Logon Detection (Rule 100002)

**Project:** P1 — Wazuh SIEM Home Lab
**Phase:** 1e wrap-up / Phase 2 prep | **Platform:** VirtualBox on Windows 11 Pro
**MITRE ATT&CK:** T1110 — Brute Force (primary); T1070.006 / T1562 — Indicator Removal / Impair Defenses (bonus finding)
**Cert Alignment:** CySA+ CS0-003, SC-200, SC-500
**Date Completed:** June 2026

---

## Scenario

Lab 1 and 1b proved single-event and frequency-based detection on flat, non-domain hosts. This lab moves the home lab into a tiered Active Directory environment: a Tier 0 domain controller and a Tier 1 jump box, modeled on the same admin-tiering principles used in production enterprise environments.

The goal was twofold. First, build a working `lab.local` forest with a deliberately isolated Tier 0 domain controller and a Tier 1 jump box that carries the lab's intentional exposure surface. Second, deploy a Wazuh agent on the new domain-joined Windows Server host and validate that a custom detection rule for failed logons (Event 4625) fires correctly, repeatedly, and survives a live agent restart.

This sets up the next phase: a two-stage attack narrative where brute-force activity against the jump box (caught by this rule) escalates into a lateral-movement attempt toward the domain controller, which will need its own detection rule.

---

## Architecture

```
[Domain Controller — Tier 0]          [Jump Box — Tier 1]              [SIEM Manager]
  Windows Server 2022                   Windows Server 2025               Ubuntu 22.04
  AD DS + DNS, lab.local forest         Domain-joined to lab.local         Wazuh Manager v4.14.5
  Console-only — no RDP        <───     RDP enabled (lab subnet only)  ──► local_rules.xml
                                         SMB share (lateral-movement       alerts.log
                                         staging)
                                         Wazuh Agent v4.14.5 ────────────► 192.xx.xx.xx:1514/tcp
```

> Real IPs and hostnames are omitted per standard home lab portfolio security practice. All hosts sit on an isolated, host-only lab subnet with no external exposure.

**Tiering model:**
- **Tier 0 (Domain Controller):** Highest-trust asset. Console access only, no remote management. This mirrors the production principle that Tier 0 credentials and consoles should never be reachable from lower tiers.
- **Tier 1 (Jump Box):** Domain-joined, RDP-enabled, hosts an SMB share. This is the intentional exposure surface for brute-force and lateral-movement testing in upcoming labs.

---

## What I Built

### 1. Domain Forest on the Tier 0 Domain Controller

Stood up a new single-domain Active Directory forest (`lab.local`) on a fresh Windows Server 2022 instance:

```powershell
Install-ADDSForest -DomainName "lab.local"
Get-ADDomain   # confirms healthy forest/domain state
```

A DNS forwarder was configured to a public resolver for external resolution. The domain controller was deliberately left **console-only with no RDP enabled** — Tier 0 isolation by design. In a production tiered admin model, the DC is the asset every other tier ultimately trusts, so it should never be a remote-access target.

### 2. Tier 1 Jump Box — Domain Join + Exposure Surface

A second VM (Windows Server 2025) was joined to `lab.local` and configured as the Tier 1 jump box:

- **RDP enabled intentionally**, scoped to the host-only lab subnet (not internet-facing). This is the designed brute-force entry point for future attack scenarios.
- **SMB share** created with broad read/write permissions, intentionally permissive to support later lateral-movement scenarios.
- A shared local administrator credential is used across lab VMs for convenience — acceptable in an isolated, host-only network with zero external exposure, and explicitly not a pattern carried into anything production-facing.

### 3. Wazuh Agent Deployment on the Jump Box

Deployed a Wazuh agent (v4.14.5) on the jump box via the dashboard's agent-install wizard and confirmed it registered and connected cleanly:

- Agent shows **Active** on the manager
- `ossec.conf` confirmed correct: the Security event channel is enabled (`<location>Security</location>`, `<log_format>eventchannel</log_format>`), and Event ID 4625 is not excluded by any default exclusion query
- Agent log shows a clean startup with no connection errors to the manager

---

## What I Detected

### Custom Rule 100002 — Windows Failed Logon (Event 4625)

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

To generate test telemetry, I repeatedly ran a `runas` command against a nonexistent account on the jump box, producing Event 4625 (failed logon, unknown account) in the Windows Security log.

**Alert captured in `alerts.log`:**

```
** Alert <timestamp>: - local,
2026 Jun 14 16:58:13 (jump-box) any->EventChannel
Rule: 100002 (level 10) -> 'Custom: Windows failed logon attempt detected (Event 4625)'
win.system.eventID: 4625
win.system.channel: Security
win.system.computer: jump-box.lab.local
win.system.severityValue: AUDIT_FAILURE
win.system.systemTime: 2026-06-14T16:58:14Z

win.eventdata.targetUserName: testuser
win.eventdata.targetUserSid: S-1-0-0
win.eventdata.workstationName: jump-box
win.eventdata.logonType: 2
win.eventdata.failureReason: %%2313
win.eventdata.subStatus: 0xc0000064  (bad username/password)
win.eventdata.ipAddress: ::1
```

The `targetUserName` field matching the test account is the causal fingerprint confirming this alert traces directly to the test run, not a stale or unrelated event.

### Aggregate Validation — 17 Firings, Including 3 Post-Restart

Running an aggregate check across the full session's alert log:

```bash
sudo awk '/<jump-box> any->EventChannel/{getline; print}' /var/ossec/logs/alerts/alerts.log | sort | uniq -c | sort -rn
```

```
17 Rule: 100002 (level 10) -> 'Custom: Windows failed logon attempt detected (Event 4625)'
 2 Rule: 60642 (level 3)  -> 'Software protection service scheduled successfully.'
 1 Rule: 60132 (level 5)  -> 'System time changed'
```

Rule 100002 fired **17 times** across the session, matching every failed logon attempt — including **three alerts generated after a manual Wazuh agent service restart**, confirming the eventchannel pipeline recovers cleanly and resumes alerting immediately without re-registration or manual intervention.

### Bonus Finding — Rule 60132 (System Time Changed)

During the session, a manual clock change on the jump box was caught automatically by Wazuh's built-in rule 60132 (level 5). This maps to **MITRE T1070.006 (Indicator Removal: Timestomp)** and **T1562 (Impair Defenses)** — clock manipulation is a classic anti-forensics technique used to confuse log timelines during an investigation. This wasn't part of the planned test; it's evidence that the SIEM's default ruleset provides detection coverage beyond what was explicitly built.

---

## What I Learned

**Tier 0 isolation is a design decision, not a default.** Standing up the domain controller with console-only access took deliberate configuration choices, not just "leave RDP off." It reinforced why tiered admin models treat the DC as a different trust class entirely: every credential and management path into Tier 0 is a path an attacker would also want.

**A single field can be the difference between "it fired" and "it fired because of my test."** The `targetUserName` value in the 4625 event was what tied this specific alert to this specific test run. Without that kind of causal fingerprint, a fired rule and a coincidental fired rule look identical in a log.

**Live-stream tailing and aggregate log review answer different questions.** Early in the session, watching `alerts.log` with `tail -f | grep` appeared to show nothing happening, which looked like a possible rule or pipeline defect. An aggregate `awk`/`sort`/`uniq -c` pass across the full log told the real story: 17 firings, rule working as designed. The live tail was a timing and buffering artifact, not evidence of a problem. When a detection seems silent, check the aggregate before assuming the rule is broken.

**Restart resilience is worth testing explicitly, even when you don't expect a problem.** Restarting the Wazuh agent mid-session wasn't strictly necessary, but doing it anyway and confirming three alerts fired afterward turned an assumption ("the pipeline will recover") into a validated fact.

**Built-in rules can produce findings you didn't plan for.** The 60132 system time change detection wasn't the point of this session, but it mapped cleanly to a real ATT&CK technique and is a legitimate piece of detection coverage. Worth treating unplanned alerts as data, not noise, before dismissing them.

---

## Lab Environment

| Component | Details |
|-----------|---------|
| Host | Windows 11 Pro, VirtualBox |
| SIEM Manager | Ubuntu 22.04 LTS, Wazuh v4.14.5 |
| Domain Controller (Tier 0) | Windows Server 2022, AD DS + DNS, `lab.local` forest, console-only |
| Jump Box (Tier 1) | Windows Server 2025, domain-joined, RDP + SMB share, Wazuh agent v4.14.5 |
| Network | Isolated host-only subnet, no external exposure |

---

## Up Next

- **Two-stage attack scenario:** RDP brute force against the Tier 1 jump box (triggers rule 100002) followed by a simulated lateral-movement attempt toward the Tier 0 domain controller, which will need a new detection rule.
- **Azure integration:** Free tenant signup and Azure Arc onboarding for the jump box, extending this lab's telemetry into Microsoft Sentinel for Phase 2 (KQL detection engineering).

---

*Part of the [cloud-security-lab-notes](https://github.com/JNewbrey87/cloud-security-lab-notes) portfolio, building toward Cloud Security Analyst and SC-200/CySA+/SC-500 certifications.*
