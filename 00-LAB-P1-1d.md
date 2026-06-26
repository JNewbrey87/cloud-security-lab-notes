# Lateral Movement Detection — Rules 100004 + 100005, Two-Stage Kill Chain

**Project:** P1 — Wazuh SIEM Home Lab
**Phase:** 1c | **Platform:** VirtualBox on Windows 11 Pro
**MITRE ATT&CK:** [T1110 — Brute Force](https://attack.mitre.org/techniques/T1110/) / [T1021.002 — Remote Services: SMB/Windows Admin Shares](https://attack.mitre.org/techniques/T1021/002/)
**Cert Alignment:** CySA+ CS0-003, SC-200, SC-500
**Date Completed:** June 2026

---

## Scenario

Labs 1a and 1b built detection for two discrete events: a failed Windows logon and an SSH brute force burst. Both caught a single stage of attack. This lab chains them together and adds the next stage of the kill chain: lateral movement from a compromised jump box toward a Tier 0 domain controller.

The core question is whether Wazuh can see both sides of a lateral movement attempt — the outbound credential use from the attacker-controlled host and the incoming failed network logon on the target DC — as two correlated, high-severity events. The answer is yes, but it requires two different custom rules, two different Windows Event IDs, and an agent on the domain controller.

This lab deploys rules 100004 and 100005, installs Wazuh agent 003 on the DC, runs a two-stage attack from brute force to lateral movement, and validates the full kill chain in the Wazuh Threat Hunting dashboard.

---

## Architecture

```
[LAB-WSJP-25 — Tier 1 Jump Box]
  Windows Server 2025
  Wazuh agent 002
  192.168.10.60
  RDP-exposed, SMB share host (intentional attack surface)
        |
        |  Stage 1: runas /user:fakeuser123   → Event 4625 → Rule 100002 (T1110, level 10)
        |  Stage 2: net use \\DC\ADMIN$       → Event 4648 → Rule 100004 (T1021.002, level 12)
        |
        v
[LAB-WSDC-22 — Tier 0 Domain Controller]
  Windows Server 2022
  Wazuh agent 003
  192.168.10.50
  AD DS + DNS, lab.local — console-only, no RDP
        |
        |  Stage 2 DC-side: incoming failed network logon
        |  Event 4625 (logon type 3) → Rule 100005 (T1021.002, level 14)
        |
        v
[LAB-UBTU — Wazuh Manager]
  Ubuntu 22.04 LTS
  Wazuh Manager + Dashboard v4.14.5
  192.168.10.10
  local_rules.xml: rules 100002, 100003, 100004, 100005
```

> All hosts sit on an isolated, host-only VirtualBox network (vboxnet0, 192.168.10.0/24). No external exposure.

**Tiering rationale:** The domain controller is Tier 0 — the highest-trust asset in the domain. RDP is disabled by design. Any network logon attempt reaching it is automatically significant, which is why rule 100005 sits at level 14 (higher than anything on the jump box).

---

## MITRE ATT&CK Coverage

| Technique | ID | Rule | Event ID | Host |
|-----------|-----|------|----------|------|
| [Brute Force](https://attack.mitre.org/techniques/T1110/) | T1110 | 100002 | 4625 | LAB-WSJP-25 |
| [Remote Services: SMB/Windows Admin Shares](https://attack.mitre.org/techniques/T1021/002/) | T1021.002 | 100004 | 4648 | LAB-WSJP-25 |
| [Remote Services: SMB/Windows Admin Shares](https://attack.mitre.org/techniques/T1021/002/) | T1021.002 | 100005 | 4625 (type 3) | LAB-WSDC-22 |

---

## What I Built

### 1. Custom Rule 100004 — Lateral Movement via Explicit Credential Use (Event 4648)

Event 4648 fires on Windows when a process authenticates to a remote host using explicitly-supplied credentials — the exact footprint of `net use` or `Invoke-Command` targeting a domain resource. This fires on the source host (the jump box), capturing the outbound attempt.

**On LAB-UBTU** — open the Wazuh rules file:

```bash
sudo nano /var/ossec/etc/rules/local_rules.xml
```

Add the following inside the `<group>` block, after the closing `</rule>` tag of the last existing rule:

```xml
<!-- Rule 100004: Lateral Movement - Explicit Credential Use (Event 4648) -->
<rule id="100004" level="12">
  <if_group>windows</if_group>
  <field name="win.system.eventID">^4648$</field>
  <description>Custom: Explicit credential use toward remote host detected - possible lateral movement (Event 4648)</description>
  <mitre>
    <id>T1021.002</id>
  </mitre>
</rule>
```

Save and exit nano: `Ctrl+O` → `Enter` → `Ctrl+X`.

Validate syntax, then restart the manager:

```bash
sudo /var/ossec/bin/wazuh-logtest
# Type any text, press Enter, confirm no errors, then Ctrl+C to exit

sudo systemctl restart wazuh-manager
sudo systemctl status wazuh-manager
# Expected: Active: active (running)

sudo grep -r "100004" /var/ossec/etc/rules/
# Expected: your local_rules.xml line returned
```

**Key decisions:**
- Level 12 reflects the seriousness of explicit credential use toward a remote host; this is not routine user activity.
- The rule fires on Event 4648 specifically, not 4625. The distinction matters: 4625 is a failure record on the target; 4648 is an attempt record on the source. Having both means neither side of the exchange is invisible to the SIEM.
- `<if_group>windows</if_group>` scopes the match to the Windows eventchannel pipeline, avoiding noise from other log sources.

---

### 2. Wazuh Agent 003 — Installation on LAB-WSDC-22 (Tier 0 DC)

The DC needs its own agent to generate DC-side telemetry. Without it, lateral movement attempts are only visible from the jump box's perspective. Deploying agent 003 on a console-only Windows Server 2022 domain controller requires routing the installer through the LabShare SMB share on the jump box, since the DC has no RDP.

**On LAB-UBTU** — get the installer URL from the Wazuh dashboard:

Navigate to **Agents → Deploy New Agent → Windows**. Copy the MSI URL:
```
https://packages.wazuh.com/4.x/windows/wazuh-agent-4.14.5-1.msi
```

**On LAB-WSJP-25** — download the installer to LabShare:

```powershell
Invoke-WebRequest -Uri "https://packages.wazuh.com/4.x/windows/wazuh-agent-4.14.5-1.msi" `
  -OutFile "C:\Shares\LabShare\wazuh-agent-4.14.5-1.msi"
```

**On LAB-WSDC-22** (console-only) — copy from LabShare and install:

```powershell
Copy-Item "\\192.168.10.60\LabShare\wazuh-agent-4.14.5-1.msi" -Destination "C:\Temp\"

msiexec.exe /i "C:\Temp\wazuh-agent-4.14.5-1.msi" /q `
  WAZUH_MANAGER='192.168.10.10' `
  WAZUH_AGENT_NAME='LAB-WSDC-22'

NET START WazuhSvc
```

**On LAB-UBTU** — verify registration:

```bash
sudo /var/ossec/bin/agent_control -l
# Agent 003 (LAB-WSDC-22) should appear as Active within 1-2 minutes
# If status shows "Never connected", wait and re-run — the first check-in can take a few minutes
```

**Note on agent naming:** The agent name (`LAB-WSDC-22`) is set permanently at enrollment. It cannot be changed after the fact without re-registering the agent with a new ID. Match it to the VM name at install time.

---

### 3. Custom Rule 100005 — DC-Side Lateral Movement Arrival (Event 4625, Logon Type 3)

With agent 003 reporting from the DC, a second rule captures the arrival side of the lateral movement attempt. Event 4625 (failed logon) with logon type 3 specifically indicates a network logon attempt rather than an interactive one. On a Tier 0 DC, this is a high-confidence indicator of lateral movement.

**On LAB-UBTU** — add rule 100005 to `local_rules.xml` after rule 100004:

```xml
<!-- Rule 100005: Lateral Movement Arrival - Failed Network Logon on Tier 0 DC (Event 4625 type 3) -->
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

Save, validate syntax, and restart the manager (same steps as Section 1).

**Key decisions:**
- Level 14 — higher than rule 100004 (level 12) on the jump box. Any network authentication attempt reaching the Tier 0 DC is treated as a more significant signal than the attempt originating on a Tier 1 host.
- The `logonType` field filter is essential. Without it, the rule would also fire on interactive failed logons (type 2), which would overlap with rule 100002 and create duplicate high-severity alerts for non-lateral events.
- Rules 100004 and 100005 together give bilateral visibility: outbound attempt from the jump box and incoming impact on the DC, both correlated to [T1021.002](https://attack.mitre.org/techniques/T1021/002/).

---

## What I Detected

### Setting Up Live Monitoring

Start this on LAB-UBTU **before** running the attack. The `--line-buffered` flag is required; without it, grep buffers output and nothing appears in real time even when alerts are writing to the log.

```bash
sudo tail -f /var/ossec/logs/alerts/alerts.log | grep --line-buffered -E "Rule: 10000[245]"
```

---

### Stage 1 — Brute Force on the Jump Box

**On LAB-WSJP-25:**

```powershell
runas /user:fakeuser123 cmd    # press Ctrl+C at each password prompt
runas /user:fakeuser123 cmd
runas /user:fakeuser123 cmd
```

Each attempt generates Event 4625 (failed logon, unknown account) on LAB-WSJP-25. Rule 100002 fires immediately on each event.

**Expected output on LAB-UBTU (live tail):**

```
Rule: 100002 (level 10) -> 'Custom: Windows failed logon attempt detected (Event 4625)'
Rule: 100002 (level 10) -> 'Custom: Windows failed logon attempt detected (Event 4625)'
Rule: 100002 (level 10) -> 'Custom: Windows failed logon attempt detected (Event 4625)'
```

---

### Stage 2 — Lateral Movement Toward the DC

**On LAB-WSJP-25** — run immediately after Stage 1:

```powershell
# Primary method — SMB admin share connection with fabricated credentials
net use \\192.168.10.50\ADMIN$ /user:LAB\fakeadmin wrongpassword123
net use \\192.168.10.50\ADMIN$ /user:LAB\fakeadmin wrongpassword123
net use \\192.168.10.50\ADMIN$ /user:LAB\fakeadmin wrongpassword123

# Clean up any cached connection state
net use \\192.168.10.50\ADMIN$ /delete 2>$null
```

Each `net use` attempt generates Event 4648 (explicit credential use) on LAB-WSJP-25 and Event 4625 (logon type 3) on LAB-WSDC-22.

**PowerShell remoting variant** (generates same events, tests a different execution path):

```powershell
$cred = New-Object System.Management.Automation.PSCredential(
  "LAB\fakeadmin",
  (ConvertTo-SecureString "wrongpassword123" -AsPlainText -Force)
)
Invoke-Command -ComputerName 192.168.10.50 -Credential $cred `
  -ScriptBlock { hostname } -ErrorAction SilentlyContinue
```

**Expected output on LAB-UBTU (live tail):**

```
Rule: 100004 (level 12) -> 'Custom: Explicit credential use toward remote host detected - possible lateral movement (Event 4648)'
Rule: 100004 (level 12) -> 'Custom: Explicit credential use toward remote host detected - possible lateral movement (Event 4648)'
Rule: 100004 (level 12) -> 'Custom: Explicit credential use toward remote host detected - possible lateral movement (Event 4648)'
```

---

### Kill Chain Verification

**Post-run aggregate check — jump box side:**

```bash
sudo awk '/\(LAB-WSJP-25\) any->EventChannel/{getline; print}' \
  /var/ossec/logs/alerts/alerts.log | sort | uniq -c | sort -rn
```

**Post-run aggregate check — DC side:**

```bash
sudo awk '/\(LAB-WSDC-22\) any->EventChannel/{getline; print}' \
  /var/ossec/logs/alerts/alerts.log | sort | uniq -c | sort -rn
```

**Full alert detail for rule 100004:**

```bash
sudo grep -A 3 "Rule: 100004" /var/ossec/logs/alerts/alerts.log | tail -30
```

**Confirm all rules are loaded:**

```bash
sudo grep -r "10000[2345]" /var/ossec/etc/rules/
```

### Confirmed Alert Counts (End of Session)

| Rule | Event ID | Host | Hits | Technique |
|------|----------|------|------|-----------|
| 100002 | 4625 | LAB-WSJP-25 | 3 | [T1110](https://attack.mitre.org/techniques/T1110/) |
| 100004 | 4648 | LAB-WSJP-25 | 3 | [T1021.002](https://attack.mitre.org/techniques/T1021/002/) |
| 100002 | 4625 | LAB-WSDC-22 | 1 | [T1021.002](https://attack.mitre.org/techniques/T1021/002/) |

**Wazuh Threat Hunting dashboard (level 12+ filter, last 24h):** 11 total high-severity hits confirmed, including the 7 lab-generated alerts above. The kill chain is visible as a sequence in the dashboard: failed logons on the jump box, followed immediately by explicit credential use from the same host, followed by a network logon failure arriving at the DC.

**Rule 100005 status:** Deployed and live; DC-side network logon events (4625 type 3) are confirmed flowing to the manager. Full live-run validation with `--line-buffered` tail monitoring is pending next session.

---

## What I Learned

**Event 4648 and 4625 are two sides of the same lateral movement attempt.** Rule 100004 fires on the jump box when Windows records the explicit credential use going outbound. Rule 100005 fires on the DC when the resulting failed logon arrives. Together they give bilateral visibility: one rule shows intent, the other shows impact. Either rule in isolation is useful; together they correlate source and destination without requiring manual log correlation.

**Logon type is the filter that separates lateral movement from interactive failed logons.** Event 4625 fires for every failed authentication, whether a user mistyped a password interactively (logon type 2) or an attacker tried to authenticate over the network (logon type 3). Without the `logonType` field filter in rule 100005, every failed interactive logon on the DC would generate a level 14 alert. The filter is what makes the rule precise enough to be actionable.

**The `--line-buffered` flag is not optional when piping `tail -f` to `grep`.** grep buffers output when writing to a pipe rather than a terminal. The default behavior — silent output despite active log writes — looks exactly like a detection failure. It's a straightforward fix, but it cost time troubleshooting what looked like a Wazuh pipeline issue before identifying the real cause. Every live-monitoring command in this lab now includes `--line-buffered`.

**Wazuh's agent-to-manager pipeline has meaningful latency under light load.** Even with the agent active and healthy, up to 8 minutes elapsed between event generation on the endpoint and alert write to `alerts.log` on the manager during some test runs. For detection validation work, this matters: a tail window that shows nothing is not evidence the rule failed; it may just mean the event hasn't arrived yet. Post-run aggregate checks with `awk` and `grep -A` are more reliable for validation than live tail when latency is a factor.

**The SMB LabShare as an installation relay is the right pattern for console-only hosts.** Installing agent 003 on a Tier 0 DC with no RDP required routing the MSI through the jump box's LabShare. This is also realistic: in a production environment, software deployment to air-gapped or restricted hosts follows a similar distribution pattern through a managed share or deployment server. Designing the lab with this constraint built in makes the installation procedure closer to real-world operations.

---

## Troubleshooting

### bash history expansion — `!` in passwords

**Symptom:** `bash: 10: event not found` when a password containing `!` is passed to a command on LAB-UBTU.

**Cause:** bash treats `!number` as history expansion in interactive shells.

**Fix:** Disable history expansion for the session, or use single quotes:

```bash
set +H

# Or: single-quote the password argument directly
sudo /usr/share/wazuh-indexer/plugins/opensearch-security/tools/wazuh-passwords-tool.sh \
  -u admin -p 'YourPassword-Here'
```

---

### Wazuh password tool rejects the password

**Symptom:**
```
ERROR: The password must have a length between 8 and 64 characters and contain
at least one upper and lower case letter, a number and a symbol(.+?-).
```

**Cause:** The Wazuh password tool only accepts four symbols: `.` `+` `?` `-`. Common symbols like `!` `$` `%` `@` are all rejected, regardless of password complexity otherwise.

**Fix:** Use a password that includes only those four symbols. Example: `Wazuh-Lab2026`.

---

### `tail -f | grep` shows no output during live attack

**Symptom:** The tail window shows nothing even while attack commands are running on the jump box.

**Cause 1:** grep buffers output when writing to a pipe. Without `--line-buffered`, output only appears when the buffer fills.

**Cause 2:** Wazuh agent-to-manager pipeline latency. Events may take several minutes to appear in `alerts.log` after generation on the endpoint.

**Fix:**

```bash
sudo tail -f /var/ossec/logs/alerts/alerts.log | grep --line-buffered -E "Rule: 10000[245]"
```

If alerts still don't appear within a few minutes, run the post-run aggregate check instead:

```bash
sudo awk '/\(LAB-WSJP-25\) any->EventChannel/{getline; print}' \
  /var/ossec/logs/alerts/alerts.log | sort | uniq -c | sort -rn
```

---

## Lab Environment

| Component | Details |
|-----------|---------|
| Host | Windows 11 Pro, VirtualBox 7 |
| SIEM Manager | Ubuntu 22.04 LTS, Wazuh v4.14.5 (LAB-UBTU, 192.168.10.10) |
| Domain Controller — Tier 0 | Windows Server 2022, AD DS + DNS, `lab.local`, console-only, Wazuh agent 003 (LAB-WSDC-22, 192.168.10.50) |
| Jump Box — Tier 1 | Windows Server 2025, domain-joined, RDP + SMB share, Wazuh agent 002 (LAB-WSJP-25, 192.168.10.60) |
| Network | Isolated host-only VirtualBox network (vboxnet0), 192.168.10.0/24, static IPs, no external exposure |

---

## Up Next

- **Rule 100005 live validation:** Re-run the two-stage attack with `--line-buffered` monitoring active to confirm real-time DC-side detection against logon type 3 events.
- **Phase 2 — Azure Integration:** Onboard the jump box to Azure Arc and begin Microsoft Sentinel integration; KQL detection engineering for the same kill chain observed here.
- **Expanded attack narrative:** Post-exploitation on the DC (privilege escalation, LSASS dump simulation, new detection rules).

---

*Part of the [cloud-security-lab-notes](https://github.com/JNewbrey87/cloud-security-lab-notes) portfolio, building toward Cloud Security Analyst and SC-200/CySA+/SC-500 certifications.*
