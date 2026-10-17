# Silent CPU Fallback: Why My 70B Model Stopped Using the GPU

**Project:** AEGIS (Adaptive Execution and Generative Intelligence System)  
**Type:** Local inference debugging note  
**Platform:** ASUS ROG Flow Z13 (ZX-A3G15): Ryzen AI Max+ 395, Radeon 8060S (gfx1151), 128 GB unified memory, Ollama on Windows  
**Date:** September 2026

---

## Why I Did This One

While setting up a clean benchmark run for the [evidence ledger](02-AEGIS-Evidence-Ledger.md), I checked `ollama ps` and found llama3.3:70b running at `100% CPU`. The server log agreed: `total_vram="0 B"` and an empty `GPULayers:[]`. A 43 GB model was running entirely on the processor, and had been for an unknown stretch of time. Nothing had ever errored. This machine has a capable integrated GPU and 128 GB of unified memory, and the same model had run on that GPU back in June and July, so this was close to the last thing I expected to find.

## What I Tried: Six Wrong Guesses

I did what most people do: treated it as a tuning problem and started turning knobs. In order, I was wrong about all of these:

1. **Docker Desktop was holding memory.** Closing it changed nothing.
2. **A model was left resident** by `OLLAMA_KEEP_ALIVE`. Unloading it did nothing.
3. **A version regression**, which argued for staying on the current version instead of upgrading. This one actively pointed me away from the fix.
4. **The dedicated graphics memory setting was too small.** Armoury Crate showed 32 GB, and a 43 GB model doesn't fit in 32 GB, so this felt conclusive. It wasn't. On this machine, ROCm draws on the wider unified memory pool and actually sees 89.4 GiB, so the setting was never the ceiling.
5. **A missing ROCm architecture override**, the `HSA_OVERRIDE_GFX_VERSION` variable people set for AMD cards.
6. **Vulkan configuration**, which I got advice on twice, in opposite directions.

Every one of those was plausible. Every one was a guess about configuration. None of them was the problem.

## What Actually Settled It

I stopped guessing and listed the backend directory Ollama actually had installed:

```
%LOCALAPPDATA%\Programs\Ollama\lib\ollama
  mlx_cuda_v13
```

One backend, and it was CUDA, which is NVIDIA, on an AMD machine. No `rocm`, no `vulkan`, nothing that could use a Radeon. The install had no AMD GPU support at all, so it could only ever run on the CPU. Every setting I'd been changing was downstream of a component that wasn't there. There was nothing to tune.

## The Fix

A clean reinstall (`irm https://ollama.com/install.ps1 | iex`) moved Ollama from 0.24.0 to 0.34.0, and the backend directory then held what it should: `cuda_v12`, `cuda_v13`, `rocm_v7_1`, and `vulkan`. I cleared the `HSA_OVERRIDE_GFX_VERSION` and `OLLAMA_VULKAN` variables left over from my guessing, since ROCm 7.1 supports this GPU's gfx1151 architecture natively, and forcing an override to a different architecture would have been wrong in a whole new way. The models I'd already pulled survived the upgrade untouched, and both of the ones I was testing came back at `100% GPU`.

## Things That Looked Broken but Weren't

A few things looked like problems and weren't. Naming them here so they don't send anyone else down the same rabbit holes:

- **CrewAI reports a local Ollama failure as `OpenAI API call failed: Error code: 500`.** Nothing contacts OpenAI. Ollama is called through an OpenAI-compatible client class, and that class name is the only reason the message says "OpenAI." Read the payload inside the braces, not the label.
- **The live-render panel interleaves with plain `print()` output**, so the console can look corrupted when it's fine.
- **Raw ANSI escape codes** occasionally leak into the log.
- **The line `dropping integrated GPU; to enable, set OLLAMA_IGPU_ENABLE=1`** only applies to the Vulkan device entry. ROCm is enumerated separately and gets used anyway, so that variable isn't needed.

## What I Took Away

**Check what's installed before tuning what's installed.** The fix was one directory listing, and it came after six configuration theories. If I'd checked the backend before changing environment variables, I'd have found it in the first five minutes. Configuration debugging assumes the component is there and misconfigured; I skipped the cheaper question of whether it was there at all.

**A silent fallback looks exactly like success.** The system produced correct output the whole time it was on the CPU. The only difference was speed, and I had no timing baseline to compare against, so nothing looked wrong. My June run log recorded the GPU doing the work, and a July session showed the same. What I can't pin down is when or why it changed. The install ended up with a CUDA-only backend on an AMD machine, and I don't have solid evidence for how it got that way, so I'm not going to guess. Nothing announced the change, nothing alerted, and it went unnoticed for an unknown stretch of time.

**It's the same failure mode I care about in detection engineering.** If your only success signal is "it produced output," a component can be broken indefinitely and never tell you. The GPU case is harmless, but the pattern isn't: a detection rule that never fires, a log pipeline that silently drops events, an agent that quietly degrades to a weaker path. Each one keeps producing plausible output while the thing you actually wanted has stopped happening. You catch these by measuring against a known baseline, not by waiting for an error that never comes. For local inference, the cheap habit that would have caught this is checking the PROCESSOR column in `ollama ps` and `total_vram` in the server log before trusting any impression of performance.

**A regression is harder to catch than something that never worked.** Because the GPU had worked before, I had no reason to suspect it. Slow runs just read as "the 70b is a heavy model."

## What I'm Not Claiming

I'm not leaning on speed numbers here. The runs during the fallback window were on the CPU, so their timing data is void. A clean GPU run has since confirmed the fix, and a full two-agent crew run now finishes in a few minutes. The point of this note is the diagnosis, not the benchmark.

## Lab Environment

| Component | Details |
|-----------|---------|
| Host | ASUS ROG Flow Z13 (ZX-A3G15): Ryzen AI Max+ 395, Radeon 8060S (gfx1151), 128 GB unified memory |
| Inference | Ollama on Windows, 0.24.0 before the fix, 0.34.0 after |
| GPU backend | ROCm 7.1 (`rocm_v7_1`), restored by the reinstall |
| Diagnostics | `ollama ps` PROCESSOR column, server log `total_vram` and `GPULayers`, backend directory listing |

## What Came Next

- With the GPU path restored, the final [evidence ledger](02-AEGIS-Evidence-Ledger.md) run finished in a few minutes on the GPU, the first run where I recorded a timing figure.

Back to the start: [aegis-00](00-AEGIS-PoC-70b.md).

---

*Part of the [cloud-security-lab-notes](https://github.com/JNewbrey87/cloud-security-lab-notes) portfolio, building toward Cloud Security Analyst and the CySA+ / SC-200 / SC-500 certifications.*
