# Cloud Security Lab Notes

## Overview

This is my working archive of hands-on lab output, AEGIS research runs, and retrospective write-ups documenting the technical work that led here. I add to it continuously as lab sessions complete. The goal is a portfolio that shows work in progress, not just finished products; if something is here, I built it or wrote it myself.

## P1: Wazuh SIEM Detection Lab

Sequential detection engineering labs building toward a fully instrumented Active Directory environment with validated MITRE ATT&CK coverage.

| File | Lab | MITRE |
|------|-----|-------|
| [00-LAB-P1-1a.md](00-LAB-P1-1a.md) | Wazuh Agent Deployment + First Detection Rule (Event 4625, Rule 100002) | T1110 |
| [00-LAB-P1-1b.md](00-LAB-P1-1b.md) | SSH Brute Force Detection — Kali + Hydra + Rule 100003 | T1110, T1110.001 |
| [00-LAB-P1-1c.md](00-LAB-P1-1c.md) | Tiered AD Lab — Domain Forest Build + Rule 100002 Validation | T1110, T1070.006 |
| [00-LAB-P1-1d.md](00-LAB-P1-1d.md) | Lateral Movement Detection — Rules 100004 + 100005, Two-Stage Kill Chain | T1021.002 |

Phase 2 (Azure Arc, Sentinel, KQL detection engineering) begins August 2026.

## AEGIS Research

Local AI Red/Blue Team simulation running on-device via CrewAI and Ollama. No cloud API, no data exfiltration. Documents how model scale affects security reasoning quality across adversarial simulation runs.

| File | Run |
|------|-----|
| [aegis/AEGIS-PoC-70b.md](aegis/AEGIS-PoC-70b.md) | Run 2: llama3.3:70b Adversarial Simulation |

## GRC

| File | Description |
|------|-------------|
| [GRC-Toolkit-Lab.md](GRC-Toolkit-Lab.md) | SOC 2 + GLBA Gap Assessment using the GRC Engineering Club toolkit |

## Retrospectives

Write-ups documenting earlier technical work that shaped the path into security.

| File | Project |
|------|---------|
| [retrospective/Help-Desk-Phishing-Triage.md](retrospective/Help-Desk-Phishing-Triage.md) | Phishing Triage Methodology: FAA Help Desk (~2020) |
| [retrospective/pre-it-linux-server.md](retrospective/pre-it-linux-server.md) | Ark: Survival Evolved Dedicated Server Administration (2015-2017) |
