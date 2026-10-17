# Incident Response Capstone: Running a Full IR Cycle on the Lab Kill Chain

**Project:** CySA+ Lab Track, Incident Response (capstone)  
**Session:** 05 | **Platform:** VirtualBox on Windows 11 Pro, isolated host-only network  
**Attacker:** LAB-KALI | **Targets:** LAB-WSJP-25 (Tier 1 jump box), LAB-WSDC-22 (Tier 0 DC, lab.local) | **SIEM:** LAB-UBTU (Wazuh 4.14.5)  
**MITRE ATT&CK:** [T1110 — Brute Force](https://attack.mitre.org/techniques/T1110/), [T1021.002 — SMB/Windows Admin Shares](https://attack.mitre.org/techniques/T1021/002/)  
**Cert Alignment:** CySA+ CS0-004, Domain 3 (Incident Response and Management) primary; Domain 1 (Security Operations) and Domain 4 (Reporting and Communication) secondary  
**Date:** October 2026  

---

## Why I Did This One

This is the capstone for my CySA lab sprint, and it pulls the earlier labs into a single incident. By this point I had detection rules from my Wazuh work, a hashed memory-acquisition workflow from the RAM lab, the Arc and cloud-acquisition angle from the Azure labs, and the scan-based verification habit from the credentialed scan. What I hadn't done was run all of it as one continuous response, from the first alert through containment, evidence preservation, rollback, and the writeup.

Responding to alerts and restoring systems is familiar ground for me as an infrastructure engineer. The part I built this lab to practice is the deliberate, documented IR cycle, especially the preserve-versus-contain decision, made on purpose and written down rather than improvised. So I attacked my own lab with a kill chain I had already validated, then worked the response as if I didn't know what was coming.

## Scope and Rules of Engagement

- **In scope:** LAB-KALI (attacker), LAB-WSJP-25 (target, Tier 1 jump box), LAB-WSDC-22 (target, Tier 0 DC), LAB-UBTU (Wazuh, detection only).
- **What counts as the incident:** rules 100002, 100004, and 100005 firing in sequence is a confirmed kill-chain event that triggers a full response.
- **Contained means:** the affected host's network adapter is disconnected at the hypervisor so the attacker can't keep talking to it.
- **Recovered means:** both targets back at a known-good snapshot, agents reporting, and no alerts when the detection condition is re-run. I'll be honest later about which of those I actually verified.

All times below are UTC. The lab runs in US Central (UTC-5), so the 12:12 PM snapshot is 17:12 UTC.

## Phase 1: Preparation

Before touching anything, I took VirtualBox snapshots of both targets (`pre-cysa05-20261017`, 17:12 UTC) so I had a clean rollback point, and I confirmed all four Wazuh agents were active and the manager was running. The kill chain was the validated two-stage chain from my earlier lateral-movement lab: brute force against the jump box firing rule 100002, then an explicit-credential move toward the DC firing rules 100004 and 100005.

## Phase 2: Detection and Analysis

### Executing the Kill Chain

**Stage 1, brute force (T1110).** I generated the failed-logon telemetry with a looped local `runas` against a nonexistent account on the jump box, which produces Event 4625. That's a documented lab shortcut; in a real incident those failures would come from a remote attacker, not a local console.

I did also try the remote version first, Hydra and crackmapexec from LAB-KALI against SMB and RDP, and it did not produce any host telemetry on the jump box. I have not confirmed why. I moved on to the local method to keep the lab going, and the VMs are shut down now, so this is an open item rather than a finding. I specifically am not claiming the host firewall blocked it, because later in this same lab the jump box happily accepted an SMB file transfer, so a blanket "SMB was blocked" doesn't hold up. Root-causing that failed remote run with a packet capture is on my list.

**Stage 2, lateral movement (T1021.002).** On the jump box I ran `net use \\192.168.10.50\IPC$ /user:LAB\Administrator <password>` in a loop, targeting the DC by IP. Targeting by IP forces an NTLM fallback instead of Kerberos, which lands as Event 4625 logon type 3 on the DC rather than a Kerberos 4771. I let the loop run and stopped it with Ctrl+C, which matters for reading the alert counts below.

### What Wazuh Caught

| Rule | Level | Event | Host | Alerts |
|------|-------|-------|------|--------|
| 100002 | 10 | 4625 failed logon | LAB-WSJP-25 | 11 |
| 100004 | 12 | 4648 explicit credential use outbound | LAB-WSJP-25 | fired |
| 100005 | 14 | 4625 logon type 3 on Tier 0 DC | LAB-WSDC-22 | 217 |

All three fired in sequence, and the levels climbed with the kill chain: 10 at the endpoint failure, 12 at the lateral-movement source, 14 at the Tier 0 target. The 217 count on rule 100005 is not 217 separate attacks; it's the looped `net use` hammering the DC until I stopped it. One deliberate action, many alerts, which is itself a thing to understand when you're reading a dashboard and trying not to overcount.

### Severity and ATT&CK

I called it HIGH. Rule 100005 at level 14 means something reached for the Tier 0 domain controller, and even a failed lateral-movement attempt against a DC warrants an escalated response, because a successful one is domain-level control.

| Technique | ID | Evidence |
|---|---|---|
| Brute Force | T1110 | 11 rule 100002 alerts, Event 4625 on the jump box |
| SMB / Windows Admin Shares | T1021.002 | Rule 100004 (4648 outbound), rule 100005 (4625 type 3 on the DC) |

## Phase 3: Containment, Eradication, and Recovery

### The Preserve-versus-Contain Decision

This is the judgment call the whole lab was built around, and I want to be precise about what I actually chose, because it's easy to dress it up as something it wasn't.

I isolated the host before acquiring memory. Strict order of volatility says collect the most volatile evidence first, and live network connections are more volatile than RAM, so isolating first actually sacrifices that connection state. I did it on purpose anyway: cutting a possible path to the Tier 0 DC mattered more to me than preserving live connections. That's a containment-over-preservation tradeoff, chosen deliberately. It is not something order of volatility required; order of volatility would have pointed the other way.

In this lab it cost me almost nothing, because the `runas` and `net use` connections were transient and already closed (my later netstat came back empty). But the reasoning is the point, not the lab's easy outcome.

For production context: an EDR shrinks this tradeoff but doesn't erase it. EDR isolation usually keeps the agent's own management channel open, so you can keep collecting telemetry and memory after cutting the host off from everything else. It reduces the conflict between preserving and containing; it doesn't make it disappear. My manual disconnect-then-WinPmem workflow is the lab version of the same decision without that tooling.

### Containment

I disconnected the jump box's adapter at the hypervisor by unchecking "Virtual Cable Connected" in VirtualBox. That happened before any acquisition work. After isolation, the Wazuh dashboard went quiet, no new 100002, 100004, or 100005 alerts.

### Memory Acquisition and Chain of Custody

With the host isolated, I acquired RAM with WinPmem (staged at `C:\temp\winpmem.exe` from the earlier RAM lab). Acquisition ran 17:56:25 to 17:56:41 UTC.

| Item | Detail |
|---|---|
| Source | LAB-WSJP-25 |
| Output | `C:\temp\wsjp25-ram-cysa05.raw` |
| Size | 8,925,478,912 bytes (8.3 GB) |
| SHA-256 at acquisition | `eb2357834c45b9c3b6b7fdb04945d7eaf3410d9a40119ce057a5e972875d6473` |
| Transfer | smbclient to LAB-KALI |
| SHA-256 at destination | `eb2357834c45b9c3b6b7fdb04945d7eaf3410d9a40119ce057a5e972875d6473` |
| Integrity | match, chain of custody intact |

One honest wrinkle: to move the image off the isolated host, I briefly reconnected its NIC for the SMB transfer, which temporarily broke containment. In production that reconnect shouldn't happen; an out-of-band EDR channel or a dedicated forensic VLAN would carry the image without putting the contained host back on the network. I'm flagging it rather than hiding it, because it's exactly the kind of thing that looks clean in a writeup and isn't.

### Memory Analysis (Volatility3 2.28.2)

I ran three plugins against the verified copy.

- **netstat:** no established or waiting TCP connections, nothing to or from the Kali or DC addresses. Expected, since `runas` and `net use` are transient and had closed before I isolated. A memory image is a point-in-time snapshot, not a connection history, so the Wazuh alerts are the authoritative record of what happened, not netstat.
- **pslist:** one PowerShell process worth noting, PID 7908, created 17:45:27 UTC, running from the standard System32 path. That's the interactive admin session I used to run the attack and WinPmem. No injected or anomalous processes.
- **cmdline (PID 7908):** a plain PowerShell invocation, no encoded commands, no odd flags.

The memory evidence lines up with the Wazuh timeline and doesn't contradict it. No persistence, no injected code, nothing hiding.

### Eradication and Recovery

I eradicated by rolling both targets back to the `pre-cysa05-20261017` snapshot. I didn't keep the compromised state as a snapshot; the raw memory image and the Wazuh alert log are the evidence record. Rollback is the right move when you have a verified clean pre-incident state, the compromise is scoped to those hosts, and there's no sign of persistence that would survive it, all of which held here. In production, hypervisor rollback often isn't available or safe (stateful databases, domain-joined machines with changes since the snapshot), and the equivalent is reimaging from a hardened gold image that predates the compromise. Same logic, bigger blast radius.

Here's where I hold myself to the cysa-03 lesson that a claimed recovery is not a verified one. What I actually verified is that both VMs returned to the pre-cysa05 snapshot state in VirtualBox, with no "Current State (changed)" marker. What I did not do is re-confirm agent health after the rollback or re-run the detection condition to prove the environment was quiet and instrumented again. So I can't say the "recovered" criterion from my own scope was fully met. The post-rollback checks (agent status, clock resync, one controlled test event per rule) are the step I'd add next time, and I'd rather name that gap than paper over it.

## Phase 4: Post-Incident

### Lifecycle Mapping

I mapped the work against both IR lifecycle models.

**NIST SP 800-61r2, the four-phase lifecycle:** Preparation (snapshots, agent check, scope); Detection and Analysis (execution, Wazuh triage, ATT&CK mapping, severity call); Containment, Eradication, and Recovery (isolation, acquisition, analysis, rollback); Post-Incident Activity (this report, lessons learned).

**NIST SP 800-61r3, aligned to CSF 2.0's six functions:** Govern (the scope and rules of engagement I set up front, including the containment-versus-preservation criteria); Identify (hosts and scope); Protect (the detection controls already in place); Detect (Wazuh rules 100002/100004/100005); Respond (containment, acquisition, eradication); Recover (rollback, plus the verification I still owe). I'm noting Govern explicitly because it's easy to drop, and it's where my decision criteria actually lived.

## Lessons Learned

**What the detections caught.** The three rules fired in the right order at climbing severity, which gave a clear read on kill-chain position without manual correlation. That's the detection engineering from my earlier labs doing its job.

**What the detections missed, honestly as an open question.** My remote brute-force attempt from Kali never produced host telemetry on the jump box, and I haven't confirmed why. Whatever the cause, it points at a real structural gap: Wazuh here is host-based, so anything that never reaches the host's authentication layer is invisible to it regardless of the specific reason. But I'm not going to hand-wave a cause I didn't verify. Figuring out what actually happened on that remote run is a packet-capture task I still owe.

**One detection improvement.** Add a network-layer brute-force rule, keyed off firewall drop logs or an IDS signature for repeated SMB/RDP attempts from the same source, to cover what host-based detection structurally can't see.

**One process improvement.** The memory transfer forced me to reconnect a contained host, which broke containment. Standing up a dedicated forensic path (a separate VLAN, or EDR tooling with an out-of-band channel) before an incident means acquisition never requires putting the contained host back on the network.

## Evidence and Chain of Custody

| Item | Location | Hash / identifier |
|---|---|---|
| Memory image | `~/wsjp25-ram-cysa05.raw` (LAB-KALI) | `eb2357834c45b9c3b6b7fdb04945d7eaf3410d9a40119ce057a5e972875d6473` |
| Wazuh alerts | LAB-UBTU `/var/ossec/logs/alerts/alerts.log` | rule 100002 (11), 100004 (fired), 100005 (217) |
| Volatility output | `~/vol-netstat.txt`, `~/vol-pslist.txt` (LAB-KALI) | retained for the session |
| Rollback snapshot | VirtualBox `pre-cysa05-20261017` | 17:12 UTC, both targets |

## Lab Shortcut vs. Production Practice

| Lab shortcut | Production equivalent |
|---|---|
| Looped local `runas` to generate 4625s | Remote brute force from an attacker host, caught by firewall and network logging at the perimeter |
| NIC disconnect in the VirtualBox UI | EDR quarantine, network ACL, or VLAN reassignment from the SOC console |
| NIC reconnect to transfer the image over SMB | Out-of-band EDR channel or a dedicated forensic VLAN; no reconnect of a contained host |
| Snapshot rollback for eradication | Reimage from a hardened gold image, re-join the domain, restore stateful data from backup |
| Volatility3 on the Kali lab host | Forensic workstation with a validated toolchain, documented versions, and an analyst signature on the custody record |

---

*Part of the [cloud-security-lab-notes](https://github.com/JNewbrey87/cloud-security-lab-notes) portfolio, the capstone of the CySA+ CS0-004 lab track.*
