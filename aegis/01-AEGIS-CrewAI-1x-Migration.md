# AEGIS: Rebuilding the Stack on New Hardware and Migrating to CrewAI 1.x

**Project:** AEGIS (Adaptive Execution and Generative Intelligence System)  
**Type:** Engineering note: framework migration and runtime recovery  
**Platform:** ASUS ROG Flow Z13 (ZX-A3G15), WSL2 Ubuntu, ROCm  
**Date:** July 2026

---

## Why I Did This One

I first built the AEGIS stack on my previous PC. When I moved to the ASUS, I set it up again, and in July I brought the full PoC script over and re-ran it. It didn't run. The script still passed a LangChain `ChatOllama` object to the agents, and the CrewAI 1.x install on the new machine no longer accepts that.

What started as a quick fix turned into a full maintenance cycle: migrating the script to the CrewAI 1.x way of doing things, recovering from a broken Ollama auto-update, and re-validating the whole model stack end to end. None of it shows up in a demo, but it's most of what running local inference actually involves.

## What I Built

### 1. The CrewAI 1.x Breaking Change

CrewAI 1.x routes every model call through LiteLLM internally. It no longer accepts LangChain chat model objects; it expects either a provider-prefixed string or a `crewai.LLM` object.

**Before (breaks on CrewAI 1.x):**

```python
from langchain_ollama import ChatOllama

llm = ChatOllama(
    model=MODEL_NAME,
    base_url=OLLAMA_BASE_URL,
    temperature=0.7,
    num_predict=2048,
)
```

**After (CrewAI 1.x):**

```python
from crewai import Agent, Task, Crew, Process, LLM

llm = LLM(
    model=f"ollama/{MODEL_NAME}",   # provider prefix is required
    base_url=OLLAMA_BASE_URL,
    temperature=0.7,
    max_tokens=2048,                # LiteLLM name; was num_predict
)
```

Two things to know if you hit this:

- The `ollama/` prefix on the model string is mandatory. Without it, LiteLLM can't resolve the backend and the crew fails on its first inference call.
- `num_predict`, Ollama's token cap, is called `max_tokens` through LiteLLM. Same setting, different name.

### 2. Recovering a Broken Ollama Update

Partway through, Ollama auto-updated during a reboot and the update landed incomplete. `ollama list` still showed my models, because the metadata was intact, but `ollama run` returned a 500 error because the `llama-server` binary was missing from the install. So `ollama list` showing models is not proof the runtime works.

The fix was a clean reinstall from the official installer. That wiped the local model store, so I had to re-pull the entire stack, over 66 GB, and re-validate every model.

After a reboot, `localhost:11434` was also unreachable from WSL2 until I set Ollama to listen on `0.0.0.0`. A later clean reboot brought mirrored networking back on its own. WSL2-to-Windows loopback for Ollama isn't always stable across reboots.

### 3. Re-Validating the Model Stack

Six models plus an embedding model, all resident on one machine:

| Model | Size | Role |
|-------|------|------|
| llama3.3:70b | 42 GB | Primary reasoning, both agents |
| deepseek-r1:14b | 9.0 GB | Red Team chain-of-thought planning |
| qwen2.5-coder:14b | 9.0 GB | Code and payload generation |
| mistral:7b | 4.4 GB | Blue Team fast triage |
| hermes3:8b | 4.7 GB | Fallback and validation runs |
| llama3.2:3b | 2.0 GB | Pipeline routing, fast local tests |
| nomic-embed-text | 274 MB | RAG embeddings |

Total footprint is roughly 72 GB.

GPU profile during 70B inference that month (Radeon 8060S, unified memory):

- Dedicated GPU memory in use: 46 to 48 GB of 63.8 GB
- Compute utilization: 27% to 45%
- Temperature under load: 71 to 78°C

## What I Observed

With the stack rebuilt, the crew ran clean end to end, and both agents produced structured output.

**Red Team** came back with technique-level vectors rather than generic categories:

- Credential stuffing using reused passwords
- Blind SQL injection escalating to command execution
- Session hijacking through CSRF token prediction

**Blue Team** returned a detection and a control for each one:

- **Credential stuffing:** a Wazuh rate rule on failed logins per source IP inside a time window, plus a Fail2Ban lockout
- **SQL injection:** an IDS signature on the HTTP pattern, plus a WAF ruleset blocking known SQLi patterns
- **Session hijacking:** a KQL anomaly rule for session access from multiple IPs in a short window, plus a `SameSite=Strict` and HTTPS-only cookie policy

At the time, my takeaway was that a fully local, multi-agent pipeline could produce paired offense and defense reasoning at usable specificity with no outside dependency.

> **Retrospective note (October 2026):** looking back at this run after the [evidence ledger](02-AEGIS-Evidence-Ledger.md) work, the specific rule references the Blue Team gave (Wazuh rule 100140, Snort rule 12001, and a KQL rule named `custom_rule_session_hijack_001`) appear to be invented by the model, not real rules. I didn't check them against real rule sets at the time. The detection logic is reasonable; the rule IDs aren't something you could deploy. That's the same "correct logic, invented syntax" pattern the ledger later preserved on purpose.

## What I Took Away

**A major-version framework change can break a script that never changed.** The code was the same; the framework underneath it moved from accepting LangChain objects to routing everything through LiteLLM. When a pipeline breaks after a rebuild, check what versions came along with the new install before debugging the code.

**Surface health checks can lie.** `ollama list` said everything was fine while the runtime couldn't serve a single request. The real check is whether the thing actually does its job, which here meant `ollama run`, not whether its metadata looks right.

**Unified memory is what makes a 70B model possible on integrated graphics.** The GPU addresses system memory directly, so there's no discrete VRAM ceiling to spill past.

**Local-only has real costs.** No cloud API and no data leaving the machine also means I'm the one re-pulling 66 GB of models and fixing a broken updater. That trade is worth it for this project, but it's a trade.

## Lab Environment

| Component | Details |
|-----------|---------|
| Host | ASUS ROG Flow Z13 (ZX-A3G15): Ryzen AI Max+ 395, Radeon 8060S, 128 GB unified memory |
| Runtime | WSL2 Ubuntu, Python 3.12 venv |
| Inference | Ollama with ROCm |
| Orchestration | CrewAI 1.x (LiteLLM routing) |
| Model store | Six models plus one embedding model, about 72 GB |

## What Came Next

- **[aegis-02](02-AEGIS-Evidence-Ledger.md):** a tamper-evident evidence ledger that records what each agent decided and why.
- **[aegis-03](03-AEGIS-GPU-Investigation.md):** some time after this, the 70b model quietly fell back to the CPU; that file covers how I found it and why.

Back to the start: [aegis-00](00-AEGIS-PoC-70b.md).

---

*Part of the [cloud-security-lab-notes](https://github.com/JNewbrey87/cloud-security-lab-notes) portfolio, building toward Cloud Security Analyst and the CySA+ / SC-200 / SC-500 certifications.*
