# Phishing Triage Methodology: FAA Help Desk
### Retrospective Write-Up | ~2020 | Federal Aviation Administration

---

> *"I was the only Help Desk tech on the call. By the time it was over, I had a pretty clear picture of what the SOC actually needed from us, and we weren't delivering it."*

---

## Overview

This document describes a phishing triage methodology I put together for Help Desk agents after a major phishing incident at the FAA. It's important to be clear about what this process was and wasn't: it didn't help during the incident that prompted it. It was designed for the one-off phishing calls that came in on normal days afterward, when a single agent needed a fast, repeatable process to handle a phishing report without letting call handle time balloon while they figured out what to do.

The incident was the catalyst. The methodology was the lesson I pulled from it.

---

## Background

The trigger was a significant phishing campaign that hit the FAA user base all at once.

At peak, the Help Desk had over 200 calls in queue against a staff of 150 agents, enough volume that an outage was declared. During the incident itself, the response was straightforward and immediate: agents were RDPing into affected machines, scraping the relevant email data, escalating the ticket, and waiting on SOC to work their end. There was no time to build anything during an event like that, and no need to; the job was to move fast and keep the queue moving.

I was included in a group call with the SOC team while the incident was still active. I was the only Help Desk technician in that call, working alongside management on our side while the SOC team worked their end. That call gave me a firsthand look at what SOC was dealing with and what they needed from us to work efficiently.

After it wrapped up, I started thinking about the routine calls. Phishing reports were a regular occurrence outside of major campaigns, and we didn't have a standard process for handling them. An agent getting a phishing call had to figure it out on the fly, which drove handle time up and produced inconsistent handoffs to SOC. The KBA I built was meant to fix that: give agents a clear sequence of steps, produce a useful pre-analysis package for SOC, and keep the two teams separated in the ticketing system throughout.

---

## The Methodology

### Step 1: Intake the Report

The end user reports a suspicious email through normal Help Desk channels. The agent opens a parent ticket in Remedy and documents the user's description.

### Step 2: Extract the Email Header

The agent opens the reported email in the user's Outlook client and pulls the full email header.

In Outlook (Windows 7 environment at the time), that's accessed via:
`File → Properties → Internet headers`

Copy the full header text from that field.

### Step 3: Run It Through a Header Analyzer

Paste the copied header into a web-based email header analyzer. The analyzer parses the routing data, flags suspicious hops, and surfaces indicators that aren't readable in the raw header text. This is the step that turns a wall of technical text into something an agent can actually package for escalation.

*(The specific tool used internally has been omitted; any reputable web-based header analyzer performs this function.)*

### Step 4: Attach the Output to the Parent Ticket

Save or screenshot the analyzer output and upload it as an **internal attachment** on the parent Remedy ticket. Internal attachments are visible to agents and SOC but not to the end user.

At this point the parent ticket contains:
- The end user's original report
- The raw extracted header
- The parsed, flagged analyzer output

That's the pre-analysis package SOC receives.

### Step 5: Open a Child Ticket for SOC

Create a secondary (child) ticket in Remedy and link it internally to the parent. The child ticket is SOC's working ticket. It contains the pre-analysis findings but keeps SOC's workflow separate from the end-user-facing parent.

### Step 6: Hold the Parent While SOC Works the Child

SOC takes it from there. The Help Desk agent holds the parent ticket open and waits on the child ticket to close.

### Step 7: Close on Resolution

When SOC closes the child ticket, Remedy sends an automated email notification to the assigned Help Desk agent. The agent reviews the closure, adds any appropriate resolution notes for the end user, and closes the parent.

The end user gets a closure update from the Help Desk. SOC's ticket activity, findings, and analyst identity stay internal throughout.

---

## Why the Ticket Separation Mattered

In a federal environment, information compartmentalization isn't really optional. End users shouldn't be able to infer who investigated their report or what the SOC found beyond what was appropriate to share. The parent/child structure enforced that boundary without relying on agents to manually manage what they communicated. It was built into the process.

It also kept the SOC's queue clean. Analysts working a phishing case don't need end-user callbacks or status inquiries finding their way into their ticket flow. Keeping the two tickets separated meant each team could close out their side without interference.

---

## Retrospective

Looking back at the original incident with more experience, there's an obvious question: why didn't SOC loop in an Exchange Admin to run a tenant-wide message trace and purge?

A web-based header analyzer on a single reported email tells you a lot about that one email. It doesn't tell you how many other mailboxes received the same message and haven't reported it yet. An Exchange Admin with PowerShell access could have traced the campaign across the entire tenant and purged it from inboxes that were still sitting on it. That's the IR step that actually contains a phishing campaign at scale, and it doesn't look like it was part of the response.

Whether SOC made that call and I wasn't privy to it, whether there was a coordination gap getting to the Exchange Admin team quickly enough, or whether it genuinely wasn't on anyone's radar at the time, I can't say for certain from where I was sitting. But I can say that by the time I was doing Exchange Admin work at DISA a few years later, PowerShell-based message tracing was one of the first things I learned; and I understood exactly why it mattered.

The triage process I built was useful for what it was. It just addressed the symptom rather than the source.

---

## What This Translates To

| What I Built | Modern Equivalent |
|---|---|
| Phishing triage KBA designed from a live incident | Incident-driven runbook authoring |
| Email header extraction and pre-analysis before escalation | Email forensics, IOC identification, threat triage |
| Parent/child ticket structure for SOC isolation | Tiered incident response, separation of duties |
| Structured findings packaged for analyst handoff | Evidence documentation, SOC intake standards |

---

## Reflection

There was no ticket for "design a phishing triage process." I built it because I'd seen the gap firsthand and had enough context to do something about it. At the time I was reading security books outside of work and treating it like a hobby. Looking back, it was probably the clearest early sign that this is where I was headed.

---

*Part of the [`cloud-security-lab-notes`](https://github.com/JNewbrey87/cloud-security-lab-notes) retrospective collection. Joshua E. Newbrey | [linkedin.com/in/jnewbrey87](https://www.linkedin.com/in/jnewbrey87)*
