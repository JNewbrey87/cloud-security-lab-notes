# Ark: Survival Evolved - Dedicated Server Administration
### Retrospective Project Write-Up | Late 2015 - Late 2017 | Pre-IT Career

---

> *"I started this to have a place to record YouTube content. I ended up spending more time configuring the server than playing the game, and realized that was fine with me."*

---

## Overview

Before I ever held an IT job title, I was running a Linux-based dedicated game server for **Ark: Survival Evolved** during its Alpha development period. What started as a content creation side project turned into hands-on experience in server administration, configuration management, operational continuity, and community support; none of which I was formally trained to do at the time.

I learned by breaking things, Googling the error messages, and fixing what broke. Then I broke something else and repeated the process. By the time the project wound down in 2017, I had scaled the community from 5 players to over 100, stood up a second server to handle load during maintenance windows, and was fielding a consistent queue of user-reported issues that looked more like a ticket backlog than a gaming hobby.

This write-up documents what I actually did and why it matters technically; not what it felt like at the time.

---

## Environment

| Component | Details |
|---|---|
| **Platform** | Ark: Survival Evolved Dedicated Server (Alpha era) |
| **OS** | Linux (no GUI; 100% command-line) |
| **Hosting** | Third-party VM provider (web-based control panel, older virtualization stack) |
| **Access method** | CLI only; no desktop environment |
| **Community size** | Scaled from 5 to 100+ concurrent players |
| **Timeline** | Late 2015 to Late 2017 |

---

## What I Built and Managed

### Linux Server Administration

I deployed and administered a Linux-based dedicated server entirely through the command line; no graphical interface, no hand-holding. Day-to-day operations included:

- Installing, updating, and managing the Ark dedicated server runtime via CLI
- Installing and maintaining security software through the command line
- Monitoring server health and managing the process lifecycle manually
- Planning and executing updates without taking active players offline for very long (eventually; see **Operational Challenges** below)

---

### Configuration Management

Ark's dedicated server is almost entirely governed by `.ini` configuration files. I authored and maintained an extensive set of custom configs that controlled nearly every behavioral parameter of the server environment:

```ini
# Representative example - GameUserSettings.ini
[ServerSettings]
DifficultyOffset=1.0
XPMultiplier=2.5
TamingSpeedMultiplier=3.0
HarvestAmountMultiplier=2.0
PlayerCharacterWaterDrainMultiplier=0.5
PlayerCharacterFoodDrainMultiplier=0.5
NightTimeSpeedScale=2.0
DayTimeSpeedScale=1.0

[/Script/ShooterGame.ShooterGameMode]
bDisableStructureDecayPV=True
MaxTribeMemberNum=10
```

Custom parameters included player stat scaling, creature taming speeds, resource harvest rates, vegetation and flora growth cycles, day/night cycle pacing, tribe size limits, and server-side rule enforcement. Every change required a server restart, which meant planning maintenance windows around peak player hours; a constraint that became more complex as the community grew.

---

### Staging Environment and User Acceptance Testing

Before pushing any configuration change to the live server, I maintained a controlled test environment separate from production. When players suggested configuration changes, I'd implement the change on my personal gaming PC and spin up a password-protected local Ark session with a capped player count, then invite the suggesting player to test the result directly; putting them inside the change before it ever touched production.

This gave every proposed change a real validation step and kept players invested in the outcome of their own suggestions. If something broke in testing, it broke for one person in a controlled environment rather than for the entire server. If it worked, the player who suggested it became the first advocate for the change when it rolled out to everyone else.

The pattern (staging environment, stakeholder-involved testing, production deployment only after sign-off) is the same one used in formal change management processes today. I just didn't have a name for it yet.

---

### Staged Testing, Config Promotion, and Backup Strategy

Test1, my local testing server, ran on a spare laptop on my home network and never connected to the public internet. It existed purely as a controlled pre-production environment: isolated, no live players, no public traffic.

On patch days, the process looked like this:

- Read through the patch notes on my personal client, noting any new or changed `.ini` parameters
- Posted a server-wide in-game message to players stating the update window and estimated downtime
- Loaded Test1 locally, applied the patch and config changes, and tested for one to two hours: flying around, spawning creatures, checking for mod conflicts and unintended behavioral changes
- Once satisfied, copied the updated config file from Test1 to Prod1
- Backed up the existing Prod1 config to my personal machine before replacing it
- Force-saved the world state on Prod1 before taking it offline; I didn't trust the automation to flush cleanly, and that discipline saved me more than once
- Rebooted Prod1; the server was back up within 15 minutes and the patch was live

Total player downtime on a standard patch day: roughly 15 minutes, just a server restart.

Backup strategy ran on three tiers:

- Rolling save states: snapshots taken daily at 5pm, retained at 1-day, 3-day, and 15-day intervals
- Live automated backups: 24-hour cycle, treated as emergency-only; these failed frequently enough that I never relied on them as a primary recovery path
- Manual force-save before every maintenance window: the one backup I always trusted

---

### Community Support and Incident Response

The community side of this project was closer to a support queue than a gaming group. Responsibilities included:

- Triaging and resolving player-reported issues: bugs, configuration conflicts, access problems, and rules disputes
- Maintaining server rules and enforcement documentation
- Managing the community communication channel
- Writing and publishing how-to guides to reduce repeat support requests; the same logic behind a knowledge base

The volume of support requests grew significant enough that my YouTube content shifted from let's-play footage to tutorial and how-to content. I couldn't maintain a continuous gameplay series because server management and community support had become the primary workload. In hindsight, that shift in priority was the first clear signal of where my interests actually sat.

---

## Operational Challenges

### The Outage

Early in the project, before I understood the system well enough to prevent it, I caused a week-long outage. The original five players left and never came back.

The recovery process forced me to actually understand the system rather than just operate it. What had been trial-and-error became deliberate troubleshooting: identifying root cause, documenting what went wrong, and building a working model of how the pieces connected. The community rebuilt from zero and grew larger than it had been before the outage.

The lesson wasn't "don't break things." The lesson was that a fast, documented recovery is more valuable than a fragile system that's never been tested.

---

### Configuration Complexity at Scale

The `.ini` configuration surface for Ark is expansive; hundreds of tunable parameters with interdependencies that aren't always documented. Early misconfigurations cascaded in unexpected ways: adjusting one growth rate would interact with a spawn rate setting and produce unintended gameplay outcomes that players would immediately report.

Managing this at scale meant developing a consistent practice:

- Document every change before applying it
- Test behavioral impact before publishing to active players
- Maintain a known-good configuration baseline for rollback

---

## What This Translates To

| Hands-On Skill | Modern Equivalent |
|---|---|
| Linux CLI server administration | Linux system administration, SSH-based remote management |
| `.ini` configuration file authoring at scale | Configuration management, infrastructure-as-code principles |
| Isolated staging environment for pre-production patch validation | Pre-production testing, change management discipline |
| Tiered rolling backup strategy with manual save discipline | Data retention planning, backup verification, RPO awareness |
| Security software deployment via CLI | Endpoint security tooling, CLI-based agent deployment |
| Player-involved UAT before production deploy | Stakeholder acceptance testing, change advisory process |
| Support queue management and documentation | ITSM, incident response, knowledge base development |
| Outage, root cause analysis, and rebuild | Incident response, disaster recovery, lessons-learned processes |

---

## Reflection

I didn't know it was called systems administration. I didn't know the configuration work I was doing mapped to concepts with formal names. I just knew that the server was the interesting part; more interesting than the game, more interesting than the content, more interesting than the community metrics.

That instinct turned out to be directionally correct. Everything that came after, including DoD contracts, federal infrastructure work, and regulated banking environments, built on a foundation that started here; in a Linux terminal, trying to figure out why a handful of gamers couldn't connect to a dinosaur game.

---

*Part of the [`cloud-security-lab-notes`](https://github.com/JNewbrey87/cloud-security-lab-notes) retrospective collection. Joshua E. Newbrey | [linkedin.com/in/jnewbrey87](https://www.linkedin.com/in/jnewbrey87)*
