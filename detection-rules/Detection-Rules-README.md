# Custom Detection Rules

Wazuh custom detection rules developed and validated across the Phase 1 and Phase 2 lab series. Every rule here was written by hand, fired against a real simulated attack, and confirmed in the Wazuh dashboard or `alerts.log` before being committed. Nothing in this catalog is theoretical.

The deployable ruleset is [`local_rules.xml`](local_rules.xml).

## Deployment

On the Wazuh manager:

```bash
sudo cp local_rules.xml /var/ossec/etc/rules/local_rules.xml
sudo systemctl restart wazuh-manager
```

Custom rule IDs start at 100000 per Wazuh convention (the 100000+ range is reserved for user-defined rules and never collides with the built-in ruleset).

## Rule Catalog

| Rule ID | Level | Trigger | MITRE ATT&CK | Validated In |
|---------|-------|---------|--------------|--------------|
| 100002 | 10 | Windows failed logon (Event 4625) | [T1110](https://attack.mitre.org/techniques/T1110/) | P1-1a |
| 100003 | 10 | SSH brute force — 3+ PAM auth failures in 60s, same source IP | [T1110](https://attack.mitre.org/techniques/T1110/), [T1110.001](https://attack.mitre.org/techniques/T1110/001/) | P2-1b |
| 100004 | 12 | Explicit credential use outbound (Event 4648) — lateral movement source | [T1021.002](https://attack.mitre.org/techniques/T1021/002/) | P1-1d |
| 100005 | 14 | Failed network logon on Tier 0 DC (Event 4625, logon type 3) — lateral movement arrival | [T1021.002](https://attack.mitre.org/techniques/T1021/002/) | P1-1d |
| 100006 | 12 | New Windows local account created (Event 4720) | [T1136.001](https://attack.mitre.org/techniques/T1136/001/) | P2-1b |
| 100007 | 12 | New Windows service installed (Event 7045) | [T1543.003](https://attack.mitre.org/techniques/T1543/003/) | P2-1b |
| 100008 | 10 | Windows account lockout (Event 4740) | [T1110](https://attack.mitre.org/techniques/T1110/) | P2-1b |
| 100009 | 5 | SSH auth failure with remote host attribution (Ubuntu 25.04 PAM single-failure) | [T1110](https://attack.mitre.org/techniques/T1110/) | P2-1b |

## Kill Chain Coverage

Rules 100002, 100004, and 100005 form the validated two-stage kill chain documented in [00-LAB-P1-1d.md](../00-LAB-P1-1d.md):

```
100002  brute force on Tier 1 jump box        (T1110)
   │
100004  explicit credential use outbound      (T1021.002)  — source host
   │
100005  failed network logon on Tier 0 DC     (T1021.002)  — target DC
```

Rule levels are deliberately tiered: activity reaching the Tier 0 domain controller (100005, level 14) is scored higher than the same technique on a Tier 1 host (100004, level 12), which is scored higher than a single endpoint failure (100002, level 10). Severity tracks position in the kill chain, not just event type.

## Rule Notes

### 100003 — the Ubuntu 25.04 PAM log path

This rule was originally written for Ubuntu 22.04, keyed off Wazuh's standard SSH failure decoder (rule 5700). On Ubuntu 25.04 it silently stopped firing. Root cause: 25.04 uses `sshd-session`, which routes authentication failures through PAM. The dominant brute-force event decodes as rule **2502** ("User missed the password more than one time"), not 5700. The rule now keys off `if_matched_sid=2502`.

Two prerequisites on the target host:

- `PerSourcePenalties no` in `/etc/ssh/sshd_config` — 25.04 rate-limits repeated connections at the transport layer before authentication, which suppresses all auth logging (and therefore all detection). This is a real hardening feature; disabling it is a lab-only choice.
- `/var/log/auth.log` registered as a `localfile` (syslog format) in `ossec.conf`.

Full write-up: [00-LAB-P2-1b.md](../00-LAB-P2-1b.md).

### 100005 — NTLM fallback, not Kerberos

When `net use` targets the domain controller by IP address, Windows falls back to NTLM authentication rather than Kerberos. A failed attempt therefore generates Event 4625 (logon type 3) on the DC — **not** Event 4771 (Kerberos pre-authentication failure). Detection engineering that assumes 4771 for lateral-movement-to-DC will miss this path entirely. The `logonType 3` field filter is essential; without it, interactive failed logons (type 2) also trigger the rule.

### 100009 — diagnostic supplement

Ubuntu 25.04 sshd-session emits two PAM formats: a singular "PAM 1 more authentication failure" (rule 5711) and a plural "PAM N more authentication failures" (rule 2502). Rule 100009 captures the single-failure format when it carries a `rhost=` field, giving remote-host attribution on events that would otherwise pass without a source IP. It runs alongside 100003 rather than feeding it.

## Active Response

Rule 100003 is wired to Wazuh's built-in `firewall-drop` active response. On trigger, the manager inserts an `iptables` DROP for the attacker's source IP and removes it after 180 seconds. Configuration and validation (confirmed via `iptables -L INPUT -n`) are documented in [00-LAB-P2-1b.md](../00-LAB-P2-1b.md).

```xml
<active-response>
  <command>firewall-drop</command>
  <location>local</location>
  <rules_id>100003</rules_id>
  <timeout>180</timeout>
</active-response>
```

Automated blocking at the SIEM layer is high-confidence in a lab; in production it belongs behind analyst confirmation or a SOAR playbook, since a false positive blocks legitimate traffic. The capability is demonstrated here; the judgment about when to automate it is the point.

## Platform

All rules validated on Wazuh v4.14.5 running on an isolated VirtualBox host-only network (no external exposure). Windows telemetry via the Wazuh agent's `eventchannel` collector; Linux telemetry via `/var/log/auth.log`.

---

*Part of the [cloud-security-lab-notes](https://github.com/JNewbrey87/cloud-security-lab-notes) portfolio.*
