# Scoping: llama.cpp as an alternative to LiteRT-LM

Status: **Step 1 (conversion + quality/speed validation) done and passed convincingly, August 2,
2026. iOS Swift integration not started.** This document exists to capture the reasoning and the
concrete steps so a future session can pick this up without re-deriving context.

⚠️ **Likely unnecessary — read `Aquinas_Backend/CLAUDE.md`'s "LiteRT-LM iOS conversion" section
(August 3, 2026 entries) before continuing this migration.** This document is premised on LiteRT-LM
having a real GPU quality ceiling on-device. That premise no longer holds: both the higher-
precision `8fc4emb` candidate and a stock Gemma 4 package that failed on the iOS *Simulator*'s GPU
delegate were confirmed working cleanly on GPU on a real physical iPhone 17 (3.86 GB / 9.3 s cold
load / 2.44 s generation for the candidate — roughly 2x the current package, not a regression).
The original rejection was based entirely on a Simulator-only artifact. Remaining before this
candidate can actually ship: sustained-memory/thermal validation across a real multi-turn
conversation and a proper blind answer-quality comparison — see the backend doc for the exact gate
list. That's a validation task, not a runtime migration. Don't start the work below until that
path is confirmed closed out or explicitly abandoned.

## Step 1 result: conversion validated, quality confirmed

The fine-tuned checkpoint (`models/Aquinas-Final-HF`) was converted to GGUF and quantized at three
precision levels using a new `Aquinas_Backend/tools/llama.cpp` checkout (Homebrew's `llama.cpp`
formula ships the compiled `llama-cli`/`llama-quantize` binaries but not the Python conversion
scripts, so the checkout is needed for `convert_hf_to_gguf.py`) and a new
`Aquinas_Backend/gguf_conversion_env` venv.

- `models/Aquinas-Final-GGUF/aquinas-f16.gguf` — 9.3GB base conversion, unquantized.
- `aquinas-Q5_K_M.gguf` — 3.45GB. `aquinas-Q6_K.gguf` — 3.65GB. `aquinas-Q8_0.gguf` — 4.72GB.
- One real conversion bug hit and fixed: `transformers==4.57.6` (pinned by llama.cpp's own
  requirements) chokes on this checkpoint's `tokenizer_config.json` — `extra_special_tokens` is
  stored as a list (`["<|video|>"]`) but that transformers version's
  `_set_model_specific_special_tokens` requires a dict. Fixed by converting from a **staged copy**
  (`models/Aquinas-Final-HF-gguf-stage`, symlinked large files + patched
  `tokenizer_config.json` with `extra_special_tokens: {"video_token": "<|video|>"}`) — the
  canonical checkpoint was never modified. Re-apply this same staging step for any future
  conversion until either the checkpoint's tokenizer config or the pinned transformers version
  changes.

**Quality test** (Q6_K, on Mac via `llama-cli -ngl 99`, manually-formatted prompt — see note below):
asked the two questions that hallucinated in the live iOS app tonight.
- *"What does doxology mean?"* → *"Doxology is an act of praising or glorifying someone or
  something. In a religious context, it is an act of worship, thanksgiving, or praise directed to
  God."* Correct. (App's answer: confused it with a council/gathering of deputations — wrong.)
- *"What was the Council of Trent?"* → correct dates (1545–1563), correct purpose (Counter-
  Reformation response, doctrinal reaffirmation, clerical reform), well-structured. (App's answer
  that night: gave the *doxology* answer for this question too — cross-contaminated.)

**Speed**: 83–84 tokens/sec generation, 148–156 tokens/sec prompt processing, on Metal GPU
(`-ngl 99`) on the development Mac. This is a Mac GPU number, not an iPhone number — re-measure on
the base supported iPhone before drawing conclusions about on-device speed, but it confirms the
GPU path itself works end-to-end with no crash, no CPU fallback, and no quality compromise.

**Important gotcha for future testing**: `llama-cli`'s automatic HF chat-template application
(the default when no `-no-cnv` is passed) produced **zero generated tokens** against this
checkpoint's real, complex Jinja chat template (tool-calling macros, `strip_thinking` filter, the
project's custom `<|turn>`/`<turn|>` role markers) — likely a template-engine compatibility gap
between llama.cpp's minimal built-in Jinja implementation (`minja`) and whatever the checkpoint's
template actually needs. **This does not block the real iOS integration**: `LiteRTAquinasModel.swift`
never relies on automatic chat-template application today either — it already manually constructs
prompt strings with explicit role markers. The working test above used the same approach: a
manually-built prompt (`<|turn>system\n...<turn|>\n<|turn>user\n...<turn|>\n<|turn>model\n`, using
the project's actual `sot_token`/`eot_token` values from `tokenizer_config.json`) with `-no-cnv` and
`-r "<turn|>"` as a stop sequence. Use this manual-formatting approach for any further CLI testing,
and build prompts manually in the Swift integration exactly as `LiteRTAquinasModel.swift` does now.

**CLI harness quirk, not a model issue**: without `-st`/`--single-turn` (which did not reliably
prevent it) and with stdin not explicitly closed, `llama-cli` drops into an idle interactive loop
after generation completes, printing empty `> ` prompts indefinitely. Always redirect
`< /dev/null` and watch for the `[ Prompt: ... | Generation: ... ]` summary line, then kill the
process — don't let it run unattended in the background.

**Housekeeping for next session**: disk is down to ~14GB free after this (the F16 base plus three
quantized variants total ~21GB). The F16 base (`aquinas-f16.gguf`, 9.3GB) can be deleted once a
precision level is chosen and re-derived from it isn't expected soon — it's only needed as the
source for further `llama-quantize` runs, not for actually running the model.

## Backend decision: llama.cpp over MLX Swift

Both were seriously considered. MLX Swift was the initial lean (first-party Apple framework, nicer
Swift API, and the framework LM Studio used for the quality comparison that started this
investigation) but was ruled out for now on a concrete blocker, not a preference: **the `gemma4`
architecture is not registered in the official `mlx-swift-lm` model loader as of August 2, 2026**
(open issues `ml-explore/mlx-swift-lm#282` and `ml-explore/mlx-swift#389`). The Python side
(`mlx-lm`) supports Gemma 4 and quantized weights already exist on Hugging Face
(`mlx-community/gemma-4-*`), but the Swift inference code that actually runs the architecture
(per-layer embeddings, shared KV cache, dual RoPE) doesn't exist in the official library. A
community "sidecar" Swift port of the text decoder exists as a workaround, but building on an
unofficial third-party port for the actual core of the app was judged too risky compared to
llama.cpp, where Gemma 4 E2B support already lives in the shared C++ core (`ggml`) that every
language binding, including Swift, calls into generically — no separate per-language architecture
port needed. GGUF conversions of Gemma 4 E2B are already published by `ggml-org` and have worked
since shortly after the model's April 2026 launch.

This is a moving-target situation: if `mlx-swift-lm` gains official Gemma 4 support later, MLX
Swift is worth reconsidering — it remains the more natively-Apple, better-backed framework
long-term. Re-check `ml-explore/mlx-swift-lm` issue #282 before assuming this section is still
accurate.

A benchmark found during this investigation claims LiteRT-LM is the fastest runtime specifically
for Gemma models on iPhone — but that comparison is GPU-vs-GPU at matched precision, and it isn't
the comparison that applies here. The actual choice is higher-precision quality on LiteRT-LM's
**CPU only** (its GPU delegate cannot run that precision at all, confirmed twice tonight) versus
higher-precision quality on llama.cpp's **GPU**. That's a GPU-vs-CPU gap, not a GPU-vs-GPU one —
GPU-accelerated inference is typically 5-20x+ faster than CPU for the same model, which should
swamp any small runtime-efficiency edge LiteRT-LM has at a precision it can actually accelerate.
Worth confirming with a real measurement once the GGUF conversion exists, but there's no reason to
expect llama.cpp-on-GPU to lose to LiteRT-LM-on-CPU.

## Why this is on the table

As of August 2, 2026, `Aquinas-iOS` runs entirely on Google's LiteRT-LM, using the bundled
`Aquinas-Final-LiteRT` package (4-bit dynamic-weight decoder, `dynamic_wi4_afp32`). That package
is fast and GPU-accelerated, but answer quality on real-world factual questions is noticeably weak
— confirmed by a same-size (`gemma-4-E2B`) comparison in LM Studio (MLX), which answered the same
questions correctly and in depth where the on-device app hallucinated.

The obvious next step — re-export the fine-tuned checkpoint at higher precision
(`dynamic_wi8_emb4_afp32`, the `8fc4emb` candidate) — was already tried by this project and is
rejected. It fails to initialize on iPhone/simulator GPU with:

```
Failed to create DelegateKernelLiteRtMetal: UNAVAILABLE: Current device can not allocate tensor
with this shape for predefined/external descriptor: Requested allocation size - 402653184 bytes.
Max allocation size for this GPU - 268435456 bytes.
```

This is a **256MB single Metal-texture allocation ceiling** inside LiteRT-LM's own TFLite GPU
delegate (`ml_drift`/`delegate_metal.mm`), which represents that tensor as a Metal *texture*
rather than a plain buffer. Metal buffers have a far higher size ceiling than Metal textures on the
same hardware — so this is specifically an implementation choice inside Google's delegate code, not
a fundamental Metal/hardware limit. The plain (non-fine-tuned) stock `gemma-4-E2B-it` package also
fails on GPU, for an unrelated reason (`texture binding has argument index 31 that is greater than
30`) — a second, independent Metal-delegate limitation in the same LiteRT-LM stack. Both packages
load and generate correctly on CPU, confirming the weights themselves are sound; only LiteRT-LM's
GPU delegate is the blocker.

The current LiteRT-LM exporter also only supports three fixed quantization recipes
(`dynamic_wi4_afp32`, `dynamic_wi8_emb4_afp32`, `dynamic_wi8_afp32`) — there is no 6-bit or custom
middle ground, and the untested full-8-bit recipe would almost certainly make the oversized-tensor
problem worse, not better, since it would enlarge the exact tensor that's already over the ceiling.

llama.cpp is mature, widely deployed for on-device LLM inference on iOS specifically, and its
Metal GPU tensor handling is not known to hit this class of texture-allocation bug — it generally
uses plain Metal buffers for large tensors rather than textures. That's a credible, evidence-based
reason to expect this specific problem to simply not exist on this runtime, not just optimism.

## What would actually be required

This is a genuine rewrite of the on-device model boundary, not a configuration change. Rough scope:

### 1. New conversion/export pipeline (`Aquinas_Backend`) — DONE, see Step 1 result above
- Base F16 conversion and Q5_K_M/Q6_K/Q8_0 quantization exist in `models/Aquinas-Final-GGUF/`,
  produced via `tools/llama.cpp/convert_hf_to_gguf.py` + `llama-quantize`, using the
  `gguf_conversion_env` venv and the staged-checkpoint workaround documented above.
- Still needed: turn the manual steps above into a real `scripts/export_gguf_aquinas.py` alongside
  the existing LiteRT exporter (don't replace it until the new path is proven), and pick a single
  precision level once iPhone-side validation (not just tonight's Mac numbers) is done.
- Remaining validation gates before picking a precision: cold-load time, memory, and tokens/sec
  measured on the base supported iPhone specifically (tonight's numbers are Mac GPU only).

### 2. New iOS integration layer (`Aquinas-iOS`)
Everything built and hardened tonight in `LiteRTAquinasRuntime.swift` is written against
LiteRT-LM's specific Swift API (`Engine`, `Conversation`, `sendMessageStream`, `SamplerConfig`,
etc.) and would need an equivalent rewritten against llama.cpp's API:
- No official first-party Swift package exists — this means either wrapping `llama.cpp`'s C API
  directly (a bridging header + a hand-written Swift wrapper around `llama_context`,
  `llama_decode`, streaming token callbacks, etc.), or adopting a community Swift wrapper (e.g.
  `LocalLLMClient` or similar — vet whichever is current at implementation time for maintenance
  activity and license before depending on it for the app's core).
- Session/generation lifecycle: streaming responses, cancellation, and the
  `ModelRuntimeLifecycleManager` load/idle-unload state machine all need re-implementing against
  the new framework's actual load/generate/unload primitives.
- The repetition and mixed-script corruption guards (`LiteRTGenerationGuard`) are framework-agnostic
  text processing and should port with minimal change.
- The entitlements fix from tonight (Increased Memory Limit, Extended Virtual Addressing) is
  general iOS memory/address-space configuration, not LiteRT-specific, and should carry over
  regardless of runtime.

### 3. What ports over largely unchanged
- All prompt/system-instruction construction (`conversationSystemInstruction`, grounding injection,
  personality instructions, key-term extraction prompt, definition prompts).
- `AquinasGroundingProviding` and the grounding bootstrap corpus.
- `ModelTaskQueue`, the task UI, and everything above the `AquinasModel` protocol boundary — the
  app only needs a new `AquinasModel`-conforming type (a `LlamaCppAquinasModel` or similar) wired
  into `AquinasApplicationRuntime`, mirroring how `LiteRTAquinasModel` is wired in today.

### 4. Re-validation
Every gate this project already applies to a new LiteRT package (cold load, sustained latency,
memory, thermal, blind answer quality on the base supported iPhone) applies again — this is a new
model-delivery path, not a drop-in swap. Additionally measure tokens/sec against the current
LiteRT-LM baseline specifically, given the open question above about LiteRT-LM's raw-speed
advantage for Gemma models.

## Recommendation

Worth pursuing as a deliberate, scoped project — not an emergency fix, since the current LiteRT-LM
path is stable as of tonight's crash/freeze fix. The technical case for llama.cpp clearing the
GPU-quality ceiling is credible: Gemma 4 E2B is already officially supported in its shared C++ core
(unlike MLX Swift's current architecture gap), its Metal GPU path doesn't share LiteRT-LM's
texture-allocation implementation choice, and GGUF's quantization ecosystem gives far more precision
granularity to tune than LiteRT-LM's three fixed recipes. The cost is real: no first-party Swift
package, so the integration layer needs either a hand-written C-API wrapper or a vetted community
one, plus a full rewrite of the on-device runtime layer, needing its own validation pass before it
could replace the current shipping path.

The cheap first step (convert + quantize + check quality and speed before writing Swift code) is
done and passed clearly — see "Step 1 result" above. Next cheap step, still before any Swift work:
get these same GGUF files onto the base supported iPhone (e.g. via a minimal test harness, or even
just confirming `llama-cli`'s iOS build story) and re-measure cold-load time and tokens/sec there,
since tonight's numbers are Mac GPU only. Only after that should the Swift integration layer
(section 2 above) start.
