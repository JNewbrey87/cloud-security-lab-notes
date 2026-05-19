# Ark: Survival Evolved - Dedicated Server Administration### Retrospective Project Write-Up | Late 2015 - Late 2017 | Pre-IT Career
---
> *"I started this to have a place to record YouTube content. I ended up spending more time configuring the server than playing the game, and realized that was fine with me."*
---
### Overview 
Before I ever held an IT job title, I was running a Linux-based dedicated game server for **Ark: Survival Evolved** during its Alpha development period. What started as a content creation side project turned into hands-on experience in server administration, configuration management, operational continuity, and community support; none of which I was formally trained to do at the time.
I learned by breaking things, Googling the error messages, and fixing what broke. Then I broke something else and repeated the process. By the time the project wound down in 2017, I had scaled the community from 5 players to over 100, stood up a second server to handle load during maintenance windows, and was fielding a consistent queue of user-reported issues that looked more like a ticket backlog than a gaming hobby.
This write-up documents what I actually did and why it matters technically; not what it felt like at the time.
---
### Environment
| Component | Details ||---|---|| **Platform** | Ark: Survival Evolved Dedicated Server (Alpha era) || **OS** | Linux (no GUI; 100% command-line) || **Hosting** | Third-party VM provider (web-based control panel, older virtualization stack) || **Access method** | CLI only; no desktop environment || **Community size** | Scaled from 5 to 100+ concurrent players || **Timeline** | Late 2015 to Late 2017 |
---
## What I Built and Managed
### Linux Server Administration
Deployed and administered a Linux-based dedicated server entirely through the command line; no graphical interface, no hand-holding. Day-to-day operations included:
- Installing, updating, and managing the Ark dedicated server runtime via CLI- Installing and maintaining security software through the command line- Monitoring server health and managing the process lifecycle manually- Planning and executing updates without taking active players offline (eventually; see **Operational Challenges** below)
---
### Configuration Management
Ark's dedicated server is almost entirely governed by `.ini` configuration files. I authored and maintained an extensive set of custom configs that controlled nearly every behavioral parameter of the server environment:
```ini# Representative example - GameUserSettings.ini[ServerSettings]DifficultyOffset=1.0XPMultiplier=2.5TamingSpeedMultiplier=3.0HarvestAmountMultiplier=2.0PlayerCharacterWaterDrainMultiplier=0.5PlayerCharacterFoodDrainMultiplier=0.5NightTimeSpeedScale=2.0DayTimeSpeedScale=1.0
[/Script/ShooterGame.ShooterGameMode]bDisableStructureDecayPV=TrueMaxTribeMemberNum=10```
Custom parameters included player stat scaling, creature taming speeds, resource harvest rates, vegetation and flora growth cycles, day/night cycle pacing, tribe size limits, and server-side rule enforcement. Every change required a server restart, which meant planning maintenance windows around peak player hours; a constraint that became more complex as the community grew.
---
### Staging Environment and User Acceptance Testing
Before pushing any configuration change to the live server, I maintained a local test environment on a separate laptop running a personal copy of the game files. When players suggested configuration changes, I would implement the change in that smaller environment first and invite the player who suggested it to test the result directly; putting them inside the change before it ever touched production.
This gave every proposed change a real validation step and kept players invested in the outcome of their own suggestions. If something broke in testing, it broke for one person in a controlled environment rather than for the entire server. If it worked, the player who suggested it became the first advocate for the change when it rolled out to everyone else.
The pattern, staging environment, stakeholder-involved testing, and production deployment only after sign-off, is the same one used in formal change management processes today.
---
### High Availability Design: Load Balancing Across Two Servers
When the player base crossed 100 concurrent users, single-server maintenance became a real problem; taking the primary offline for updates meant full service interruption during peak hours. The solution:
- Stood up a second dedicated Linux server on the same hosting platform- Configured the backup server as an overflow and maintenance target- During update windows, shifted active players to the backup server- Applied patches and updates to the primary with no interruption to active sessions- Reversed the configuration once the primary was back online and verified
This was not enterprise load balancing with automated failover; it was manual, deliberate, and effective. The underlying principle, which is maintaining service continuity by decoupling the update process from the live environment, is the same one that shows up in rolling deployments and blue/green infrastructure patterns today.
---
### Community Support and Incident Response
The community side of this project was closer to a support queue than a gaming group. Responsibilities included:
- Triaging and resolving player-reported issues: bugs, configuration conflicts, access problems, and rules disputes- Maintaining server rules and enforcement documentation- Managing the community communication channel- Writing and publishing how-to guides to reduce repeat support requests; the same logic behind a knowledge base
The volume of support requests grew significant enough that my YouTube content shifted from let's-play footage to tutorial and how-to content. I could not maintain a continuous gameplay series because server management and community support had become the primary workload. In hindsight, that shift in priority was the first clear signal of where my interests actually sat.
---
## Operational Challenges
### The Outage
Early in the project, before I understood the system well enough to prevent it, I caused a week-long outage. The original five players left and did not come back.
The recovery process forced me to actually understand the system rather than just operate it. What had been trial-and-error became deliberate troubleshooting: identifying root cause, documenting what went wrong, and building a working model of how the pieces connected. The community rebuilt from zero and grew larger than it had been before the outage.
The lesson was not "do not break things." The lesson was that a fast, documented recovery is more valuable than a fragile system that has never been tested.
---
### Configuration Complexity at Scale
The `.ini` configuration surface for Ark is expansive; hundreds of tunable parameters with interdependencies that are not always documented. Early misconfigurations cascaded in unexpected ways: adjusting one growth rate would interact with a spawn rate setting and produce unintended gameplay outcomes that players would immediately report.
Managing this at scale meant developing a consistent practice:
- Document every change before applying it- Test behavioral impact before publishing to active players- Maintain a known-good configuration baseline for rollback
---
## What This Translates To
| Hands-On Skill | Modern Equivalent ||---|---|| Linux CLI server administration | Linux system administration, SSH-based remote management || `.ini` configuration file authoring at scale | Configuration management, infrastructure-as-code principles || Manual load balancing across two servers | High availability design, maintenance window planning || Security software deployment via CLI | Endpoint security tooling, CLI-based agent deployment || Staging environment for config testing | Pre-production validation, change management discipline || Player-involved UAT before production deploy | Stakeholder acceptance testing, change advisory process || Support queue management and documentation | ITSM, incident response, knowledge base development || Outage, root cause analysis, and rebuild | Incident response, disaster recovery, lessons-learned processes |
---
## Reflection
I did not know it was called systems administration. I did not know the configuration work I was doing mapped to concepts with formal names. I just knew that the server was the interesting part; more interesting than the game, more interesting than the content, more interesting than the community metrics.
That instinct turned out to be directionally correct. Everything that came after, including DoD contracts, federal infrastructure work, and regulated banking environments, built on a foundation that started here; in a Linux terminal, trying to figure out why 100 players could not connect to a dinosaur game.
---
*Part of the [`cloud-security-lab-notes`](https://github.com/JNewbrey87/cloud-security-lab-notes) retrospective collection.**Joshua E. Newbrey | [linkedin.com/in/jnewbrey87](https://www.linkedin.com/in/jnewbrey87)* 
