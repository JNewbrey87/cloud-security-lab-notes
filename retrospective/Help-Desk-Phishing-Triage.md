# Phishing Triage Methodology: FAA Help Desk

**Type:** Retrospective  
**Role:** Help Desk technician, Federal Aviation Administration  
**MITRE ATT&CK:** [T1566 — Phishing](https://attack.mitre.org/techniques/T1566/)  
**Cert Alignment:** CySA+ CS0-004, Domain 1 (Security Operations) and Domain 3 (Incident Response and Management)  
**Date:** 2019  

---

> *"I was the only Help Desk tech on the call. By the time it was over, I had a pretty clear picture of what the SOC actually needed from us, and we weren't delivering it."*

## Why I'm Writing This One Up

This is a phishing triage process I put together for Help Desk agents after a major phishing incident at the FAA. To be clear about what it was and wasn't: it didn't help during the incident that prompted it. It was built for the one-off phishing calls that came in on normal days afterward, when a single agent needed a fast, repeatable way to handle a report without call handle time ballooning while they figured out what to do.

The incident was the catalyst. The process was the lesson I pulled from it.

## Background

The trigger was a large phishing campaign that hit the FAA user base all at once.

At peak, the Help Desk had over 200 calls in queue against a staff of 150 agents, enough volume that an outage was declared. During the incident itself, the response was simple and immediate: agents RDP'd into affected machines, scraped the relevant email data, escalated the ticket, and waited on SOC to work their end. There was no time to build anything during an event like that, and no need to; the job was to move fast and keep the queue moving.

I was pulled into a group call with the SOC team while the incident was still active. I was the only Help Desk technician on it, working alongside management on our side while SOC worked theirs. That call gave me a firsthand look at what SOC was dealing with and what they needed from us to work efficiently.

After it wrapped up, I started thinking about the routine calls. Phishing reports came in regularly outside of major campaigns, and we didn't have a standard way to handle them. An agent getting a phishing call had to figure it out on the fly, which drove handle time up and made the handoffs to SOC inconsistent. The knowledge base article (KBA) I wrote was meant to fix that: give agents a clear sequence of steps, hand SOC a useful pre-analysis package, and keep the two teams separated in the ticketing system the whole way through.

## The Process

### Step 1: Take the Report

The user reports a suspicious email through normal Help Desk channels. The agent opens a parent ticket in Remedy and documents the user's description.

### Step 2: Pull the Email Header

The agent opens the reported email in the user's Outlook client and pulls the full header. In Outlook on the Windows 7 environment at the time, that was `File → Properties → Internet headers`. Copy the full header text from that field.

### Step 3: Run It Through a Header Analyzer

Paste the header into a web-based email header analyzer. The analyzer parses the routing data, flags suspicious hops, and surfaces indicators that aren't readable in the raw text. This is the step that turns a wall of technical text into something an agent can package for escalation.

*(The specific tool used internally is left out; any reputable web-based header analyzer does the same job.)*

### Step 4: Attach the Output to the Parent Ticket

Save or screenshot the analyzer output and upload it as an **internal attachment** on the parent Remedy ticket. Internal attachments are visible to agents and SOC but not to the end user.

At this point the parent ticket holds:

- The user's original report
- The raw header
- The parsed, flagged analyzer output

That's the pre-analysis package SOC receives.

### Step 5: Open a Child Ticket for SOC

Create a child ticket in Remedy and link it internally to the parent. The child is SOC's working ticket. It carries the pre-analysis findings but keeps SOC's workflow separate from the user-facing parent.

### Step 6: Hold the Parent While SOC Works the Child

SOC takes it from there. The Help Desk agent holds the parent ticket open and waits for the child to close.

### Step 7: Close on Resolution

When SOC closes the child ticket, Remedy emails the assigned Help Desk agent automatically. The agent reviews the closure, adds resolution notes appropriate for the user, and closes the parent.

The user gets a closure update from the Help Desk. SOC's ticket activity, findings, and analyst identity stay internal throughout.

## Why the Ticket Separation Mattered

In a federal environment, compartmentalizing information isn't really optional. Users shouldn't be able to infer who investigated their report or what SOC found beyond what's appropriate to share. The parent/child structure enforced that boundary without relying on each agent to manage what they said. It was built into the process.

It also kept SOC's queue clean. Analysts working a phishing case don't need user callbacks or status questions landing in their ticket flow. Keeping the two tickets apart meant each team could close out its side without interference.

## Looking Back

With more experience, there's an obvious question about the original incident: why didn't SOC loop in an Exchange admin to run a tenant-wide message trace and purge?

A header analyzer on one reported email tells you a lot about that email. It doesn't tell you how many other mailboxes got the same message and haven't reported it. An Exchange admin with PowerShell access could have traced the campaign across the whole tenant and purged it from the inboxes still holding it. That's the step that actually contains a phishing campaign at scale, and it doesn't look like it was part of the response.

Whether SOC made that call and I wasn't privy to it, whether there was a coordination gap getting to the Exchange team fast enough, or whether it just wasn't on anyone's radar, I can't say from where I was sitting. What I can say is that by the time I was doing Exchange admin work at DISA a few years later, PowerShell-based message tracing was one of the first things I learned, and I understood exactly why it mattered.

The triage process was useful for what it was. It just addressed the symptom rather than the source.

## What It Maps To Now

| What I Built | Modern Equivalent |
|---|---|
| Phishing triage KBA written after a live incident | Incident-driven runbook authoring |
| Header extraction and pre-analysis before escalation | Email analysis, IOC identification, threat triage |
| Parent/child ticket structure to isolate SOC work | Tiered incident response, separation of duties |
| Structured findings packaged for analyst handoff | Evidence documentation, SOC intake standards |

## What I Took Away

There was no ticket for "design a phishing triage process." I built it because I'd seen the gap firsthand and had enough context to do something about it. At the time I was reading security books outside of work and treating it like a hobby. Looking back, it was probably the clearest early sign of where I was headed.

---

*Part of the [cloud-security-lab-notes](https://github.com/JNewbrey87/cloud-security-lab-notes) portfolio, building toward Cloud Security Analyst and the CySA+ / SC-200 / SC-500 certifications.*
