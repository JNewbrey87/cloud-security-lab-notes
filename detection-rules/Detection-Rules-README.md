# Custom Detection Rules

These are the custom Wazuh rules I've written across the detection labs in this repo. Each one exists because a lab needed it, and the table below says plainly which ones I've seen fire against a real simulated attack and which ones are deployed but still waiting on a test.

The deployable ruleset is [`local_rules.xml`](local_rules.xml).

## Deployment

On the Wazuh manager:

```bash
sudo cp local_rules.xml /var/ossec/etc/rules/local_rules.xml
sudo systemctl restart wazuh-manager
```

Custom rule IDs start at 100000. Wazuh reserves the 100000+ range for user-defined rules, so they never collide with the built-in ruleset.

## Rule Catalog

| Rule ID | Level | Trigger | MITRE ATT&CK | Status |
|---------|-------|---------|--------------|--------|
| 100002 | 10 | Windows failed logon (Event 4625) | [T1110](https://attack.mitre.org/techniques/T1110/) | Validated: [P1-1a](../00-LAB-P1-1a.md), and on a domain host in [P1-1c](../00-LAB-P1-1c.md) (17 firings) |
| 100003 | 10 | SSH brute force: 3+ PAM auth failures in 60 seconds from the same source IP | [T1110](https://attack.mitre.org/techniques/T1110/), [T1110.001](https://attack.mitre.org/techniques/T1110/001/) | Validated: first version in [P1-1b](../00-LAB-P1-1b.md) (Ubuntu 22.04); rebuilt and revalidated in [P2-1b](../00-LAB-P2-1b.md) (Ubuntu 25.04) |
| 100004 | 12 | Explicit credential use toward a remote host (Event 4648), lateral movement source | [T1021.002](https://attack.mitre.org/techniques/T1021/002/) | Validated: [P1-1d](../00-LAB-P1-1d.md) (3 hits) |
| 100005 | 14 | Failed network logon on the Tier 0 DC (Event 4625, logon type 3), lateral movement arrival | [T1021.002](https://attack.mitre.org/techniques/T1021/002/) | Deployed in [P1-1d](../00-LAB-P1-1d.md); validated live in a follow-up session on June 27, 2026 |
| 100006 | 12 | New Windows local account created (Event 4720) | [T1136.001](https://attack.mitre.org/techniques/T1136/001/) | Added in [P2-1b](../00-LAB-P2-1b.md); not yet fired in a lab |
| 100007 | 12 | New Windows service installed (Event 7045) | [T1543.003](https://attack.mitre.org/techniques/T1543/003/) | Added in [P2-1b](../00-LAB-P2-1b.md); not yet fired in a lab |
| 100008 | 10 | Windows account lockout (Event 4740) | [T1110](https://attack.mitre.org/techniques/T1110/) | Added in [P2-1b](../00-LAB-P2-1b.md); not yet fired in a lab |
| 100009 | 5 | SSH auth failure with remote host attribution (Ubuntu 25.04 PAM single-failure format) | [T1110](https://attack.mitre.org/techniques/T1110/) | Added in [P2-1b](../00-LAB-P2-1b.md); not yet fired in a lab |

## Kill Chain Coverage

Rules 100002, 100004, and 100005 cover the two-stage kill chain from [P1-1d](../00-LAB-P1-1d.md):

```
100002  brute force on the Tier 1 jump box     (T1110)
   |
100004  explicit credential use outbound       (T1021.002), source host
   |
100005  failed network logon on the Tier 0 DC  (T1021.002), target DC
```

I tiered the rule levels on purpose. Activity reaching the Tier 0 domain controller (100005, level 14) scores higher than the same technique on a Tier 1 host (100004, level 12), which scores higher than a single failed logon on an endpoint (100002, level 10). Severity follows how far into the environment the attacker has gotten, not just the event type.

## Rule Notes

### 100003: The Ubuntu 25.04 PAM Log Path

I first wrote this rule on Ubuntu 22.04, keyed off Wazuh's standard SSH failure rules. When I rebuilt the lab on Ubuntu 25.04, it silently stopped firing. The cause: 25.04 uses `sshd-session`, which routes authentication failures through PAM, and the main brute-force event now decodes as rule **2502** ("User missed the password more than one time"). The rule now keys off `if_matched_sid` 2502.

It needs two things on the target host:

- `PerSourcePenalties no` in `/etc/ssh/sshd_config`. Ubuntu 25.04 rate-limits repeated connections before authentication happens, which means no auth logs and nothing to detect. It's a real hardening feature; turning it off is a lab-only choice.
- `/var/log/auth.log` registered as a `localfile` (syslog format) in `ossec.conf`.

Full write-up: [P2-1b](../00-LAB-P2-1b.md).

### 100005: NTLM Fallback, Not Kerberos

When `net use` targets the domain controller by IP address, Windows falls back to NTLM instead of Kerberos. A failed attempt then shows up on the DC as Event 4625 with logon type 3, **not** Event 4771 (Kerberos pre-authentication failure). A detection that assumes 4771 for lateral movement toward a DC would miss this path entirely. The logon type 3 filter matters too: without it, failed interactive logons (type 2) would trigger the rule as well.

### 100009: A Diagnostic Supplement

Ubuntu 25.04's `sshd-session` writes two PAM formats: a singular "PAM 1 more authentication failure" (rule 5711) and a plural "PAM N more authentication failures" (rule 2502). Rule 100009 catches the single-failure format when it carries an `rhost=` field, which attaches a source IP to events that would otherwise come through without one. It runs alongside 100003 rather than feeding it.

## Active Response

Rule 100003 is wired to Wazuh's built-in `firewall-drop` active response. When it fires, the manager adds an `iptables` DROP for the attacker's source IP and removes it after 180 seconds. I confirmed the block with `iptables -L -n` during [P2-1b](../00-LAB-P2-1b.md).

```xml
<active-response>
  <command>firewall-drop</command>
  <location>local</location>
  <rules_id>100003</rules_id>
  <timeout>180</timeout>
</active-response>
```

Auto-blocking at the SIEM is fine in a lab, where a false positive just locks out my attacker VM for three minutes. In production it belongs behind analyst confirmation or a SOAR playbook, because a false positive there blocks real traffic. I built it to learn how it works and where that judgment call sits.

## Platform

All rules run on Wazuh v4.14.5 on an isolated VirtualBox host-only network with no outside exposure. Windows telemetry comes through the Wazuh agent's `eventchannel` collector; Linux telemetry comes from `/var/log/auth.log`.

## What's Next

**On the roadmap:** translating these rules into KQL analytic rules for Microsoft Sentinel.

---

*Part of the [cloud-security-lab-notes](https://github.com/JNewbrey87/cloud-security-lab-notes) portfolio, building toward Cloud Security Analyst and the CySA+ / SC-200 / SC-500 certifications.*
