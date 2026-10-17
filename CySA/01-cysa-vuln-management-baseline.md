# Vulnerability Management Baseline: Nessus Scan of the Lab Environment

**Project:** CySA+ Lab Track, Vulnerability Management  
**Session:** 01 | **Platform:** VirtualBox on Windows 11 Pro, isolated host-only network  
**Scanner:** Nessus Essentials, uncredentialed Basic Network Scan  
**MITRE ATT&CK:** [T1557.001 — Adversary-in-the-Middle: LLMNR/NBT-NS Poisoning and SMB Relay](https://attack.mitre.org/techniques/T1557/001/)  
**Cert Alignment:** CySA+ CS0-004, Domain 2 (Vulnerability Management), objectives 2.1 through 2.4  
**Date:** October 2026

---

## Why I Did This One

My homelab started out as a detection lab: a Wazuh SIEM with eight custom rules and a validated kill chain across a tiered Active Directory setup. What it didn't have was any vulnerability management. There was no scanner in the lab at all.

Scanning and patching are already part of my infrastructure work, so running Nessus wasn't new to me. What I wanted from this lab was the part that tends to get skipped when you're working through a patch queue: slowing down on the results to figure out which findings actually matter, re-scoring them against what each machine is worth, and building a remediation plan I could defend instead of just patching whatever shows up red. CySA+ puts Vulnerability Management at 26% of the exam, so this was also where my studying needed the most hands-on time.

## Architecture

```
[Nessus Essentials]  ──uncredentialed──►  [LAB-WSDC-22]  Tier 0 Domain Controller   192.168.10.50
  Basic Network Scan                       [LAB-WSJP-25]  Tier 1 Jump Box            192.168.10.60
                                           [LAB-UBTU]     Wazuh Manager              192.168.10.10
```

Everything sits on an isolated host-only VirtualBox network with no outside exposure. I ran the scan from a host on the same segment, so it shows what an attacker already inside the lab network would see.

## What I Ran

I started with a Basic Network Scan and no credentials, on purpose. An uncredentialed scan only sees what's visible from the outside: open ports, service banners, protocol behavior, and config flags you can detect without logging in. All three hosts came back `Auth: Fail`, which confirmed nothing was assessed from the inside.

I wanted that outside view first so I'd have something to compare a credentialed scan against later. A credentialed scan reads installed software, registry settings, and local policy, and it usually finds a lot more. Having both pictures is how you tell what an attacker on the network sees versus what's actually going on inside the box.

## What the Scan Returned

The scan took about fifteen minutes.

| Host | Role | IP | Findings | Medium | Info |
|------|------|----|----------|--------|------|
| LAB-WSDC-22 | Tier 0 Domain Controller | 192.168.10.50 | 56 | none confirmed | 56 |
| LAB-WSJP-25 | Tier 1 Jump Box | 192.168.10.60 | 55 | 5 | 50 |
| LAB-UBTU | Wazuh Manager | 192.168.10.10 | 42 | 1 | 41 |

The per-host totals add up to 153, which tripped me up at first, because the Vulnerabilities tab said 43. The difference is that the per-host count lists the same plugin once for every host it fires on, while the Vulnerabilities tab counts each unique vulnerability once. Same scan, two different units. I made a note to always say which one I'm reporting.

Almost all of it was informational, just Nessus describing what it found. Once I set that aside, the scan came down to **one confirmed finding I could act on**: SMB signing not required on the jump box. There were other Medium groups (five on the jump box, one on the Wazuh manager) that I didn't drill into on this pass, so I treated those as unverified rather than counting them. The domain controller showed no Medium findings at all from the outside, which, given how important that box is, made me want to scan it with credentials rather than call it clean.

No Critical or High findings showed up anywhere. That lines up with patch state more than anything: both Windows Servers had the October 2026 cumulative updates (KB5126646, KB5126655) installed before I scanned.

## The Finding: SMB Signing Not Required

| Field | Value |
|-------|-------|
| Plugin ID | 57608 |
| Severity | Medium |
| CVSS v3 Base | 5.3 |
| CVSS v3 Vector | AV:N/AC:L/PR:N/UI:N/S:U/C:N/I:L/A:N |
| Temporal Score | 4.6 |
| Affected Host | LAB-WSJP-25 (192.168.10.60), the jump box |
| Port | 445/tcp (CIFS) |
| Exploit Available | Yes, publicly available |

SMB signing verifies that SMB traffic actually came from the authenticated sender and wasn't changed on the way. Without it, someone on the network can intercept SMB authentication and relay the NTLM credentials to log in somewhere else, no password cracking needed. That's what tools like Responder automate, and it's a common way to move laterally (MITRE T1557.001).

The `AV:N` in the vector means anything that can reach port 445 can try it, not just machines on the same local segment. And it was on the jump box, the one machine whose whole job is handling admin sessions. That's why I put it at the top of the list even though a 5.3 doesn't sound like much.

## Re-Scoring for My Environment

A CVSS base score describes the vulnerability on its own, not what it means for a particular environment. So I went through each machine and weighed how much confidentiality, integrity, and availability matter for it, then used that to adjust the scores.

**LAB-WSDC-22, the domain controller (Tier 0).** Confidentiality High, since it holds every domain credential hash, Kerberos ticket, and GPO. Integrity High, since quietly changing AD objects compromises the whole domain. Availability High, since nobody authenticates without it. That's High/High/High. Nothing actionable showed up from the outside, but with a box this critical, I read that as a reason to scan with credentials, not a reason to relax.

**LAB-WSJP-25, the jump box (Tier 1).** Confidentiality and Integrity High, because privileged sessions and admin credentials pass through it, and owning it gives an attacker a trusted starting point. Availability Medium, since there are other ways to do admin work if it's down. That's High/High/Medium. The SMB signing finding hits Integrity (`I:L` in the vector), and with the jump box's Integrity requirement set to High, the environmental score comes out above the 5.3 base.

**LAB-UBTU, the Wazuh manager.** Confidentiality Medium, since it's internal security telemetry rather than regulated data. Integrity High, because tampering with rules or logs would blind detection across every monitored host. Availability High, because if it's down, all eight custom rules (100002 through 100009) stop firing. That's Medium/High/High. One Medium finding showed up here that I still need to drill into.

Working through it that way, the thing that ended up at the top of my list was a 5.3, and it only got there because of the context. If I'd sorted by base score, I would have ranked it wrong.

![Nessus findings summary across the three lab hosts](../screenshots/cysa-01-findings-summary.png)

![SMB Signing Not Required, Plugin 57608 detail](../screenshots/cysa-01-smb-signing-finding.png)

## The Fix

This is a policy change, not a patch, and it doesn't need downtime. Through Group Policy on the jump box:

```
Computer Configuration → Windows Settings → Security Settings →
Local Policies → Security Options →
  Microsoft network server: Digitally sign communications (always) → Enabled
  Microsoft network client: Digitally sign communications (always) → Enabled
```

Or with PowerShell:

```powershell
Set-SmbServerConfiguration -RequireSecuritySignature $true -Force
Set-SmbClientConfiguration -RequireSecuritySignature $true -Force
```

## Checking That It Worked

The next day (October 14), I ran another uncredentialed scan against the jump box and the domain controller to confirm the fix. The Wazuh manager didn't make it into that run, so its status check is still on my list.

| Check | Result |
|-------|--------|
| Plugin 57608 (SMB signing) | **Gone from both hosts.** Remediation confirmed. |
| Critical or High findings | None |
| Total | 33 vulnerabilities across the 2 hosts |

That run also turned up four Medium findings, all SSL/TLS related, on the Windows hosts:

| Finding | Severity | CVSS |
|---------|----------|------|
| SSL Certificate Cannot Be Trusted | Medium | 6.5 |
| SSL Self-Signed Certificate | Medium | 6.5 |
| TLS Version 1.0 Protocol Detection | Medium | 6.5 |
| TLS Version 1.1 Deprecated Protocol | Medium | 6.5 |

They all come from default Windows Server settings, since services like RDP and WinRM still offer the older TLS versions out of the box. On a production network I'd fix them by disabling TLS 1.0 and 1.1 (with a tool like IIS Crypto or directly in the registry) and putting real certificates in place. In an isolated lab with no outside exposure, I'm accepting them as a known risk for now and writing that down instead of ignoring them.

## What I Took Away

**An outside scan tells you about the perimeter, not the patch state.** A clean Critical/High result from an uncredentialed scan felt good, but all it really said was that an attacker on the network wouldn't find easy wins. It didn't say the machines were configured well inside. That was the reason the credentialed scan of the domain controller came next.

**I can't prioritize by base score alone.** The finding that ended up on top was a 5.3, and it got there because of two things the base score leaves out: a temporal score of 4.6, since there's a public exploit, and the jump box's high integrity requirement. The scan also had the opposite case, a Low finding (CVSS 2.1) with an EPSS of 0.32, meaning roughly a 32% estimated chance of exploitation in the next thirty days. If I'd only looked at base scores, I would have skipped right past that one.

**It connected back to my detection work.** SMB relay is the same lateral movement path my earlier kill-chain lab was built around. Rules 100004 and 100005 catch explicit credential use and failed network logons landing on the domain controller. Turning on SMB signing closes the hole, and the rules still catch the attempt if someone tries. Seeing vulnerability management and detection from both ends of the same attack was the most useful part of this lab for me.

**Writing it up showed me what to study next.** Turning the scan into this writeup made it obvious where I wanted more depth: how NTLM relay actually works step by step, the order of volatility for evidence collection, and the differences between SAST, DAST, IAST, and SCA.

## Lab Environment

| Component | Details |
|-----------|---------|
| Scanner | Nessus Essentials, Basic Network Scan, uncredentialed |
| Targets | LAB-WSDC-22 (Server 2022 DC), LAB-WSJP-25 (Server 2025 jump box), LAB-UBTU (Ubuntu, Wazuh manager) |
| Host | VirtualBox on Windows 11 Pro; isolated host-only network (192.168.10.0/24) |
| Patch state | October 2026 cumulative updates (KB5126646, KB5126655) on both Windows Servers before the scan |

## What Came Next

- **[cysa-03](03-cysa-dc-credentialed-scan.md):** the credentialed scan of the domain controller that this baseline pointed toward.
- **[cysa-05](05-cysa-ir-capstone.md):** the incident response capstone that replayed the kill chain against these same machines.

Still on my list from this one:

- Drill into the remaining Medium groups on both Windows Servers and the Wazuh manager.
- Disable LLMNR and NBT-NS, which go hand in hand with SMB signing.
- Confirm SMB signing is actually enforced on the domain controller instead of assuming the default policy covers it.
- Check Wazuh service status on LAB-UBTU, which missed the verification run.

---

*Part of the [cloud-security-lab-notes](https://github.com/JNewbrey87/cloud-security-lab-notes) portfolio, building toward Cloud Security Analyst and the CySA+ / SC-200 / SC-500 certifications.*
