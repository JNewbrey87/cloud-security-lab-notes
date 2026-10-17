# AEGIS PoC Run 2: llama3.3:70b Adversarial Simulation

**Project:** AEGIS (Adaptive Execution and Generative Intelligence System)  
**Type:** Local AI Red Team / Blue Team simulation, proof of concept  
**Platform:** ASUS ROG Flow Z13 (ZX-A3G15), WSL2 Ubuntu, ROCm  
**Date:** June 2026  

---

## About AEGIS

AEGIS is my local Red Team / Blue Team AI simulation: one AI agent attacks, another defends, and all of the inference runs on my own hardware. No cloud API, no metered tokens, nothing leaving the machine. It's personal research, not a production system.

Most of it is an extension of my thesis work. My master's thesis, the [Cloud Forensic Readiness and Preservation Framework (CFRPF)](https://github.com/JNewbrey87/CFRPF), is about preserving evidence in cloud environments, including what automated systems decided and why. AEGIS is where I try those ideas in code. The rest is that I wanted to toy with AI past just chatting with a model, and learn how to build a stack of agents that actually work together. It's also adjacent to my day job, where I'm looking into newly-added AI agents.

I don't know what I don't know about agentic AI yet, so the approach is the same as the rest of this repo: build the thing, run it, watch where it breaks, and write down what I learned. The AEGIS files are numbered in the order I did them, starting here.

## Why I Did This One

Run 1 used llama3:8b to prove the stack worked. The framework ran, both agents finished their tasks, and I had a baseline. For Run 2, I moved up to llama3.3:70b, about 42 GB of weights, to answer one question: does the bigger model produce better security analysis, or just longer output?

The short answer is better, and the difference shows most in the Blue Team's defensive recommendations.

## Stack

| Component | Detail |
|-----------|--------|
| Model | llama3.3:70b (42 GB weights) |
| Inference backend | ROCm (AMD Radeon 8060S, unified memory) |
| Agent framework | CrewAI 1.14.6 |
| Runtime | Python 3.12, WSL2 Ubuntu (aegis-env-312 venv) |
| Ollama endpoint | localhost:11434 (WSL mirrored networking) |
| Tracing | Disabled; defaulted off after a 20-second prompt timeout |

## What I Built

The crew is two agents running in sequence on a shared threat scenario.

**Red Team Analyst** gets a target environment and has to identify attack vectors at the technique level. Category answers like "SQL injection" don't satisfy the `expected_output`; the agent has to describe how the technique works and what the attacker gains.

**Blue Team Analyst** receives the Red Team's output and answers each vector with one detection rule and one defensive control. One answer per vector, not a laundry list.

That constraint is the point. Forcing the Blue Team agent to commit to a single detection per vector shows whether the model understands which control actually closes the gap, or whether it's pattern-matching to a checklist. The 8b run exposed this; the 70b answered it.

```python
from crewai import Agent, Task, Crew, LLM

llm = LLM(model="ollama/llama3.3:70b", base_url="http://localhost:11434")

red_agent = Agent(
    role="Red Team Analyst",
    goal="Identify attack vectors against a target system",
    backstory="You are an adversarial security analyst who thinks like an attacker.",
    llm=llm,
    verbose=True
)

blue_agent = Agent(
    role="Blue Team Analyst",
    goal="Detect and counter attack vectors identified by the Red Team",
    backstory="You are a defensive security analyst. You receive threat intel from Red Team and build detection rules in response.",
    llm=llm,
    verbose=True
)

red_task = Task(
    description="Identify three realistic attack vectors against a cloud-hosted web application with user authentication. Be specific about technique, not just category.",
    expected_output="Three attack vectors, each with: technique name, how it works, and what the attacker gains.",
    agent=red_agent
)

blue_task = Task(
    description="For each attack vector the Red Team identified, propose one detection rule and one defensive control.",
    expected_output="A response for each of the three vectors: detection rule + defensive control.",
    agent=blue_agent
)

crew = Crew(agents=[red_agent, blue_agent], tasks=[red_task, blue_task], verbose=True)
result = crew.kickoff()
```

## Hardware During Inference

The Z13 has one 128 GB LPDDR5X memory pool shared between the CPU and GPU. At the time, the dedicated graphics setting was about 64 GB, and I assumed that was the ceiling the model had to fit under. I later learned that's not how it works on this machine: ROCm draws on the wider unified pool, so the setting isn't the real limit (more on that in [aegis-03](aegis-03-gpu-cpu-fallback.md)).

What the run looked like:

- **GPU compute:** 100%, with a sawtooth pattern on the Compute 0 lane, which is what active transformer inference looks like
- **Dedicated GPU memory:** 42.5 GB at model load, growing to 53.0 GB by the end of the run as the KV cache filled
- **System RAM:** about 60 of 63.6 GB (94%)
- **GPU temperature:** 81°C sustained, within spec for full-load inference on this chassis
- **CPU:** about 10%, just CrewAI orchestration

The near-idle CPU confirmed ROCm was doing the inference instead of falling back to the processor. That was the thing I was watching for. On the 8b run I couldn't be completely sure the GPU was doing the work; this run showed it plainly.

## The Threat Scenario

**Target:** cloud-hosted web application with user authentication  
**Red Team task:** three realistic attack vectors at technique-level specificity  
**Blue Team task:** for each vector, one detection rule and one defensive control

## Red Team Output

### Vector 1: JWT Injection via Malformed Token Payload

**Technique:** manipulate the JWT payload to carry a valid username with a wrong or missing password, exploiting weak or missing signature validation on the server. The attacker sends crafted tokens whose header specifies `alg: none`, bypassing signature checks on servers that accept unsigned tokens.

**Attacker gain:** an authenticated session without valid credentials; the attacker can impersonate any user whose username is known, including admins.

### Vector 2: Stored XSS via Unvalidated Search Input

**Technique:** inject persistent JavaScript into a search field with no server-side input validation. The script is stored in the database and runs in the browser of every later user who loads the affected page or results.

**Attacker gain:** session cookie theft, account hijacking, and actions taken as other users, including privileged ones.

### Vector 3: SSRF via Misconfigured Proxy

**Technique:** send requests through a misconfigured proxy to reach internal services, cloud metadata endpoints (such as `169.254.169.254`), or RFC 1918 address space that was never meant to be reachable from the application tier.

**Attacker gain:** access to internal services and databases, cloud instance metadata including IAM credentials, and lateral movement into otherwise isolated segments.

## Blue Team Output

### Response to Vector 1: JWT Injection

**Detection rule:** alert on more than 5 failed logins in 60 seconds from one source IP. Separately, flag any inbound JWT whose header specifies `alg: none` or whose payload doesn't match the expected schema.

**Defensive control:** enforce server-side JWT signature verification and reject any token specifying `alg: none`. Rate-limit every authentication endpoint. Use a vetted JWT library rather than a custom parser.

### Response to Vector 2: Stored XSS

**Detection rule:** watch web application logs for search queries containing XSS indicators like `<script>`, `javascript:`, `onload=`, and `onerror=`. Send matches to the alert queue for analyst review instead of relying only on WAF blocking.

**Defensive control:** input validation and output encoding on every user-supplied field. A WAF ruleset for XSS as a second layer, not the primary control. Content Security Policy headers to limit impact if an injection gets through anyway.

### Response to Vector 3: SSRF via Proxy

**Detection rule:** watch proxy logs for outbound requests to non-allowlisted domains, any RFC 1918 destination (`10.x`, `172.16-31.x`, `192.168.x`), and cloud metadata addresses (`169.254.169.254`). Alert on unusual routing regardless of response code.

**Defensive control:** restrict the proxy to an allowlist of destinations. Add egress filtering at the network layer as a separate enforcement point that doesn't depend on the proxy config. Segment the internal network to limit the blast radius if SSRF works anyway.

## 8b vs. 70b: What the Upgrade Changed

| Dimension | llama3:8b (Run 1) | llama3.3:70b (Run 2) |
|-----------|-------------------|----------------------|
| Attack vector specificity | Category level ("XSS attack," "brute force") | Technique level: mechanism, payload path, specific attacker gain |
| JWT vector | Named the class of attack | Named the `alg: none` bypass as the exploitation path |
| Blue Team defensive depth | One surface-level control | Layered controls, with reasoning about primary vs. secondary enforcement |
| SSRF response | "Fix the proxy configuration" | Egress filtering independent of the proxy, segmentation, metadata endpoint coverage |
| KV cache growth | Minimal | About 10 GB (42.5 to 53 GB) |
| Speed | Faster | Slower per token; the depth of the output was worth it |

The SSRF answer shows the difference best. The 8b model answered at the level of the misconfiguration: fix the proxy. The 70b model answered at the level of the threat: assume the proxy can be bypassed or misconfigured again, and build controls that don't depend on it being right. Egress filtering plus segmentation is layered defense, not a config patch, and that's the kind of answer that holds up under review.

## What I Took Away

**Model scale changed the quality of the reasoning, not just the length.** The 70b answers weren't longer versions of the 8b answers; they were built differently. The 8b named threat categories and prescribed the most obvious single fix. The 70b reasoned about the threat and prescribed controls that still hold if the simple fix fails. In security work, where layered defense means never trusting one control, that difference matters.

**KV cache growth isn't a measure of reasoning.** I originally read the roughly 10 GB of cache growth as a sign of longer reasoning chains. Looking back, that doesn't hold up. The KV cache grows with the amount of context the model is holding, and a 70b model needs more cache per token than an 8b model anyway, so the two numbers can't be compared as evidence of quality. What the growth does show is the model holding more context as the run goes on, and that the memory budget for a 70b run has to account for it, not just the weights.

**ROCm was doing the work.** Compute 0 at 100% with the sawtooth pattern, and the CPU around 10% the whole time, settled whether inference was GPU-accelerated. A 42 GB model running fully in GPU-addressable memory is something most machines can't do without a very large discrete card.

**Constrained tasks surface what a model can actually do.** Asking for one detection and one control per vector, instead of "how would you defend against this," forced the Blue Team agent to make a decision. That decision was what I was actually measuring. With an open-ended prompt, both models would have produced similar-looking lists, and the gap in depth would have stayed hidden.

## Lab Environment

| Component | Details |
|-----------|---------|
| Host | ASUS ROG Flow Z13 (ZX-A3G15): Ryzen AI Max+ 395, Radeon 8060S, 128 GB unified memory |
| Runtime | WSL2 Ubuntu, Python 3.12 venv |
| Inference | Ollama with ROCm, llama3.3:70b |
| Orchestration | CrewAI 1.14.6, two sequential agents |
| Network | Local only; no cloud API calls |

## What Came Next

Looking back, the "up next" list I wrote after this run was more ambitious than what I actually did. What I really did next:

- **[aegis-01](aegis-01-crewai-migration.md):** setting the full stack back up on this machine, migrating the PoC script to CrewAI 1.x, and recovering from a broken Ollama update.
- **[aegis-02](aegis-02-evidence-ledger.md):** a tamper-evident evidence ledger that records what each agent decided and why. My original idea of "automating run logging" grew into this.
- **[aegis-03](aegis-03-gpu-cpu-fallback.md):** finding out the 70b model had quietly fallen back to the CPU, and why.

Ideas from the original list I haven't gotten to:

- A Blue Team run on a code-focused model (qwen2.5-coder) to see whether it produces structured Sigma rules instead of plain-language controls.
- Broader scenarios beyond a web app with authentication, such as ICS/OT, cloud-native misconfigurations, and supply chain.

---

*Part of the [cloud-security-lab-notes](https://github.com/JNewbrey87/cloud-security-lab-notes) portfolio, building toward Cloud Security Analyst and the CySA+ / SC-200 / SC-500 certifications.*
