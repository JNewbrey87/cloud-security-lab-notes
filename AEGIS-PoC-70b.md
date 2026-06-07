# AEGIS PoC: Local Adversarial Simulation with llama3.3:70b

**Project:** AEGIS (Adversarial Execution and Generative Intelligence System)  
**Lab Type:** Local AI Red/Blue Team Simulation  
**Date:** 2026-06-07  
**Status:** Completed

---

## What This Demonstrates

A fully local, multi-agent adversarial security simulation running on consumer hardware with no cloud dependency. Two AI agents (Red Team and Blue Team) were orchestrated via CrewAI against a defined threat scenario; inference ran entirely on-device using ROCm compute acceleration.

This is a follow-up to the initial 8b PoC, upgrading to a 42GB parameter model to validate depth-of-output improvements at scale.

---

## Stack

| Component | Detail |
|-----------|--------|
| Model | llama3.3:70b (42 GB weights) |
| Inference Backend | ROCm (AMD Radeon 8060S, unified memory architecture) |
| Agent Framework | CrewAI 1.14.6 |
| Runtime | Python 3.12 / WSL Ubuntu |
| Ollama Endpoint | localhost:11434 |
| Tracing | Disabled (local use; no telemetry) |

---

## Hardware Metrics During Inference

- GPU Utilization: 100% (Compute lane, sawtooth pattern confirming active token generation)
- Dedicated GPU Memory: 53.0 / 63.8 GB (42.5 GB at model load; grew to 53 GB as KV cache expanded)
- System RAM: ~60 / 63.6 GB (94%)
- GPU Temperature: 81°C (sustained full-load, within operating spec)
- CPU: ~10% (orchestration only; GPU handled all inference)

The Z13's unified memory architecture allowed the full 42GB model to load directly into GPU-addressable VRAM without offloading to system RAM or disk. CPU stayed near idle throughout the run.

---

## Threat Scenario

**Target:** Cloud-hosted web application with user authentication  
**Red Team Task:** Identify three realistic attack vectors (technique-level specificity required)  
**Blue Team Task:** For each vector, produce one detection rule and one defensive control

---

## Red Team Output

### Vector 1: JWT Injection via Malformed Token Payload
- **Technique:** Manipulate the JWT payload to include a valid username with an incorrect or absent password, exploiting weak or missing signature validation on the server side
- **Attacker Gain:** Authenticated session access without valid credentials; further exploitation of session context

### Vector 2: Stored XSS via Unvalidated Search Input
- **Technique:** Inject persistent JavaScript into a search field lacking server-side input validation; the script executes for every subsequent user who loads the affected page
- **Attacker Gain:** Session cookie theft, account hijacking, unauthorized actions on behalf of other users

### Vector 3: SSRF via Misconfigured Proxy
- **Technique:** Craft requests through a misconfigured proxy layer to reach internal services or RFC-1918 address space not intended to be externally reachable
- **Attacker Gain:** Access to internal systems and databases; lateral movement potential; further vulnerability chaining

---

## Blue Team Output

### Vector 1 Response: JWT Injection
- **Detection Rule:** Monitor authentication logs for failed login attempts exceeding 5 attempts within 60 seconds from a single IP; flag JWT payloads with structural anomalies
- **Defensive Control:** Enforce server-side JWT signature verification and payload inspection; implement rate limiting on all authentication endpoints

### Vector 2 Response: Stored XSS
- **Detection Rule:** Monitor web application logs for search queries containing known XSS indicators: `<script>`, `javascript:`, `onload=`; route matches to alert queue
- **Defensive Control:** Input validation and output encoding on all user-supplied fields; deploy WAF ruleset targeting XSS patterns

### Vector 3 Response: SSRF via Proxy
- **Detection Rule:** Monitor proxy logs for outbound requests to non-whitelisted domains or any destination in RFC-1918 space; alert on anomalous routing patterns
- **Defensive Control:** Restrict proxy to allowlisted destinations only; implement egress filtering; segment the internal network to limit lateral movement on breach

---

## 8b vs 70b Comparison

| Dimension | llama3:8b | llama3.3:70b |
|-----------|-----------|--------------|
| Attack vector specificity | Category-level (e.g., "XSS attack") | Technique-level (mechanism, payload path, gain) |
| Blue Team defensive depth | Single control, surface-level | Layered controls; e.g., egress filtering + segmentation |
| SSRF response quality | "Fix the proxy config" | Egress filtering, segmentation, RFC-1918 monitoring |
| KV cache growth during run | Minimal | ~10 GB growth (42.5 to 53 GB) confirming longer reasoning chains |
| Wall time per task | Faster | Slower per token; depth justified the trade-off |

The 70b model's SSRF Blue Team response was the standout: recommending egress filtering layered with network segmentation rather than a single config-level fix. That reflects genuine depth, not just longer output.

---

## Notes

- Tracing defaulted to disabled after the 20-second prompt timeout; acceptable for local-only use
- To re-enable: set `CREWAI_TRACING_ENABLED=true` or pass `tracing=True` to the Crew constructor
- Next planned variation: replace the Blue Team model with `qwen2.5-coder:7b` to evaluate structured Sigma rule output vs. natural language controls

---

## Related Work

- AEGIS PoC: First Run Log (llama3:8b baseline)
- AEGIS Local Stack Setup
- AEGIS Component Architecture
