# Credentialed Vulnerability Scanning: Finding What the Perimeter Cannot See

**Project:** CySA+ Lab Track, Vulnerability Management  
**Session:** 03 | **Platform:** VirtualBox on Windows 11 Pro, isolated host-only network  
**Target:** LAB-WSDC-22 (Windows Server 2022, Tier 0 Domain Controller, lab.local)  
**Tool:** Nessus Essentials, credentialed Basic Network Scan  
**MITRE ATT&CK (relevance):** [T1553.002 — Subvert Trust Controls: Code Signing](https://attack.mitre.org/techniques/T1553/002/)  
**Cert Alignment:** CySA+ CS0-004, Domain 2 (Vulnerability Management): authenticated scanning, finding triage, remediation, verification  
**Date:** October 2026  

---

## Scenario

The first lab in this track ran an uncredentialed baseline that returned a clean HIGH/MEDIUM/LOW result on the domain controller: 56 findings, all informational. I flagged that as a reason to scan with credentials rather than a clean bill of health, because an uncredentialed scanner sees only the external attack surface and cannot read what is inside the host. This lab tests that claim directly, and then does the thing a scan is actually for: it closes the loop. Authenticated scan, prioritize the finding, remediate it, re-scan to prove the fix. That full cycle, not the scan itself, is the Vulnerability Management workflow.

## Scan Configuration

| Field | Value |
|-------|-------|
| Policy | Basic Network Scan, Windows credentialed (password auth) |
| Target | LAB-WSDC-22 (192.168.10.50) |
| Account | Domain Administrator (lab shortcut, see below) |
| Remote Registry | Disabled on the DC at 12:13:16; Nessus auto-started it for each scan and restored the disabled start type afterward (confirmed in the Service Control Manager log) |

Two honesty notes belong up front, because a vulnerability scan of a Tier 0 asset is as much about posture as about findings.

First, I used a Domain Admin account and scanned the DC from an adjacent segment. That violates Tier 0 separation and is a deliberate lab shortcut, not correct practice. A production credentialed-scan program uses a dedicated least-privilege scan account with only the rights Nessus needs (Remote Registry read, WMI, specific SMB access), and runs the scanner from inside Tier 0 isolation. I am documenting the shortcut as a shortcut.

Second, Remote Registry is the service a Windows credentialed scan needs to read local configuration. I disabled it on the DC at 12:13:16, before the control scan described below, to test whether that would block authenticated scanning. It did not: Nessus auto-starts Remote Registry when it finds it disabled, scans, and restores the original start type when it finishes, which the DC's own logs confirm. Leaving the service running would widen the host's attack surface, so the detail worth capturing is that the scanner manages the service's lifecycle itself and hands it back disabled.

## What Credentials Revealed

The authenticated scan returned 52 findings including one HIGH; the uncredentialed baseline had returned 56, all informational. It would be easy and wrong to read the drop from 56 to 52 as false-positive cleanup. Credentialed and uncredentialed scans run different plugin sets, so the count moved because the composition changed, not because the scan got "cleaner." A plugin-ID-level diff would characterize the real delta, but Nessus Essentials paywalls scan export to the Professional tier, so that diff was not possible here; results were captured by screenshot and transcription instead. The delta I can state with confidence is qualitative and it is the whole point: the HIGH finding was completely absent from the uncredentialed scan, because it can only be seen from the inside.

## The HIGH Finding: CVE-2013-3900

| Field | Value |
|-------|-------|
| Plugin ID | 166555 (type: local) |
| Title | WinVerifyTrust Signature Validation (EnableCertPaddingCheck) |
| CVSS v3 | 8.8 |
| VPR | 9.5 |
| EPSS | 0.4465 |
| CISA KEV | Yes, added 2022-01-10 (BOD 22-01) |

Windows does not enforce strict Authenticode PE structure validation by default. The mitigation is two registry values, and both were absent on the DC:

```
HKLM\Software\Microsoft\Cryptography\Wintrust\Config\EnableCertPaddingCheck
HKLM\Software\Wow6432Node\Microsoft\Cryptography\Wintrust\Config\EnableCertPaddingCheck
```

Without them, WinVerifyTrust can be bypassed by a PE binary that carries a valid Authenticode signature alongside malicious content stuffed into padding bytes; the file validates as signed while hiding its payload. The attack requires a user to execute the delivered binary, so it is not self-propagating, but the CISA KEV listing confirms it is exploited in the wild. Microsoft left the mitigation opt-in because strict validation can break some legitimately signed third-party installers, which is exactly why, on a real DC, this is a tested change and not a blind edit.

The reason a credentialed scan found it and the baseline did not is the plugin type: 166555 is a local check that reads registry state directly. An unauthenticated scanner cannot inspect the registry; it can only infer configuration from network behavior. This entire class of finding, registry values, local policy, installed software versions, is structurally invisible without credentials.

## Prioritization

This is a 2013 CVE that sat unmitigated on this host for years, and it is a good argument for prioritizing on more than a base score. CVSS 8.8 describes theoretical impact. EPSS at 0.4465, from FIRST's Exploit Prediction Scoring System (Nessus surfaces the score but does not compute it), estimates a roughly 44% probability of exploitation in the next thirty days, and the VPR of 9.5 plus the KEV listing reflect confirmed active exploitation. An old CVE with high exploitation likelihood should outrank a newer finding with a scarier-sounding base score but a low EPSS. Age is not the variable that matters; exploitation probability is.

## Remediation

The `Wintrust\Config` key did not exist under either path and had to be created before the value could be set, on both the 64-bit and the 32-bit (Wow6432Node) paths:

```powershell
New-Item -Path "HKLM:\Software\Microsoft\Cryptography\Wintrust\Config" -Force
New-ItemProperty -Path "HKLM:\Software\Microsoft\Cryptography\Wintrust\Config" `
  -Name "EnableCertPaddingCheck" -Value 1 -PropertyType DWORD -Force

New-Item -Path "HKLM:\Software\Wow6432Node\Microsoft\Cryptography\Wintrust\Config" -Force
New-ItemProperty -Path "HKLM:\Software\Wow6432Node\Microsoft\Cryptography\Wintrust\Config" `
  -Name "EnableCertPaddingCheck" -Value 1 -PropertyType DWORD -Force
```

Both returned `EnableCertPaddingCheck : 1`, confirmed. In a production environment this would not be an immediate edit on a live DC; it would be a change request with a rollback plan and application-compatibility testing first, precisely because the mitigation can break legitimately signed installers. The lab is where you get to apply it directly and watch what happens.

## Scan Sequence: Control, Then Verification

A verification scan only proves a fix worked if it runs under the same conditions as the scan that found the problem. So there were three credentialed runs, not two, and the middle one was a control.

The initial scan (11:41 to 12:00) found CVE-2013-3900. Before touching the vulnerability, I disabled Remote Registry on the DC at 12:13:16 and ran a second scan (12:14 to 12:30) as a control. That run answered a specific question: does the finding depend on Remote Registry's manual state, or will it reproduce when the service starts the way Nessus starts it? Nessus auto-started Remote Registry, completed the scan, and returned the same 52 findings as the initial run, CVE-2013-3900 among them. The finding was real and reproducible, and the scan conditions were now pinned.

Only then did I apply the remediation (shown above, around 12:38), and the verification scan (12:40 to 12:56) ran under conditions identical to the control: Remote Registry disabled beforehand, Nessus auto-starting it. This time CVE-2013-3900 was absent; the total dropped from 52 to 50, with every remaining finding informational. Because the control and the verification differed in exactly one variable, the registry fix, the finding's disappearance is attributable to the remediation and not to a change in how the scan ran. That is the difference between "I changed something and the finding went away" and "I isolated the fix as the cause."

| Scan | Window | Total | CVE-2013-3900 | Remote Registry |
|------|--------|-------|---------------|-----------------|
| Lab 01, uncredentialed baseline | prior session | 56 | not visible (uncredentialed) | n/a |
| Credentialed, initial | 11:41–12:00 | 52 | present | enabled |
| Credentialed, control | 12:14–12:30 | 52 | present | disabled beforehand; Nessus auto-started |
| Credentialed, verification | 12:40–12:56 | 50 (all INFO) | absent | disabled beforehand; Nessus auto-started |

Scan, find, fix, prove, and control the one scan you use as proof. A finding without a verification scan is an assertion; a verification scan without a matched control is a weaker one than it looks.

### Independent Evidence: the Service Control Manager Log

The DC's own System log corroborates the Remote Registry behavior without relying on Nessus's account of itself. Querying the Service Control Manager events:

```powershell
Get-WinEvent -LogName System |
  Where-Object { $_.Id -in 7036,7040 -and $_.Message -match "Remote Registry" } |
  Select-Object TimeCreated, Id, Message | Format-List
```

Event ID 7040 records service start-type changes; Event ID 7036 records the service entering the running and stopped states. Together they show Remote Registry being started around each scan window and its start type returned to disabled afterward, which confirms Nessus managed the service's lifecycle itself rather than leaving it running. This is the same move as resolving the memory lab's `shutdown.exe` from Event ID 1074: when the tooling's behavior is load-bearing, confirm it from the host's own logs instead of taking the tool's word for it.

## What I Took Away

**A clean uncredentialed result is often blindness, not hardening.** The DC looked spotless from the outside and carried a KEV-listed HIGH on the inside. The uncredentialed baseline was not wrong, it was answering a narrower question than it appeared to; naming which question a scan actually answered is half of reading one correctly.

**Prioritization lives in the gap between CVSS and EPSS.** The finding that mattered was an old CVE with a high exploitation probability, and the only way to rank it correctly is to read likelihood and active-exploitation signals alongside the base score, not instead of each other.

**Posture is part of the finding, and you verify it from the logs.** The Tier 0 credential shortcut and Remote Registry's lifecycle are not footnotes; a credentialed-scan program that ignores how it authenticates and what it leaves running has traded one exposure for another. Here the DC's Service Control Manager log confirmed Nessus restored the disabled start type after each scan, which is exactly the kind of claim worth checking against the host rather than taking on faith.

**Verification is the deliverable.** The remediation was two registry values; the thing worth showing is the before-and-after scan that proves they closed the finding. That is the shape of real vulnerability management, and it is the shape this lab was built to practice.

## Lab Environment

| Component | Details |
|-----------|---------|
| Target | LAB-WSDC-22, Windows Server 2022, Tier 0 DC, lab.local forest |
| Scanner | Nessus Essentials, Local Scanner on LAB-KALI, credentialed Basic Network Scan |
| Credential | Domain Administrator (documented lab shortcut; least-privilege account is the production pattern) |
| Network | Isolated host-only VirtualBox network (192.168.10.0/24), no external exposure |

## Follow-On Items

- Create a dedicated least-privilege scan account for future credentialed scans; on a DC this needs an AD-based approach rather than a local group.
- Complete the plugin-ID diff between the uncredentialed and credentialed scans, which requires the Professional tier or an API workaround.
- Next session: Azure VM forensic snapshot, extending the forensic work from the memory-capture lab into the cloud.

---

*Part of the [cloud-security-lab-notes](https://github.com/JNewbrey87/cloud-security-lab-notes) portfolio, building toward Cloud Security Analyst and the CySA+ / SC-200 / SC-500 certifications.*
