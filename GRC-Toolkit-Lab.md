# GRC Toolkit Lab - SOC2 & GLBA Gap Assessment

**Date:** May 19, 2026  
**Environment:** WSL2 (Ubuntu) on Windows 11 
**Tool:** [claude-grc-engineering](https://github.com/GRCEngClub/claude-grc-engineering) by GRC Engineering Club  
**Frameworks Assessed:** SOC 2 (AICPA Trust Services Criteria), GLBA Safeguards Rule (16 C.F.R. Part 314)  
**Status:** Phase 1 complete - findings documented; remediation in progress

---

## Objective

This lab is foundational work in a broader series. I ran the github-inspector assessment here first to get comfortable with the toolkit's architecture, invocation method, and output format before applying it to more complex environments; specifically, the Hyper-V lab sessions on the home build and the Azure tenant I'm standing up for AZ-500 and SC-200 prep. Getting the toolchain functional and documented at a simpler scope first means less friction in later labs where the configuration surface is significantly larger.

Beyond the setup value, I wanted a real findings artifact tied to frameworks I work with daily. I work in a GLBA and PCI-DSS regulated banking environment, so running a gap assessment against those frameworks against my own GitHub repos isn't a synthetic exercise; it's applying a compliance lens to actual public-facing work. The goal was to establish hands-on familiarity with automated GRC tooling, produce a genuine findings artifact, and identify remediation actions I can actually close.

---

## Environment & Prerequisites

The toolkit requires a Unix environment; Windows native PowerShell isn't supported. WSL2 was already configured on my machine (Ubuntu distro on LAB-PC, confirmed via `wsl --status`), so all commands below run inside that environment.

**Dependencies installed inside WSL2:**

```bash
# GitHub CLI
sudo apt install gh
gh auth login   # authenticated as JNewbrey87 via browser flow

# Claude Code (routed to user-owned directory to avoid permission issues)
mkdir -p ~/.npm-global
npm config set prefix '~/.npm-global'
echo 'export PATH=~/.npm-global/bin:$PATH' >> ~/.bashrc
source ~/.bashrc
npm install -g @anthropic-ai/claude-code
```

Both `claude --version` and `gh auth status` returned clean output before proceeding.

---

## Setup

The toolkit uses a plugin architecture, but the `/plugin` slash command documented in the quickstart isn't wired up in the current Claude Code release. I used the local clone method instead:

```bash
cd ~
git clone https://github.com/GRCEngClub/claude-grc-engineering.git
cd claude-grc-engineering
claude --plugin-dir .
```

Launching Claude Code with `--plugin-dir .` loads the repo's `CLAUDE.md` as context, giving Claude Code full knowledge of the toolkit's capabilities, runbooks, and the Secure Controls Framework (SCF) crosswalk. Slash commands don't auto-register this way, but plain-language prompts produce the same output since the underlying behavior is prompt-driven. The slash commands are just documented shortcuts.

---

## What I Scanned

**Scope:** My personal GitHub account (`@me` - JNewbrey87)

**Repositories evaluated:**

| Repository | Purpose |
|---|---|
| `cloud-security-lab-notes` | Active lab documentation and portfolio artifacts |
| `JNewbrey87` (profile repo) | GitHub profile README |
| `CFRPF` | Cloud Forensic Readiness and Preservation Framework (graduate thesis) |

The github-inspector connector evaluated branch protection configuration, GitHub Actions setup, Dependabot status, secret scanning state, code scanning enablement, and deploy key configuration across all three repos.

---

## How I Ran the Assessment

Since slash commands weren't registering, I invoked the assessment via a plain-language prompt inside the Claude Code session:

> *"Set up the github-inspector connector for my GitHub account (JNewbrey87) and collect findings from all my personal repos, then run a gap assessment against SOC2 and GLBA frameworks."*

Claude Code cross-referenced the loaded `CLAUDE.md`, ran the collection and crosswalk steps agenically, and wrote the full report bundle to disk.

**Report artifact:** `./gap-assessment-20260519231045-a7c11d5b/`

---

## Coverage Summary

| Framework | Controls Evaluated / Total | Pass Rate |
|---|---|---|
| SOC 2 | 26 / 399 (7%) | 0% |
| GLBA | 1 / 52 (2%) | 0% |

**Why the coverage percentages are low:** The github-inspector connector maps to 4 SCF controls. The full SOC 2 and GLBA control libraries are far broader than what any single connector can evaluate; this one is scoped specifically to repository-level security posture. The low percentages are expected and documented in the toolkit. The 0% pass rate is the number that actually matters; it means every control that was evaluated returned a finding.

---

## Findings

### Tier 1 - Certification Blocker (1)

**CHG-02 | Configuration Change Control** - HIGH severity

All three repositories have no branch protection rules on `main`; direct pushes without review are permitted.

- **SOC 2 mapping:** CC8.1, CC8.1-POF1-14, CC2.2, CC3.4
- **GLBA mapping:** §314.4(c)(7) - Safeguards Rule, monitoring and testing requirement
- **Remediation:** Enable branch protection on each repo with at least one required review and status checks. Estimated 15 minutes per repo; can be done via GitHub UI or automated via Terraform. This single fix clears the blocker across 17 SOC 2 sub-controls.

---

### Tier 2 - Active Findings (2)

**MON-01.4 | Dependabot Alerts**

Automated dependency vulnerability alerts are not enabled on any of the three repositories.

- **SOC 2 mapping:** CC7.1
- **GLBA mapping:** Dependency monitoring gap under the Safeguards Rule
- **Remediation:** Enable Dependabot alerts under each repo's Security settings. Approximately 5 minutes per repo.

**IAO-04 | Code Scanning / SAST**

Static application security testing is not configured on any of the three repositories.

- **SOC 2 mapping:** CC8.1
- **GLBA mapping:** §314.4(c)
- **Remediation:** Enable CodeQL via a GitHub Actions workflow. The easiest path is GitHub's built-in "Set up code scanning" button per repo, or a shared template PR applied across all three.

---

### Inconclusive (1)

**MON-01 | Secret Scanning**

Secret scanning status could not be confirmed; the connector likely needs GitHub Advanced Security or an expanded OAuth scope to read this setting.

- **Resolution:** Re-run the collect step with the `security_events` scope added: `gh auth refresh --scopes=repo,read:org,security_events`

---

## Priority Remediation Order

1. **Enable branch protection on all three repos** - 30 minutes total. Clears the Tier 1 blocker and satisfies the most impactful control cluster in the report.
2. **Enable Dependabot alerts** - 5 minutes per repo via Security settings.
3. **Enable CodeQL** via GitHub Actions - shared workflow template is the most efficient approach.
4. **Re-run collect with `security_events` scope** - resolves the MON-01 inconclusive result and gives a complete picture of secret scanning posture.

After completing steps 1 through 3, re-run the gap assessment to capture the before/after delta. That remediation cycle artifact will show a measurable improvement in pass rate across the evaluated controls.

---

## Framework Context

**Why GLBA?** I work in a GLBA-regulated banking environment. Running a gap assessment against the Gramm-Leach-Bliley Act Safeguards Rule against my own GitHub repos isn't a stretch; it connects lab work directly to the regulatory environment I operate in daily. The CHG-02 finding under §314.4(c)(7) isn't theoretical; it's a real control gap in repositories that are publicly visible and tied to my professional portfolio.

**Why SOC 2?** SOC 2 Trust Services Criteria (specifically the Common Criteria / CC series) is the dominant attestation framework for cloud-hosted services and SaaS environments. Understanding how control gaps surface in SOC 2 terms is directly relevant to Cloud Security Analyst roles, where SOC 2 reports come up constantly as a baseline assurance reference.

---

## What This Lab Demonstrates

- Hands-on use of open-source GRC tooling built on the Secure Controls Framework (1,468 controls, 249 framework mappings)
- Ability to run a structured gap assessment and interpret findings in both framework-native and plain-language terms
- Working knowledge of specific GLBA Safeguards Rule subsections and their mapping to technical controls
- Understanding the difference between low coverage (expected, tool-scoped) and 0% pass rate (genuine; every evaluated control failed)
- A documented remediation backlog with effort estimates, organized by priority tier

---

## Next Steps

- [ ] Enable branch protection on `cloud-security-lab-notes`, `JNewbrey87`, and `CFRPF`
- [ ] Enable Dependabot alerts on all three repos
- [ ] Configure CodeQL via GitHub Actions
- [ ] Refresh gh auth scope and resolve MON-01 secret scanning status
- [ ] Re-run collection and gap assessment; commit before/after delta as Phase 2 of this lab
