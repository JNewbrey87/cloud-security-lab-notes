# The GPU That Was Never On

**Project:** AEGIS (Adaptive Execution and Generative Intelligence System)  
**Type:** Local inference debugging note  
**Platform:** ASUS ROG Flow Z13 (Ryzen AI Max+ 395, Radeon 8060S / gfx1151, 128 GB unified), Ollama on Windows  
**Companion to:** [AEGIS-Evidence-Ledger.md](AEGIS-Evidence-Ledger.md)  

---

## The symptom

While setting up a clean benchmark run for my local Red/Blue agent project, I checked `ollama ps` and found llama3.3:70b running at `100% CPU`. The server log agreed: `total_vram="0 B"` and an empty `GPULayers:[]`. A 43 GB model was being inferenced entirely on the processor, and had been for an unknown period. Nothing had ever errored. The machine has a capable integrated GPU and 128 GB of unified memory, so this was close to the last thing I expected to find.

## Six wrong guesses

I did what most people do, which is treat it as a tuning problem and start turning knobs. In order, I was wrong about all of the following:

1. **Docker Desktop was holding memory.** Closing it changed nothing about the processor split.
2. **A model was left resident** by `OLLAMA_KEEP_ALIVE`. Unloading it did nothing.
3. **A version regression**, which argued for staying pinned rather than upgrading. This one actively pointed me away from the fix.
4. **An undersized VRAM carve-out.** The Armoury Crate "Dedicated Graphics Memory" figure reads 32 GB, and a 43 GB model does not fit in 32 GB, so this felt conclusive. It was wrong; ROCm draws on the wider unified pool and actually sees 89.4 GiB, so 32 GB was never the ceiling.
5. **A missing ROCm architecture override**, the `HSA_OVERRIDE_GFX_VERSION` variable people set for AMD cards.
6. **Vulkan configuration**, which I was advised on twice in opposite directions.

Every one of those was plausible. Every one was a guess about configuration. None of them was the problem.

## What actually settled it

I stopped guessing and listed the backend directory Ollama actually had installed:

```
%LOCALAPPDATA%\Programs\Ollama\lib\ollama
  mlx_cuda_v13
```

One backend. A CUDA backend, which is NVIDIA, on an AMD machine. No `rocm`, no `vulkan`, nothing that could touch a Radeon. The install had no AMD GPU support of any kind, so it could only ever have run on the CPU. Every knob I had been turning was downstream of a binary that was never present; there was nothing to tune, because the thing I was tuning did not exist.

## The fix

A clean reinstall (`irm https://ollama.com/install.ps1 | iex`) moved Ollama from 0.24.0 to 0.34.0, and the backend directory then held what it should: `cuda_v12`, `cuda_v13`, `rocm_v7_1`, and `vulkan`. I cleared the leftover `HSA_OVERRIDE_GFX_VERSION` and `OLLAMA_VULKAN` machine variables from my earlier guessing, since ROCm 7.1 supports this GPU's gfx1151 architecture natively and forcing an override to a different architecture would have been wrong in a new way. Pulled models survived the upgrade untouched. Both models now report `100% GPU`.

## Lesson one: inventory what is installed before tuning what is installed

The fix was a single directory listing, and it came after six configuration theories. If I had inventoried the backend before adjusting environment variables, I would have found it in the first five minutes. Configuration debugging assumes the component is present and misconfigured; I skipped the cheaper prior question of whether it was present at all.

## Lesson two, the one that matters for a security audience

A silent fallback looks exactly like success. The system produced correct output the entire time it was on the CPU; the only thing that differed was speed, and I had no baseline to compare against, so nothing looked wrong. This machine ran this same 70B model on the GPU months ago, when I first set it up; a June 2026 lab note of mine recorded it as "GPU-exclusive, insanely fast," and that may well have been accurate at the time. What I cannot pin down is when it regressed to CPU, most likely during an Ollama update that quietly changed the installed backend, because nothing announced the change and there was no baseline to catch it against. Nothing alerted, nothing errored, and the regression survived an unknown stretch of use.

This is the same failure mode I care about in detection engineering. If your only success signal is "it produced output," a component can be broken indefinitely and never tell you. The GPU case here is benign, but the pattern is not: a detection rule that never fires, a log pipeline that silently drops events, an agent that degrades to a weaker path. Each one keeps producing plausible output while the thing you actually wanted has quietly stopped happening. You find these by measuring against a known baseline, not by waiting for an error that never comes.

## Cosmetic artifacts, for anyone reproducing this

A few things looked like problems and were not, worth naming so they don't send you down the same rabbit holes:

- CrewAI reports a local Ollama failure as `OpenAI API call failed: Error code: 500`. Nothing contacts OpenAI; Ollama is routed through an OpenAI-compatible client class, and that class name is the entire reason the string says "OpenAI." Read the payload inside the braces, not the label.
- The live-render panel interleaves with plain `print()` output, so the console can look corrupted when it is fine.
- Raw ANSI escape codes occasionally leak through into the log.
- The line `dropping integrated GPU; to enable, set OLLAMA_IGPU_ENABLE=1` applies only to the Vulkan device entry. ROCm is enumerated separately and used, so that variable is not needed.

## What I am not claiming

I am not leaning on speed numbers here. Every run during the regression window was on the CPU, which makes that timing data void; a clean GPU run has since confirmed the fix, and a full two-agent crew run completes in a few minutes. The point of this note is the diagnosis, not the benchmark.

---

*Part of the [cloud-security-lab-notes](https://github.com/JNewbrey87/cloud-security-lab-notes) portfolio. AEGIS is personal research, not a production system.*
