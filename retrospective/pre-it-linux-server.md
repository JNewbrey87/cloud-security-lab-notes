# Ark: Survival Evolved Dedicated Server Administration

**Type:** Retrospective, pre-IT career  
**Platform:** Linux dedicated game server on a third-party hosted VM, CLI only  
**Date:** Late 2015 to late 2017  

---

> *"I started this to have a place to record YouTube content. I ended up spending more time configuring the server than playing the game, and realized that was fine with me."*

## Why I'm Writing This One Up

Before I ever held an IT job title, I ran a Linux dedicated server for **Ark: Survival Evolved** during its alpha. It started as a content creation side project and turned into hands-on experience with server administration, configuration management, keeping a service running, and supporting a community, none of which I'd been trained to do.

I learned by breaking things, Googling the error messages, and fixing what broke. Then I broke something else and did it again. By the time the project wound down in 2017, the community had grown from 5 players to over 100, I'd stood up a second server to carry load during maintenance windows, and I was working a steady queue of player-reported issues that looked more like a ticket backlog than a gaming hobby.

## Environment

| Component | Details |
|---|---|
| Platform | Ark: Survival Evolved dedicated server (alpha era) |
| OS | Linux, no GUI, command line only |
| Hosting | Third-party VM provider (web control panel, older virtualization stack) |
| Testing | Local test server on a spare laptop, isolated from the internet |
| Community size | Grew from 5 to 100+ players; peak of about 15 online at once (server capacity was 100) |
| Timeline | Late 2015 to late 2017 |

## What I Built and Managed

### Linux Server Administration

I deployed and ran the server entirely from the command line. Day-to-day work included:

- Installing, updating, and managing the Ark dedicated server runtime from the CLI
- Installing and maintaining security software from the command line
- Monitoring server health and managing the process lifecycle by hand
- Planning and running updates without keeping players offline for long (eventually; see **The Outage** below)

### Configuration Management

Ark's dedicated server is governed almost entirely by `.ini` files. I wrote and maintained a large set of custom configs that controlled nearly every behavior on the server:

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

The custom settings covered player stat scaling, taming speeds, harvest rates, plant growth cycles, day/night pacing, tribe size limits, and server rule enforcement. Every change needed a server restart, so maintenance windows had to be planned around peak hours, which got harder as the community grew.

### Player Testing Before Production

When players suggested a config change, I didn't push it straight to the live server. I'd set it up on my gaming PC, start a password-protected local Ark session with a capped player count, and invite the player who suggested it to test it themselves. They got to play inside the change before it ever touched production.

That gave every proposed change a real validation step, and it kept players invested in their own suggestions. If something broke, it broke for one person in a controlled session instead of the whole server. If it worked, the player who asked for it became the first advocate when it rolled out to everyone else.

Staging, testing with the person who asked for the change, and deploying only after they signed off is the same pattern formal change management uses. I just didn't have a name for it yet.

### Patch Days, Config Promotion, and Backups

Patch testing ran on a separate box: Test1, a local test server on a spare laptop on my home network. It never connected to the public internet. No live players, no public traffic, just a pre-production copy.

On patch days, the routine was:

- Read the patch notes on my own client and note any new or changed `.ini` settings
- Post a server-wide in-game message with the update window and expected downtime
- Load Test1, apply the patch and config changes, and test for one to two hours: flying around, spawning creatures, checking for mod conflicts and behavior changes
- Once satisfied, copy the updated config from Test1 to Prod1
- Back up the existing Prod1 config to my own machine before replacing it
- Force-save the world on Prod1 before taking it down; I didn't trust the automation to flush cleanly, and that habit saved me more than once
- Reboot Prod1; it was back up within 15 minutes with the patch live

Player downtime on a normal patch day was about 15 minutes, just the restart.

Backups ran in three tiers:

- **Rolling save states:** daily snapshots at 5pm, kept at 1-day, 3-day, and 15-day intervals
- **Automated live backups:** a 24-hour cycle, treated as emergency-only, since they failed often enough that I never relied on them as the main recovery path
- **Manual force-save before every maintenance window:** the one backup I always trusted

### Community Support

The community side was closer to a support queue than a gaming group:

- Triaging and resolving player-reported issues: bugs, config conflicts, access problems, and rules disputes
- Maintaining the server rules and how they were enforced
- Running the community communication channel
- Writing and publishing how-to guides to cut down on repeat questions, which is the same idea as a knowledge base

Support volume grew enough that my YouTube content shifted from let's-play footage to tutorials and how-tos. Server management and support had become the main workload, and I couldn't keep up a gameplay series alongside it. Looking back, that shift was the first clear sign of where my interests actually were.

## Operational Challenges

### The Outage

Early on, before I understood the system well enough to prevent it, I caused a week-long outage. The original five players left and never came back.

Recovering forced me to understand the system instead of just operating it. Trial and error turned into deliberate troubleshooting: finding the root cause, writing down what went wrong, and building a working model of how the pieces fit together. The community rebuilt from zero and grew larger than it had been before.

The lesson wasn't "don't break things." It was that a fast, documented recovery is worth more than a fragile system that's never been tested.

### Configuration Complexity at Scale

Ark's `.ini` surface is huge: hundreds of settings with interdependencies that aren't always documented. Early mistakes cascaded in ways I didn't expect. Adjusting one growth rate would interact with a spawn rate and produce gameplay side effects that players reported right away.

Managing that at scale meant building a few habits:

- Write down every change before applying it
- Test the behavior change before it reaches active players
- Keep a known-good config baseline to roll back to

## What It Maps To Now

| Hands-On Skill | Modern Equivalent |
|---|---|
| Linux CLI server administration | Linux system administration, SSH-based remote management |
| `.ini` configuration authoring at scale | Configuration management, infrastructure-as-code principles |
| Isolated test server for patch validation | Pre-production testing, change management discipline |
| Tiered rolling backups plus manual force-saves | Data retention planning, backup verification, RPO awareness |
| Security software deployment from the CLI | Endpoint security tooling, CLI-based agent deployment |
| Player testing before production deploys | Stakeholder acceptance testing, change advisory process |
| Support queue and how-to guides | ITSM, knowledge base development |
| Outage, root cause, and rebuild | Incident response, disaster recovery, lessons learned |

## What I Took Away

I didn't know it was called systems administration. I didn't know the config work I was doing had formal names. I just knew the server was the interesting part: more interesting than the game, the content, or the community numbers.

That instinct turned out to be right. Everything after it, including DoD contracts, federal infrastructure work, and regulated banking environments, built on a foundation that started here: in a Linux terminal, trying to figure out why a handful of gamers couldn't connect to a dinosaur game.

---

*Part of the [cloud-security-lab-notes](https://github.com/JNewbrey87/cloud-security-lab-notes) portfolio, building toward Cloud Security Analyst and the CySA+ / SC-200 / SC-500 certifications.*
