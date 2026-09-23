# Rotated Ternary Compression — Research Plan

**Status:** proposed research plan; no model, runtime, or app behavior has changed.

**Planned implementation branch:** `research/rotated-ternary-poc` (creation is a future action).

**Review:** September 18, 2026. Targets below are research acceptance proposals, not measurements. This review changes the plan only.

**Purpose:** determine whether a Gemma 4 12B Aquinas checkpoint can be represented and executed in a rotated ternary format that fits a base iPhone-class memory budget while retaining enough theological reasoning quality to improve on the current deployed mobile model.

**Product objective takes precedence:** improve useful, grounded, reliable local Aquinas answers within the phone's memory, latency, and energy limits. Rotated ternary 12B is one candidate means, not a required product outcome. Section 1A determines whether its implementation is justified before custom compression begins.

This is an experiment, not a release commitment. The first task is to establish whether the idea survives measurement; it is not to force a compressed model into the app.

**For an implementing agent:** start with Section 11. It defines the task order, repository map, required artifacts, implementation contracts, and handoff procedure. Sections 2–7 define the technical requirements and acceptance gates. No implementation task is complete merely because its code exists.

## 0. Current progress and next action

**Last updated:** September 18, 2026 — planning review and implementation-handoff design.

**Current stage:** planning complete enough to begin preflight; implementation has not started.

**Current task:** **RT-00 — Workspace inventory**, `not_started`.

**Last passed implementation gate:** none. No conversion, training, runtime changes, or physical-device tests have been performed as part of this plan review.

### Completed so far

- Reviewed and strengthened the research scope, mathematical contract, storage arithmetic, evaluation protocol, resource limits, and phase gates.
- Inspected the local `Aquinas-Final-HF/config.json`; its declared architecture does not establish the existence of the proposed tuned 12B teacher.
- Checked the referenced Google configuration and Prism packing documentation; identified rotation-width compatibility and tied-embedding storage as mandatory decisions.
- Established the fallback approach: if no tuned 12B teacher can be verified, evaluate a pinned stock instruction-tuned 12B before deciding whether adaptation is warranted.
- Defined RT-00 through RT-13, repository entry points, component contracts, required evidence records, and an agent handoff procedure.
- Added a root-cause and alternative-selection gate so custom 12B compression proceeds only when it addresses a demonstrated product gap better than simpler feasible options.
- Checked the plan's task numbering, referenced implementation entry-point paths, and basic Markdown formatting. These document checks do not validate model or runtime behavior.

### Implementation tracker

All statuses below refer to work under this plan, not historical experiments in sibling repositories. Detailed requirements and dependencies are in Section 11.

| Task | Status | Evidence / remaining work |
| --- | --- | --- |
| RT-00 — Workspace inventory | `not_started` | Verify repositories/worktrees, instructions, active experiments, native source provenance, and research artifact locations; create the initial handoff record |
| RT-01 — Teacher and decisions | `not_started` | Complete checkpoint inventory and select a verified teacher; resolve or explicitly track target, budget, and execution-machine decisions |
| RT-02 — Evaluation and baseline | `not_started` | Diagnose failure causes; compare bounded alternatives; record the Section 1A route decision; build separated datasets, evaluator, numerical acceptance configuration, measured baselines, and memory ledger |
| RT-03 — Runtime compatibility audit | `not_started` | Audit pinned runtime source, reproduce its build, and establish normal-model parity and backend coverage |
| RT-04 — Format and algebra reference | `not_started` | Specify and test packing, transforms, scales, tensor treatment, and independent reference decoding |
| RT-05 — Packed native kernels | `not_started` | Implement and validate CPU/iOS Metal operations on representative real shapes |
| RT-06 — Early phone feasibility | `not_started` | Measure disposable-probe allocations and timings against the declared phone budget |
| RT-07 — Streamed full conversion | `not_started` | Convert, reload-validate, evaluate, and run a full-model phone feasibility test |
| RT-08 — Calibration and sensitivity | `not_started` | Run bounded calibration/reconstruction and precision ablations; select by development evidence |
| RT-09 — Conditional QAT | `not_started` | Determine necessity after calibration; if justified and budgeted, validate a pilot, resume fidelity, and refinement |
| RT-10 — Optimize and freeze candidate | `not_started` | Meet measured memory/speed targets and freeze artifact/runtime/configuration hashes |
| RT-11 — App contract and lifecycle validation | `not_started` | Validate local tasks, install/rollback, cancellation, lifecycle, memory, and thermal behavior |
| RT-12 — Blind final evaluation | `not_started` | Run the sealed evaluation and report quality gates and uncertainty |
| RT-13 — Decision and handoff | `not_started` | Deliver an evidence-backed promote/continue/stop decision and reproducibility package |

### Open decisions and blockers

- **Teacher identity:** no Aquinas-tuned 12B checkpoint has been verified. RT-01 must resolve this before teacher-dependent experiments.
- **Device/context:** base iPhone 17 with 4,096 total tokens remains a provisional target.
- **Budget/access:** research-time and paid-compute ceilings, remote-compute permission, teacher execution resources, and device availability remain unresolved; this does not prevent bounded inventory work.
- **Technical choices:** rotation block policy, packing, embedding precision, and a measured numerical memory cap await their assigned investigation tasks.

**Immediate next action:** perform RT-00. Read applicable repository instructions, inspect repository/worktree status and active experiments, identify the native runtime source behind the existing XCFramework, then create `research/rotated-ternary/STATUS.md` in the Foundations repository with the verified workspace map and next RT-01 action. This is a proposed output path, not an existing completion artifact.

### Progress-update rules

This section is the canonical quick progress summary. The detailed `STATUS.md`, run manifests, and gate records are its supporting evidence; they do not replace updating this document.

1. Update the date, current task, tracker row, evidence links, open blockers, and next action whenever a task starts, passes, fails, is blocked, or is skipped with a reason, and at every agent handoff.
2. Use the task states defined in Section 11. Mark a task `passed` only when its completion criteria have linked evidence. Describe partial work explicitly without marking the entire task passed.
3. Link the latest run/report and gate record from the relevant tracker row. Keep detailed commands, logs, hashes, and failed attempts in the supporting records rather than expanding this summary indefinitely.
4. Update this section and the detailed handoff record together. If they disagree, inspect the evidence and reconcile them before dependent work; do not assume the more optimistic status is correct.
5. Replace superseded open questions with the decision and its evidence. Preserve historical results in phase reports. Never present historical tests, planning edits, or proposed paths as completed implementation work.

## 1. Decision summary

The proposed product target is an Aquinas-capable **Gemma 4 12B** used for text-only inference. An existing Aquinas-tuned 12B checkpoint has **not yet been verified**. The inspected `../Aquinas_Backend/models/Aquinas-Final-HF/config.json` instead declares `Gemma4ForConditionalGeneration`, hidden size 1,536, and 35 layers. Do not assume its weights or adapters transfer to the 12B architecture. Phase 0 must identify the intended teacher and establish its quality; if only a stock 12B exists, domain adaptation is a separate, budgeted dependency.

Qwen remains valuable as a technical reference because the public Bonsai work and its custom runtime were validated on Qwen hybrid-attention models. We may use the existing Qwen/llama.cpp experiment to understand the format and validate a minimal kernel path, but Qwen is not the planned replacement model.

The target format is inspired by the publicly described Bonsai 2 representation:

- large language-model matrices represented by ternary codes `{-1, 0, +1}`;
- one FP16 scale per group of 128 weights;
- a fixed, signed, blockwise Walsh–Hadamard rotation applied to activations at inference time (the reference width is 1,024; Gemma dimensions require the explicit policy in Section 3);
- small, numerically sensitive tensors retained at BF16/FP16 where measurements justify it;
- custom kernels that consume packed weights directly rather than dequantizing a whole tensor to FP16.

The public documentation describes this representation and inference behavior. It does **not** publish the full conversion/calibration/refinement recipe that preserves Bonsai’s reported quality. Our work must therefore treat quality retention as an open research question.

## 1A. Choose the intervention before building it

### Working diagnosis, not an assumed model-size problem

The project history contains evidence of conversion/runtime degradation, factual/citation errors, weak use of supplied passages, and training-format failures. These findings do not establish that every current failure has the same cause or that a larger model alone fixes it. `MODEL-INTEGRATION.md` and the Qwen case study contain statements from different implementation dates; RT-00/02 must verify the current artifact, active retrieval path, and actual phone behavior rather than resolve historical inconsistencies by assumption.

Use a **40–60-case development diagnostic set**, including known failures and fresh variants, before an expensive model search. Include reasoning without retrieval, source-sensitive questions with verified evidence, absent/irrelevant/conflicting evidence, and multi-turn/local task contracts. This is a screening set, not the sealed final suite and not sufficient evidence for promotion.

For matched cases, compare:

1. The current production package and pipeline, with raw engine output and postprocessing/retries recorded separately.
2. The same underlying tuned checkpoint at a reliable higher precision/runtime, where available. Verify semantic prompt, template, token limits, and retrieved passages match. This separates deployment loss from limitations already present in the teacher.
3. The deployed model with the actual retrieved passages versus a manually verified relevant passage supplied through the same prompt contract. Separately compare absent evidence on source-dependent cases. This distinguishes retrieval failure from inability to use evidence; it is a diagnostic, not permission to silently change retrieval for one contender in a model benchmark.
4. A selected stock instruction-tuned model versus its tuned counterpart, if both already exist and their lineage is verified. This tests whether adaptation helped behavior while harming instruction-following, grounding, or general capability.

Record each failure as `retrieval`, `evidence-use`, `reasoning/capacity`, `adaptation/data`, `conversion/numerics`, `prompt/template`, `application-state`, or `unresolved`, with the discriminating comparison and evidence. Multiple labels are allowed; do not infer a root cause from a fluent or incorrect answer alone.

### Bounded alternative comparison

Limit the initial comparison to the current model, **one** credible smaller model/standard-precision alternative, and the proposed 12B quality reference. Choose candidates from available verified artifacts first. Add another candidate only for a specific unresolved hypothesis and within the phase budget; do not turn preflight into an open-ended leaderboard search.

| Route | When it is the better next step | Evidence required before choosing it |
| --- | --- | --- |
| Correct current export/runtime/pipeline | The same weights answer well before deployment but fail afterward, or conversation state/template behavior explains the regression | Matched-input parity/localization of the defect and a bounded correction estimate; repeat known regression cases |
| Improve local retrieval and evidence use | Correct passages are missing, or a model cannot reliably answer from supplied evidence | Retrieval coverage/ranking/metadata tests, gold-passage answer tests, and current phone integration verification; prefer instruction-tuned baselines before adding training |
| Use a smaller model at supported precision | It meets the answer-quality and device gates without a new quantizer/runtime format | Same production-task and retrieval comparison, physical-device fit, and measured sustained speed/reliability |
| Use an already published low-bit model | A compatible, licensed artifact offers a lower-cost test of the memory/quality tradeoff | Exact artifact/runtime provenance, relevant task quality, and phone measurements; a public benchmark or desktop load is insufficient |
| Develop rotated-ternary Gemma 12B | The 12B teacher has a material useful-quality advantage and normal representations cannot deliver it within the phone budget | Teacher advantage under matched evidence, remaining compression-loss allowance, credible size/kernel plan, and approved research resources |

An official [Gemma 4 12B QAT Q4_0 GGUF](https://huggingface.co/google/gemma-4-12B-it-qat-q4_0-gguf) is available as a potential supported-format control (checked September 18, 2026). QAT changes the source weights; label it as a separate model variant, not a same-weights packing ablation or BF16 teacher. Four-bit storage for 12 billion weights already implies about 6 GB of codes before overhead, so its usefulness as a Mac quality reference does not establish phone fit. No download or fit result is implied here.

**RT-02 route-selection gate:** write `route-decision.md` with the diagnostic findings, alternative comparison, measured versus estimated quality/memory/latency, maintenance/compute cost, and chosen next intervention. For custom compression, demonstrate a meaningful 12B teacher advantage on development evidence, not merely a win against a damaged deployment. Proposed screening requirement: at least a five-point gain on the frozen 0–100 reasoning/usefulness rubric versus the best feasible control, with paired uncertainty reported and no critical grounding regression. If the evidence is ambiguous, add a bounded diagnostic or mark the route inconclusive; do not infer a quality advantage from parameter count.

Before committing to refinement, freeze the maximum acceptable loss from that teacher such that the expected compressed candidate would still beat the best feasible control. The Section 7 five-point non-inferiority allowance is a ceiling, not automatically an affordable loss: if the teacher's advantage is smaller than that allowance, tighten it or reject the route. Final acceptance must beat the current mobile package **and the strongest feasible control selected before the sealed test**; being better than the current package alone may not justify a custom runtime.

Record one of `repair_current_path`, `improve_grounding`, `evaluate_supported_model`, `proceed_rotated_ternary`, or `inconclusive`. This selects the next workstream; it does not authorize deployment or silently expand this compression project into a corpus/training rewrite. If another route wins, produce RT-13's bounded handoff for that route and pause custom-compression implementation. RT-03 read-only reconnaissance and inexpensive reference mathematics may inform the decision, but RT-05 onward require `proceed_rotated_ternary`.

### Separate quality recovery from behavior adaptation

First establish what the chosen instruction-tuned teacher can do using correct prompts and evidence. Then measure losses introduced by compression. Restore those losses with the smallest effective calibration/refinement scope. Add Aquinas behavior adaptation only for a remaining demonstrated gap; do not change the model's behavior dataset and compression recipe in the same unexplained experiment. If adaptation is necessary, compare pre/post-adaptation development quality, recheck teacher advantage, and validate the final compressed artifact again. Retrieval corpora remain supporting evidence rather than automatically becoming fine-tuning data; any approved behavior-training examples need correct prompt/evidence/answer structure and separate data provenance.

Include energy/thermal cost in route selection. A larger ternary model still executes a larger graph, transforms activations, and reads packed weights each generated token; smaller files alone do not imply better latency or battery behavior. A simple screening lower bound is `weight bytes read per token × tokens/s` for required weight bandwidth (about 25.6 GB/s for 3.2 GB read once at 8 tokens/s). Measure actual traffic, kernel cost, and sustained performance; the bound excludes cache/activation traffic and is not a throughput prediction.

## 2. Success criteria and hard limits

### Product goal

A text-only local Aquinas model that is materially more capable than the present phone package and remains practical on the base supported phone.

### Initial engineering target

| Measure | Target | Gate |
| --- | --- | --- |
| Packed language weights | Aspirational 2.5–3.3 GB; derive actual size from tensor inventory and packing | No full FP16 expansion during inference; include retained tensors and metadata |
| Phone resident memory | Must remain below the device’s practical per-app limit with a usable KV cache | Measure on physical hardware; simulator is not enough |
| Decode throughput | Interactive; initial target ≥8 tokens/s, aspirational target ≥15 tokens/s | Physical-device measurement after warm-up |
| Context | Proposed 4,096 total tokens, including prompt, retrieval, history, reasoning, and output; reserve 512 for output | Confirm minimum requirement; 2,048 is a separately labeled fallback, not a silent pass |
| Quality | Beat the current mobile package on a blinded Aquinas quality suite without new corruption/repetition regressions | Evaluate before any app promotion |
| Reliability | No sustained memory termination, invalid UTF-8/mixed-script corruption, or uncaught repetition failure in the test protocol | Test cancellation, lifecycle, thermal, and topic shifts |

### Stop conditions

Stop or change direction if any of these occur:

1. A correct native ternary runtime cannot fit the model plus minimum useful context on the physical phone.
2. The calibrated model remains materially worse than the existing mobile package on the blinded evaluation suite.
3. Quality recovery needs full-scale retraining or hardware resources beyond the project’s approved budget.
4. The runtime requires a non-maintainable fork that cannot be isolated, tested, and updated safely.
5. The selected teacher does not itself beat the mobile baseline, or the checkpoint identity cannot be established. Resolve the model-selection problem before compression.
6. The declared phase time/compute budget is exhausted without meeting its exit gate. Record an inconclusive or failed result; do not extend runs automatically.
7. A simpler feasible route meets the product gates with comparable or better quality and lower lifecycle cost. Stop custom compression and record the successful alternative; preserving this research direction is not itself a success criterion.

### Decisions to freeze before experiments

- **Device:** provisionally the base iPhone 17, 8 GB RAM, identified in `MODEL-INTEGRATION.md`; owner confirmation pending. Record exact OS/build and app entitlements. There is no assumed universal per-app memory limit.
- **Teacher:** exact local path or repository revision, original precision, Aquinas adapter/fusion provenance, tensor inventory, tokenizer and chat-template hashes. A dequantized checkpoint must be labeled by its original precision rather than called an uncompressed teacher.
- **User experience:** proposed warm time to first visible token ≤5 seconds for a 512-token prompt, cold load ≤15 seconds, and sustained decode ≥8 tokens/s. Measure end-to-end latency, including hidden reasoning and metadata passes. Freeze any revised thresholds before candidate selection.
- **Budget:** research-time ceiling, paid-compute ceiling, whether remote compute/data transfer is allowed, and per-phase trial limits remain owner decisions. No paid compute or long refinement run is authorized by this document alone. Until resolved, limit work to bounded preflight and reference proofs.
- **Memory:** Phase 0 must set a numerical peak physical-footprint cap and headroom policy for the supported device and app configuration, using known-good measurements and observed constraints. Target at least 15% headroom against the chosen operating cap; passing once near termination is insufficient. Do not deliberately drive the production app to jetsam to discover a limit.

## 3. What the rotation does — and does not do

For an original dense projection `W`, a lossless change of coordinates can be expressed as:

```text
W × x  =  (W × Rᵀ) × (R × x)
```

where `R` is the fixed signed Hadamard rotation. Before ternarization, this equality is exact. The rotation spreads concentrated values across a block, making the rotated matrix more amenable to a shared-scale ternary approximation.

The stored matrix is then a lossy approximation `T ≈ W × Rᵀ`, with grouped scales. At runtime the activation is transformed as `R × x`, and the low-bit matrix multiply produces `T × (R × x)`.

The runtime does **not** later reconstruct the original full-precision weights. The rotation is lossless in exact arithmetic; finite-precision transforms introduce rounding. Replacing the rotated values with ternary codes adds quantization loss. Calibration and refinement attempt to minimize the behavioral loss from that step.

### Mandatory transform and tensor contract

For column-vector notation, define `W` as `[out, in]`, normalized `H = H_raw / sqrt(b)`, a diagonal sign matrix `D`, and choose `R = H D` within each block. Then `Rᵀ = D H`, `RᵀR = I`, and the converter computes `W D H`. Applying `R` twice is **not** generally the inverse. Document the equivalent row-major/batched convention in the implementation. If the chosen Prism format uses a different ordering, implement and test that exact ordering instead of assuming compatibility.

The format contract must specify block boundaries, normalization, sign placement, sign bit ordering or PRNG algorithm/version/seed, per-tensor transform sharing, group axis, scale storage, code mapping, padding, endianness, alignment, and original versus stored dimensions. Persist the actual signs or their verifiable checksum. Unsupported versions, missing required transforms, and invalid shapes must fail before allocation or inference.

The published Google 12B configuration has hidden size **3,840**, intermediate size **15,360**, and tied embeddings with vocabulary **262,144**. A width-1,024 transform does not tile the hidden dimension. Compare an explicitly recorded mixed-block decomposition (for example 1,024 + 1,024 + 1,024 + 512 + 256), a smaller common block size, or zero-padding to 4,096. Padding requires matching activation padding and extra stored columns, counted in every byte and kernel estimate. These are research variants, not automatically Bonsai-compatible. Verify the selected local checkpoint independently. [Google configuration](https://huggingface.co/google/gemma-4-12B-it/blob/main/config.json)

Apply rotations at individual linear inputs; keep the residual stream in its original basis. Do not commute transforms through normalization, nonlinearities, RoPE, attention, or residual additions without a separate equivalence proof. Shared Q/K/V or gate/up inputs may reuse a transformed activation only when their complete transform contracts match. Bias stays in the output basis.

Treat tied embeddings/output heads separately: rotating stored rows changes embedding lookup results. Either retain an unrotated representation, implement and verify the inverse transform on gathered embeddings, or account for untied storage. Preserve embedding scaling and output-head behavior. Do not assume this matrix is a small precision exception.

### Actual storage arithmetic

For an illustrative **12 billion eligible weights**, before padding, retained tensors, and container overhead:

| Representation | Bits per weight including group scales | Bytes (decimal GB) |
| --- | ---: | ---: |
| Ternary information lower bound + FP16/128 scale | `log2(3) + 16/128 ≈ 1.710` | 2.565 GB; not an implemented random-access format |
| Compact format at 1.75 bits/weight | 1.75 | 2.625 GB |
| Two-bit codes + FP16/128 scale | 2.125 | 3.188 GB |

Prism documents `PTQ1_0` at 1.75 bits/weight and `PQ2_0` at about 2.13; storage and prompt-processing speed differ. Select the actual layout before asserting fit, and test each packing against the same quantized values to separate packing from model quality. [Prism format documentation](https://github.com/PrismML-Eng/Bonsai-demo/blob/main/MODEL-FORMATS.md)

Use `sum(packed payload + scale bytes + padding + retained tensor bytes) + metadata`, with tied storage counted once only if the runtime actually shares it. At the published dimensions, the embedding table alone has about 1.007 billion entries, or **2.013 GB at FP16**. Keeping it FP16 rather than group-128 two-bit ternary adds approximately **1.746 GB** to the illustrative all-ternary estimate. This can invalidate the 3.3 GB target before KV cache is allocated; evaluate intermediate precision for large sensitive matrices as an explicit ablation.

## 4. Research environment and source-control plan

1. Start from the existing `Aquinas-iOS-llama-cpp-12b` / corresponding backend experiment rather than altering the working LiteRT path.
2. Create branch `research/rotated-ternary-poc` and a separate worktree.
3. Keep the current app branch, production LiteRT manifest, and device seed untouched.
4. Record the precise base-model revision, tokenizer revision, conversion configuration, calibration corpus revision, runtime commit, compiler options, device/OS build, and all hashes in an experiment manifest.
5. Treat PrismML’s public runtime as a reference dependency. Preserve its license and attribution; do not represent unpublished conversion details as known or recovered.
6. Record separate repository/worktree paths and base commits for backend conversion, native runtime, and iOS probe; a single branch name does not isolate sibling repositories. Inventory existing work and do not reset or repurpose an occupied experiment checkout.

No research artifact may replace the current deployed model until it clears the gates in this document.

## 5. Phased implementation plan

### Phase 0 — Preflight and baselines

**Objective:** establish a reproducible baseline before changing a weight.

1. Inventory available checkpoints and verify the selected Gemma 4 12B teacher, tokenizer, tuning provenance, and intended text-only configuration. If no tuned 12B is found, first evaluate a pinned stock instruction-tuned 12B using Aquinas prompts/retrieval. Add adaptation only if this baseline shows a specific recoverable domain gap and a separate resource estimate justifies it; do not transfer incompatible adapters.
2. Measure the dense/BF16 or existing Mac runtime on the Aquinas evaluation prompts.
3. Measure the current mobile package with the real production conversation path, not a bare debug prompt.
4. Create a compact, versioned blind evaluation suite covering:
   - Aquinas’s theological definitions and distinctions;
   - source-sensitive factual questions with grounding;
   - multi-step moral reasoning;
   - cited uncertainty and refusal boundaries;
   - topic shifts, follow-ups, and long responses;
   - repetition, malformed output, and multilingual-corruption regressions.
5. Capture disk, memory, load time, prefill speed, decode speed, and output samples for each baseline.
6. Produce a tensor census with names, shapes, exact parameter counts, alias/tie relationships, dtype, quantization history, and planned treatment. Inventory every tensor needed for text inference; do not remove tensors solely because their names appear multimodal. Verify text-only equivalence after exclusions.
7. Add a standard supported quantization of the same teacher (for example Q4/Q5, on Mac if necessary) as a control. Define the later rotation-only, unrotated ternary, rotated ternary, calibrated, and refined comparisons with fixed data/seeds; execute each when its implementation exists. Distinguish runtime, packing, rotation, and learning gains.
8. Build a memory ledger for weights, KV, prefill activations, FWHT scratch, graph/work buffers, logits, app/retrieval overhead, and transient loader/staging/repacking copies. Include the largest individual Metal allocation. Estimate KV from the actual layer/head dimensions and cache dtype: `sum_l(tokens_allocated_l × kv_heads_l × (key_dim_l + value_dim_l) × bytes_per_element)`, adjusted for implemented sharing, separate K/V dtypes, quantization metadata, and padding. Sliding-window attention does not guarantee a bounded allocation unless the runtime implements it.
9. Execute Section 1A's diagnostic comparisons and bounded alternative screening, then record the route-selection decision. Compression-specific implementation beyond the minimal reference work requires a justified `proceed_rotated_ternary` decision.

**Exit gate and deliverables:** identified teacher that improves on the mobile baseline, versioned baseline manifest/evaluator, exact byte ledger, frozen operating targets, and a checked-in measurement report. A BF16 12B model alone is approximately 24 GB; the 24 GB Mac cannot be assumed to run it with usable working memory. Declare a feasible teacher execution strategy (larger authorized machine, bounded offload, or an explicitly labeled higher-bit proxy) before scheduling baseline or reconstruction runs. A proxy does not establish loss against a BF16 teacher.

### Phase 1 — Runtime reconnaissance

**Objective:** identify the smallest safe Gemma-compatible extension of the existing llama.cpp experiment.

1. Diff PrismML’s llama.cpp fork against its upstream base to isolate:
   - custom low-bit tensor layouts;
   - Hadamard/FWHT graph operations and hints;
   - Metal/CUDA/CPU kernel changes;
   - model-loader metadata and architecture-specific handling.
2. Determine whether Gemma 4 needs architecture changes beyond ordinary llama.cpp tensor mapping.
3. Confirm that the iOS build can consume the required native library without weakening app isolation or changing production runtime selection.
4. Implement unit tests for the signed FWHT: norm preservation, deterministic signs, inverse-equivalence expectations, and FP16 numerical tolerance.
5. Record a support matrix for every required tensor type and graph operation on CPU, macOS Metal, and **iOS Metal**. Desktop support is not evidence of phone support. Audit prefill and decode separately, GPU fallback, maximum buffer sizes, and implicit dense expansion or weight repacking.
6. Verify architecture-specific attention, cache sharing, RoPE, normalization, logit handling, tied embeddings, tokenizer, EOS, and prompt-template behavior against the teacher. Compare fixed teacher-forced logits/activations before free generation so numerical drift is not confused with prompt or tokenization errors.

**Exit gate:** a normal Gemma checkpoint still produces baseline-equivalent output through the isolated research runtime.

### Phase 2 — Offline mathematical conversion proof-of-concept

**Objective:** prove the transform and container are correct on a small, controllable subset before converting a 12B model.

1. Begin with synthetic matrices using the target model's real shapes, including non-1,024-divisible widths, then a real block from the selected teacher. A smaller Gemma checkpoint may have a different architecture and is not sufficient compatibility evidence.
2. Define the orientation explicitly. The converter must produce a stored matrix approximating `W × Rᵀ` for the runtime’s tensor convention.
3. Generate and persist fixed sign vectors and transform metadata. They must be deterministic and included in the artifact manifest.
4. Implement group-128 ternary assignment and FP16 group scales.
5. Compare dense and transformed-dense layer outputs before quantization; they should match within numerical tolerance.
6. Compare transformed-dense and ternary layer outputs after quantization; record error distributions, worst groups, and output/logit divergence.
7. Add a round-trip/format reader test so that every packed code and scale is interpreted identically by the converter and runtime.
8. Define the initial quantizer reproducibly: minimize group weight MSE with nonnegative scale `s`, codes in `{-1,0,+1}`, fixed tie-breaking, and deterministic initialization/iteration limits; for fixed nonzero codes update `s = dot(w,t)/dot(t,t)`. Specify all-zero groups, rounding to stored FP16 scales, overflow/underflow, and rejection of non-finite values. Report zero-code fraction and error after scale serialization. Activation-aware calibration is a later, distinct objective.
9. Use an independent FP64/FP32 algebra reference and CPU unpacker. Test impulses, zeros, outliers, random and non-contiguous inputs, partial blocks, all codes, invalid reserved codes, truncated files, and mismatched metadata. Freeze absolute/relative error tolerances by dtype and shape before assessing kernel results; exclude numerical near-zero denominators from relative-only assertions.

**Exit gate:** reference transform equivalence and format tests pass; CPU and minimal iOS Metal kernels consume packed representative tensors directly and match the independent reference for prefill and decode. Inspect buffer allocations to rule out full-weight expansion. This small correctness kernel precedes Phase 5 optimization.

### Phase 2B — Early physical-device feasibility

**Objective:** reject an impossible phone target before expensive calibration.

Use a disposable probe bundle with representative real layer shapes and the planned full-model allocation layout. Measure packed matvec/matmul, FWHT, largest-buffer creation, staging peaks, and projected full-model decode cost. Include the minimum KV/app headroom from the ledger. Synthetic allocation and summed layer timing are screening evidence only; they cannot establish real model fit or quality.

**Exit gate:** no known allocation/format blocker and a plausible measured route to the frozen memory/speed targets. If this fails, change packing, context, or model size through a documented decision before full conversion. Run an actual full-model phone load/prefill/decode smoke test immediately after Phase 3 and before any long Phase 4 refinement, even if its answers are poor.

### Phase 3 — Naïve end-to-end Gemma conversion

**Objective:** obtain a real, runnable baseline for measuring how much quality is lost before trying to recover it.

1. Classify every Gemma tensor into one of:
   - ternary + rotation eligible;
   - ternary without rotation, if architecture requires it;
   - retained BF16/FP16 because it is stateful or numerically sensitive;
   - explicitly out of scope (for example, optional vision weights in the first text-only experiment).
2. Convert the whole text model in a streamed/chunked fashion; never require a second full dense checkpoint on disk.
3. Package the provisional model with unambiguous custom-format metadata so unsupported runtimes reject it rather than silently generate invalid text.
4. Run it first on Mac with a small context and deterministic decoding.
5. Measure perplexity/logit divergence plus the complete Aquinas development evaluation suite. Preserve all failures, not just fluent examples. Reserve the sealed final suite for the selected candidate.
6. Write to a temporary artifact, validate tensor counts/shapes/checksums in a fresh process, then atomically finalize it. Never overwrite the source or last accepted artifact. Bound chunk size and scratch space; partially completed output is not runnable.

**Exit gate:** every expected text tensor is accounted for; teacher-forced numeric checks and end-to-end decoding pass; full-model physical-device feasibility is recorded. Low quality is permitted at this stage, silent format errors and infeasible memory are not.

**Expected outcome:** this model may be poor. Its value is a quantified error baseline and a working end-to-end stack.

### Phase 4 — Calibration and quality recovery

**Objective:** improve the ternary approximation without assuming access to PrismML’s unpublished recipe.

Work incrementally; evaluate after each change.

1. **Scale and threshold calibration:** optimize per-group scales and ternary thresholds against representative Aquinas prompts and general language data.
2. **Layerwise reconstruction:** optimize one layer or block at a time to reproduce teacher activations from the uncompressed Aquinas checkpoint.
3. **Sensitivity study:** promote only demonstrated-sensitive tensors, prioritizing small tensors for BF16/FP16. Large matrices, particularly tied embeddings, require explicit intermediate-precision or FP16 byte accounting and a renewed memory gate. Every precision exception must state its byte cost and measured gain.
4. **Low-bit refinement/QAT:** if calibration is insufficient, refine scales and latent weights with a straight-through ternary constraint and teacher-logit/hidden-state distillation. Keep the training scope, data license, compute budget, and checkpoints explicit.
5. **Regression control:** require the quality suite to improve without degrading grounding, safety, entity preservation, or the repetition/corruption protections.
6. **Training feasibility:** profile a single layer/block first, including teacher activations, gradients, latent weights, optimizer state, and checkpoint writes. Do not co-reside dense teacher and full trainable student on the 24 GB Mac. Use bounded activation shards and explicitly priced offload/remote execution if needed. Full-model QAT remains conditional on a credible resource plan.
7. **Reproducible recovery:** save all trainable and frozen quantization state, signs, optimizer/scheduler, RNG, data order, and precision map. Prove fresh-process reload and continuation on a short run before a long job. Stop on non-finite values, held-out regression beyond the frozen tolerance, disk/swap limits, or monitor/checkpoint failure. Prior DWQ work documented restart-fidelity and memory failures; reuse those lessons, not unvalidated checkpoints (`Aquinas-QAT-DWQ-Writeup.md`).

This phase is the principal unknown. It may establish that a 12B model needs a more capable training run than the project can justify. That is a valid result.

### Phase 5 — Kernel and memory optimization

**Objective:** turn a correct model into a memory-resident, interactive one.

1. Profile one-token decode to determine time spent in low-bit GEMM, unpacking, scale application, activation rotation, cache access, and launches.
2. Fuse ternary unpacking, group-scale application, and matrix multiplication. The implementation must not materialize full FP16 weight matrices.
3. Fuse fixed sign multiplication into the FWHT load path where feasible.
4. Start with Metal for the intended Apple devices; retain a CPU reference backend for correctness tests.
5. Measure `tg128` (single-stream generation) and `pp512` (prompt processing) separately.
6. Introduce a bounded low-bit KV cache only after baseline cache behavior is validated. Measure long-context quality separately from short-context quality.
7. Re-run mathematical and development-set quality checks after kernel/precision changes. The sealed final test follows candidate freeze. Measure temporary per-tile unpacking separately from retained full matrices; bounded register/threadgroup tiles are acceptable. Report actual backend placement and any CPU fallback.

**Exit gate:** peak resident memory and throughput are measured on the base supported phone, not inferred from artifact size.

### Phase 6 — iOS research integration

**Objective:** test the model safely without disturbing the shipping LiteRT experience.

1. Add a research-only model/runtime selection behind a Debug-only flag or separate probe target.
2. Use a disposable app container and preserve existing device data; never remove the production app container for a model experiment.
3. Enforce model hash, byte count, minimum free-space check, bounded context, cancellation, and one live-engine ownership.
4. Test cold load, warm load, 128-token decode, prompt processing, sustained generation, background/foreground transitions, memory pressure, and device temperature.
5. Run blinded answer review before showing candidate outputs as a potential product replacement.
6. Cover every intended local task contract: prose, key-term metadata, definitions, compaction, and structured outputs. Record raw model output and the final postprocessed answer separately, including every retry and its latency. Reject corrupt candidates even if a recovery layer hides the symptom.
7. Verify a production app-data backup before device experiments. Test atomic install, truncated/wrong-hash artifacts, insufficient space, failed initialization, rollback to the known-good package, and engine resource release after repeated cancel/reload. Preserve production bundle identity and data.

### Phase 7 — Decision and promotion

Produce one of three explicit outcomes:

- **Promote:** candidate meets quality, memory, reliability, and speed gates; prepare a versioned downloadable artifact and controlled app integration plan.
- **Continue research:** runtime and footprint work, but calibration/refinement has a plausible measured next step.
- **Stop:** the quality/engineering cost does not justify replacing the current package; retain reusable runtime and evaluation findings.

Promotion requires an independent code review, repeated physical-device testing, and an update to `MODEL-INTEGRATION.md`. It does not happen merely because the model loads.

## 6. Storage and machine budget

The original plan recorded **104 GiB free at creation**; this is a historical observation, not a current measurement. Recheck before every run. Use decimal GB for model bytes and GiB for system resource measurements, with both recorded in manifests.

| Resource | Planning allowance | Practice |
| --- | ---: | --- |
| Gemma 12B dense/BF16 source | ~24–30 GB | Reuse an existing verified checkpoint; do not duplicate it casually |
| Ternary artifact | Derive from the tensor ledger; 2.5–3.3 GB is aspirational | Keep versioned only when associated with a measurement manifest |
| Conversion/calibration scratch | ~15–30 GB | Stream tensors; clean failed temporary outputs promptly after recording results |
| Evaluation/runtime builds | ~5–15 GB | Periodically remove rebuildable Derived Data and stale build products |
| Reserve for OS, swap, and recovery | ≥25 GiB | Hard stop before storage pressure compromises the machine |

Before long conversion or refinement, verify free disk, available memory, swap growth, battery/power state, active builds, and a predeclared stop threshold. Do not run concurrent heavyweight exports, iOS builds, or model training on the 24 GB Mac without measuring the combined pressure.

Budget the simultaneous peak, including input shards, output, teacher activation caches, optimizer/resume checkpoints, interrupted-run recovery, and download staging. Require free disk to exceed the remaining worst-case writes **plus 25 GiB reserve** before starting; estimate headroom for orderly checkpoint/exit rather than waiting to cross the reserve. Monitor at least every 10 seconds plus load/save boundaries; freeze swap-growth and memory-pressure stop thresholds after a bounded pilot. Keep only explicitly identified regenerable scratch eligible for cleanup; preserve raw logs and the last reload-validated checkpoint. Record phase wall-time, compute cost, attempted trials, and next decision.

## 7. Evaluation protocol

Quality cannot be established by one attractive answer or conventional perplexity alone.

Each candidate is compared against both the uncompressed Aquinas checkpoint and the current mobile package using the same production prompt path and deterministic decoding where required. Record the prompt revision, retrieval context, grounded source availability, generation settings, output, latency, and evaluator result.

Here “uncompressed Aquinas checkpoint” means the verified teacher selected in Phase 0; use its actual name and original precision in reports. If no tuned 12B exists, compare against the selected stock 12B and keep the current tuned model as a separate baseline. Match semantic prompt content and retrieval evidence; use each model's correct tokenizer/chat template rather than forcing identical token IDs across architectures. Fix truncation, output/reasoning budgets, EOS, sampling and repetition settings. Cross-model perplexity is not directly comparable when tokenizers differ; use it chiefly for same-teacher compression diagnostics.

Primary score dimensions:

1. Correct theological/philosophical distinctions and coherent explanations.
2. Faithfulness to retrieved evidence, citations, uncertainty, and named entities.
3. Multi-step reasoning and follow-up coherence.
4. No repetition loops, mixed-script corruption, malformed structured output, or inappropriate confident fabrication.
5. Device memory, responsiveness, thermal behavior, and recovery from cancellation/lifecycle events.

Blind review must hide which model produced each answer. A candidate must improve the aggregate result and must not conceal serious regressions behind average scores.

### Data separation and candidate selection

- Version disjoint calibration/training, development, and sealed final-test sets. Deduplicate by source passage, paraphrase family, and conversation, not only exact text. Keep final-test answers and teacher activations out of calibration. Record data provenance/licenses and domain/general-language proportions.
- Tune on development results only. Record all candidate trials and selection rules; evaluate the selected candidate on the sealed set once per declared research round. Using final-test feedback to retune converts it to development data and requires a new holdout.
- Proposed final suite: at least **200 independent question/conversation units**, stratified over the dimensions above, with at least 30 examples in each critical grounding/reasoning/regression category (categories may overlap). Use a pilot to assess whether this provides enough power for the intended gain; an uncertain interval is inconclusive, not success.
- Randomize paired answer order and blind model/version identity. Use anchored rubrics and source-backed answer keys. Two independent reviewers assess critical cases and a shared subset; resolve disagreements and report agreement. Model judges may assist but cannot alone establish theological accuracy or promotion.

### Proposed quality acceptance rules

Freeze these rules before looking at candidate test results; changing them creates a new experiment revision.

1. Against both the current mobile package and the strongest feasible control frozen in RT-02: paired preference score `wins + 0.5 × ties` divided by evaluation units is at least **55%**, and its conversation-clustered bootstrap 95% confidence interval lies above 50% for each comparison. If the current package is itself the strongest feasible control, one comparison suffices. Predeclare the two comparisons and require both to pass; do not select the easier comparator after final results.
2. Against the selected 12B teacher: the paired rubric score on a frozen 0–100 scale has a lower 95% confidence bound no worse than **−5 points**, or the tighter margin frozen by Section 1A when needed to preserve the teacher's advantage. Apply the same proposed non-inferiority margin to separately reported grounding and reasoning categories; insufficient evidence does not clear the gate. These are project proposals, not established scientific constants.
3. No newly introduced confirmed critical failure (for example fabricated source attribution on a grounding test, hidden-reasoning leakage, malformed required output, or an unbounded repetition loop) may be waived because the average improves. Preserve failures and rerun their regression cases after a fix. Language detection must distinguish legitimate quotations/names from unintended script corruption.
4. Evaluate both the raw engine and the production pipeline; include retry rates, extra generation work, and latency in the comparison. A better response achieved through repeated hidden retries is not free quality improvement.

### Physical-device acceptance protocol

Use the frozen device/context and a Release-equivalent optimized probe, plus a Debug integration check. Record build/OS, power mode, charging, ambient conditions, thermal state, GPU placement, and batch sizes. Keep device conditions comparable across baselines.

- At least 10 fresh-process loads and 30 warm prompt/decode measurements across three sessions. Report median, p95 latency, peak physical footprint, and p10 decode throughput; proposed p10 threshold is ≥8 tokens/s. Distinguish process-cold from a genuinely cold file cache rather than claiming the former measures both.
- Measure prompt lengths 128, 512, 2,048, and near the supported context limit with reserved output space. Include `pp512`, `tg128`, a longer answer, full-context prefill, multi-turn growth, and context eviction/compaction. Report generated token counts and time to first **visible** token, including reasoning, metadata, and recovery passes.
- At least one 30-minute sustained workload and 300 short automated requests across the supported local task types; report thermal slowdown over time, cancellation at load/prefill/decode, background/foreground, and memory warnings. Do not require unsupported background generation. Zero observed fatal failures is required for this pilot; with 300 independent trials it only bounds a failure rate near 1% at 95% confidence, and correlated runs provide weaker evidence.
- Sample process physical footprint and Metal allocations through load, prefill, decode, and teardown; reconcile allocator measurements to avoid double-counting shared unified-memory buffers. Check memory returns to a stable plateau across repeated engine destruction/recreation, and review termination diagnostics. Artifact size and RSS alone are insufficient fit evidence.
- Pass the operating cap/headroom at peak context and load transitions, and the latency/speed gates under sustained use. Record any permitted thermal-state exclusion in advance. A tiny-context or freshly cooled success does not substitute for the declared workload.

### Minimum experiment record

Each report links: hypothesis; parent/candidate IDs; repository commits and dirty diff; model/tokenizer/template/data hashes; tensor precision/packing/transform map; software/compiler versions; seeds and optimizer state; calibration sample/token counts; device/context/cache settings; numerical tolerances; commands; resource peaks; raw outputs and retry traces; per-category scores/confidence intervals; failures; cost; and a signed-off gate decision. Record “not measured” rather than filling missing evidence with estimates. Promotion additionally requires verified artifact licensing/distribution terms and a reproducible build/load manifest.

## 8. Non-goals for the first experiment

- Recreating or claiming to recreate PrismML’s unpublished training recipe.
- Replacing the current LiteRT model before physical-device and blind-quality gates pass.
- Supporting vision, tools, speculative decoding, or maximal context in the first candidate.
- Advertising an artifact size as device-fit evidence without resident-memory measurement.
- Training a model from scratch or running an unbounded full-model optimization job.

## 9. Immediate next actions

1. Inventory checkpoints and verify whether a tuned 12B teacher exists. Otherwise use a pinned stock instruction-tuned 12B baseline first, with a separate adaptation decision if needed.
2. Freeze device/context, research-time and compute budgets, and feasible teacher execution. Create isolated worktrees only when implementation starts.
3. Build the Phase 0 evaluation manifest, separated datasets, baseline suite, and tensor/memory ledger; specifically resolve the tied embedding cost and non-divisible rotation widths.
   Run the Section 1A failure diagnosis and alternative screening before committing to native compression implementation; record the route decision and the best feasible control.
4. Audit the pinned Prism runtime diff, type identifiers, and CPU/macOS/iOS support; identify reusable code versus Gemma-specific work.
5. Implement the standalone signed-FWHT and packed rotated-linear-layer reference tests, then a small iOS kernel/allocation probe before full conversion or refinement.

The next action after Phase 0 is a review of measured baselines and the runtime audit—not automatic model conversion.

## 10. Evidence and remaining uncertainty

Primary references checked September 18, 2026; pin exact source commits in Phase 0 because these URLs track moving branches:

- [Google Gemma 4 12B configuration](https://huggingface.co/google/gemma-4-12B-it/blob/main/config.json): architectural dimensions and tied embeddings; the chosen local checkpoint remains authoritative.
- [Prism model-format specification](https://github.com/PrismML-Eng/Bonsai-demo/blob/main/MODEL-FORMATS.md): packing costs, format migration, and required transform support. A known tensor type can load yet yield invalid output when rotation support is missing; test rejection on unsupported runtimes, not just your own loader.
- [Prism runtime overview](https://github.com/PrismML-Eng/Bonsai-demo): reference runtime and upstream status; public claims are not validation of an independently converted Gemma model.
- Local `MODEL-INTEGRATION.md`, `QWEN-TESTING-CASE-STUDY.md`, and `Aquinas-QAT-DWQ-Writeup.md`: deployed behavior, physical-device history, and prior training/restart failures. Historical results must be tied to exact artifact versions rather than treated as current baselines.

This review verified document assumptions and selected checkpoint metadata. It did not run conversion, audit the complete native runtime, establish a tuned 12B checkpoint, measure current machine resources, or validate phone performance. Those remain explicit experiment gates.

## 11. Agent execution and handoff guide

### Start here and resume here

1. Read this plan and the applicable `AGENTS.md` / `CLAUDE.md` in each repository before editing that repository. Read `MODEL-INTEGRATION.md` for integration work and the prior DWQ failure/restart addenda before refinement. Follow repository-required implementation skills and checks.
2. Inspect current repository status, existing worktrees, run records, and any ongoing processes. Preserve unrelated changes and active experiments. Existing experiment directory names are discovery hints, not proof of model identity or isolation.
3. Locate the experiment's `STATUS.md` and resolve its manifest and evidence paths. If none exists, begin **RT-00**. Do not infer completion from this plan, a filename, or another agent's prose; verify recorded artifacts and test results.
4. Select the first incomplete task whose dependencies are satisfied. Missing phone access does not block metadata inventory or reference mathematics; it does block a physical-device gate. A missing budget blocks expensive work, not read-only investigation.
5. Before an experiment, record its hypothesis, inputs, command/configuration, numerical/resource limits, expected output, and success criterion. Use new run IDs. Run the smallest discriminating test first.
6. Finish by updating the task record, experiment manifest, evidence links, and exact next action. Record failures and unresolved questions with the same care as successes. Never mark a dependent task passed while a prerequisite is failed, blocked, or inconclusive.

**Current implementation state:** consult Section 0 and its linked evidence records. At the initial planning handoff, the first implementation action is RT-00, followed by RT-01. Owner discretion permits choosing stock 12B for the initial baseline if no tuned 12B can be verified; it does not establish model quality or authorize paid compute.

### Repository map and proposed artifact locations

Paths below are relative to the shared `Developer` directory. Verify and record actual worktree paths in RT-00. Existing file locations were inspected during this review; **proposed** paths do not exist merely because they appear here.

| Location | Responsibility / first files to inspect |
| --- | --- |
| `Aquinas-Foundations/` | Plan, cross-repository decisions, and concise evidence reports. Keep this repository documentation-only. |
| `Aquinas_Backend/` | Checkpoint discovery, converter, calibration, mathematical reference, and evaluation orchestration. Inspect `models/Aquinas-Final-HF/config.json`, `scripts/evaluate_prompt_quality.py`, and `scripts/benchmark_latency_quality.py`; audit suitability before reuse. |
| `Aquinas-iOS-llama-cpp-12b/` | Existing runtime integration reference: `Aquinas-iOS/Services/LlamaCPPChatSession.swift`, `LlamaCPPAquinasModel.swift`, `AquinasModel.swift`, and `Features/Developer/LlamaCPPDeviceProbe.swift`. Read its instructions before creating an isolated descendant. |
| Native runtime source checkout, to resolve in RT-00 | GGUF reader/writer compatibility, tensor types, CPU kernels, graph builder, Metal kernels, and reproducible XCFramework build. A vendored binary is not a source checkout; identify its provenance before replacing it in the research target. |
| Backend research worktree: `research/rotated_ternary/` (**proposed**) | Versioned implementation modules, configuration schemas, tests, and a README containing verified commands. Reuse repository conventions if they prescribe a different layout; record the mapping. |
| Backend research worktree: `runs/rotated-ternary/<run_id>/` (**proposed**) | Manifests, logs, metrics, local datasets/activations, and candidate artifact references. Exclude large models, private data, caches, and credentials from Git. |
| `Aquinas-Foundations/research/rotated-ternary/` (**proposed**) | `STATUS.md`, decision register, phase reports, and links/hashes for evidence held elsewhere. Keep enough small fixtures and sanitized results versioned to reproduce checks. |

RT-00 must record how to rebuild the native library and which source commit produces each binary hash, plus the Xcode project, scheme, build configuration, minimum OS, probe bundle identifier, and physical-device destination. Do not invent these from an old command. Put verified commands into the implementation README as soon as discovered.

### Work breakdown and required completion evidence

| Task | Dependencies | Work and required output | Done when |
| --- | --- | --- | --- |
| **RT-00 — Workspace inventory** | None | Repository/worktree map, applicable instructions, clean/dirty status, native source/binary provenance, artifact storage plan, initial `STATUS.md` | Every intended edit/build location is identified; production resources are distinguished from disposable research resources |
| **RT-01 — Teacher and decisions** | RT-00 | Checkpoint inventory and tensor census; teacher-selection decision; target/budget decision register; feasible teacher execution strategy | Teacher identity and original precision are verified; unresolved decisions have owners and dependent tasks; no assumed tuned 12B |
| **RT-02 — Evaluation and baseline** | RT-01 for model runs | Diagnostic set and failure attribution; bounded alternative comparison and `route-decision.md`; dataset split manifests, rubric, adapters, development baselines, memory ledger, frozen acceptance configuration | Phase 0 and Section 1A gates pass on measured development evidence; best feasible control is frozen; final holdout is sealed; resource targets are numerical |
| **RT-03 — Runtime compatibility audit** | RT-00; selected architecture from RT-01 | Pinned fork/upstream diff, operation/backend matrix, build recipe, normal-model parity report, gap/maintenance estimate | Phase 1 gate passes; each required iOS operation has an implementation or a bounded explicit implementation task |
| **RT-04 — Format and algebra reference** | RT-01 and RT-03 format findings | Versioned format contract, precision/transform map, pure reference converter and independent reader, malformed-file and algebra fixtures | Shape, rotation, scale, code, padding, tied-embedding, and round-trip checks pass under frozen tolerances |
| **RT-05 — Packed native kernels** | RT-04; RT-02 selects `proceed_rotated_ternary` | CPU reference integration and minimal iOS Metal matvec/matmul/FWHT; allocation traces; graph placement tests | Phase 2 gate passes for actual model shapes, both prefill/decode, with no full-weight expansion |
| **RT-06 — Early phone feasibility** | RT-02, RT-05 | Disposable probe, layer timings, allocation stress within declared limits, full-model ledger projection | Phase 2B gate passes; remaining fit/speed uncertainty is quantified |
| **RT-07 — Streamed full conversion** | RT-02, RT-03, RT-06 | Naïve full text artifact, manifest, tensor coverage report, fresh-process validation, development evaluation, full-model phone smoke report | Phase 3 gate passes; actual full-model allocation is feasible before long refinement |
| **RT-08 — Calibration and sensitivity** | RT-07 | Fixed calibration/dev splits; scale/threshold and block-reconstruction trials; precision ablations and updated byte ledger | Best candidate is selected by the declared development rule; measured gains/costs justify proceeding or stopping |
| **RT-09 — Conditional QAT** | RT-08; explicit resource budget | Single-block pilot, objective/configuration, exact-resume test, monitored refinement, fresh-process accepted checkpoint | Phase 4 quality target is reached or a bounded negative result is documented; skip with evidence if calibration already suffices |
| **RT-10 — Optimize and freeze candidate** | RT-07; selected quality candidate from RT-08/09 | Kernel/cache profiling, optimized artifact/runtime pair, numeric/development regressions, full device metrics | Memory and speed targets pass; artifact/runtime/config hashes are frozen before final evaluation |
| **RT-11 — App contract and lifecycle validation** | RT-10 | Research integration, local-task contract results, install/rollback checks, repeated lifecycle/thermal/cancellation report | Phase 6 and Section 7 device gates pass on the frozen configuration |
| **RT-12 — Blind final evaluation** | RT-10, RT-11 | Sealed test outputs, blinded reviews, confidence intervals, per-category and critical-failure report | All quality gates pass, fail, or are explicitly inconclusive; no retuning on this holdout |
| **RT-13 — Decision and handoff** | RT-12, or earlier documented stop | Promote/continue/stop report, independent review where promotion is proposed, reproducibility package, integration/rollback plan, updated source-of-truth docs | Evidence supports the decision; release is separately authorized and any continued research has a bounded next hypothesis |

Read-only RT-03 reconnaissance and RT-02 dataset preparation may proceed while teacher details are resolved. Their gates still depend on the selected architecture/teacher. Later performance optimization may begin earlier when it resolves a measured blocker, but must not bypass correctness or consume undeclared resources. Task ordering does not request parallel agents or concurrent heavyweight runs.

### Implementation boundaries and interface contracts

Implement these responsibilities as small modules, using the existing repository's conventions. The names describe **required interfaces**, not currently installed APIs or commands.

| Component | Inputs → outputs | Required invariant |
| --- | --- | --- |
| Inventory / planner | Source config and tensor metadata → census, precision/transform map, byte ledger | Enumerates every expected tensor and alias; no model-sized allocation just to inspect metadata |
| Reference transform | Array, block plan, signs, normalization, direction → transformed array | Handles forward/transpose distinctly; validated in independent high precision |
| Reference ternarizer | Rotated group, assignment configuration → codes and serialized scales | Deterministic and finite; measures error using the stored scale, not an unrounded training value |
| Packer and independent reader | Codes/scales/specification ↔ bytes and decoded values | Exact code round-trip; explicit invalid-code, version, length, bounds, and overflow checks |
| Streaming converter | Verified source + immutable conversion config → temporary artifact + report → validated final artifact | Bounded memory/scratch; no silent tensor omission, alias duplication, or source mutation |
| Native loader / graph integration | Validated metadata + packed tensors + runtime capability map → executable graph | Applies exactly one intended rotation per linear input; rejects missing capability instead of silently using a dense or unrotated path |
| Kernel backend | Packed weights, scales, transformed input, dimensions/strides → output | Matches the independent reference; bounded scratch; reports backend/fallback; supports batch/prefill and single-token decode |
| Calibration / reconstruction | Teacher, current student, calibration split, objective/config → candidate + optimization state | Never reads sealed evaluation targets; records which inputs come from teacher versus progressively quantized student |
| Evaluation adapter | Model identity, semantic request, retrieved evidence, generation config → raw trace + presented output + metrics | Uses correct tokenizer/template; exposes truncation, retries, stop reason, and task contract; no hidden model-specific prompt advantage |
| Device probe | Artifact/runtime hashes + workload config → timestamped resource/performance/lifecycle report | Disposable bundle, one engine owner, explicit cancellation and teardown, no production-data replacement |

For block reconstruction, distinguish (a) matching teacher outputs on teacher inputs and (b) compensating accumulated upstream error using student inputs with teacher targets. Predeclare which is tested. Mask padded tokens, freeze non-target parameters, report both training and development reconstruction errors, and keep the teacher fixed. For distillation, specify KL direction, temperature and scaling, hidden-state layer mapping, token masks, loss weights, optimizer/learning rate, trainable parameter set, STE rule, and scale constraints. These choices belong in trial configs; “use QAT” is not a sufficient implementation specification.

Start with a deterministic weight-MSE reference quantizer; its exact initialization, tie rule, scale floor, stopping criterion, and iteration cap must be in the format/reference tests. Keep simple two-bit and compact packings behind the same logical codes/scales interface. Repacking identical codes must not change logits beyond the declared numeric tolerance. Do not simultaneously change packing, transform, quantizer, cache precision, and training recipe in one ablation.

### Required run files and validation

Each run directory must contain the following small records or explicit references to immutable records. Create machine-readable schemas and a validator in RT-02/04; a schema with `null` for unknown information is valid for investigation, but a gate depending on that field cannot pass.

| Record | Minimum contents |
| --- | --- |
| `manifest.json` | `schema_version`, `run_id`, parent run, task ID, hypothesis, status, timestamps; repository commits/dirty-diff hashes; source/tokenizer/template/data hashes; conversion/training config hashes; runtime/binary/device identity; artifact locations/byte counts/checksums |
| `config.json` | Frozen seeds, model/task/token budgets, dtype/packing/transform policy, numerical tolerances, calibration/training settings, resource/quality/performance limits, trial/time/compute budget |
| `tensor-map.json` | Source/runtime names, original/stored shapes, alias owner, input/output orientation, dtype/code layout, scale grouping, block/sign metadata, retained/excluded reason, exact byte counts |
| `metrics.json` | Units and sample counts; numeric errors, quality scores/intervals, task failures/retries, latency distributions, peak memory, thermal conditions, cost; explicit measured/estimated/not-measured labels |
| `commands.jsonl` and `logs/` | Working directory, executable/tool version, exact arguments, sanitized environment, start/end, exit status, stdout/stderr references; no credentials |
| `gate.json` | Gate ID, prerequisite evidence, thresholds/config hash, evidence hashes, per-criterion outcome, overall pass/fail/inconclusive/blocked, reviewer, unresolved issues |
| `report.md` | Hypothesis, result, failures, practical interpretation, and one concrete next action; links to the records above |

Artifact validation must reject duplicate tensor names, incompatible dimensions, out-of-bounds offsets, invalid scales, unexpected tensor omissions, and incompatible runtime/transform versions. Hash/model byte checks must precede expensive model allocation. Conversion completion requires reload in a fresh process; training completion also requires continuation equivalence under the declared determinism tolerance. A crashed, partial, or unverified run stays non-promotable even if it produced an apparently usable file.

### Decisions, blockers, and completion semantics

Maintain a decision register with `id`, question, current option, evidence, owner, status (`provisional` / `frozen` / `superseded`), date, and affected task IDs. Initial entries are teacher identity/adaptation, device/context, research and compute budget, teacher execution machine, rotation block policy, packing, embedding precision, and measured memory cap. Implementing agents may resolve technical choices through the prescribed measurements within authorized scope; ask the owner only for missing preferences, access, scope changes, or spending authorization.

Task states are `not_started`, `in_progress`, `blocked`, `passed`, `failed`, or `skipped_with_reason`. A blocked task names the missing input and work that remains possible. A failed gate leads to a bounded new hypothesis or the stop report, not an undocumented threshold relaxation. RT-09 is optional; skipping correctness, physical-device fit, or final evaluation cannot justify promotion. “Continue research” is a decision with remaining work, not overall success.

At every handoff, update `STATUS.md` with:

```text
Current objective and task ID:
Repository/worktree paths and commits (including dirty changes):
Last passed gate and evidence paths/hashes:
Current candidate/source/runtime/config identities:
Latest attempted command and result:
Active processes, device state, and cleanup still required:
Open decisions/blockers and their dependent tasks:
Next smallest action, working directory, inputs, expected output:
Commands already verified / commands still to discover:
Resource budget used and remaining:
```

A new agent should be able to validate this record and execute the next task without recovering conversation history. Do not invent runnable commands before implementations and checkout paths exist: discover/build them in their assigned tasks, then document and verify their actual invocation. Completing this research means delivering reproducible evidence and an explicit decision; successful phone deployment cannot be promised before the research gates pass.
