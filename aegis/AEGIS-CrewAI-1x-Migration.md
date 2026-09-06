# AEGIS: Migrating the PoC to CrewAI 1.x + Full 6-Model Stack Validation

**Project:** AEGIS — Adaptive Execution and Generative Intelligence System
**Type:** Local AI Red/Blue Team Simulation — engineering note
**Platform:** ROG Flow Z13 (Ryzen AI Max+ 395, Radeon 8060S, 128 GB unified), WSL2 Ubuntu, ROCm
**Follows:** [AEGIS-PoC-70b.md](AEGIS-PoC-70b.md) (Run 2)

---

## Background

AEGIS is a personal research project: a Red Team AI agent and a Blue Team AI agent run against each other via CrewAI, entirely on-device. No cloud API, no metered tokens, nothing leaving the machine. Run 2 validated the adversarial simulation on llama3.3:70b.

This note covers a maintenance cycle that turned into a real debugging session: bringing the proof-of-concept forward onto CrewAI 1.x, recovering from a broken Ollama auto-update, and re-validating the full six-model stack end to end. The kind of work that doesn't show up in a demo but is most of what running local inference actually is.

> Real hostnames, internal addresses, and environment-specific identifiers are omitted per standard portfolio security practice.

---

## The CrewAI 1.x Breaking Change

The PoC was written against a pre-1.0 CrewAI that accepted a LangChain `ChatOllama` object as the agent `llm`. CrewAI 1.x routes all model calls through LiteLLM internally and no longer accepts LangChain chat model objects; it expects either a provider-prefixed string or a `crewai.LLM` object.

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

- The `ollama/` provider prefix on the model string is mandatory. Without it LiteLLM cannot resolve the backend and the crew fails at first inference.
- `num_predict` (Ollama's token cap) is surfaced as `max_tokens` through the LiteLLM interface. Same knob, different name.

---

## Ollama Recovery

Mid-cycle, Ollama auto-updated during a reboot and the update landed incomplete: `ollama list` still returned models (metadata intact), but `ollama run` returned a 500 because the `llama-server` binary was missing from the install tree. `ollama list` reporting models is not proof the runtime is healthy.

Fix was a clean reinstall from the official installer. That wiped the local model store, so the entire stack had to be re-pulled (66 GB+). Rebuilt all six models and re-validated.

Post-reboot, `localhost:11434` was unreachable from WSL2 until Ollama bound to `0.0.0.0`; a subsequent clean reboot re-established mirrored networking on its own. Worth knowing that WSL2-to-Windows loopback for Ollama is not always stable across reboots.

---

## Validated Stack

| Model | Size | Role |
|-------|------|------|
| llama3.3:70b | 42 GB | Primary reasoning, both agents |
| deepseek-r1:14b | 9.0 GB | Red Team chain-of-thought planning |
| qwen2.5-coder:14b | 9.0 GB | Code / payload generation |
| mistral:7b | 4.4 GB | Blue Team fast triage |
| hermes3:8b | 4.7 GB | Fallback / validation runs |
| nomic-embed-text | 274 MB | RAG embeddings |
| llama3.2:3b | 2.0 GB | Pipeline routing / fast local test |

Total footprint roughly 72 GB, all resident on one machine.

**GPU profile during 70B inference** (Radeon 8060S, unified memory):

- Dedicated VRAM in use: 46–48 GB of 63.8 GB allocated
- Compute utilization: 27–45%
- Temperature under load: 71–78 °C

Running a 42 GB model on integrated graphics is only possible because the unified memory architecture lets the GPU address system RAM directly; there's no discrete-VRAM ceiling to spill over.

---

## Simulation Output

With the stack rebuilt, the adversarial crew ran clean end to end. Both agents completed and produced structured output. A representative slice:

**Red Team** identified attack vectors at technique-level specificity (not generic categories):

- Credential stuffing via reused passwords
- Blind SQL injection escalating to command execution
- Session hijacking through CSRF token prediction

**Blue Team** returned a detection and a control for each vector:

- Credential stuffing → Wazuh rate rule on failed logins per source IP within a window + Fail2Ban lockout
- SQL injection → IDS signature on the HTTP pattern + a WAF rule set blocking known SQLi patterns
- Session hijacking → KQL anomaly rule on session access from multiple IPs in a short window + `SameSite=Strict` and HTTPS-only cookie policy

The point of the exercise isn't the specific vectors; it's that a fully local multi-agent pipeline can produce paired offense/defense reasoning at usable specificity with no external dependency.

---

## What This Demonstrates

- Migrating a multi-agent pipeline across a framework major-version break (LangChain-object → LiteLLM string/object)
- Diagnosing a partial-update runtime failure where surface indicators (`ollama list`) lie about health
- Operating a 70B model on integrated graphics via unified memory
- Local-only AI orchestration: no cloud API, no data egress, full control of the inference path

---

*Part of the [cloud-security-lab-notes](https://github.com/JNewbrey87/cloud-security-lab-notes) portfolio. AEGIS is personal research, not a production system.*
