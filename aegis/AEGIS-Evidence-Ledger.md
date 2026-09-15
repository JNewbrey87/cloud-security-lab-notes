# AEGIS Evidence Ledger: Tamper-Evident Custody for Agent Decisions

**Project:** AEGIS (Adaptive Execution and Generative Intelligence System)  
**Type:** Local AI Red/Blue Team simulation, evidentiary layer  
**Platform:** ASUS ROG Flow Z13 (Ryzen AI Max+ 395, Radeon 8060S, 128 GB unified), local inference via Ollama  
**Companion:** [The GPU That Was Never On](AEGIS-GPU-Investigation.md) | **Follows:** [AEGIS-PoC-70b.md](AEGIS-PoC-70b.md)  

---

## The problem

Autonomous agents are starting to make security decisions at machine speed, and almost nothing preserves what they decided or why in a form that survives later editing. When an agent's action is questioned by an investigator, a regulator, or a court, the surviving record is usually the outcome, not the reasoning, the model, or the inputs. This is the AI audit gap my thesis work is built around; the evidence ledger is the smallest honest attempt I could make at closing part of it in code.

The framing that matters: this is chain of custody for AI decisions, not AI outputs. It maps to CFRPF Component 2 (Immutable Evidence Preservation) and the framework's cross-cutting AI Auditability principle.

## What I built

An append-only JSONL ledger where every record is hash-chained to the one before it, plus a bridge that writes to it from a live CrewAI Red Team / Blue Team crew. The ledger uses the Python standard library only: hashlib, json, datetime, pathlib, argparse. No dependencies, no services, no network, which is the right posture for something whose entire job is to be trustworthy.

Each record carries the agent's decision fields (telemetry observed, rule triggered, response selected, reasoning, event type) and the chain linkage. The linkage is deliberately boring, because boring is auditable:

| Field | Meaning |
|---|---|
| `payload_sha256` | Hash of the record's content fields; catches any later edit to the content |
| `prev_hash` | The previous record's `record_hash`; the actual chain link |
| `record_hash` | `SHA-256(prev_hash + payload_sha256)` |

The first record's `prev_hash` is 64 zeros, the genesis link. Verification runs four checks per record and exits 0 on pass, 1 on fail, so it drops straight into a scheduled job without modification.

The bridge is worth two notes. It never imports CrewAI; it reads whatever the crew returned defensively, because the production target is a LangGraph migration and the evidentiary layer should not need rewriting when the orchestration substrate changes. And structured-output failure degrades rather than losing: if a model fails to fill the schema, the bridge writes a single record marked `(unstructured)` instead of raising, so a parse failure costs granularity, not evidence.

## Proof it is tamper-evident

The verification demo is three lines, and the middle one is the point:

```
STEP 1  verify the intact chain            -> PASS
STEP 2  naive edit of seq 2, hashes as-is  -> FAIL at seq 2 (content check)
STEP 3  edit seq 2 AND re-sign that record -> FAIL at seq 3 (broken link)
```

Step 3 is the argument for chaining over plain per-record hashing. Even when an attacker repairs the hash on the record they altered, the link to the very next record breaks, and verification points straight back at the tampered entry. Both FAILs are the tool working correctly, not errors. The live chain has been verified four times by two independent methods, including an out-of-band recomputation of every `record_hash` from `SHA-256(prev_hash + payload_sha256)` outside the tool that wrote them, so the pass is not just the tool agreeing with itself.

## Proof it is useful: three ways the agents failed, all preserved

The ledger now holds twelve records across three live runs. All three runs completed in structured mode, meaning the models filled the typed schema and produced one record per attack vector; I did not expect that from a 7B model, and the graceful-degradation path went unused. What those records preserved is a progression of failures, and the progression is better evidence than three clean runs would have been.

| Records | Model | Failure mode |
|---|---|---|
| seq 1 to 3 | mistral:7b | Plausible rules aimed at the wrong telemetry source |
| seq 5 to 7 | llama3.3:70b | Correct detection logic, invented rule syntax and fabricated rule IDs |
| seq 9 to 11 | llama3.3:70b | No defensive work at all; schema satisfied, content transposed from the attacker |

**Mode one, mistral:7b.** Its detection rules are structurally valid, confidently worded, and operationally wrong. All three watch for the attacker's own tooling running as a local process on the defender's host: JWT Hunter, Burp Suite, Hydra, curl. An attacker runs those from their own machine, so a Wazuh rule watching your web server for a process named `hydra` will never fire. The same run labeled Sysmon Event ID 2 as process creation; Event ID 2 is FileCreateTime, and process creation is Event ID 1. Its defensive controls were largely fine; it was specifically the detection engineering that collapsed, which is its own quiet lesson about where small models are weak.

**Mode two, llama3.3:70b.** The larger model fixed the category error and monitored the defender's own telemetry instead: authentication failure rates, input-field content, outbound request patterns, each with thresholds. Its weakness was different and subtler; the detection logic was sound, but the rule IDs are invented. `SIG-2023-001` is not a Sentinel convention, and Sentinel uses KQL, so what it produced is correct thinking in prose, not deployable syntax.

The cleanest single comparison is SSRF, the only vector both models produced, which makes it a controlled head-to-head where the rest is not:

- **mistral:7b** watched for `curl` or `hydra` process starts on the monitored host.
- **llama3.3:70b** specified an egress firewall policy plus detection of outbound requests to internal services.

Egress control is the canonical SSRF mitigation. The small model was not less polished; it was solving an imaginary problem.

**Mode three is the closing argument, and it is the strongest thing this project has produced.** On the final run, the Blue Team agent did no defensive work at all. It transposed the Red Team's attack description straight into the defensive fields, field for field. The `detection_platform` values came back as Hydra, Burp Suite's Intruder, and Metasploit's CookieJar, which are attack tools, when the task named Wazuh, Sentinel, or Snort. The `response_selected` for the first vector is literally the attacker's objective, "Gain unauthorized access to user accounts," presented as the defensive response, and the `reasoning` field is the attacker's objective text verbatim.

Here is what makes it matter: every automated signal reported success. Pydantic schema validation passed. The pipeline reported `4 records written (structured)`, not `fallback`. The hash chain verified intact. No error appeared anywhere. An agent did none of its assigned work, and nothing in the pipeline noticed. Structured output means well-formed, not correct; schema validation is not semantic validation. Any governance pipeline gating on "did it validate" would have green-lit a run where the agent produced nothing of value.

One honest hedge on this run: the same model produced genuine detection engineering six days earlier, so the collapse is a regression, not a fixed property of the model. Two things changed between the runs, the processor and the context length, and neither should alter semantics. The leading explanation is run-to-run variance at temperature 0.7, since "fill the schema with the nearest available text" is a known structured-output failure mode on local models. That is a hypothesis, not an established finding; a few more runs would settle it, and they now cost about four minutes each.

## Why the ledger is the point

The ledger never caught any of these errors; it has no idea the mistral rules are misaimed or that the final run did nothing. Its job is to preserve, verbatim and tamper-evidently, exactly what each agent decided and what it claimed as its reasoning, so that someone reading seq 9 through 11 six months from now sees immediately that the Blue Team never did its job. Without the ledger, the only surviving artifact from that run would be a console line reading "run completed successfully, 4 records written (structured)." Swapping the model behind a security decision is a governance event that is otherwise invisible; with a provenance chain you cannot quietly rewrite, the decision, the model, and the stated rationale are all fixed and later-auditable.

## An honest note about these records

The records in this ledger that predate 2026-09-11 were produced on CPU, and the later ones ran on the GPU; that is the one distinction the chain does not encode on its own. It is worth being precise about how that happened, because it was a regression, not a machine that never worked. This same 70B model ran genuinely on the GPU in June and July; at some later point the installed inference backend changed, the machine fell back to CPU with nothing to announce it, and the path was not restored until 2026-09-11. The two ledger runs from that CPU window are still valid evidence, produced correctly on a slower path, not weaker records. I chose to carry the CPU-versus-GPU distinction in this one sentence rather than add code or a second file, so the chain is not self-describing on the point; that is a mild tension with the framework's own principle that evidence should stand alone, and I would rather name it than paper over it. The infrastructure is sound, and a full two-agent crew run completes in a few minutes on the GPU.

## Limitations

- **Tamper-evidence is within the file.** The chain catches edits to records it holds; it does not prove the file was not deleted wholesale, and there is no external timestamp authority. Real WORM adds an external anchor, and that is the next layer, not a gap being hidden.
- **The reasoning field records what the agent said its reasoning was.** It does not verify the decision was correct; all three failure modes demonstrate exactly that.
- **Detection rules are LLM-generated text.** Nothing here has been deployed to a live Wazuh, Snort, or Sentinel instance.
- **Only the SSRF pair is a true head-to-head.** The Red Team generated different vectors on each run, so the rest is apples to oranges and should not be read as a matched comparison.
- **The failure-handling path is untested.** A `crew_failure` record path exists in the code, but it has never been exercised, so I am not claiming failure handling works.
- **The chain does not distinguish CPU-era from GPU-era records.** The external sentence above is required to interpret it.
- **The Blue Team collapse is unexplained.** Variance is the leading hypothesis, not a conclusion.
- **Still CrewAI.** The LangGraph migration has not started.

## What's next

A short series of repeat runs to establish whether the Blue Team collapse is temperature variance or systematic; if systematic, the fix is prompt-side, not infrastructure. Exercising the untested `crew_failure` path so a crashed crew still writes an evidence record. And eventually the LangGraph migration, at which point the bridge's framework-agnostic design is meant to carry over untouched.

---

*Part of the [cloud-security-lab-notes](https://github.com/JNewbrey87/cloud-security-lab-notes) portfolio. AEGIS is personal research, not a production system.*
