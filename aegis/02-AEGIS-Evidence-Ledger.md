# AEGIS Evidence Ledger: Tamper-Evident Custody for Agent Decisions

**Project:** AEGIS (Adaptive Execution and Generative Intelligence System)  
**Type:** Evidentiary layer for the Red Team / Blue Team simulation  
**Platform:** ASUS ROG Flow Z13 (ZX-A3G15), local inference through Ollama  
**Date:** September 2026

---

## Why I Did This One

AI agents are starting to make security decisions at machine speed, and almost nothing preserves what they decided, or why, in a form that survives later editing. When an agent's action gets questioned by an investigator, a regulator, or a court, what usually survives is the outcome: not the reasoning, not the model, not the inputs. That gap is what my thesis work is built around. This ledger is the smallest honest attempt I could make at closing part of it in code.

The framing that matters to me: this is chain of custody for AI *decisions*, not AI outputs. It maps to two parts of [CFRPF](https://github.com/JNewbrey87/CFRPF): Immutable Evidence Preservation, and the framework's cross-cutting AI Auditability principle.

## What I Built

An append-only JSONL ledger where every record is hash-chained to the one before it, plus a bridge that writes to it from the live CrewAI Red Team / Blue Team crew. The ledger uses only the Python standard library: hashlib, json, datetime, pathlib, and argparse. No dependencies, no services, no network, which felt like the right posture for something whose whole job is to be trustworthy.

Each record carries the agent's decision fields (telemetry observed, rule triggered, response selected, reasoning, event type) plus the chain linkage. The linkage is deliberately boring, because boring is auditable:

| Field | Meaning |
|---|---|
| `payload_sha256` | Hash of the record's content fields; catches any later edit to the content |
| `prev_hash` | The previous record's `record_hash`; the actual chain link |
| `record_hash` | `SHA-256(prev_hash + payload_sha256)` |

The first record's `prev_hash` is 64 zeros, the genesis link. Verification runs four checks per record and exits 0 on pass and 1 on fail, so it can drop straight into a scheduled job.

Two design choices in the bridge are worth calling out:

- **It never imports CrewAI.** It reads whatever the crew returns, defensively. I plan to move AEGIS to LangGraph eventually, and the evidence layer shouldn't need a rewrite when the orchestration underneath it changes.
- **Structured-output failure degrades instead of losing evidence.** If a model fails to fill the schema, the bridge writes one record marked `(unstructured)` instead of throwing an error. A parse failure costs detail, not evidence.

## Proving It's Tamper-Evident

The verification demo is three steps, and the third is the point:

```
STEP 1  verify the intact chain            -> PASS
STEP 2  naive edit of seq 2, hashes as-is  -> FAIL at seq 2 (content check)
STEP 3  edit seq 2 AND re-sign that record -> FAIL at seq 3 (broken link)
```

Step 3 is why I chained the records instead of just hashing each one. Even if an attacker fixes the hash on the record they changed, the link to the next record breaks, and verification points straight back at the tampered entry. Both FAILs are the tool working correctly.

The live chain has been verified four times by two independent methods. One of them recomputed every `record_hash` from `SHA-256(prev_hash + payload_sha256)` outside the tool that wrote them, so the PASS isn't just the tool agreeing with itself.

## What the Ledger Preserved: Three Ways the Agents Failed

The ledger holds twelve records across three live runs. All three runs finished in structured mode, meaning the models filled the typed schema and produced one record per attack vector. I didn't expect that from a 7B model, and the fallback path never got used. What those records preserved is a progression of failures, and that turned out to be better evidence than three clean runs would have been.

| Records | Model | Failure mode |
|---|---|---|
| seq 1 to 3 | mistral:7b | Plausible rules aimed at the wrong telemetry source |
| seq 5 to 7 | llama3.3:70b | Correct detection logic, invented rule syntax and fabricated rule IDs |
| seq 9 to 11 | llama3.3:70b | No defensive work at all; schema satisfied, content copied from the attacker |

### Mode One: mistral:7b, Watching the Wrong Machine

The detection rules were structurally valid, confidently worded, and operationally wrong. All three watched for the attacker's own tools running as processes on the defender's host: JWT Hunter, Burp Suite, Hydra, curl. An attacker runs those from their own machine, so a Wazuh rule watching your web server for a process named `hydra` will never fire. The same run also labeled Sysmon Event ID 2 as process creation; Event ID 2 is FileCreateTime, and process creation is Event ID 1. The defensive controls were mostly fine. It was specifically the detection engineering that fell apart, which says something about where small models are weak.

### Mode Two: llama3.3:70b, Right Idea, Made-Up Syntax

The larger model fixed the category error and watched the defender's own telemetry: authentication failure rates, input-field content, outbound request patterns, each with thresholds. Its weakness was subtler. The logic was sound, but the rule IDs were invented. `SIG-2023-001` isn't a Sentinel convention, and Sentinel uses KQL, so what it produced was correct thinking in prose, not something you could deploy.

The cleanest comparison is SSRF, the only vector both models produced, which makes it the one true head-to-head:

- **mistral:7b** watched for `curl` or `hydra` process starts on the monitored host.
- **llama3.3:70b** specified an egress firewall policy plus detection of outbound requests to internal services.

Egress control is the standard SSRF mitigation. The small model wasn't just less polished; it was solving a problem that doesn't exist.

### Mode Three: llama3.3:70b, No Defensive Work at All

This is the strongest thing the project has produced so far. On the final run, the Blue Team agent did no defensive work. It copied the Red Team's attack description straight into the defensive fields, field for field. The `detection_platform` values came back as Hydra, Burp Suite's Intruder, and Metasploit's CookieJar, which are attack tools, when the task named Wazuh, Sentinel, or Snort. The `response_selected` for the first vector is literally the attacker's objective, "Gain unauthorized access to user accounts," presented as the defense, and the `reasoning` field is the attacker's objective word for word.

Here's what makes it matter: every automated signal reported success. Pydantic schema validation passed. The pipeline reported `4 records written (structured)`, not `fallback`. The hash chain verified intact. No error appeared anywhere. An agent did none of its assigned work, and nothing in the pipeline noticed. Structured output means well-formed, not correct; schema validation is not semantic validation. Any governance pipeline gating on "did it validate" would have approved a run where the agent produced nothing of value.

One hedge on this run: the same model did real detection engineering six days earlier, so this is a regression, not a fixed trait of the model. Two things changed between those runs, the processor and the context length, and neither should change what the model says. My leading guess is run-to-run variance at temperature 0.7, since "fill the schema with the nearest available text" is a known failure mode for structured output on local models. That's a hypothesis, not a finding. A few more runs would settle it, and they now take about four minutes each.

## A Note About These Records

Records from before September 11, 2026 were produced on the CPU; later ones ran on the GPU. That's the one distinction the chain doesn't capture on its own, so I want to be precise about it. This was a regression, not a machine that never worked: the same 70B model ran on the GPU in June and July, then at some point the installed inference backend changed, the machine fell back to the CPU without any warning, and the GPU path wasn't restored until September 11 ([aegis-03](03-AEGIS-GPU-Investigation.md) covers how I found it).

The two ledger runs from that CPU window are still valid evidence. They were produced correctly, just on a slower path. I chose to note the CPU-versus-GPU split in this paragraph instead of adding code or a second file, which means the chain isn't self-describing on this point. That's a mild tension with my own framework's principle that evidence should stand on its own, and I'd rather name it than hide it.

## Limitations

- **Tamper-evidence only covers what's inside the file.** The chain catches edits to the records it holds; it can't prove the whole file wasn't deleted, and there's no external timestamp authority. Real WORM storage adds an external anchor. That's the next layer, not a gap I'm hiding.
- **The reasoning field records what the agent said its reasoning was.** It doesn't verify the decision was right; all three failure modes show exactly that.
- **The detection rules are LLM-generated text.** None of them have been deployed to a live Wazuh, Snort, or Sentinel instance.
- **Only the SSRF pair is a true head-to-head.** The Red Team produced different vectors on each run, so the rest of the comparison is apples to oranges.
- **The failure-handling path is untested.** A `crew_failure` record path exists in the code but has never been exercised, so I'm not claiming failure handling works.
- **The chain doesn't mark CPU-era versus GPU-era records.** The note above is needed to read it correctly.
- **The Blue Team collapse is unexplained.** Variance is the leading hypothesis, not a conclusion.
- **Still CrewAI.** The LangGraph migration hasn't started.

## What I Took Away

**The ledger's job is to preserve, not to judge.** It never caught any of these errors. It has no idea the mistral rules are aimed at the wrong machine or that the final run did nothing. What it does is preserve, word for word and tamper-evidently, what each agent decided and what it claimed as its reasoning, so someone reading seq 9 through 11 six months from now can see right away that the Blue Team never did its job. Without the ledger, the only thing left from that run would be a console line saying it completed successfully and wrote 4 structured records.

**Validation isn't the same as correctness.** The schema check, the pipeline status, and the chain verification all passed on a run with zero useful content. Those checks prove the output is well-formed and unaltered. They say nothing about whether it's right, and a governance process needs a separate check for that.

**Changing the model behind a decision is a governance event.** Swap the model and the quality of the decisions can change completely, with nothing in the logs to show it. A provenance chain that can't be quietly rewritten fixes the decision, the model, and the stated rationale in place, so that change becomes auditable later.

**Failures make better evidence than clean runs.** Three clean runs would have shown the ledger works. Three different failures, preserved exactly, show why it's needed.

## Lab Environment

| Component | Details |
|-----------|---------|
| Host | ASUS ROG Flow Z13 (ZX-A3G15): Ryzen AI Max+ 395, Radeon 8060S, 128 GB unified memory |
| Ledger | Append-only JSONL, SHA-256 hash chain, Python standard library only |
| Bridge | Pydantic schema for Blue Team assessments; one provenance record per run plus one per vector |
| Orchestration | CrewAI Red Team / Blue Team crew |
| Models | mistral:7b (run 1), llama3.3:70b (runs 2 and 3) |
| Runs | Three live runs, twelve records, chain verified four times by two methods |

## What Came Next

- **[aegis-03](03-AEGIS-GPU-Investigation.md):** the companion to this one. While setting up these runs, I found the 70b model had been running on the CPU, and tracked down why.

Still on my list:

- A short series of repeat runs to find out whether the Blue Team collapse is temperature variance or something systematic. If it's systematic, the fix is in the prompt, not the infrastructure.
- Exercising the untested `crew_failure` path, so a crashed crew still writes an evidence record.
- The LangGraph migration, where the bridge's framework-agnostic design is supposed to carry over untouched.

Back to the start: [aegis-00](00-AEGIS-PoC-70b.md).

---

*Part of the [cloud-security-lab-notes](https://github.com/JNewbrey87/cloud-security-lab-notes) portfolio, building toward Cloud Security Analyst and the CySA+ / SC-200 / SC-500 certifications.*
