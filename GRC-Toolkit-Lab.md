# GRC Toolkit Lab: SOC 2 and GLBA Gap Assessment of My Own Repos

**Project:** GRC  
**Platform:** WSL2 (Ubuntu) on Windows 11  
**Tool:** [claude-grc-engineering](https://github.com/GRCEngClub/claude-grc-engineering) by GRC Engineering Club  
**Frameworks Assessed:** SOC 2 (AICPA Trust Services Criteria), GLBA Safeguards Rule (16 C.F.R. Part 314)  
**Cert Alignment:** CySA+ CS0-004, Domain 4 (Reporting and Communication)  
**Status:** Assessment complete; findings documented; remediation still open  
**Date:** May 19, 2026  

---

## Why I Did This One

I saw this toolkit advertised on LinkedIn and thought it might be a nifty tool to bring into the lab environment one day. Before doing that, I wanted to learn how it's put together, how to invoke it, and what the output looks like, at a scope small enough that a big configuration surface wouldn't get in the way.

My own GitHub repos made a good first target. They're real, they're public, they're tied to my professional portfolio, and any gap the tool finds is one I can actually close. So the findings here aren't synthetic.

## Environment and Prerequisites

The toolkit needs a Unix environment; native Windows PowerShell isn't supported. WSL2 was already set up on my machine at the time (Ubuntu distro, confirmed with `wsl --status`), so everything below runs inside it.

```bash
# GitHub CLI
sudo apt install gh
gh auth login   # authenticated as JNewbrey87 through the browser flow

# Claude Code, installed to a user-owned directory to avoid permission issues
mkdir -p ~/.npm-global
npm config set prefix '~/.npm-global'
echo 'export PATH=~/.npm-global/bin:$PATH' >> ~/.bashrc
source ~/.bashrc
npm install -g @anthropic-ai/claude-code
```

Both `claude --version` and `gh auth status` came back clean before I moved on.

## What I Built

### 1. Loading the Toolkit

The toolkit uses a plugin architecture, but the `/plugin` slash command from the quickstart wasn't wired up in the Claude Code release I had. I used the local clone method instead:

```bash
cd ~
git clone https://github.com/GRCEngClub/claude-grc-engineering.git
cd claude-grc-engineering
claude --plugin-dir .
```

Launching with `--plugin-dir .` loads the repo's `CLAUDE.md` as context, which gives Claude Code the toolkit's capabilities, runbooks, and the Secure Controls Framework (SCF) crosswalk. The slash commands don't auto-register this way, but since the underlying behavior is prompt-driven, a plain-language prompt gets the same result. The slash commands are just shortcuts.

### 2. Scoping the Scan

I scoped it to my personal GitHub account (`@me`, JNewbrey87) and its three repos:

| Repository | Purpose |
|---|---|
| `cloud-security-lab-notes` | Lab documentation and portfolio artifacts |
| `JNewbrey87` (profile repo) | GitHub profile README |
| `CFRPF` | Cloud Forensic Readiness and Preservation Framework (graduate thesis) |

The github-inspector connector checks branch protection, GitHub Actions setup, Dependabot status, secret scanning, code scanning, and deploy keys across each repo.

### 3. Running the Assessment

With the slash commands unavailable, I ran it with a plain-language prompt inside the Claude Code session:

> *"Set up the github-inspector connector for my GitHub account (JNewbrey87) and collect findings from all my personal repos, then run a gap assessment against SOC2 and GLBA frameworks."*

Claude Code used the loaded `CLAUDE.md` to run the collection and crosswalk steps agentically and wrote the full report bundle to disk at `./gap-assessment-20260519231045-a7c11d5b/`.

## What I Found

### Coverage

| Framework | Controls Evaluated / Total | Pass Rate |
|---|---|---|
| SOC 2 | 26 / 399 (7%) | 0% |
| GLBA | 1 / 52 (2%) | 0% |

The low coverage is expected. The github-inspector connector maps to only 4 SCF controls, and it's scoped to repository-level security posture, so it can't touch most of what SOC 2 or GLBA cover. The number that matters is the 0% pass rate: every control the tool could evaluate came back as a finding.

### Tier 1: Certification Blocker

**CHG-02, Configuration Change Control (HIGH)**

None of the three repos have branch protection on `main`, so direct pushes without review are allowed.

- **SOC 2:** CC8.1, CC8.1-POF1-14, CC2.2, CC3.4
- **GLBA:** §314.4(c)(7), the Safeguards Rule monitoring and testing requirement
- **Remediation:** enable branch protection on each repo with at least one required review and status checks. Roughly 15 minutes per repo through the GitHub UI, or automated with Terraform. This one fix clears the blocker across 17 SOC 2 sub-controls.

### Tier 2: Active Findings

**MON-01.4, Dependabot Alerts**

Automated dependency vulnerability alerts aren't enabled on any of the three repos.

- **SOC 2:** CC7.1
- **GLBA:** dependency monitoring gap under the Safeguards Rule
- **Remediation:** enable Dependabot alerts in each repo's Security settings, about 5 minutes per repo.

**IAO-04, Code Scanning (SAST)**

No static analysis is configured on any of the three repos.

- **SOC 2:** CC8.1
- **GLBA:** §314.4(c)
- **Remediation:** enable CodeQL through a GitHub Actions workflow, either with GitHub's built-in "Set up code scanning" button on each repo or one shared template applied across all three.

### Inconclusive

**MON-01, Secret Scanning**

The connector couldn't confirm secret scanning status. It likely needs GitHub Advanced Security or a wider OAuth scope to read that setting. The fix is to re-run the collect step after adding the `security_events` scope:

```bash
gh auth refresh --scopes=repo,read:org,security_events
```

### Remediation Order

1. **Branch protection on all three repos**, about 30 minutes total. Clears the Tier 1 blocker and the biggest control cluster in the report.
2. **Dependabot alerts**, about 5 minutes per repo.
3. **CodeQL** through GitHub Actions, using a shared workflow template.
4. **Re-run collection with the `security_events` scope** to resolve MON-01.

After steps 1 through 3, a second assessment run gives a before-and-after comparison of the pass rate.

## What I Took Away

**Low coverage and a 0% pass rate are two different signals.** Seven percent coverage says the connector is narrow; that's a property of the tool. Zero percent passing says every control it could check failed; that's a property of my repos. Reading the report correctly depends on keeping those apart.

**The highest-impact fix is also the cheapest.** Branch protection is a settings change, and it accounts for the only Tier 1 finding and 17 SOC 2 sub-controls. Sorting remediation by impact against effort, instead of by the order findings show up in the report, is what made the priority list obvious.

**Framework mappings make a small gap legible to an auditor.** "No branch protection" is a technical detail. "CHG-02, mapped to SOC 2 CC8.1 and GLBA §314.4(c)(7)" is something a compliance reviewer can act on. That translation from setting to control is most of the value the toolkit adds.

**Why both frameworks.** I work in a GLBA and PCI-DSS regulated banking environment, so mapping findings to the Safeguards Rule connects lab work to rules I already live under. SOC 2 isn't part of my day-to-day, but its Common Criteria are the standard assurance reference for cloud and SaaS providers, and I need to be fluent in it for the cloud security roles I'm working toward. Seeing my own gaps written in SOC 2 terms is a practical way to start.

## Lab Environment

| Component | Details |
|-----------|---------|
| Host | Windows 11, WSL2 with Ubuntu |
| Tooling | GitHub CLI, Claude Code, claude-grc-engineering (local clone, `--plugin-dir`) |
| Control framework | Secure Controls Framework (SCF) crosswalk, 1,468 controls and 249 framework mappings |
| Scope | Three public repos under JNewbrey87 |
| Report bundle | `./gap-assessment-20260519231045-a7c11d5b/` |

## What Came Next

The remediation from this assessment is still open. Still on my list:

- [ ] Enable branch protection on `cloud-security-lab-notes`, `JNewbrey87`, and `CFRPF`
- [ ] Enable Dependabot alerts on all three repos
- [ ] Configure CodeQL through GitHub Actions
- [ ] Refresh the gh auth scope and resolve the MON-01 secret scanning result
- [ ] Re-run collection and the gap assessment, and commit the before-and-after comparison as a follow-up to this lab

---

*Part of the [cloud-security-lab-notes](https://github.com/JNewbrey87/cloud-security-lab-notes) portfolio, building toward Cloud Security Analyst and the CySA+ / SC-200 / SC-500 certifications.*
