# Cloud Security Lab Notes

After finishing my college work, I wanted to keep studying. I enjoy learning, and there's the whole "idle hands" thing: I'd rather stay busy building my skills, especially with how fast AI is changing this field. This repo is where that work lands.

I'm a SysAdmin-track IT Infrastructure Engineer, and these labs are how I go deeper than scan and patch: building the detections, running the attacks against my own lab, and writing up what happened, including what broke along the way. I'm also studying in public for CySA+ (CS0-004), so every lab lists the exam domain and the MITRE ATT&CK techniques it maps to.

The labs are grouped by focus, and each group is listed in the order I did them. It's a working archive, so it's never finished. If something is here, I built it or wrote it myself.

---

## Detection Engineering and Threat Hunting (Wazuh SIEM)

Building a Wazuh SIEM from the first agent up to a tiered Active Directory lab, writing custom detection rules, and validating them against real attacks.

| Lab | Maps To |
|-----|---------|
| [Wazuh Agent Deployment and My First Detection Rule](00-LAB-P1-1a.md) | CySA+ Domain 1; T1110 |
| [SSH Brute Force Detection with Kali, Hydra, and Wazuh](00-LAB-P1-1b.md) | CySA+ Domain 1; T1110, T1110.001 |
| [Tiered AD Build and Failed Logon Detection on a Domain Host](00-LAB-P1-1c.md) | CySA+ Domain 1; T1110, T1070.006, T1562 |
| [Detecting a Two-Stage Kill Chain with Rules 100004 and 100005](00-LAB-P1-1d.md) | CySA+ Domain 1; T1110, T1021.002 |

## Governance, Risk, and Compliance

Automated GRC tooling and framework gap assessments, pointed at real assets.

| Lab | Maps To |
|-----|---------|
| [GRC Toolkit Lab: SOC 2 and GLBA Gap Assessment of My Own Repos](GRC-Toolkit-Lab.md) | CySA+ Domain 4; SOC 2, GLBA Safeguards Rule |

## Hybrid Cloud Integration and Automated Response (Azure Arc)

Pulling the same lab machines into Azure through Arc for cloud-native posture management, and hardening the on-prem detection stack with automated response.

| Lab | Maps To |
|-----|---------|
| [Azure Arc Onboarding and an MCSB Baseline for the Lab](00-LAB-P2-1a.md) | CySA+ Domain 2; T1078, T1562; MCSB v2 |
| [Rebuilding SSH Brute Force Detection for Ubuntu 25.04, Plus Active Response](00-LAB-P2-1b.md) | CySA+ Domain 1; T1110, T1110.001, T1136.001, T1543.003 |

**On the roadmap:** Microsoft Sentinel and KQL detection engineering, translating the Wazuh rules above into analytic rules in a cloud SIEM.

All eight custom detection rules from these labs are deployable from [`detection-rules/local_rules.xml`](detection-rules/local_rules.xml). The [detection-rules catalog](detection-rules/Detection-Rules-README.md) has the full rule index, MITRE mapping, and validation status.

## AI Security Research (AEGIS)

AEGIS is my local Red Team / Blue Team AI simulation, running entirely on my own hardware through CrewAI and Ollama: no cloud API, nothing leaving the machine. Most of it is extending my thesis framework, [CFRPF](https://github.com/JNewbrey87/CFRPF), into code, specifically preserving what AI agents decided and why. The rest is me learning how agent stacks actually behave.

| Lab | Maps To |
|-----|---------|
| [AEGIS PoC Run 2: llama3.3:70b Adversarial Simulation](aegis/AEGIS-PoC-70b.md) | Model scale and security reasoning quality |
| [Migrating the PoC to CrewAI 1.x and Validating the Six-Model Stack](aegis/AEGIS-CrewAI-1x-Migration.md) | Local inference operations; recovering a broken runtime |
| [AEGIS Evidence Ledger: Tamper-Evident Custody for Agent Decisions](aegis/AEGIS-Evidence-Ledger.md) | CFRPF Immutable Evidence Preservation; AI Auditability |
| [The GPU That Was Never On](aegis/AEGIS-GPU-Investigation.md) | Silent failure in local inference; verifying what the system reports |

## Vulnerability Management, Forensics, and Incident Response (CySA+ Track)

Labs built around the CySA+ CS0-004 objectives, run on the same lab machines: scanning and remediation, memory and cloud forensics, and a full incident response cycle.

| Lab | Maps To |
|-----|---------|
| [Vulnerability Management Baseline: Nessus Scan of the Lab Environment](CySA/cysa-01-vuln-management-baseline.md) | CySA+ Domain 2; T1557.001 |
| [Volatile Memory Capture and Analysis: Live Acquisition from a Windows Jump Box](CySA/cysa-02-volatile-ram-capture.md) | CySA+ Domains 3 and 1; T1059.001, T1218 |
| [Credentialed Vulnerability Scanning: Finding What the Perimeter Cannot See](CySA/cysa-03-dc-credentialed-scan.md) | CySA+ Domain 2; T1553.002 |
| [Cloud Forensic Acquisition: Snapshotting a Running Azure VM](CySA/cysa-04-azure-vm-forensic-snapshot.md) | CySA+ Domains 3 and 1 |
| [Incident Response Capstone: Running a Full IR Cycle on the Lab Kill Chain](CySA/cysa-05-ir-capstone.md) | CySA+ Domains 3, 1, and 4; T1110, T1021.002 |

## Retrospectives

Write-ups of earlier technical work that shaped my path into security.

| Project | Maps To |
|---------|---------|
| [Phishing Triage Methodology: FAA Help Desk (~2020)](retrospective/Help-Desk-Phishing-Triage.md) | CySA+ Domains 1 and 3; T1566 |
| [Ark: Survival Evolved Dedicated Server Administration (2015-2017)](retrospective/pre-it-linux-server.md) | Linux server administration |
