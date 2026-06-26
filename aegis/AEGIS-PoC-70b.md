# AEGIS PoC Run 2: llama3.3:70b Adversarial Simulation

**Project:** AEGIS — Adaptive Execution and Generative Intelligence System  
**Lab Type:** Local AI Red/Blue Team Simulation  
**Platform:** ROG Flow Z13, WSL2 Ubuntu, ROCm  
**AEGIS Run:** 2 of N  
**Date Completed:** June 2026

---

## Background

AEGIS is a personal research project: a local adversarial simulation where a Red Team AI agent and a Blue Team AI agent run against each other via CrewAI, with all inference happening on-device. No cloud API, no metered tokens, no data leaving the machine. The goal isn't to replace a real red team; it's to build hands-on familiarity with multi-agent AI orchestration, observe how model scale affects reasoning quality in a security context, and generate a documented artifact I can point to.

Run 1 used llama3:8b to validate the stack. The framework worked, the agents completed their tasks, and I had a baseline to compare against. Run 2 upgrades to llama3.3:70b, 42GB of weights loaded directly into GPU-addressable unified memory, to answer a specific question: does the larger model produce meaningfully better security analysis, or just longer output?

The short answer is yes, and the difference is most visible in the Blue Team's defensive recommendations.

---

## Stack

| Component | Detail |
|-----------|--------|
| Model | llama3.3:70b (42 GB weights) |
| Inference Backend | ROCm (AMD Radeon 8060S, unified memory architecture) |
| Agent Framework | CrewAI 1.14.6 |
| Runtime | Python 3.12, WSL Ubuntu (aegis-env-312 venv) |
| Ollama Endpoint | localhost:11434 (WSL mirrored networking mode) |
| Tracing | Disabled; defaulted off after 20s prompt timeout |

> Real hostnames, internal addresses, and environment-specific identifiers are omitted per standard portfolio security practice.

---

## What I Built

The crew structure is two agents running sequentially on a shared threat scenario.

**Red Team Analyst** is given a target environment and tasked with identifying attack vectors at technique-level specificity. Generic category answers like "SQL injection" don't satisfy the `expected_output` constraint; the agent has to describe how the technique works and what the attacker gains from it.

**Blue Team Analyst** receives the Red Team's output and responds to each vector with one detection rule and one defensive control. The task is deliberately constrained: one answer per vector, not a laundry list.

That constraint matters. Forcing the Blue Team agent to commit to a single detection rule per vector surfaces whether the model understands which control actually closes the gap, or whether it's pattern-matching to a checklist. The 8b run exposed this; the 70b answered it.

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

---

## Hardware During Inference

The Z13 runs a unified memory architecture: one 128GB LPDDR5X pool shared between the CPU and GPU. AMD's driver carves a portion out as dedicated GPU-addressable memory; on this machine, that carve-out is approximately 64GB. Loading a 42GB model means the full weight set sits in that GPU-addressable portion with room left for the KV cache to grow during generation.

What that looked like during the run:

- GPU Compute: 100% (sawtooth pattern on the Compute 0 lane; characteristic of active transformer inference)
- Dedicated GPU Memory: 42.5 GB at model load, grew to 53.0 GB by end of run as the KV cache expanded
- System RAM: ~60 / 63.6 GB (94%)
- GPU Temperature: 81°C sustained (within spec for full-load inference on this chassis)
- CPU: ~10% (CrewAI orchestration only; GPU handled all inference)

The CPU staying near idle throughout confirmed that ROCm was actually handling inference rather than falling back to CPU. That was the thing I was watching for. On the first run with the 8b model, I couldn't be completely certain the GPU was doing the work; this run confirmed it, visibly, at 100% compute utilization.

---

## The Threat Scenario

**Target:** Cloud-hosted web application with user authentication  
**Red Team Task:** Identify three realistic attack vectors at technique-level specificity  
**Blue Team Task:** For each vector: one detection rule, one defensive control

---

## Red Team Output

### Vector 1: JWT Injection via Malformed Token Payload

**Technique:** Manipulate the JWT payload to include a valid username with an incorrect or absent password, exploiting weak or missing signature validation on the server side. An attacker sends crafted tokens where the header specifies `alg: none`, bypassing signature verification on servers that accept unsigned tokens.

**Attacker Gain:** Authenticated session access without valid credentials; the attacker can impersonate any user whose username is known, including administrative accounts.

### Vector 2: Stored XSS via Unvalidated Search Input

**Technique:** Inject persistent JavaScript into a search field that lacks server-side input validation. The script is stored in the application's database and executes in the browser of every subsequent user who loads the affected page or search results view.

**Attacker Gain:** Session cookie theft, account hijacking, and the ability to perform unauthorized actions on behalf of other users, including users with elevated privileges.

### Vector 3: SSRF via Misconfigured Proxy

**Technique:** Craft outbound requests through a misconfigured proxy layer to reach internal services, metadata endpoints (e.g., `169.254.169.254` on cloud platforms), or RFC-1918 address space that was never intended to be reachable from the application tier.

**Attacker Gain:** Access to internal services and databases; cloud instance metadata including IAM credentials; lateral movement into otherwise isolated network segments.

---

## Blue Team Output

### Response to Vector 1: JWT Injection

**Detection Rule:** Monitor authentication logs for failed login attempts exceeding 5 in 60 seconds from a single source IP. Separately, flag any inbound JWT where the header specifies `alg: none` or where the payload structure deviates from the expected schema.

**Defensive Control:** Enforce server-side JWT signature verification; reject any token that specifies `alg: none`. Implement rate limiting on all authentication endpoints. Use a vetted JWT library rather than a custom parser.

### Response to Vector 2: Stored XSS

**Detection Rule:** Monitor web application logs for search queries containing known XSS indicators: `<script>`, `javascript:`, `onload=`, `onerror=`. Route matches to the alert queue for analyst review; do not rely solely on WAF blocking.

**Defensive Control:** Input validation and output encoding on all user-supplied fields. Deploy a WAF ruleset targeting XSS patterns as a second layer, not the primary control. Content Security Policy headers reduce impact if injection occurs despite validation.

### Response to Vector 3: SSRF via Proxy

**Detection Rule:** Monitor proxy logs for outbound requests to non-whitelisted domains, any destination in RFC-1918 space (`10.x`, `172.16-31.x`, `192.168.x`), and cloud metadata IP ranges (`169.254.169.254`). Alert on anomalous routing patterns regardless of response code.

**Defensive Control:** Restrict the proxy to an allowlisted set of destinations only. Implement egress filtering at the network layer as a separate enforcement point, not dependent on the application proxy configuration. Segment the internal network to limit the blast radius if SSRF is successfully exploited.

---

## 8b vs 70b: What the Upgrade Actually Changed

| Dimension | llama3:8b (Run 1) | llama3.3:70b (Run 2) |
|-----------|-------------------|----------------------|
| Attack vector specificity | Category-level ("XSS attack," "brute force") | Technique-level: mechanism, payload path, specific attacker gain |
| JWT vector | Identified the class of attack | Specified `alg: none` bypass as the specific exploitation path |
| Blue Team defensive depth | Single control, surface-level recommendations | Layered controls with explicit reasoning about primary vs. secondary enforcement |
| SSRF response | "Fix the proxy configuration" | Egress filtering independent of the proxy; network segmentation; metadata endpoint coverage |
| KV cache growth | Minimal | ~10 GB growth (42.5 to 53 GB) indicating longer internal reasoning chains |
| Wall time per task | Faster | Slower per token; output depth justified the trade-off |

The SSRF Blue Team response is the clearest illustration of what changed. The 8b model answered at the layer of the misconfiguration: fix the proxy. The 70b model answered at the layer of the threat: assume the proxy can be bypassed or misconfigured again, and build controls that don't depend on it being correct. Egress filtering at the network layer plus segmentation is a layered defensive posture, not a config patch. That's the kind of answer that actually holds up under review.

---

## What I Learned

**Model scale changes the quality of reasoning, not just the length of output.** The 70b's answers weren't longer versions of the 8b's answers; they were structurally different. The 8b model identified threat categories and prescribed the most obvious single control. The 70b model reasoned about the threat model and prescribed controls that hold even if a simpler fix fails. That's a meaningful distinction in security work, where layered defense depends on not assuming any single control is sufficient.

**The KV cache growth confirms something about how the model thinks.** In the 70b run, the GPU's dedicated memory grew from 42.5 GB at model load to 53 GB by the end of the Blue Team task. That ~10 GB growth is the KV cache accumulating context across the generation. The model is holding more prior state in memory as it reasons through the response. The 8b run showed minimal cache growth. I don't want to overinterpret this, but the correlation with output quality is worth noting.

**ROCm is doing the work.** I confirmed GPU Compute at 100% on the Compute 0 lane with a sawtooth pattern that matches transformer token generation. CPU stayed at 10% throughout. On the first run I had some uncertainty about whether inference was actually GPU-accelerated; this run removed it. The unified memory architecture on the Z13 is genuinely capable of running a 42GB model fully in GPU-addressable memory without offloading, which is not something most machines can do without dedicated VRAM at that scale.

**Constrained task design surfaces model capability.** Requiring one detection rule and one defensive control per vector instead of an open-ended "how would you defend against this" forced the Blue Team agent to make a decision. That decision quality is what I was actually measuring. If I had asked for a list of recommendations, both models would have produced similar-looking output. The constraint made the depth difference visible.

---

## Up Next

- **Run 3:** Swap the Blue Team model to `qwen2.5-coder:7b` and evaluate whether it produces structured Sigma rule output vs. the natural language controls generated here. A model purpose-built for code and structured output may produce more immediately operationalizable detections.
- **Broader scenario coverage:** The cloud web app with authentication is a reasonable starting scenario but narrow. Expanding to ICS/OT, cloud-native misconfigurations, and supply chain scenarios would test whether technique-level specificity holds across different threat models.
- **Run logging automation:** Currently capturing run output manually. Building a lightweight logger that writes structured output to the vault on run completion is the obvious next step.

---

*Part of the [cloud-security-lab-notes](https://github.com/JNewbrey87/cloud-security-lab-notes) portfolio. Building toward Cloud Security Analyst, CySA+ CS0-003, SC-200, and SC-500. [linkedin.com/in/jnewbrey87](https://www.linkedin.com/in/jnewbrey87)*
