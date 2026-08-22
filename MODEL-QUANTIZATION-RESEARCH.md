# Model Quantization Research

> Status: exploratory research, not implementation status. This document tracks the open question
> of whether a larger base model, quantized further down, can beat the currently shipped fine-tuned
> checkpoint on-device — and what it would take to find out. Nothing here has been promoted into
> `AquinasApplicationRuntime`; see [MODEL-INTEGRATION.md](MODEL-INTEGRATION.md) for what is actually
> live. Rejected-candidate byte/hash specifics for prior probes live in
> `../Aquinas-iOS/CLAUDE.md`; this document tracks the current line of inquiry rather than
> duplicating that history.

## The question

Public quantization work gets very large models (30B+ parameters) down to a few gigabytes — small
enough that they sound competitive with the current shipped Aquinas checkpoint (2.72 GB, 4-bit,
~3–4B-parameter class). Is a bigger base model, quantized harder, actually a better on-device
option than the current small model at a gentler quantization?

## How extreme quantization actually works

The techniques that get large models down to 2-bit are not naive rounding. They rely on
calibration: AWQ and GPTQ profile which weights are most sensitive to error and protect them (mixed
precision rather than uniform), and GGUF's k-quant formats (`Q2_K` through `Q8_0`) group weights and
use a calibration pass (an "importance matrix," or imatrix) to decide which groups can tolerate more
compression. This is why 2-bit quantization of a large model is usable at all — a flat 2-bit round of
every weight would not be.

## The bigger-vs-smaller tradeoff has a known breakpoint

For a fixed memory budget, more parameters at lower precision generally beats fewer parameters at
higher precision — but only down to roughly **4-bit**. Below that (3-bit, and especially 2-bit),
quantization error starts dominating and the relationship can reverse: a smaller model at 4/5-bit
can outperform a larger model crushed to 2-bit. Recent 2-bit Qwen benchmarks (imatrix-calibrated,
and newer methods like GSQ) show real quality recovery versus naive 2-bit, but still call out
measurable degradation versus 4-bit. This is the regime the "bigger, more quantized" theory
actually lives in, and it is exactly the regime where the tradeoff is least settled — it has to be
measured for a given model and use case, not assumed.

## Runtime ceiling: LiteRT-LM tops out around INT4

The app's production runtime (`LiteRTAquinasRuntime`, via the vendored LiteRT-LM Swift wrapper) has
no built-in path below INT4. Community `.litertlm` conversions confirm INT4 is the practical floor
of that toolchain today; MediaPipe's older LLM Inference API is in the same position and is now
maintenance-only, pointed at LiteRT-LM.

Going below INT4 (GGUF `Q2_K`/`IQ2`/`IQ1` quant levels) means llama.cpp, which does support that
range with a mature Metal backend on iOS. That is a real architectural fork — a second inference
runtime alongside the existing vendored LiteRT-LM `Engine.swift`/`Conversation.swift` wrapper and
its documented native-deadlock workaround (see `../Aquinas-iOS/CLAUDE.md`) — not a drop-in swap. It
has not been attempted; it remains the fallback path if staying on LiteRT-LM turns out to be a dead
end.

## Candidates surveyed

| Candidate | Format / runtime | Size | Status |
| --- | --- | --- | --- |
| Qwen3-4B mixed-INT4 | LiteRT-LM `.litertlm` | 2.66 GB | **Rejected** (prior probe): failed simulator Metal init, a 388,956,160-byte tensor exceeded the 256 MB per-allocation limit, base iPhone 17 killed with signal 9 during initialization. Full detail in `../Aquinas-iOS/CLAUDE.md`. |
| Qwen (family), GGUF `Q2_K` | llama.cpp, would need the runtime fork above | ~varies by size | Not attempted. Quality at this bit-width is the least proven of the options here — see breakpoint discussion above. |
| Gemma 4 26B-A4B (MoE, 3.8B active) | GGUF only — no `litert-community` LiteRT-LM build found | 14.2–16.9 GB (`Q4_K_XL`/`Q4_K_M`) | **Ruled out.** MoE reduces active *compute* per token, not resident *memory* — all 26B total parameters must still be loaded. At 14+ GB this doesn't fit an 8 GB device's budget even before accounting for OS/app overhead, and reaching a size that would fit means quantizing into the least-proven 2-bit-class range on top of an unproven architecture for this stack. It also has no native LiteRT-LM build, so pursuing it would additionally force the llama.cpp fork. Two compounding risks on one candidate. |
| Gemma 4 12B (dense) | LiteRT-LM `.litertlm`, native — `litert-community/gemma-4-12B-it-litert-lm` | 6.55 GB (`gemma-4-12B-it.litertlm`, CPU) / 5.99 GB (`gemma-4-12B-it-gpu.litertlm`) | **Current probe candidate.** Native LiteRT-LM build exists; no runtime fork needed. |

The Gemma 4 12B dense model is the only candidate that both plausibly fits an 8 GB device and stays
entirely inside the existing shipped runtime, so it is the current target for a feasibility probe —
the same kind of pass/fail check already run against the current 2.72 GB checkpoint and the rejected
Qwen3-4B candidate.

### Expected cost, not yet measured

Dense 12B at the same 4-bit precision as the current checkpoint means roughly 3x the active weight
bytes streamed per generated token versus the current ~3–4B model — on-device decoding is generally
memory-bandwidth-bound, so this is expected to cost real latency, not just load time. Against the
current checkpoint's measured baseline (2.72 GB / 4.33 s cold load / 1.27 s one-sentence generation
on a base iPhone 17, 8 GB RAM), the working estimate going in is a cold load in the high single
digits of seconds and generation in the 3–4x range for comparable output — unverified until probed.

## Next step

`../Aquinas-iOS/Features/Developer/LiteRTDeviceProbe.swift` already supports pointing the existing
diagnostic probe at an arbitrary external model file — no code change is required. Run:

```
--litert-probe --litert-probe-auto --litert-model-path /absolute/path/to/gemma-4-12B-it-gpu.litertlm
```

against a Debug build on a base iPhone (or the arm64 iOS Simulator first) and record cold-load time,
generation time, and stability against the baseline above. This requires a local macOS environment
with Xcode and either the simulator or a connected device — it cannot be run from a cloud/remote
session without a macOS toolchain.

Per the existing gate in `../Aquinas-iOS/CLAUDE.md`: never promote a candidate before simulator and
base-iPhone load, latency, memory, stability, and blind answer-quality checks all pass. This probe
is the load/latency/memory/stability half of that gate; blind answer-quality comparison against the
current fine-tuned checkpoint's voice and factual behavior is a separate, later step, and this
stock (non-fine-tuned) Gemma 4 12B checkpoint has none of the current checkpoint's Aquinas-specific
fine-tuning regardless of how the performance numbers land.
