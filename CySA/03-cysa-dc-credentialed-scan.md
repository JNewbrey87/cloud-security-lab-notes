# Credentialed Vulnerability Scanning: Finding What the Perimeter Cannot See

**Project:** CySA+ Lab Track, Vulnerability Management  
**Session:** 03 | **Platform:** VirtualBox on Windows 11 Pro, isolated host-only network  
**Target:** LAB-WSDC-22 (Windows Server 2022, Tier 0 Domain Controller, lab.local)  
**Tool:** Nessus Essentials, credentialed Basic Network Scan  
**MITRE ATT&CK (relevance):** [T1553.002 — Subvert Trust Controls: Code Signing](https://attack.mitre.org/techniques/T1553/002/)  
**Cert Alignment:** CySA+ CS0-004, Domain 2 (Vulnerability Management): authenticated scanning, finding triage, remediation, verification  
**Date:** October 2026  

---

## Why I Did This One

In my [first lab in this series](01-cysa-vuln-management-baseline.md), I ran an uncredentialed Nessus scan against my homelab, and my domain controller came back clean: 56 findings, all informational. I wasn't sure I trusted that, since an uncredentialed scan can only see what's exposed from the outside.

Vulnerability scanning and patching are already part of my infrastructure work, so a credentialed Nessus scan wasn't new to me. What I wanted from this lab was the part that tends to get skipped when you're working through a patch queue: slowing down on one finding to understand why it actually matters, proving the fix with a controlled re-scan instead of trusting a single pass, and checking the host's own logs to see what the scanner really did on the box.

## Scan Configuration

| Field | Value |
|-------|-------|
| Policy | Basic Network Scan, Windows credentialed (password auth) |
| Target | LAB-WSDC-22 (192.168.10.50) |
| Account | Domain Administrator (lab shortcut, see below) |
| Remote Registry | Started by Nessus for every run; disabled on the DC at 12:13:16 before the control scan; Nessus restored the disabled start type afterward (confirmed in the Service Control Manager log) |

Two shortcuts I want to be upfront about, since scanning a Tier 0 machine is as much about how you do it as what you find.

First, I used a Domain Admin account and scanned the DC from the next segment over. That breaks Tier 0 separation, and it's a lab shortcut, not how I'd do it for real. A production setup would use a dedicated least-privilege scan account with only the rights Nessus needs (Remote Registry read, WMI, specific SMB access), and run the scanner from inside the Tier 0 boundary.

Second, a Windows credentialed scan relies on the Remote Registry service to read local configuration. I disabled it on the DC at 12:13:16, before the control scan described below, to see whether that would block the scan. It didn't. Nessus starts Remote Registry when it needs it, runs the scan, and sets the start type back to how it found it when it's done. That means Nessus does change the target while it scans, but the footprint is small and I was able to document it from the DC's own logs.

## What Credentials Turned Up

The credentialed scan came back with 52 findings, including one HIGH. The uncredentialed baseline had 56, all informational. At a glance, the drop from 56 to 52 looks like the credentialed scan weeding out noise, but that's not what happened. Credentialed and uncredentialed scans run different sets of plugins, so the count moved because the mix changed. A plugin-by-plugin comparison would show exactly what changed, but Nessus Essentials won't export scans (that's a Professional feature), so I captured the results with screenshots and notes instead. What I can say for sure is the part that matters: the HIGH finding wasn't in the uncredentialed scan at all, because you can only see it from inside the box.

## The HIGH Finding: CVE-2013-3900

| Field | Value |
|-------|-------|
| Plugin ID | 166555 (type: local) |
| Title | WinVerifyTrust Signature Validation (EnableCertPaddingCheck) |
| CVSS v3 | 8.8 |
| VPR | 9.5 |
| EPSS | 0.4465 |
| CISA KEV | Yes, added 2022-01-10 (BOD 22-01) |

By default, Windows doesn't strictly validate the structure of Authenticode-signed files. The fix is two registry values, and neither existed on the DC:

```
HKLM\Software\Microsoft\Cryptography\Wintrust\Config\EnableCertPaddingCheck
HKLM\Software\Wow6432Node\Microsoft\Cryptography\Wintrust\Config\EnableCertPaddingCheck
```

Without them, someone can take a validly signed executable, stuff extra malicious content into its padding, and it still passes signature checks. A user has to run the file, so it doesn't spread on its own, but it's on CISA's Known Exploited Vulnerabilities list, so it's being used in real attacks. Microsoft made the fix opt-in because strict checking can break some legitimately signed third-party installers, which is why on a production DC I'd test it before rolling it out.

The reason only the credentialed scan caught it is the plugin type. 166555 is a local check that reads the registry directly. An uncredentialed scanner can't look at the registry at all; it can only guess at configuration from how the machine behaves on the network. So this whole category of finding, registry values, local policy, installed software versions, is invisible without credentials.

## Why This One Went to the Top

This CVE is from 2013 and had been sitting on this machine unmitigated the whole time. It's a good example of why I don't prioritize on base score alone. CVSS 8.8 describes how bad it could be. EPSS at 0.4465, which comes from FIRST's Exploit Prediction Scoring System (Nessus displays it but doesn't calculate it), estimates roughly a 44% chance of exploitation in the next thirty days. Add the VPR of 9.5 and the KEV listing, and it's clearly being exploited out there. An old CVE that's actively exploited should outrank a newer one with a scarier base score and a low EPSS.

## The Fix

The `Wintrust\Config` key didn't exist under either path, so I had to create it before I could set the value, on both the 64-bit and the 32-bit (Wow6432Node) side:

```powershell
New-Item -Path "HKLM:\Software\Microsoft\Cryptography\Wintrust\Config" -Force
New-ItemProperty -Path "HKLM:\Software\Microsoft\Cryptography\Wintrust\Config" `
  -Name "EnableCertPaddingCheck" -Value 1 -PropertyType DWORD -Force

New-Item -Path "HKLM:\Software\Wow6432Node\Microsoft\Cryptography\Wintrust\Config" -Force
New-ItemProperty -Path "HKLM:\Software\Wow6432Node\Microsoft\Cryptography\Wintrust\Config" `
  -Name "EnableCertPaddingCheck" -Value 1 -PropertyType DWORD -Force
```

Both came back `EnableCertPaddingCheck : 1`. On a production DC this would go through a change request with a rollback plan and compatibility testing first, because of the installer-breakage risk. In the lab I could just apply it and see what happened.

## Proving It: A Control Scan, Then Verification

I didn't want to just re-scan once and call it fixed. If something else changed between scans, a clean result wouldn't prove the registry fix did anything. So I ran three credentialed scans, and the middle one was a control.

The first scan (11:41 to 12:00) found CVE-2013-3900. Before fixing anything, I disabled Remote Registry on the DC at 12:13:16 and ran a second scan (12:14 to 12:30). The question I wanted that run to answer was whether the finding would still show up with Remote Registry disabled beforehand and Nessus starting it on its own. It did: same 52 findings as the first run, CVE-2013-3900 included. So the finding was real, and I had my scan conditions locked in.

Then I applied the fix (around 12:38) and ran the verification scan (12:40 to 12:56) under exactly the same conditions as the control. This time CVE-2013-3900 was gone, and the total dropped from 52 to 50, all informational. Since the control and the verification only differed in one way, the registry fix, I could say the fix is what made the finding go away, not something about how the scan ran.

| Scan | Window | Total | CVE-2013-3900 | Remote Registry |
|------|--------|-------|---------------|-----------------|
| cysa-01, uncredentialed baseline | prior session | 56 | not visible (uncredentialed) | n/a |
| Credentialed, initial | 11:41–12:00 | 52 | present | started by Nessus |
| Credentialed, control | 12:14–12:30 | 52 | present | disabled beforehand; started by Nessus |
| Credentialed, verification | 12:40–12:56 | 50 (all INFO) | absent | disabled beforehand; started by Nessus |

### Checking the DC's Own Logs

I didn't want to just take Nessus's word for how it handled Remote Registry, so I checked the DC's Service Control Manager events:

```powershell
Get-WinEvent -LogName System |
  Where-Object { $_.Id -in 7036,7040 -and $_.Message -match "Remote Registry" } |
  Select-Object TimeCreated, Id, Message | Format-List
```

Event ID 7040 logs changes to a service's start type, and 7036 logs a service starting and stopping. Together they showed Remote Registry starting up around each scan and its start type going back to disabled afterward. That matched what I expected, and now I had the host's own record of it. It's the same habit as checking Event ID 1074 for `shutdown.exe` in my memory capture lab: if a tool's behavior matters to your conclusion, confirm it from the machine's logs.

## What I Took Away

**A clean outside scan doesn't mean a clean machine.** The DC looked spotless from the outside and had a KEV-listed HIGH on the inside. The uncredentialed scan wasn't wrong. It was answering a narrower question than I first gave it credit for.

**Exploitation likelihood matters as much as the base score.** The finding that mattered most was an old CVE that's actively exploited. I only ranked it correctly because I looked at EPSS, VPR, and the KEV listing alongside the CVSS score.

**How you scan is part of the result.** The Domain Admin shortcut and what Nessus does to Remote Registry aren't side notes. If a scanning setup ignores how it logs in and what it leaves running, it's just traded one exposure for another. Checking the DC's logs is how I confirmed Nessus put things back the way it found them.

**The control scan is what makes the verification hold up.** The fix itself was two registry values. The scan sequence is what proves those values closed the finding, and the control run is what rules out anything else explaining the result.

## Lab Environment

| Component | Details |
|-----------|---------|
| Target | LAB-WSDC-22, Windows Server 2022, Tier 0 DC, lab.local forest |
| Scanner | Nessus Essentials, Local Scanner on LAB-KALI, credentialed Basic Network Scan |
| Credential | Domain Administrator (documented lab shortcut; least-privilege account is the production pattern) |
| Network | Isolated host-only VirtualBox network (192.168.10.0/24), no external exposure |

## What Came Next

- **[cysa-04](04-cysa-azure-vm-forensic-snapshot.md):** cloud forensic acquisition on an Azure VM, carrying the evidence work from my memory capture lab into the cloud.
- **[cysa-05](05-cysa-ir-capstone.md):** the incident response capstone, which pulled this and the other labs into one incident.

Still on my list from this one:

- Set up a dedicated least-privilege scan account for future credentialed scans. On a DC that needs an AD-based approach rather than a local group.
- Do the plugin-by-plugin comparison between the uncredentialed and credentialed scans, which needs the Professional tier or an API workaround.

---

*Part of the [cloud-security-lab-notes](https://github.com/JNewbrey87/cloud-security-lab-notes) portfolio, building toward Cloud Security Analyst and the CySA+ / SC-200 / SC-500 certifications.*
