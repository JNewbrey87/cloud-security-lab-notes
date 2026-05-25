# Cloud Security Lab Notes

## Overview

This is my working archive of hands-on lab output, study notes, and reusable artifacts from building cloud security depth in Microsoft Azure. I add to it continuously as cert work and practical lab sessions accumulate. The goal is a portfolio that shows work in progress, not just finished products; if something is here, I built it or wrote it myself.

## What's in Here

### KQL Queries
Kusto Query Language queries I've written for Microsoft Sentinel, covering threat detection, log analysis, and security monitoring use cases. These come out of SC-200 study and actual lab sessions in Sentinel.

### Sentinel Analytic Rules
Custom detection rules and alert configurations for Microsoft Sentinel, developed during SC-200 prep. Logic behind each rule is documented, not just the syntax.

### Wazuh Detection Rules
Custom detection rules written in Wazuh's XML rule syntax, targeting Windows and Linux security events. Rules are MITRE ATT&CK mapped and documented with design rationale, not just syntax. Current coverage: T1110 (Brute Force / Event ID 4625).

### Azure Policy Definitions
Policy-as-code artifacts for governance, compliance enforcement, and configuration management in Azure environments. Part of the AZ-500 cert work and a longer-term interest in treating compliance as an engineering problem rather than a checkbox exercise.

### GRC Toolkit Labs
Hands-on gap assessment work using the [GRC Engineering Club toolkit](https://github.com/GRCEngClub/claude-grc-engineering), starting with a SOC2 and GLBA assessment against my own GitHub repositories. This lab series is foundational work toward auditing IaC-defined Azure environments as the lab builds out.

### Lab Notes
Structured notes from SC-200, AZ-500, and CySA+ study covering key concepts, things that tripped me up, and practical application. Less polished than the other sections; more honest.

### Retrospective Write-Ups
Documented case studies from earlier in my career, translated into technical write-ups that map what I actually did to modern security concepts. Current entries cover Linux server administration and staging environments (2015-2017) and a phishing triage methodology developed from a live FAA incident (~2020). These aren't certifications or lab simulations; they're the work that preceded the formal path.

## Cert Pipeline

| Certification | Status |
|---|---|
| CompTIA Security+ | Active (through 2027) |
| SC-200 (Microsoft Security Operations Analyst) | Future Track |
| CySA+ | Actively Studying |
| SC-500 (Microsoft Security Operations) | Future Track |

## A Note on Usage

Everything here reflects personal study and lab work. Nothing contains employer-specific data or proprietary information. Queries and rules are written for educational and portfolio purposes; if something's useful to you, take it.

## Author

**Joshua E. Newbrey**  
[LinkedIn](https://www.linkedin.com/in/jnewbrey87/) | [GitHub Profile](https://github.com/JNewbrey87)
