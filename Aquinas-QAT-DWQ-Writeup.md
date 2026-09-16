# Aquinas Model: Quantization-Aware Training (DWQ) — Session Writeup

**Dates:** August 10–13, 2026
**Goal:** Record the original 4-bit DWQ experiment and the later controlled attempt to apply the
same mechanism to the shipping 8-bit-decoder/4-bit-embedding package on this M4 Pro/24 GB Mac.

---

## TL;DR (final status through Aug 13)

**The Aug 11 circular-definition/repetition alarm was caused by the wrong debug probe, not a DWQ
quality defect. However, that successful 4-bit experiment targeted an artifact production had
already replaced. The later retry against the shipping 8-bit-decoder/4-bit-embedding scheme produced
no valid checkpoint on this 24 GB Mac: runs encountered non-finite gradients, validation regression,
or Metal OOM. A final controlled retry also found nondeterministic validation, preventing trustworthy
checkpoint/resume verification. Production remains unchanged, and further DWQ work on the current
shipping 8-bit checkpoint is **not recommended** because its quantization error is already tiny and
the likely product benefit does not justify the remaining research and hardware cost.**

The properly-trained (512-sample) DWQ correction exported and device-tested cleanly, but kept producing circular or repetitive answers — and so did the unmodified production baseline, often worse (one run looped the same clause for 65 seconds). The suspect shifted a few times (undertraining, then the base model/training data) before the real cause turned up: the debug probe used all night (`--litert-probe`) sends a bare, out-of-distribution question with a stripped-down system prompt, bypassing the real production system prompt, which already has an explicit instruction against exactly this failure mode. Retesting through the actual production path (`--litert-probe --litert-quality-probe`, which exercises the real `LiteRTAquinasModel.respond` code path) made the problem disappear entirely for both models — coherent answers, no repetition, and DWQ came out consistently faster with zero quality cost (on one prompt, byte-identical output to the baseline, just quicker, consistent with this app's deterministic decoding).

Net result: the 4-bit DWQ mechanism and bridge worked end to end and showed a modest speed edge in
that historical comparison, but it was not a comparison against the shipping 8-bit package. The
8-bit retry has not produced a safe or deployable artifact. This document preserves both the
investigation and the authoritative controlled-retry protocol at its end.

---

## Background: why this started

You asked whether "Gemma 4 quantization-aware training" could make the on-device model faster and/or smarter. Worked out over the conversation:

- **QAT (quantization-aware training)** simulates low-precision rounding *during* training, so the model learns weights that survive quantization better than a model quantized only *after* training (post-training quantization, PTQ — what the current shipped pipeline does).
- It doesn't inherently make inference *faster* at a fixed bit-width — it makes the model *smarter per bit*: better quality at the same 4-bit size, or comparable quality at a smaller/faster size.
- Doing this "for real" (retraining a QAT-native Gemma 4 checkpoint from scratch) would have been a multi-week, high-risk undertaking requiring new training infrastructure.

The breakthrough: your existing MLX toolchain already ships a lighter-weight, laptop-feasible version of this idea — **DWQ (distillation-based weight quantization)**, built into `mlx-lm` (already installed, v0.31.3). Instead of retraining the whole model, DWQ:
1. Quantizes the model to 4-bit.
2. Unfreezes *only* the quantization scales/biases (~3% of total parameters for this model).
3. Trains those against a KL-divergence loss vs. the full-precision model's outputs, using your own theology training data.

Much cheaper than full QAT (minutes to an hour, not weeks) because it's correcting quantization error, not re-deriving the whole model.

---

## Part 1: proving the mechanism (first pass, smoke-test scale)

### Found and fixed the real training pipeline
- Confirmed your current fine-tune uses MLX LoRA (`lora_config_v4.yaml`) on `google/gemma-4-E2B-it`, fused, then exported via `litert_torch` (Google's ai-edge-torch toolkit) with PTQ (`dynamic_wi4_afp32`) — no QAT anywhere in the current path.
- Found DWQ already present in `mlx_lm.quant.dwq`. Discovered the real CLI entry point is the installed `mlx_lm.dwq` command, not `python -m mlx_lm.quant.dwq` (which silently no-ops — that file has no `__main__` guard).
- Installed the missing `datasets` package DWQ's data loader needs.

### Ran a 16-sample DWQ smoke test
- ~2 minutes wall time. Validation loss dropped 0.246 → 0.147.
- Had to load the model with `strict=False`: this checkpoint's config declares `num_kv_shared_layers=20` (later transformer layers reuse earlier layers' key/value projections), and `mlx-lm`'s Gemma4 implementation never materializes those redundant weight tensors — but the real HF checkpoint still stores them, so a strict load fails.

### Built a bridge from MLX's format back to a deployable HF checkpoint
DWQ's output lives entirely in MLX's own quantized tensor format, which your production export tool (`transformers.AutoModel.from_pretrained` → `litert_torch`) doesn't understand — it errored trying to interpret MLX's quantization config as an HF scheme.

Wrote `Aquinas_Backend/scripts/dwq_bridge_reconstruct.py`, which:
1. Dequantizes DWQ's corrected weights back to dense bf16.
2. Renames `mlx-lm`'s internal key naming (`language_model.model.X`) to the checkpoint's real naming (`model.language_model.X`).
3. Splices back in ~60 shared-KV-layer tensors mlx-lm never loaded (copied verbatim — DWQ never touched them).
4. Splices back in the vision/audio towers verbatim (mlx-lm's Gemma4 support is text-only).
5. Verifies via assertion that the final tensor set exactly matches the original checkpoint's **2011 keys, 1:1**, before writing anything.

### Exported and tested on real hardware
After working through a multi-attempt disk-space saga (documented in the original version of this writeup — short version: this Mac started the night at 12GB free out of 460GB, and the export process needs 40GB+ of scratch space), the smoke-test-scale package exported successfully: **2,722,385,120 bytes — the exact same size as the 4-bit package used as the baseline in that session.**

Simulator testing hit a real, pre-existing native-library deadlock (confirmed via a `sample` stack trace to be inside `CLiteRTLM`'s own worker threads, unrelated to our changes — your *existing* shipped package hit the identical hang under identical simulator conditions). We pivoted to your physical iPhone ("Ry") instead:

- Built and signed for device, pushed the model file into the app's sandboxed Documents folder via `devicectl`, launched the probe.
- **Cold load: 4.21s. Generation: 1.17s.** Matches (fractionally beats) your documented production baseline (4.33s / 1.27s) on the same hardware.
- **But the generated text was bad**: *"Prudence is the notion of prudence in action, specifically referring to the prudent person's conduct and decisions."* — circular, low-information, a real quality regression.

**Diagnosis:** 16 training samples is far too few to generalize (it's ~0.3% of your 5,517-example corpus). It can reduce loss on that tiny slice without actually correcting quantization broadly, which shows up as exactly this kind of degenerate phrasing. This is precisely why blind answer-quality checks matter — the size/speed/structural signals told us nothing about this.

---

## Part 2: doing it properly (512 samples)

### Two real bugs found and fixed along the way
1. **Memory thrashing at `batch_size=4`.** First attempt at scale used the CLI's default batch size with `max_seq_length=1025`; system swap ballooned and step time went from ~3s to 250+ seconds (projected 11+ hour completion). Diagnosed via `vm.swapusage` (3.4GB swapped, ~78MB RAM free) and fixed by dropping to `batch_size=1`.
2. **NaN divergence.** With `batch_size=1` restored, training ran cleanly until step ~237, where a single degenerate training example produced a NaN loss that got permanently baked into the Adam optimizer's internal moving-average state — corrupting every step after it, even though the individual steps were otherwise fine. Fixed by patching `mlx_lm/quant/dwq.py`'s `step()` function to skip the parameter update (not just skip logging) whenever the loss isn't finite, so one bad example can't poison the whole run.

### Real result
With the guard in place, 512-sample training completed cleanly in ~24 minutes:
- Initial validation loss: 0.258
- Mid-training checkpoint (step 400): 0.049
- **Final validation loss: 0.048**

That's roughly 5x more correction than the smoke test achieved, using real data instead of a tiny slice — a legitimate, substantial improvement.

### The export saga (unresolved tonight)
Bridging this 512-sample result through to a deployable package hit the same disk-space wall as before, compounded by a new issue:

- Four export attempts were killed tonight. The first three died from straightforward disk exhaustion at the very last stage (vision encoder compilation/bundling) — each time, the safety monitor caught it *right* before the disk would have hit 0, with clean recovery every time.
- **Root cause of the repeated near-misses**: killed export attempts leave their temp files behind (cleanup only runs on a successful finish). We found **43GB of leftover junk** from two earlier kills silently eating our "free" headroom on a later retry — clearing it was the single biggest fix of the night.
- After clearing that and freeing other real headroom (Trash, two already-rejected model candidates, an abandoned 20GB GGUF export path, Xcode's DerivedData and iOS DeviceSupport caches, and — with your explicit sign-off — the currently-shipped `Aquinas-Final-LiteRT` package, which is regenerable from the untouched `Aquinas-Final-HF` source), the fourth attempt got much further with ~70GB free.
- It ultimately died differently: the vision-adapter compilation step is memory-hungry, and **system swap climbed to 42GB out of a 43GB cap** while disk dropped in parallel — a compounding, harder-to-predict failure mode than plain disk exhaustion. I killed it deliberately rather than risk a messier crash, and both disk and swap recovered cleanly immediately after.

At that point — a new failure mode, several attempts in, and you flagging you were getting tired — we agreed to stop for the night rather than push a fifth attempt.

---

## Part 3: the export finally completes (Aug 11 morning)

Picked back up fresh: 71GB free, no leftover junk, phone plugged in and paired.

### Export succeeded on the second attempt of the morning
- **First attempt**: disk and swap climbed together during the vision-adapter compile step (the same spot that killed all four attempts the night before). This time I killed it *proactively* around 17GB free / 33GB swap, before hitting either hard threshold — a judgment call, not a forced kill.
- **In hindsight, that kill was probably premature.** Disk was declining gradually, not accelerating toward zero, and every prior run had shown this exact stage recovering hard once quantization kicks in afterward. For that historical retry only, disk-hitting-zero became the sole hard kill trigger and swap was logged rather than used independently. **That was not a safe general policy and is superseded by the final protocol's predeclared disk, swap, memory-pressure, and responsiveness stops.**
- **Second attempt succeeded outright.** Disk did dip to as low as 5GiB during the worst of it (swap peaked around 43GB), but never crossed zero, and the whole export finished in about 11 minutes. Output: `Aquinas_Backend/models/dwq_litert_test/model.litertlm`, **2,722,385,120 bytes** — same size as production, this time built from the real (512-sample, 0.048-loss) correction, not the smoke test.

### Device test
Pushed the file to the phone via `devicectl` (same flow as before), forced a fresh launch with `--terminate-existing` so the new model path actually took effect, and ran the probe:

- **Cold load: 3.66s** — even faster than last night's smoke-test result (4.21s) and than the documented production baseline (4.33s).
- **Generated text**: *"Prudence is the moral virtue of prudence, which is the prudent consideration of what is right and proper."*

Structurally this is a clean pass (loads fast, generates fast, no crash). But the answer is circular again — different wording than last night's smoke-test failure ("the notion of prudence in action...") but the same shape of problem, despite training on 32x more data and reaching a validation loss 3x lower.

### The reframe that turned out to be correct
DWQ's loss is a **distillation loss** — it measures how closely the corrected model's outputs match the *original, full-precision, unquantized* model's outputs. A lower loss means better fidelity to that teacher, not necessarily better absolute quality. The hypothesis was: if the base Aquinas model already produces this kind of circular phrasing on this specific prompt, DWQ doing its job well would faithfully reproduce that behavior, not fix it. This turned out to be directionally right, and worse than expected — see below.

---

## Part 4: the comparison test, and closing the investigation (Aug 11 afternoon)

### Two build-quality fixes made along the way
- **Added `--skip-vision`** to `export_litert_aquinas.py`. On-device multimodal input is post-launch, and the vision-adapter compile step was also the single largest memory/disk consumer in every export — the exact step that had killed multiple attempts (including two more this afternoon, both on the production baseline export, even from a clean ~58-69GB starting point with no other memory pressure). Skipping it isn't just a workaround: it's the actually-correct scope for what's needed right now. With it, a text-only export finished in **6:38** — roughly half the time, no near-misses at all.
- Confirmed disk headroom alone doesn't fully explain the export failures — a concurrent Claude Code session doing unrelated memory-heavy work on the same Mac was a real contributing factor to at least one earlier failure. Worth remembering for any future export: check what else is running, not just `df`.

### The comparison result
Built the current production model as a text-only package (`Aquinas-Final-LiteRT`, 2,556,106,704 bytes, no DWQ), pushed it to the phone, ran the identical "Prudence" probe prompt used against the DWQ candidate:

- **Cold load: 4.44s** (roughly in line with DWQ's 3.66s and the documented baseline).
- **Generated text**: fell into a **severe repetition loop** — *"Prudence is the virtue that directs human action by reason, especially in regard to prudence, which is the virtue that directs human action by reason..."* repeated verbatim dozens of times over **65 seconds** before stopping.

Compared side by side:

| | Baseline (no DWQ) | DWQ-corrected (512-sample) |
|---|---|---|
| Cold load | 4.44s | 3.66s |
| Generation | 65s, runaway repetition loop | Fast, single clean (if circular) sentence |
| Failure mode | Severe — degenerate repeated-phrase loop | Mild — topically circular but grammatically normal, stops cleanly |

**Conclusion: the circularity is inherited from the base model / training data, not introduced by DWQ.** If anything, DWQ's correction came out *more* stable on this exact prompt, not less — it didn't inherit the worst of the baseline's behavior. This matches what your CLAUDE.md documents as a known class of problem your production app's own runtime already guards against (repeated-phrase rejection, preserving the coherent prefix) — meaning a real user probably wouldn't see the baseline's 65-second loop in practice, but at the raw model level, on this prompt, DWQ is the better-behaved of the two.

### The real cause: wrong test tool, not a training-data problem
Two follow-up findings changed the diagnosis again, in a good way:

1. **The training-data theory didn't hold up.** Checked directly: literal immediate sentence repetition appears in only 1 of 5,517 training examples — not the systemic pattern first suspected.
2. **The actual cause was the test methodology.** All 5,517 training examples use one fixed instruction template ("Explain this Scholastic concept using the method of the Summa") — zero examples use a bare "What is X?" question. The debug probe we'd been using all night (`--litert-probe`) sends exactly that bare, out-of-distribution question, with a minimal system message ("Answer clearly and in one concise sentence") that has none of the production system prompt's safeguards. The real production system prompt (`conversationSystemInstruction` in `LiteRTAquinasModel.swift`) explicitly instructs: *"Do not imitate archaic source prose, invent quotations, announce what will be examined later, or pad an answer by repeating the term or conclusion."* — a direct guard against exactly the failure we kept seeing.

There's a second, purpose-built probe mode already in the codebase for this: `--litert-quality-probe`, which routes through the real production conversation path (`LiteRTAquinasModel.respond`), engaging the actual system prompt. It requires `--litert-probe` to also be present (the outer flag that routes to the probe view at all; `--litert-quality-probe` is checked inside that view) — passing only `--litert-quality-probe` alone just opens the regular app.

### Retested with the correct tool — the issue is gone
Ran both models through `--litert-probe --litert-quality-probe` with the real production prompt, on the default quality-probe question ("How can justice and mercy work together when someone repeatedly does wrong?"):

| | Baseline (no DWQ) | DWQ-corrected (512-sample) |
|---|---|---|
| Cold load | 3.94s | 3.73s → 3.92s (re-run) |
| Generation | 4.35s | 2.95s → 2.79s (re-run) |
| Response | Coherent, twofold justice/mercy distinction (one minor "seem to to" typo) | **Identical text**, including the same typo |

Both answers were clean, coherent, on-topic — no repetition, no circularity. On a re-run, the DWQ model produced **byte-identical output** to the baseline, just consistently faster. That's explained by the app's deterministic decoding (per CLAUDE.md, sampled decoding corrupts this 4-bit checkpoint, so generation is fixed/greedy) combined with DWQ only adjusting quantization scales/biases on a small fraction of weights — not enough to flip any token choice on this decoding path, but enough to measurably speed things up.

**Bottom line: the entire night's chase — circular "Prudence" definitions, the 65-second repetition loop — was a testing-methodology artifact from using the bare debug probe, not a real defect in either model.** With the actual production prompt engaged, both models perform well, and DWQ shows a real (if modest) generation-speed edge with zero quality cost.

### One shipped improvement along the way
Added a **"Copy results" button** to the device probe UI (`LiteRTDeviceProbe.swift`) — copies model size, cold load, generation time, and response text as one formatted block, since manually retyping all of that for comparison was the actual workflow all night. Built, installed, and confirmed working on-device.

---

## What this investigation actually proved

- **DWQ is a real, working, laptop-feasible technique** for correcting post-training-quantization error, done properly (512+ samples, not a smoke test) — a legitimate, substantial improvement in distillation loss (0.258 → 0.048), and it does not appear to introduce or worsen the specific quality problem investigated here.
- **The full mechanism works end-to-end** and is now proven with real, on-device, side-by-side comparison data: MLX training → bridge to HF format → production export pipeline → real device test, twice over (DWQ and baseline).
- **The circular-definition problem is not a DWQ defect.** The raw-probe comparison initially
  made the base model look responsible, but the production-path retest showed that the apparent
  defect was caused by the bare, out-of-distribution probe. It is not evidence against either
  model or the training data.
- **Two real, reusable pipeline improvements came out of this**: the NaN-guard patch to `mlx_lm/quant/dwq.py`, and the `--skip-vision` flag on the production export script (a real scope correction given multimodal is post-launch, independent of anything DWQ-related).
- **Two real bugs in the export tooling were found and worked around**: killed export attempts silently leave temp files behind (not cleaned up automatically), and concurrent memory-heavy processes on the same Mac can tip an export into failure even with generous disk headroom.

## Where things were on disk after the 4-bit investigation (historical snapshot)

This snapshot was superseded by the cleanup in Addendum 2 and must not be used to infer current
files or machine state.

- `Aquinas_Backend/models/dwq_bridged_hf/` — the properly-trained (512-sample, 0.048 loss) reconstructed HF checkpoint (~9.6GB). No longer actively needed unless DWQ work resumes later; regenerable from `dwq_bridge_reconstruct.py`.
- `Aquinas_Backend/models/dwq_litert_test/model.litertlm` — the DWQ candidate (2.72GB), device-tested. Also still on the phone as `model.litertlm` in the app's Documents folder.
- `Aquinas_Backend/models/Aquinas-Final-LiteRT/model.litertlm` — the text-only production baseline (2.56GB) built for this comparison. Also still on the phone as `model_baseline.litertlm`.
- `Aquinas_Backend/scripts/dwq_bridge_reconstruct.py` — the reusable DWQ→HF bridge script, kept for reference/future use.
- `Aquinas_Backend/scripts/dwq_full_run.py` — the 512-sample DWQ training script, kept for reference/future use.
- `Aquinas_Backend/scripts/export_litert_aquinas.py` — now has a permanent `--skip-vision` flag (default off, matching prior behavior) — this is a real, kept improvement independent of the DWQ investigation.
- The `mlx_lm/quant/dwq.py` NaN-guard patch is applied directly to the installed package in `aquinas_env` — not vendored/tracked anywhere else, so a fresh `pip install`/venv rebuild would need it reapplied if DWQ work ever resumes.
- Disk is back to a clean ~50GB+ free; no hung processes, no leftover temp files.

---

## Addendum (Aug 12 afternoon) — the comparison above was against a checkpoint we'd already moved past

Follow-up review found that the "current shipped package" this whole investigation compared
DWQ against (2.72 GB, `dynamic_wi4_afp32`, 4-bit) was **not actually what was shipping** by
the time this session ran. Five days earlier, commit `f204b5a` (Aug 6) had already switched
production to **8-bit decoder weights + 4-bit embeddings**
(`dynamic_wi8_emb4_afp32`, 3.86 GB, `sha256: 9a6345f1a6cd39283f957977c84d31cc63b8dd56f2b8fffeb784940f63365282`,
see `Aquinas-iOS/Services/LiteRTModelStore.swift:15-16`) — a switch made specifically to fix
the 4-bit checkpoint's own repetition/looping problem, the same symptom category this whole
DWQ night chased. So the Part 4 comparison table (DWQ vs. baseline) was old-4-bit vs.
old-4-bit; it never touched what's actually in the app today, and everything in "Where
historical disk snapshot above describes a correction of a checkpoint no longer shipped.

**This doesn't invalidate the mechanism-level conclusions** (DWQ works, the MLX→HF bridge
is correct, the NaN-guard and `--skip-vision` fixes are real and reusable) — it just means
the specific corrected artifact isn't deployable, and the quality comparison needs redoing
against the real production scheme.

Follow-up research also confirmed this is a coherent, low-friction retry, not a rethink:
DWQ's MLX-side correction and the later `dynamic_wi8_emb4_afp32` litert export are fully
independent pipeline stages (DWQ produces dense bf16, the litert export applies its own
quantization recipe afterward, with no shared state), and both already point at the same
base checkpoint (`Aquinas_Backend/models/Aquinas-Final-HF`) — confirmed identical via a
byte-for-byte match of `aquinas_conversion_manifest.json` in both `Aquinas-Final-HF` and
`dwq_bridged_hf`. So there's no checkpoint-staleness problem to solve first, only a
retarget-to-8-bit problem.

### Next-session handoff plan

> **Historical plan, already attempted.** Addenda 2–4 contain the outcome. Do not execute this plan;
> use the final authoritative protocol at the end of the document.

**Goal:** free disk space by discarding the stale 4-bit DWQ artifacts, retry DWQ properly
against the 8-bit scheme actually in production, and reconcile the two now-contradictory
CLAUDE.md sections. Estimated ~1.5-2+ hours (training + export + device comparison), with
real risk of repeating the disk/swap near-misses documented above — this is why it's a
handoff rather than a same-session follow-up.

1. **Free disk space (confirm before deleting — destructive, ~12GB).** Delete
   `Aquinas_Backend/models/dwq_bridged_hf/` (~9.6GB) and
   `Aquinas_Backend/models/dwq_litert_test/model.litertlm` (~2.72GB, sha `04b6c19e...` — also
   untracked/unexplained by any script, treat as stale scratch). Keep the reusable tooling:
   `dwq_bridge_reconstruct.py`, `dwq_full_run.py`, the `--skip-vision` flag on
   `export_litert_aquinas.py`, and the NaN-guard patch already applied to the installed
   `mlx_lm/quant/dwq.py` (not git-tracked — re-verify it's still present; reapply from Part 2
   above if a venv rebuild wiped it).

2. **Fix the DWQ unfreeze blocker.** The installed `mlx_lm/quant/dwq.py`'s `unfreeze()`
   (`m.bits < 8` predicate, ~line 96) skips every layer when targeting `bits=8` — DWQ would
   silently train nothing until this is loosened (e.g. `m.bits <= 8`, or an explicit "unfreeze
   whatever was just quantized" check). Same file/mechanism as the NaN-guard patch — not
   git-tracked, so record the exact diff here once applied.

3. **Retarget training/bridge scripts to 8-bit.** `dwq_full_run.py` (currently hardcodes
   `bits=4, group_size=64`, ~line 37) → `bits=8`. `dwq_bridge_reconstruct.py` (currently
   `mx.dequantize(..., bits=4)`, ~line 69) → `bits=8` to match. Source checkpoint stays
   `models/Aquinas-Final-HF` — unchanged, already confirmed to be the same base weights used
   for the current production 8-bit export.

4. **Retrain, bridge, export.** Run `dwq_full_run.py` at `bits=8` against `Aquinas-Final-HF`
   (expect similar ~24min/512-sample timing to the 4-bit run, same batch-size=1/NaN-guard
   caveats as Part 2 above). Bridge to dense bf16 via `dwq_bridge_reconstruct.py`. Export via
   `export_litert_aquinas.py --source <bridged checkpoint> --quantization-recipe
   dynamic_wi8_emb4_afp32` — the exact recipe string production uses today.

5. **Device-test with the correct probe.** Push to the phone via `devicectl`, test with
   `--litert-probe --litert-quality-probe` only (the real production-path probe — **not**
   bare `--litert-probe`, which is what caused the false "circular definition" alarm the
   first time, per Part 4 above). Run multiple varied prompts, not just one, comparing cold
   load / generation time / blind quality against the current shipped package
   (`LocalModels/gemma-4-E2B-it.litertlm`, sha `9a6345f1...`).

6. **If it passes, swap production; if not, stop and record why.** Only if the new package
   is at least as good on quality and meets/beats current speed and size: update
   `LiteRTModelManifest.aquinas` in `LiteRTModelStore.swift:15-16` with the new
   `byteCount`/`sha256`, replace `Aquinas-iOS/Aquinas-iOS/LocalModels/gemma-4-E2B-it.litertlm`
   with the new file (gitignored dev seed, no repo tracking needed), and verify with
   `xcodebuild -project Aquinas-iOS.xcodeproj -scheme Aquinas-iOS -destination 'platform=iOS Simulator,name=iPhone 17' build`
   from `/Users/ryanbaltodano/Developer/Aquinas-iOS`. Either way, append the outcome here.

7. **Reconcile CLAUDE.md.** `Aquinas-iOS/CLAUDE.md` currently has two sections describing the
   production model that disagree: an older section (~lines 111-118) still describes the
   retired 2.72GB/4-bit package as current, while a separate, correct section (~lines
   226-228) documents the real Aug 1 `8fc4emb` 8-bit candidate. Update the older section to
   match the real current manifest and remove the contradiction.

---

## Addendum 2 (Aug 12, later) — 8-bit retry started, hit a learning-rate wall, paused mid-fix

> **Historical record only — do not execute this addendum's “What to do next.”** Addenda 3 and 4
> invalidated its learning-rate-only diagnosis, and the final controlled-retry protocol at the end
> of this document supersedes every restart or promotion direction above it.

Picked up the handoff plan above in this session. At that historical point, steps 1–3 were done and
verified, and step 4 appeared to be blocked only on a learning-rate fix. Addendum 3 later disproved
that diagnosis; no agent should resume from this point.

### What's actually done (verified, safe to build on)

1. **Disk freed.** Deleted `Aquinas_Backend/models/dwq_bridged_hf/` (~9.6GB) and
   `Aquinas_Backend/models/dwq_litert_test/` (~2.5GB, including the unexplained untracked
   `model.litertlm`). Disk went from 61GB free to 73GB free. Both directories no longer exist.

2. **`unfreeze()` blocker fixed and confirmed working.** In the *installed* package (not
   git-tracked — lives at
   `Aquinas_Backend/aquinas_env/lib/python3.14/site-packages/mlx_lm/quant/dwq.py`, line ~95),
   changed:
   ```python
   # before
   and m.bits < 8
   # after
   and m.bits <= 8
   ```
   Confirmed working empirically: the first full-scale run below printed
   `Trainable parameters: 3.131% (144.920M/4628.569M)` — essentially identical to the 4-bit
   run's trainable-parameter ratio in Part 2 above, proving 8-bit layers are now actually
   being unfrozen and trained rather than silently skipped. **This patch is not git-tracked
   and will be lost on a venv rebuild** — re-apply it first thing if `aquinas_env` ever gets
   recreated. The pre-existing NaN-guard patch in the same file's `step()` (documented in
   Part 2 above) is still present and unaffected.

3. **Scripts retargeted to 8-bit**, both in `Aquinas_Backend/scripts/`. They are real
   repo-local files but remain untracked as of Aug 13 — preserve or add them explicitly if the
   tooling should become permanent:
   - `dwq_full_run.py`: `OUT_PATH` renamed `models/dwq_full_run` → `models/dwq_full_run_8bit`
     (so it can't collide with old 4-bit runs); `quantize_model(..., bits=4)` →
     `quantize_model(..., bits=8)` at line ~37.
   - `dwq_bridge_reconstruct.py`: `DWQ_MODEL` → `models/dwq_full_run_8bit`, `OUTPUT` →
     `models/dwq_bridged_hf_8bit` (renamed for the same collision-avoidance reason); the
     `mx.dequantize(..., bits=4)` call at line ~69 → `bits=8`.
   - Source checkpoint is unchanged: both still read from `models/Aquinas-Final-HF`.

### What's blocking step 4, diagnosed in detail

**The first full 512-sample run at the tool's default `learning_rate=1e-6` diverged
immediately** — this is a different failure mode than the 4-bit run's single-bad-example NaN
blip in Part 2 above, and the existing NaN-guard does not fix it:

```
step 1: loss=0.0967
step 2: loss=45.25
step 3: loss=95.5
step 4: loss=nan   <- and every step after this stayed nan (guard skips the update,
                       but the *weights* were already corrupted by steps 2-3's huge
                       updates before any NaN appeared, so the forward pass keeps
                       producing nan regardless)
```

**Root cause:** 8-bit quantization starts far closer to the full-precision teacher than
4-bit did. Initial validation loss on the full run was **0.003** (vs. 0.258 for the 4-bit
run in Part 2 — roughly 85x less quantization error to correct in the first place). With
that little real signal, `learning_rate=1e-6` (both this project's prior default and
`mlx_lm.dwq`'s own CLI default, per `dwq.py` line ~284) overshoots hard on the much flatter
8-bit loss landscape, and the runaway compounds within 3-4 steps.

**A 32-sample smoke test at `learning_rate=1e-7` (10x lower) confirms this is fixable by
just lowering the learning rate** — full log:
```
Trainable parameters: 3.131% (144.920M/4628.569M)
Validation: it=0, loss=0.023
step 1: loss=0.0034
step 2: loss=0.0032
step 3: loss=0.0052
step 4: loss=0.0034
```
Four steps in a row, all small (0.003-0.005 range) and stable — no explosion, unlike `1e-6`'s
0.09 → 45 → 95 → nan in the same number of steps. This run was intentionally stopped after 4
steps (mid-session pause, not a failure) once the stability pattern was clear; it was never
meant to run to completion as a 32-sample run anyway (32 samples is a smoke test, same caveat
as the original 16-sample smoke test in Part 1 — too few to generalize, only useful for
checking stability, not for producing a deployable correction).

**One operational trap hit along the way, worth knowing about for any future run:** after
killing the diverged `1e-6` run with `pkill`, the process didn't actually die — it stayed
alive in macOS's uninterruptible-sleep state (`ps` showed status `U`), still holding its
full memory allocation, which drove swap to 96% full (18.6GB/19.4GB) and made an unrelated
smoke test crawl (2m35s for a single step that normally takes ~1-2s). Diagnosed via
`ps aux | grep dwq` + `sysctl vm.swapusage`; fixed with `kill -9 <pid>`, not plain `kill`/
`pkill`. **Always confirm no stray `dwq_full_run.py`/smoketest processes are alive
(`ps aux | grep dwq`) and that swap has recovered (`sysctl vm.swapusage`, should be a couple
GB at most when idle) before starting a new training run** — a repeat of this will silently
corrupt timing and stability signals the same way it nearly did here.

### What was proposed next at the time (obsolete; retained as investigation history)

The numbered steps below explain what the next agent attempted; they are **not** a current runbook.
In particular, the later trials disproved the claim that only the learning rate remained to be
fixed, and the current script still lacks cache controls and resumable checkpoints.

1. Confirm clean state: `ps aux | grep -i dwq` should show nothing; `sysctl vm.swapusage`
   should be low (a few GB, not climbing).
2. Edit `Aquinas_Backend/scripts/dwq_full_run.py` line ~43, change
   `optimizers.Adam(learning_rate=1e-6, bias_correction=True)` to
   `optimizers.Adam(learning_rate=1e-7, bias_correction=True)` — this was believed to be the one
   remaining fix at the time; Addendum 3 disproved that belief.
3. Run it for real: `cd Aquinas_Backend && source aquinas_env/bin/activate && python3
   scripts/dwq_full_run.py`. Expect roughly the same wall-clock shape as the 4-bit run in
   Part 2 (~24 min for 512 samples at `batch_size=1`), though a 10x lower learning rate may
   mean the correction is smaller/slower to converge — watch the loss curve; if it's still
   trending down and stable near the end rather than plateaued, more samples or a slightly
   higher rate (e.g. `3e-7`) could be worth trying, but only after confirming `1e-7` doesn't
   diverge over a full run (the smoke test only proved 4 steps' worth of stability).
   If it diverges again even at `1e-7`, drop to `1e-8` next rather than assuming more data
   will fix an exploding-gradient pattern — the smoke test data points at learning rate as
   the variable that matters, not sample count.
4. The plan at that time was to continue per steps 4 (bridge/export) through 7 in the original
   handoff above. Later evidence makes that continuation conditional on the final protocol's
   training, checkpoint-reload, validation, and clean-process gates:
   - `python3 scripts/dwq_bridge_reconstruct.py` → produces `models/dwq_bridged_hf_8bit`.
   - `python3 scripts/export_litert_aquinas.py --source models/dwq_bridged_hf_8bit
     --quantization-recipe dynamic_wi8_emb4_afp32` → produces the comparable `.litertlm`.
   - Device-test with `--litert-probe --litert-quality-probe` (never bare `--litert-probe` —
     see Part 4 above for why that produces false failures), multiple varied prompts, vs. the
     current shipped package (`LocalModels/gemma-4-E2B-it.litertlm`, sha `9a6345f1...`).
   - If it passes: update `LiteRTModelManifest.aquinas` in
     `Aquinas-iOS/Aquinas-iOS/Services/LiteRTModelStore.swift:15-16`, replace the bundled dev
     seed file, verify with the standard `xcodebuild ... build` command from
     `/Users/ryanbaltodano/Developer/Aquinas-iOS`.
   - Either way, append the real outcome here.
   - Reconcile `Aquinas-iOS/CLAUDE.md`'s two contradictory model sections (~111-118 vs
     ~226-228) regardless of the retrain's outcome — this is independent of whether the
     8-bit DWQ retry itself succeeds.

### Current machine state as of this pause

- Disk: 72GB free (`df -h /`).
- Swap: ~2.7GB used, healthy (`sysctl vm.swapusage`).
- No `dwq`-related processes running (confirmed via `ps aux | grep dwq` immediately before
  writing this).
- `Aquinas_Backend/models/` contains no `dwq_full_run_8bit` or `dwq_bridged_hf_8bit` yet —
  the diverged run never reached its save step, and the stable smoke test was 32-sample/
  intentionally stopped early, so nothing has been written to either output path.

---

## Addendum 3 (Aug 13) — 8-bit retry closed: no deployable correction produced

Resumed from Addendum 2 and completed the investigation far enough to make a definitive
deployment decision. **Production remains unchanged.** No 8-bit DWQ checkpoint completed the
training and validation gates, so there was nothing valid to bridge, export, or device-test.
The shipping `dynamic_wi8_emb4_afp32` package remains exactly 3,862,121,696 bytes with SHA-256
`9a6345f1a6cd39283f957977c84d31cc63b8dd56f2b8fffeb784940f63365282`.

### What the retry established

1. **Learning rate was not the whole failure.** Full runs at both `1e-7` and `1e-8` with the
   original 1,025-token window produced small, finite losses for the first two steps, then a NaN
   forward pass at step 3. The existing loss-only guard acted one step too late.

2. **Finite loss hid non-finite gradients.** Strengthened the installed, untracked
   `mlx_lm/quant/dwq.py` guard to flatten and check every gradient before Adam updates. The failed
   step then reported `loss_finite=True, grads_finite=False`; 45 gradients were non-finite across
   late transformer layers, including projection scales/biases, norms, and layer scalars. Skipping
   the update kept the next forward loss finite and proved that corrupt gradients—not the finite
   loss value—were poisoning the model and optimizer. Casting the trainable path to float32 did
   not fix it. A zero-cotangent/padding fast-path experiment in MLX's Metal KL backward kernel also
   did not fix it and was reverted.

3. **Context length strongly affected numerical stability, but no usable full run emerged.** The
   initial failures correlated with long batches (for example 638, 721, and 725 tokens). At a
   512-token cap, shorter formerly-failing points passed, but non-finite gradients still occurred
   frequently across later batches, so that run was stopped rather than training on a biased
   subset. A 256-token cap reached step 104 with zero skipped gradients and improving loss, then
   Metal terminated the process with `kIOGPUCommandBufferCallbackErrorOutOfMemory` before saving.

4. **The laptop-safe window did not produce a validated improvement.** At 128 tokens and `1e-7`,
   training reached the step-199 validation gate without skipped gradients, but validation loss
   worsened from `0.006` to `0.009`; the run was stopped as an objective regression. The final
   128-token/`1e-8` attempt stayed finite but hit the same Metal out-of-memory error at step 39.
   Repeated health checks showed roughly 75 GiB free disk and about 3.8–4.1 GiB swap used; the
   failures were Metal command-buffer allocation failures, not disk exhaustion or a stray DWQ
   process.

### Exact retained installed-package patch

The earlier `m.bits <= 8` unfreeze change remains. The NaN guard in the installed
`Aquinas_Backend/aquinas_env/.../mlx_lm/quant/dwq.py` now also imports `tree_flatten`, checks
`mx.isfinite` across every flattened gradient, evaluates that check before applying Adam, logs the
count and first failing parameter names, and skips the update unless both loss and all gradients
are finite. This is useful diagnostic hardening, but it is still an untracked site-packages edit
and will disappear if the virtual environment is rebuilt.

`dwq_full_run.py` remains retargeted to 8-bit and records the final conservative diagnostic
settings (`max_seq_length=128`, `learning_rate=1e-8`) with an explicit warning that they did not
complete. `dwq_bridge_reconstruct.py` remains correctly retargeted to dequantize 8-bit output, but
was not run because no valid input checkpoint exists. Neither `models/dwq_full_run_8bit` nor
`models/dwq_bridged_hf_8bit` exists.

### Final decision

Close the 8-bit DWQ deployment retry. The production 8-bit package already starts extremely close
to its bf16 teacher (initial distillation loss around 0.003–0.007 depending on the truncated
validation window), leaving little correction signal. On this M4 Pro/24 GB machine and current MLX
implementation, attempts either produce broad non-finite gradients, run out of Metal memory, or
worsen held-out loss. Exporting or device-comparing any of those partial results would be invalid.

Also reconciled `Aquinas-iOS/CLAUDE.md`: it now identifies the real 3.86 GB
`dynamic_wi8_emb4_afp32` package and matching manifest as production, describes the 2.72 GB 4-bit
artifact as retired, and no longer treats the 8-bit package's Simulator-only GPU failure as a
production rejection.

---

## Addendum 4 (Aug 13) — corrected memory interpretation and parked restart plan

This addendum refines Addendum 3's conclusion after comparing the failed DWQ memory topology with
the Aquinas training that previously succeeded on this same M4 Pro/24 GB Mac. **It supersedes the
"What to do next" instructions in Addendum 2. Do not resume from those older steps without first
reading this section.**

### Why prior Aquinas training fit while 8-bit DWQ did not

The earlier Aquinas fine-tune was MLX LoRA training, not full-weight base-model retraining in the
ordinary sense. The base model stayed frozen; only small adapter matrices were trainable, and the
adapters were fused into the base checkpoint afterward. That workflow requires one base model,
its forward/backward activations, and optimizer state for the comparatively small adapters.

DWQ has a less obvious but substantially heavier peak-memory topology even though it updates only
about 3.13% of parameters:

- A dense bf16 teacher remains loaded to produce target logits.
- A separate quantized student remains loaded at the same time.
- The student performs a backward pass through the full language graph.
- Gradients and Adam state exist for approximately 144.92 million trainable quantization
  scales/biases and other unfrozen quantized-module parameters.
- Teacher logits, student logits, activations, gradients, optimizer state, Metal scratch buffers,
  and MLX's reusable allocation cache can overlap at a step boundary.

The 8-bit student is also materially larger than the successful 4-bit DWQ student. The 4-bit DWQ
run proved that this Mac and the bridge/export mechanism can handle the technique at the smaller
student size; it does not imply that the 8-bit teacher-plus-student peak will fit.

### Exact capacity evidence from this Mac

`mx.device_info()` reported:

```text
memory_size:                     25,769,803,776 bytes (24 GiB)
max_recommended_working_set:     19,069,665,280 bytes (~17.76 GiB)
max_buffer_length:               14,302,248,960 bytes
```

The DWQ script already set the MLX wired limit to the full reported
`max_recommended_working_set_size`, approximately 74% of physical unified memory. Observed MLX
peaks were roughly 18.65–20.12 GB, directly against that ceiling, while macOS reported about
3.8–4.1 GB of swap use during the later trials. The 256-token run remained numerically clean
through step 104 and then failed with a Metal command-buffer out-of-memory error. The 128-token
`1e-8` run failed the same way at step 39. Those are transient-allocation failures: the displayed
peak does not necessarily include the next allocation that Metal could not satisfy.

This means the machine was genuinely near its capacity, but the failures do **not** establish that
24 GB can never complete the job. Many Metal-heavy attempts ran back-to-back in one boot. MLX's
buffer cache, Metal allocator fragmentation, and resources not immediately returned to the system
may have reduced usable headroom on later attempts. The fact that the final 128-token run failed
earlier than the 256-token run is evidence that session history/allocator state contributed; it is
not evidence that 128 tokens inherently require more memory.

### Memory estimate for a future machine

- **24 GB:** marginal but not conclusively impossible. Only retry after a reboot with explicit
  cache controls and checkpointing.
- **32–36 GB:** likely enough for this specific lightweight 8-bit DWQ job, giving several more
  gigabytes of Metal working-set and OS headroom.
- **48 GB:** recommended practical target. It provides comfortable space for teacher, 8-bit
  student, optimizer/gradients, activations, transient buffers, and macOS without tuning against
  the hard limit.
- **64 GB or a 40–48+ GB accelerator:** preferable if moving beyond DWQ into more conventional
  QAT, longer contexts, broader parameter updates, or repeated hyperparameter sweeps.

These are engineering estimates from the measured run, not guarantees. The current production
8-bit model begins extremely close to the bf16 teacher, so additional hardware may make training
complete without making the correction valuable.

### Do not raise the wired-memory percentage first

Overriding macOS's GPU wired-memory limit above the recommended ~75% might allow one transient
allocation to succeed, but it is the wrong first lever on a 24 GB machine. Wired memory is kept
resident and cannot be reclaimed like ordinary pageable memory. Raising the GPU share to 80–85%
would leave only about 3.9–5.2 GB of physical memory for macOS and every other process, while the
observed runs were already swapping at the default ceiling. That could trade a contained Metal OOM
for severe system thrashing, a frozen desktop, or GPU-driver/system instability. It also cannot fix
the independent `1e-7` result where held-out loss worsened from 0.006 to 0.009.

Prefer reducing MLX cache retention and improving resumability. Only consider a wired-limit
override as an explicitly supervised diagnostic after a clean cache-controlled run, with a backup,
no other applications open, active memory monitoring, and acceptance that the Mac may need a hard
restart. It is not an approved unattended workflow.

### Preliminary future-agent restart plan (superseded by the final protocol below)

This was the first cache/checkpoint-aware restart outline. It remains useful evidence, but the final
agent handoff below adds the required operational gates and is the only executable runbook.

1. **Confirm the objective first.** Production already uses the verified 3.86 GB
   `dynamic_wi8_emb4_afp32` artifact. Do not restart merely to make DWQ complete; require a concrete
   quality, size, or latency hypothesis that justifies replacing it.
2. **Use a fresh machine state.** Reboot the Mac. Close Figma, Xcode builds, browsers with heavy
   tabs, local model servers, video calls, and other GPU/AI processes. Verify no DWQ process exists,
   swap is low, and disk has at least 70–80 GiB free before training/export.
3. **Re-verify installed patches.** A virtual-environment rebuild removes the untracked
   `mlx_lm/quant/dwq.py` changes. Confirm `m.bits <= 8` and the pre-Adam all-gradient finiteness
   guard are present. Preserve the guard even if the underlying numerical bug is fixed; a skipped
   update must be logged and treated as a failed training-quality signal, not silently ignored.
4. **Control MLX allocation caching before raising wired memory.** Add explicit instrumentation for
   active, cached, and peak Metal memory. Start with a small cache limit (or zero for the diagnostic
   pass), clear reusable cache at safe validation/checkpoint boundaries, and keep the wired limit at
   or below `max_recommended_working_set_size`. Record exact settings and timing in this file.
5. **Add resumable checkpoints.** The current script saves only after all 512 samples, so every OOM
   discards the run. Save the corrected student plus optimizer/restart state at least before and
   after validation gates. Verify a checkpoint can actually reload before relying on it.
6. **Use validation-gated escalation.** Begin with the retained 128-token/`1e-8` configuration only
   as a memory/numerical diagnostic. Evaluate at or before step 199. Continue only if every update
   has finite gradients and validation improves over its initial value. If stable, test 256 tokens
   from a fresh process; do not infer full-context quality from a truncated 128-token run.
7. **Stop on any objective regression.** Do not bridge/export a run with skipped gradients,
   non-finite values, Metal OOM recovery, or worse held-out loss. The earlier 128-token/`1e-7` run's
   0.006 → 0.009 validation change is a hard failure, even though its forward losses looked normal.
8. **Bridge and export only after a clean final checkpoint.** Then use
   `dwq_bridge_reconstruct.py`, export with `dynamic_wi8_emb4_afp32` and `--skip-vision`, and compare
   against the exact shipping package using `--litert-probe --litert-quality-probe` on the base
   iPhone with multiple blind prompts. Never use the bare probe for quality judgment.
9. **Promote only on a real win.** Require at least equal blind quality plus a meaningful measured
   benefit in latency, size, stability, or sustained memory. Update the manifest and bundled seed
   only after all gates pass.

### Parked status

No deployable 8-bit DWQ artifact exists, no partial checkpoint should be recovered, and production
must remain unchanged. The retained scripts and installed-package diagnostics are research tooling,
not a pending release candidate. Reopen this method only with a fresh-memory/cache experiment or a
larger-memory training environment and a reason to believe the already-small 8-bit quantization
error is worth correcting.

---

## Agent handoff — authoritative controlled-retry protocol (Aug 13)

**This is the only current retry runbook. It comes after and supersedes every “next step,” restart,
kill-threshold, export, and promotion direction above, including Addendum 2.** Earlier sections are
an investigation record, not executable instructions. The 24 GB M4 Pro is marginal. A retry is an
attended diagnostic first and a candidate-producing run only after every gate below passes.

### 0. Write down the objective gate before changing the machine

Do not run merely to see whether 8-bit DWQ can finish. Add a dated retry record under this section
before starting, with:

- one measurable hypothesis against the shipping
  `dynamic_wi8_emb4_afp32` package (for example, an explicitly defined latency, stability, or blind
  quality improvement);
- the baseline artifact's byte count and full SHA-256 (currently recorded as 3,862,121,696 bytes and
  `9a6345f1a6cd39283f957977c84d31cc63b8dd56f2b8fffeb784940f63365282`; re-measure rather than assume);
- the prompts, repetitions, device, pass/fail threshold, and rejection rule fixed in advance; and
- numerical stop thresholds for disk reserve, swap growth/memory pressure, and monitor cadence; and
- paths where the raw training, memory, export, and device logs will be preserved.

If the hypothesis or threshold is not measurable, stop. The initial 8-bit distillation loss was
already only about 0.003–0.007 depending on the validation window, so completion alone is not a win.

### 1. Establish a clean, reproducible machine baseline

1. Reboot immediately before the diagnostic. Do not stack it after another Metal/MLX run.
2. Close Xcode builds, Simulator, Figma, browsers with heavy tabs, local model servers, video calls,
   and all other GPU/AI work. Prevent sleep, but do not leave the run unattended.
3. Inspect processes rather than relying on `grep` alone: confirm there is no live
   `dwq_full_run.py`, smoke-test, export, Python worker, or other unexpected MLX/Metal workload.
   If an old process resists normal termination or remains in `U` state, do not immediately escalate
   to a stronger kill; record its PID/state, stop the retry, and recover with a reboot.
4. Record `df -h /`, `sysctl vm.swapusage`, `mx.device_info()`, relevant process memory, and the
   initial values of MLX active, cache, and peak memory. Require at least 70–80 GiB free for the
   combined training/bridge/export workflow, low and stable idle swap, and no unexplained memory
   pressure. If not clean, stop before loading either model.
5. Inspect known export temporary locations and output paths. Remove stale data only after resolving
   and listing exact targets and confirming it is regenerable scratch—not a checkpoint or production
   artifact. Killed exports previously left 43 GB behind; never use a broad cleanup command.

### 2. Verify code and installed-package state before execution

The repo-local `Aquinas_Backend/scripts/dwq_full_run.py` and
`dwq_bridge_reconstruct.py` are currently untracked research files. The installed
`Aquinas_Backend/aquinas_env/lib/python3.14/site-packages/mlx_lm/quant/dwq.py` edits are also
untracked and disappear on a virtual-environment rebuild. Before each retry:

1. Record `git status`, the environment's Python/MLX/`mlx-lm` versions, and checksums or a saved diff
   of all three files. Do not assume line numbers in earlier addenda are still current.
2. Inspect the installed `dwq.py` and verify both observed patches are present: the quantized-module
   predicate accepts 8-bit modules (`m.bits <= 8` in the tested package), and the pre-Adam guard
   evaluates finiteness of every flattened gradient as well as the loss, logs failing names/count,
   and skips the update if either is non-finite. Verify the run still reports approximately 3.131%
   trainable parameters (144.920M/4628.569M); otherwise stop.
3. Treat any skipped update as a failed quality signal, not successful resilience. A candidate run
   must have zero skipped/non-finite updates.
4. Inspect the scripts' source/output paths, bit width, group size, context length, learning rate,
   sample count, batch size, gradient checkpointing, seed, and validation cadence. The retained
   `128`-token/`1e-8` settings are a failed conservative diagnostic, not an approved final recipe.

### 3. Implement and validate the missing memory and checkpoint controls

Do not start a 512-sample retry with the current script unchanged. It presently sets the wired limit
to `max_recommended_working_set_size`, prints only peak memory periodically, and saves only after
training. It does **not** set a cache limit, clear the cache at controlled boundaries, save resumable
training state, or reload-test a checkpoint.

Before the run, inspect the installed MLX APIs and the optimizer/model serialization contract, then
implement and smoke-test the following. Do not invent a save/reload API from this document:

- capture MLX's prior cache and wired limits before changing them, and restore those exact returned
  byte values in a `finally` path on success, rejection, exception, or interrupt;
- set and log an explicit small cache limit (zero is acceptable for the first diagnostic), and use
  the verified `mx.get_active_memory()`, `mx.get_cache_memory()`, `mx.get_peak_memory()`,
  `mx.reset_peak_memory()`, and `mx.clear_cache()` calls for instrumentation/control at safe
  post-evaluation validation/checkpoint boundaries—not in the middle of lazy work;
- checkpoint the corrected student parameters **and** every state item needed for an exact restart,
  including optimizer state, iteration/data position, seed/order assumptions, configuration, initial
  and latest validation results, and skip counters; and
- launch a fresh process, reload an early checkpoint, validate it, and prove that its iteration,
  validation result, optimizer/restart state, and next-step behavior match the pre-exit run. A file
  that merely deserializes is not a validated resume point.

Preserve the reload-test log. If the exact optimizer/state restoration cannot be verified against
the installed versions, stop and implement/inspect it before spending the full run's memory budget.

### 4. Wired-memory policy

The default diagnostic limit is the reported `max_recommended_working_set_size` (measured here as
19,069,665,280 bytes, about 74% of 24 GiB), not a hand-entered percentage. Change it only inside the
training process after recording the old value returned by `mx.set_wired_limit(limit)`, and restore
that old value on every exit as described above.

An **80% system wired limit may be tried once as a supervised diagnostic only** after a fresh-boot,
cache-controlled, checkpoint-reload-validated run fails solely on a reproducible transient Metal
allocation while loss and every gradient remain finite and validation improves. It is not the first
run and not an unattended setting. Before changing the macOS system limit, inspect the current
`mx.device_info()` and the installed MLX documentation, calculate the exact byte/MB value from actual
physical memory, preserve the prior system setting, ensure a backup exists, and define the rollback.
The installed MLX documentation identifies the macOS control as
`sudo sysctl iogpu.wired_limit_mb=<size-in-megabytes>`; first inspect and record the current
`iogpu.wired_limit_mb` value, and use that same control to restore the captured value. If that key or
behavior differs on the current OS/MLX build, stop and inspect rather than guessing.
After the diagnostic, restore the prior macOS setting and the prior per-process MLX wired limit, then
reboot and verify the normal reported limit/state before doing other work.

**85% is not a default, escalation step, or approved retry setting on this 24 GB Mac.** The observed
runs were already swapping near the recommended ceiling. Do not normalize a successful 80%
diagnostic into later runs; its result must be compared with the cache-controlled default and
recorded as an exceptional condition.

### 5. Monitoring cadence and hard stop conditions

Run in the foreground with a separate, timestamped monitor. At baseline, model load, teacher/student
creation, every logged training interval, before and after validation/checkpoint/cache clear, and
during bridge/export, record at least:

- iteration, train loss, initial/latest validation loss, skipped-update count, throughput, and
  checkpoint identity;
- MLX active/cache/peak memory, `vm.swapusage`, free disk, and relevant process state/RSS; and
- wired/cache limits plus any change from baseline.

Stop the run and reject its current checkpoint on any of the following:

- non-finite loss or gradient, any skipped update, unexplained loss explosion, or validation worse
  than its same-window initial baseline;
- Metal command-buffer error/OOM, process hang or `U` state, sustained severe memory pressure or
  rapidly growing swap, loss of desktop responsiveness, monitor failure, or an inability to write
  the next safety checkpoint;
- disk headroom entering the predeclared safety reserve or falling in an unexplained/unbounded way;
- missing/corrupt checkpoint, failed fresh-process reload validation, unexpected trainable-parameter
  count, configuration drift, or contamination by another heavy process.

Do not copy or promote an in-memory model after OOM recovery. Do not resume past a bad update. Keep
the last previously reload-validated checkpoint for diagnosis only; it is not automatically a
candidate. If the system becomes unresponsive, prioritize a controlled reboot and filesystem/output
inspection before another attempt.

### 6. Conservative staged configuration and validation gates

1. Run a very short 128-token/`1e-8`, batch-size-1 diagnostic only to verify instrumentation,
   checkpoint/reload fidelity, finite gradients, memory behavior, and deterministic restart. It does
   not establish quality and must not be bridged.
2. From a new process (and a fresh reboot if memory did not return to baseline), repeat at 256 tokens.
   Require zero skipped updates and validation improvement at an early gate no later than the
   historical step-199 gate. The prior 256-token run was clean to step 104 and then OOMed; crossing
   that point is a memory test, not evidence of final value.
3. Only after both diagnostics pass at the normal recommended wired limit may a dated configuration
   be proposed for the 512-sample candidate run. Fix its learning rate, context length, cache policy,
   checkpoint cadence, and acceptance threshold before launch. Do not tune reactively inside a run.
4. Require improving same-window validation at every gate, zero skipped updates, a clean final
   validation, and successful fresh-process reload/validation of the final checkpoint. A truncated
   context result supports only that context; do not claim full-context quality from 128 tokens.

Never bridge, export, device-test, or promote a partial, OOM-affected, resumed-without-validation,
non-finite, skipped-gradient, or validation-regressing run.

### 7. Bridge, export, and device evaluation only after the training gates pass

1. From a fresh process, point `dwq_bridge_reconstruct.py` only at the accepted 8-bit checkpoint.
   Inspect its configured paths first. Preserve its exact-key-set assertion against
   `models/Aquinas-Final-HF`; if the assertion or output inspection fails, stop.
2. Inspect `export_litert_aquinas.py --help` and the current source before invoking it. For the
   currently comparable text-only product scope, use the verified recipe
   `dynamic_wi8_emb4_afp32` with `--skip-vision`, an explicit source, and a new non-production
   output directory. Record the exact invocation, output byte count, SHA-256, logs, scratch-space
   peak, and cleanup targets. Do not overwrite the shipping package or manifest.
3. Use the project's previously verified physical-device install/launch flow only after inspecting
   the current app/device tooling and resolving the exact device and container paths. Force a fresh
   launch so the candidate path is actually loaded. Do not infer an exact `devicectl` invocation
   from this write-up where none is recorded.
4. Evaluate on the base iPhone through **both** flags,
   `--litert-probe --litert-quality-probe`, which reaches `LiteRTAquinasModel.respond` and the real
   production prompt. Never use bare `--litert-probe` for quality judgment. Run the predeclared
   blind prompt set and repetitions against candidate and exact shipping baseline, recording cold
   load, generation time, response, stability, and reviewer result. Keep model identity/hash with
   every result so phone-side files cannot be confused.

Promotion requires at least equal blind quality and the measurable benefit declared in step 0.
Only then may a separate, explicitly reviewed change replace the bundled seed and update
`LiteRTModelManifest.aquinas`, followed by the standard project build verification. Training
completion, lower distillation loss, or one fast prompt is insufficient. Production remains
unchanged unless that separate promotion succeeds.

### 8. Failure recovery, cleanup, and record of truth

After any success or failure, restore the captured MLX cache and wired limits. If the system wired
limit was exceptionally changed, restore it, reboot, and verify the restored state. Confirm all DWQ,
export, and monitoring processes have exited; record final swap, disk, and MLX/process state. Inspect
outputs for incomplete checkpoints and exporter temp data. Quarantine or remove only exact,
enumerated scratch targets after recording them; preserve the last reload-validated diagnostic
checkpoint and raw logs until the outcome is reviewed.

Append the complete outcome under a dated subheading immediately below this protocol. This Markdown
file is the decision record and must contain the hypothesis, code/environment identity, clean-state
baseline, exact settings and commands actually used, checkpoint/reload evidence, monitoring extrema,
every stop/skip event, validation results, artifact hashes/sizes, device comparison, cleanup and
limit restoration, and the final reject/promote decision. Raw logs may live elsewhere, but their
paths and hashes must be recorded here. Also update `POST-LAUNCH-FEATURES.md` if the parked status or
deployability conclusion changes. Do not rewrite the earlier history to make a later retry look
cleaner than it was.

### Current handoff state

No valid `dwq_full_run_8bit` checkpoint or `dwq_bridged_hf_8bit` output exists. The retained training
script is deliberately a failed diagnostic configuration and is not ready to run under this
protocol. Cache control, full-state resumable checkpoints, and fresh-process reload validation must
be implemented and verified first. The installed package patches and all three DWQ scripts must be
treated as untracked research state. Production stays on the recorded 3.86 GB
`dynamic_wi8_emb4_afp32` artifact unless a future run clears every gate above.

### Controlled retry record — Aug 13, 2026

**Final status:** rejected at the 128-token deterministic validation/resume gate; production is
unchanged. The objective and preflight below were fixed before execution, followed by the complete
outcome record at the end of this section.

#### Objective and promotion gate fixed before model execution

The measurable hypothesis is that a clean 8-bit DWQ correction can reduce median production-path
generation latency by at least **5%** versus the exact shipping `dynamic_wi8_emb4_afp32` artifact on
the base iPhone named `Ry`, without increasing the package by more than 1%, worsening p95 generation
latency by more than 5%, causing any runtime/repetition failure, or losing any blind pairwise quality
judgment. Completion or lower distillation loss alone is a rejection.

The shipping baseline was re-measured before execution:

- path: `Aquinas-iOS/Aquinas-iOS/LocalModels/gemma-4-E2B-it.litertlm`
- byte count: **3,862,121,696**
- SHA-256: `9a6345f1a6cd39283f957977c84d31cc63b8dd56f2b8fffeb784940f63365282`

If training and export gates pass, candidate and baseline will each run three cold repetitions for
each of these fixed questions through `--litert-probe --litert-quality-probe` plus the explicit
`--litert-probe-question` value:

1. `How can justice and mercy work together when someone repeatedly does wrong?`
2. `What is the difference between prudence and mere caution?`
3. `Why does Aquinas think human law cannot forbid every vice?`
4. `How can free will remain real if God already knows what I will choose?`
5. `What makes an act courageous rather than reckless?`
6. `How should someone respond when conscience and an authority appear to conflict?`

Every result will retain artifact hash, prompt, repetition, cold-load time, generation time, full
response, and runtime status. Pair order and artifact labels will be randomized for blind review.
Promotion requires no candidate quality loss across all 18 pairs, no repetition/runtime failure,
median generation at least 5% faster, candidate p95 no more than 5% slower, and size growth no more
than 1%. Any failed condition rejects the candidate.

#### Predeclared operational thresholds and logs

- monitor cadence: 10 seconds plus event snapshots at baseline, model load, validation, checkpoint,
  cache clear, bridge, and export boundaries;
- disk gate before loading: at least 70 GiB free; hard stop during training/bridge/export at 60 GiB;
- swap hard stop: growth of more than 8 GiB from the run's idle baseline;
- immediate reject/stop: any skipped or non-finite update, validation regression, Metal error/OOM,
  `U` process state, severe memory pressure, lost responsiveness, monitor failure, checkpoint write
  failure, configuration drift, or another heavy process appearing;
- run records: `Aquinas_Backend/runs/dwq-controlled/2026-08-13-128-prepare/`,
  `Aquinas_Backend/runs/dwq-controlled/2026-08-13-128-resume/`, and—only after those pass—the
  corresponding dated `256` and `candidate` directories;
- bridge/export records: `Aquinas_Backend/runs/dwq-controlled/2026-08-13-export/`;
- device records: `Aquinas_Backend/runs/dwq-controlled/2026-08-13-device/`.

The retry will use the normal `max_recommended_working_set_size`; no system wired-limit override is
authorized by this record.

#### Preflight/tooling work completed so far

- Reboot was confirmed at Aug 13 20:55 EDT; idle swap was 0 and `memory_pressure -Q` reported 90%
  system memory free before model load.
- Exact regenerable Xcode caches were enumerated, checked for live users, and removed to establish
  disk headroom: `DerivedData` (9.1 GiB), two `XCTestDevices` clones (24 GiB), and one iOS
  `DeviceSupport` cache (6.5 GiB). They are not recoverable as deleted cache contents, but Xcode will
  regenerate them. No Simulator container, checkpoint, source model, or production artifact was
  removed. APFS reported 74 GiB free afterward.
- `dwq_full_run.py` was replaced with a controlled runner that captures/restores the prior MLX cache
  and wired limits in `finally`, sets an explicit cache limit, records active/cache/peak memory,
  rejects the first bad gradient instead of skipping it, persists student parameters plus full Adam
  state/data position/configuration/validation/skip counters, and supports deterministic
  fresh-process resume verification. `dwq_monitor.py` is a separate timestamped process monitor for
  RSS/state, disk, swap growth, and memory pressure. A small synthetic MLX checkpoint/optimizer
  round-trip passed before loading the real model.
- Current file identities before the first real diagnostic:
  - `dwq_full_run.py`: `30f04fa32c865fc17ff1aec60bab7b62a104853ebfd327f732942f72f48e8176`
  - `dwq_monitor.py`: `28495384ca516e488a5b3de4675127a2406c73237c577df903dc8b0692950d75`
  - `dwq_bridge_reconstruct.py`: `2fd6a8202c8c8e5135cdf812b2412db8b1f5d691e2eadf8690621494f411fd93`
  - installed patched `mlx_lm/quant/dwq.py`:
    `84d85be8bbb4fa140334e3eefc1af14c65b60b6a02f386b963577899439894d6`

Preflight found the Ollama application and its local server still running. A normal application quit
request was canceled, so the protocol's clean-process gate is not yet satisfied. No force-kill or
model load was attempted after that cancellation.

#### Controlled retry outcome — rejected at the 128-token resume gate

After the user quit Ollama and rebooted, the clean-state baseline passed: boot time was Aug 13 at
21:24:57 EDT, free disk was 74 GiB, swap use was zero, `memory_pressure -Q` reported 90% free, the
production artifact still matched the byte count/hash above, and no Ollama, MLX, Python training,
export, Xcode build, Simulator, or local-model workload was live. Figma's background agent remained
at approximately 43 MiB RSS; it was recorded and accepted as non-heavy. No system wired-limit
override was used.

The first attempted run stopped before validation because the new parameter gate incorrectly used
the packed quantized array count as its denominator. The numerator was the expected 144,919,843,
but the packed denominator produced a false 11.13% ratio. The runner was corrected to use
`mlx_lm.utils.get_total_parameters`, exactly matching MLX's own logical count. This attempt loaded
no training batch, wrote no checkpoint, and restored the prior cache and wired limits.

The corrected two-update preparation pass then reported the required
`3.131% (144.920M/4628.569M)` trainable parameters. It used 128 tokens, batch size 1, `1e-8`, seed
123, 512 train samples, 32 validation samples, cache limit zero, and the normal
19,069,665,280-byte recommended wired limit. Initial validation was `0.006634833`; update losses
were `0.004333496` and `0.006622314`, with no skipped/non-finite update. MLX peak memory was
18,566,660,787 bytes and cache memory stayed zero. It saved a float32 trainable accumulator and
full Adam state at iteration 2, cleared cache, computed an expected next update, and restored both
MLX limits.

The mandatory fresh-process test rejected that checkpoint: its same-window validation fingerprint
was `0.0080108642578125` after reload versus `0.0091552734375` before exit. Investigation found that
the checkpoint stored trainable values and Adam state but recreated the frozen quantized student
tensors, which is not an exact restart contract. The runner was strengthened to persist/reload the
entire quantized student as well as the float32 accumulator and Adam state, and to require two
identical validation fingerprints in the same process before writing a checkpoint.

The strengthened preparation run exposed a deeper blocker before serialization: its two immediate,
same-process evaluations of the same first two validation batches produced
`0.0060882568359375` and `0.00543212890625`, outside the fixed `1e-7` tolerance. Its initial full
validation also differed from the prior identical process (`0.006122981` versus `0.006634833`). The
two update losses remained finite (`0.006011963`, `0.004852295`), peak MLX memory again stayed under
the recommended ceiling at 18,566,660,787 bytes, swap stayed zero, free disk never entered the
60-GiB reserve, and no monitor stop occurred. The runner stopped before saving the incomplete
`checkpoint-000002` directory and restored both MLX limits.

This failure invalidates validation-gated training and exact restart fidelity under the installed
MLX/Metal implementation. Per sections 3, 5, and 6, the retry stops here. No 256-token diagnostic,
candidate training, bridge, export, device comparison, artifact replacement, or manifest update was
performed. Production remains on the exact 3,862,121,696-byte artifact with SHA-256
`9a6345f1a6cd39283f957977c84d31cc63b8dd56f2b8fffeb784940f63365282`.

Raw evidence is preserved below `Aquinas_Backend/runs/dwq-controlled/`:

- `2026-08-13-128-prepare.log` — SHA-256
  `4eaa8fcc08af24650eb9bb556616ed26071fb8805459569f59222fc8fc67f7f0`
- `2026-08-13-128-prepare-2.log` — SHA-256
  `5edcac4925a6da0a805a76b5b0a27fa887e36dd87fee1b8da685cf36b9502fb4`
- `2026-08-13-128-resume.log` — SHA-256
  `8d2825ef5b90b6bde736f6bcea173777b9adbee0e8511fd69827dda11bfb06e2`
- `2026-08-13-128-prepare-3.log` — SHA-256
  `be3592646b84ea67df7e1b6a94a4cad279464ba235427fa42661263abe73ffa7`

The final controlled runner is `a620f374fd12b83573e09790ab05fa9e8afee4678ed3e28c3cab50617dab15a2`.
Monitor records show zero swap use/growth, no `U` process state, minimum free disk of
78,093,807,616 bytes, and maximum monitored process RSS of 9,933,488 KiB. Final machine state was
73 GiB free, zero swap, 85% memory free, no DWQ/monitor/export process, and restored normal MLX
limits. Rejected/incomplete checkpoints and logs are retained for review; none is a candidate.

#### Product recommendation — do not continue DWQ for the current shipping checkpoint

Treat this investigation as closed, not as an experiment awaiting another near-term retry. The
shipping 8-bit decoder already begins extremely close to its bf16 teacher, while the controlled
attempts encountered non-finite gradients, validation regression, Metal memory limits, and finally
nondeterministic validation that prevents reliable resume and improvement judgments. More memory
could address OOM risk, but it would not by itself fix numerical/deterministic evaluation or make
the already-small correction materially valuable.

Keep the bridge, controlled runner, monitor, and historical logs as research tooling. Do not spend
additional product time or hardware budget on DWQ for this exact 8-bit checkpoint. Reconsider only
if at least one of these conditions changes:

- a future production checkpoint has measurably worse quantization error or a clear quantization-
  related quality/stability problem;
- the product intentionally moves to a smaller lower-bit package where DWQ has meaningful error to
  correct and a concrete size/latency payoff;
- deterministic teacher/student validation is first proven and substantially stronger hardware is
  already available; or
- a predeclared benchmark shows a credible user-visible benefit large enough to justify the work.

For the current app, prioritize checkpoint/data quality, production-path evaluation, and runtime
optimization instead. Production should remain unchanged.
