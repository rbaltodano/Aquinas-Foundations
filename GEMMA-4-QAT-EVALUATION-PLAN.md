# Gemma 4 E2B / E4B QAT — Evaluation and Optional Adaptation Plan

**Status:** proposed execution plan; no model downloads, training, app changes, or device tests have been performed for this plan.

**Created:** September 18, 2026.

**Objective:** determine whether an existing Gemma 4 E2B or E4B QAT model can deliver better Aquinas answers locally on the base supported iPhone, using an existing runtime and bounded integration effort. Test untouched models first. Fine-tune only a promising candidate with a demonstrated behavior gap and a verified export path.

**Relationship to other work:** this is a separate, preferred initial evaluation workstream before custom [rotated-ternary compression](ROTATED-TERNARY-COMPRESSION-PLAN.md). It can conclude successfully without that compression project. It does not authorize replacing the production model. [MODEL-INTEGRATION.md](MODEL-INTEGRATION.md) remains the integration source of truth; reconcile historical statements against the actual checked-out code and installed artifact.

**For an implementing agent:** read Section 0 for current progress, Section 8 for task dependencies and completion evidence, and Section 12 for the start/resume procedure. Sections 1–7 describe the experiments; Sections 9–11 define implementation interfaces, validation, and failure handling. This plan does not require conversation history to execute.

## 0. Current progress

**Last updated:** September 18, 2026 — QAT-00 inventory and baseline reconciliation completed.

**Current task:** QAT-01/M2-L, fresh prebuilt-QAT LiteRT-LM smoke test, `in_progress`.

**Last passed experimental gate:** none.

**Completed planning work:** identified official E2B/E4B mobile-QAT and Q4_0 model repositories; distinguished source checkpoints from deployment artifacts; defined baseline, phone, quality, and optional adaptation tests. Publication of a checkpoint is not evidence that our pinned iOS runtime supports it.

The implementation review additionally defines per-candidate dependency tracking, task/API contracts, controlled comparison rules, numerical/export checks, workload coverage, resource accounting, stop/recovery decisions, and the final review package. These are documented requirements; no implementation gate has passed.

| ID | Task | Status | Remaining work / evidence |
| --- | --- | --- | --- |
| QAT-00 | Inventory and isolation | `passed` | Evidence: [`research/gemma-4-qat/workspace.md`](research/gemma-4-qat/workspace.md), [`baseline-identity.json`](research/gemma-4-qat/baseline-identity.json), [`STATUS.md`](research/gemma-4-qat/STATUS.md), [`decisions.md`](research/gemma-4-qat/decisions.md). B0 is the active iOS 3,862,121,696-byte `dynamic_wi8_emb4_afp32` package, SHA-256 `9a6345f1a6cd39283f957977c84d31cc63b8dd56f2b8fffeb784940f63365282`; two local backend LiteRT files are older distinct artifacts. Device enumeration is blocked by unavailable CoreDevice/CoreSimulator services; host memory/process reads require refresh before heavyweight work. |
| QAT-01 | Resolve artifacts and runtime compatibility | `in_progress` | M2-L is now a verified prebuilt E2B QAT LiteRT-LM route: the local artifact exactly matches the published package's revision/file/hash. Prior physical base-iPhone GPU smoke evidence exists; perform a fresh isolated run before passing the candidate. M2/M4 source export remains blocked; Q2/Q4 GGUF has an unproven llama.cpp route. Evidence: [`LITERT-QAT-SUPPORT-CLARIFICATION.md`](research/gemma-4-qat/LITERT-QAT-SUPPORT-CLARIFICATION.md), [`candidate-registry.json`](research/gemma-4-qat/candidate-registry.json), and [`compatibility.md`](research/gemma-4-qat/compatibility.md). |
| QAT-02 | Evaluation harness and baseline | `not_started` | Prepare separated datasets, matched requests, rubric, current-model measurements, and frozen gates |
| QAT-03 | Stock E2B evaluation | `not_started` | Desktop correctness, early phone fit, then development quality/performance |
| QAT-04 | Stock E4B evaluation | `not_started` | Repeat E2B protocol; account for actual memory and embeddings rather than model name |
| QAT-05 | Select route and candidate | `not_started` | Compare feasible stock candidates; select stock, bounded adaptation, repair, or stop |
| QAT-06 | Conditional fine-tuning/export pilot | `not_started` | Prove train/save/reload/export feasibility before a useful-length training run; skip if unnecessary |
| QAT-07 | Conditional Aquinas adaptation | `not_started` | Train within budget, validate export loss, and compare to untouched QAT; skip if unnecessary |
| QAT-08 | Final phone and blind acceptance | `not_started` | Freeze one candidate; complete lifecycle/thermal/quality tests and independent review |
| QAT-09 | Decision and handoff | `not_started` | Deliver adopt/continue/stop evidence and any separately authorized integration proposal |

**Open decisions:** device/OS access; minimum context; numerical phone footprint cap; phase time/trial budget; whether paid or remote compute is permitted; exact mobile artifact/export support. Use base iPhone 17, 8 GB RAM, and 4,096 total tokens as provisional targets based on project history. Do not infer authorization for remote data transfer or compute spending.

**Immediate next action:** run the already-local M2-L package in a disposable LiteRT-LM phone probe and record fresh identity/load/generation/template/EOS results. M2/M4 source export remains a separate blocked route. Do not download all variants or retrain the current model.

**Update rule:** update this section at every task transition and handoff. States are `not_started`, `in_progress`, `blocked`, `passed`, `failed`, and `skipped_with_reason`. Link evidence in each completed row; partial work is not a passed gate. Keep the detailed handoff record in sync with this summary. Plan edits are not implementation progress.

## 1. Scope and model selection

### What “QAT model” means here

Google provides QAT-trained checkpoints in distinct families: Q4_0-oriented source/deployment formats and a mobile-specific mixed-precision family. Mobile source weights in Transformers format are not an already validated LiteRT package for our app. E2B/E4B names denote effective model size; embeddings and other tensors still contribute to storage, loading, and training memory. Measure the full tensor/artifact inventory. [Google model overview](https://ai.google.dev/gemma/docs/core)

QAT source weights can be adapted, but fine-tuning, adapter merging, and re-export may change quantization behavior. Availability of ordinary Gemma LoRA support does not prove support for the mobile-QAT scheme. QLoRA is also not automatically equivalent to preserving Google's deployment quantizer.

### Fixed initial candidate matrix

**Support clarification:** published E2B `.litertlm` packages already use QAT according to a LiteRT Community maintainer, even though their filename lacks `qat`. See [the evidence and next action](research/gemma-4-qat/LITERT-QAT-SUPPORT-CLARIFICATION.md). Register the exact prebuilt artifact as `M2-L` after provenance/hash verification; its runtime smoke test is independent of the mobile Transformers export route. Verify E4B package provenance separately. A blocked source exporter must not block investigation of a prebuilt package or cause it to be labeled non-QAT.

| Candidate ID | Checkpoint / role | Initial treatment |
| --- | --- | --- |
| B0 | Exact currently deployed Aquinas model and production pipeline | Mandatory baseline; capture installed hash and settings |
| B1 | Corresponding pre-export tuned checkpoint, if verified and runnable | Diagnostic for existing export/runtime loss; not a phone-fit claim |
| M2 | [google/gemma-4-E2B-it-qat-mobile-transformers](https://huggingface.co/google/gemma-4-E2B-it-qat-mobile-transformers) | Primary E2B mobile-QAT source; resolve a supported deployment artifact in QAT-01 |
| M4 | [google/gemma-4-E4B-it-qat-mobile-transformers](https://huggingface.co/google/gemma-4-E4B-it-qat-mobile-transformers) | Primary E4B mobile-QAT source; same verification requirements |
| Q2 | [google/gemma-4-E2B-it-qat-q4_0-gguf](https://huggingface.co/google/gemma-4-E2B-it-qat-q4_0-gguf) | Supported-format control/fallback where mobile route is blocked or needs comparison |
| Q4 | [google/gemma-4-E4B-it-qat-q4_0-gguf](https://huggingface.co/google/gemma-4-E4B-it-qat-q4_0-gguf) | Corresponding E4B control; screen size before phone load |
| R12 | [google/gemma-4-12B-it-qat-q4_0-gguf](https://huggingface.co/google/gemma-4-12B-it-qat-q4_0-gguf) | Optional Mac quality reference, only if smaller candidates leave a meaningful gap |

Evaluate M2 and M4 first when their deployment route is available. Q2/Q4 are different QAT variants, not repackings of M2/M4; label comparisons accordingly. Do not treat results from Q2 as completed testing of M2. Avoid downloading every source precision and format: inspect metadata, identify the needed files, and process one model at a time.

Unsloth documents mobile-derived `UD-Q2_K_XL` GGUFs for both sizes and conversion-fidelity differences between quantization formats. These are optional third-party deployment candidates with distinct IDs (for example M2-U/M4-U), not official Google GGUF artifacts or proof of iOS support. Verify source lineage, exact file, tensor types, converter revision where available, and our runtime's CPU/Metal support before use. Published desktop measurements are not acceptance evidence. [Unsloth's QAT and mobile-conversion documentation](https://unsloth.ai/docs/models/gemma-4/qat)

**Non-goals:** inventing a quantizer, custom ternary kernels, full-model QAT recovery, speculative decoding, maximum context, or simultaneous training of both sizes. Initial tests are text-only. Any modality removal must follow a supported export path and pass text equivalence checks; do not manually strip arbitrary tensors or per-layer embeddings.

## 2. Workspace, artifact, and resource contracts

Read `AGENTS.md` / `CLAUDE.md` in every repository before editing. This Foundations repository remains documentation-only. Create isolated worktrees when implementation starts, after inspecting existing dirty work and active jobs.

| Location, relative to `Developer/` | Responsibility / starting points |
| --- | --- |
| `Aquinas-Foundations/` | This plan, decisions, sanitized phase reports; proposed `research/gemma-4-qat/STATUS.md` |
| `Aquinas_Backend/` research worktree | Model/source inspection, evaluator, optional training/export. Inspect `scripts/evaluate_prompt_quality.py`, `benchmark_latency_quality.py`, `export_litert_aquinas.py`, and `export_litert_aquinas_stage.py` before reuse |
| Verified current iOS checkout | Actual model identity, prompt/task contracts, grounding and lifecycle behavior; resolve checkout path rather than assume a historical directory is current |
| `Aquinas-iOS-llama-cpp-12b/` | Existing llama.cpp integration reference: `LlamaCPPAquinasModel`, `LlamaCPPChatSession`, and `LlamaCPPDeviceProbe`; use an isolated descendant if selected |
| `Aquinas-iOS-main/Aquinas-iOS/Features/Developer/LiteRTDeviceProbe.swift` | LiteRT probe reference; verify current native framework and exporter compatibility |
| Native runtime source, to locate | Pin source commit, build recipe, compiler/SDK, framework hash, device backend coverage, and Xcode probe scheme/bundle |
| Backend research worktree, proposed `research/gemma4_qat/` | Versioned adapters, configuration/schema validation, relevant tests, and README with verified commands |
| Backend research worktree, proposed `runs/gemma4-qat/<run_id>/` | Immutable configs, artifact references, logs, outputs, metrics, and gate reports; keep models/caches/private data out of Git |

Proposed paths are not existing deliverables. Prefer established repository conventions where present and record the mapping. Do not invent runnable commands or scheme names; discover them in QAT-00/01, verify a smoke invocation, and record exact working directories and arguments.

The development Mac was previously described as having 24 GB unified memory; measure current resources. Before each large download/load/export, account for simultaneous source, destination, staging, caches, build products, and a **25 GiB free-disk reserve**. Declare maximum run duration, memory-pressure and swap-growth limits, remaining write budget, and graceful-stop behavior. Monitor at least every 10 seconds and at load/save boundaries. Never run concurrent heavyweight model jobs or iOS builds without a measured combined budget.

Reuse verified files; download only pinned revisions/files, verify byte count and hash, and stage outputs atomically. Retain logs and last validated checkpoints. Clean only explicitly identified regenerable research scratch. No paid job or long training run starts without a numerical budget and permitted execution location.

## 3. Tasks, implementation steps, and exit gates

### QAT-00 — Establish current reality

1. Record all repository/worktree paths, commits, dirty changes, applicable instructions, active processes, available runtimes, and device access.
2. Identify B0 from the actual app artifact/configuration, including model hash, tokenizer/template, context, decoder/backend settings, and fallback behavior. Existing docs contain historical states; do not assume their latest-looking paragraph describes the running app.
3. Verify whether factual retrieval, MiniLM, corpus export, and structured task processing are actually local in the selected app build. Record corpus/index/model versions and unresolved integration gaps.
4. Inventory local model files and B1 provenance; match adapters/base checkpoints rather than inferring identity from folder names.
5. Create the status record, resource/decision register, and disposable test-bundle strategy. Preserve production data and verify its backup before physical-device experiments.

**Output:** `workspace.md`, `baseline-identity.json`, `decisions.md`, initial `STATUS.md`.

**Exit gate:** current production identity and intended research edit/build paths are verified. Missing device access may defer phone work but does not stop metadata/evaluator preparation.

### QAT-01 — Resolve deployment support before model experimentation

1. Inspect M2/M4/Q2/Q4 file metadata and configs; pin repository revision, requested files, hashes, license, tokenizer/template, tensor dtypes, embeddings, quantization configuration, context/cache requirements, and total bytes.
2. Build a per-candidate matrix: source format → deployment artifact → loader → CPU/Metal kernels → iOS build. Check static activation/scaling requirements, mixed-width weights, KV format, per-layer embeddings, and largest buffer. Do not infer supported execution from the model card's generic loading snippet.
3. Prefer a verified ready-made mobile package for the existing LiteRT path, if available. Otherwise test a bounded, supported export of the official mobile source. If that route requires unsupported quantization/runtime work, record it as blocked and examine the documented mobile-derived GGUF route or the distinct Q4_0 control.
4. Pin native and converter versions. Rebuild the disposable probe reproducibly, recording architecture/deployment target, scheme, bundle identifier, compiler options, and source-to-framework hash linkage.
5. Run a tiny loader/template/EOS smoke test before a full evaluation. Capability checks and artifact validation must precede large allocation. Record any CPU fallback; desktop success is not phone compatibility.

**Output:** `candidate-registry.json`, `compatibility.md`, verified command README, smoke logs.

**Exit gate:** per candidate, an explicit supported test route with successful smoke evidence. Unsupported candidates receive a blocked or failed outcome, not a passed gate. A working route for one size may advance independently. Do not substitute an ordinary non-QAT model and label the task passed. Freeze a bounded troubleshooting budget before investigating any unsupported route; a custom runtime project needs its own decision.

### QAT-02 — Build comparisons that identify the cause of improvement

1. Create a **60-case development screen** covering definitions/distinctions, reasoning/application, source-dependent questions, absent/conflicting evidence, multi-turn/topic shifts, and local structured tasks. Include historical citation, repetition, corruption, and shallow-answer failures plus fresh variants; tag overlapping categories.
2. Prepare a separate **sealed final set of at least 200 independent questions/conversations**, with at least 30 examples in each critical reasoning, grounding, and regression category. Keep any future training set separate by passage, paraphrase family, and conversation. Pilot variance may require more examples; insufficient statistical power is inconclusive.
3. Snapshot semantic system/personality/task instructions, retrieved passages and identifiers, history, output budgets, and expected contracts. Adapt only the tokenizer/chat-template syntax each model requires. Check rendered prompt tokens, truncation, role boundaries, EOS, and channel parsing explicitly.
4. Use deterministic decoding for numeric diagnosis. For user-facing quality, start with B0's actual production settings and a declared common candidate policy. A bounded sampled-decoding or reasoning-mode trial is allowed on development data, with fixed seeds/repeats and equal trial allowance per size; freeze the chosen policy before final evaluation. Do not assume B0's sampling corruption applies to all QAT models.
5. Diagnose retrieval separately: compare actual retrieved evidence with gold relevant passages for a subset. Freeze the same evidence across model comparisons. Track whether failures come from retrieval, evidence use, base capability, export, template, or app state.
6. Measure B0 and feasible B1 on development cases. Disable network/backend recovery for local-model scoring; report failures rather than scoring a server answer as phone quality. Evaluate app recovery separately.

**Output:** dataset manifests, anchored rubric, request/output schemas, baseline report, frozen acceptance config.

**Exit gate:** the evaluator reproduces baseline results within declared tolerance, exposes errors/retries, and keeps test data separate from tuning. Baseline failure is evidence to diagnose, not permission to lower candidate standards.

### QAT-03 and QAT-04 — Untouched E2B, then untouched E4B

Dependencies: QAT-01/02 for the candidate. Complete this protocol for each size independently; a blocked mobile route does not invalidate a separately reported Q4_0 result.

1. Load the selected stock deployment artifact on Mac. Run 5–10 smoke cases covering normal generation, source use, a follow-up, structured output, long-answer stopping, and cancellation where supported. Check NaNs, malformed tokens, wrong template, unexpected prompt echo, and output-channel handling.
2. Where the matching source checkpoint is runnable, compare fixed-input tokenization, teacher-forced logits and decoded behavior through export/runtime boundaries. Freeze numeric tolerances before assessing results; do not require byte-identical free generation across different numerical backends. A desktop deployment-format run without source parity remains explicitly labeled as such.
3. Build a memory ledger: weights/embeddings, persistent and transient copies, KV, prefill/graph buffers, logits, app/retrieval overhead, and largest allocation. Run early disposable-phone load, short generation, then 2K/4K total context screening only when headroom permits. Screen before expensive desktop evaluation or adaptation.
4. Run the development screen for viable routes. Record raw visible answer, presented answer, task validity, citations/evidence use, stop reason, retries, latency, and failure category. Keep private reasoning out of persisted app history and evaluation text; record token counts/timing and channel violations instead.
5. Measure load, prefill, decode, first visible token, full answer completion, metadata work, and peak physical footprint. Run a preliminary sustained session to expose thermal decline and growing allocations.
6. Test the candidate through the actual research app's prompt/task/grounding path after basic engine correctness. Mark engine-only success separately from app integration.

**Output:** per-candidate development report, numerical/export diagnostics where available, phone smoke metrics, output traces and failures.

**Exit gate:** to pass, the candidate has a measured development quality/footprint/speed profile and successful phone/app smoke checks. Otherwise record a specific failed, blocked, or inconclusive result; that record permits a stop/selection decision but is not a pass. A simulator pass, fluent example, or model-file size is not a phone acceptance result. Neither size is presumed superior.

### QAT-05 — Decide whether adaptation is needed

Compare the viable candidates against B0 using blinded development review. Prefer the candidate that meets quality requirements at lower measured latency/memory/maintenance cost; do not choose by parameter count or style alone.

Record one outcome:

- **Stock finalist:** quality and device behavior are promising; skip QAT-06/07 and proceed to final acceptance.
- **Adapt one candidate:** a feasible model has a specific voice, task-format, or evidence-use gap that a small supervised adaptation might resolve. Specify the target behavior, existing stock performance, training/export route, and expected measurable gain.
- **Repair integration/retrieval:** evidence points to a bounded pipeline defect. Document a separate repair scope and rerun affected comparisons; do not train around the defect.
- **Stop / inconclusive:** neither candidate meets the target, or compatibility/resources prevent a valid test. Preserve results and identify the next discriminating experiment.

If the remaining issue appears to be reasoning capability, optionally evaluate R12 on the same development requests on Mac. This estimates a potential benefit of larger-model research; it does not demonstrate phone fit or require resuming the ternary plan. Missing references are not established solely by comparing model sizes.

**Output and exit gate:** `selection.md` names the finalist/route, supporting evidence, frozen targets, and any approved adaptation budget. No training is required just to make the model “Aquinas.”

### QAT-06 — Prove the adaptation-to-phone path cheaply

This task is conditional on QAT-05. Google describes fine-tuning its released QAT weights, but this does not establish an end-to-end mobile-schema workflow for our pinned toolchain. [Google QAT announcement](https://blog.google/innovation-and-ai/technology/developers-tools/quantization-aware-training-gemma-4/)

1. Identify the exact trainable source matching the selected variant. A Q4_0 unquantized source is not the mobile-QAT source. Confirm supported modules, frozen quantizer/scales where applicable, tokenizer, and tensor names. Do not reuse adapters from a different base revision or size without compatibility proof.
2. Choose a supported LoRA-style pilot with a frozen base and the smallest useful target scope. Specify rank/alpha, modules, dropout, learning rate, optimizer, sequence length, loss masks, precision, seeds, microbatch/accumulation, maximum steps, and resource limits. A training backend that lacks the model's required quantization behavior is a blocked route, not a reason to silently strip it.
3. Prove an **unmodified round trip first**: source → intended exporter → phone-loadable artifact. Compare with the untouched deployed QAT candidate to separate exporter loss from training effects. Recalibration must use only permitted calibration data.
4. Run a tiny smoke adaptation (for example 10–20 updates on a small licensed diagnostic sample). This proves mechanics, not quality. Save adapter/base identity plus optimizer, scheduler, RNG and data position; reload in a fresh process and verify continuation within declared determinism tolerance.
5. Test the actual deployment method: separate adapters only if the runtime supports them correctly, otherwise merge into the verified source and re-export using a validated quantization path. Recheck scale/activation calibration, tensor coverage, hashes, output behavior, and memory. Never assume merging preserves QAT fidelity.

**Output:** training/export feasibility report, pilot configuration, reload evidence, and phone smoke artifact.

**Exit gate:** the full training-to-phone chain works within budget and does not introduce unexplained export loss. Otherwise stop adaptation, keep the best stock model eligible, and document whether recovery would require a separate QAT project.

### QAT-07 — Bounded Aquinas adaptation

1. Build a small audited instruction dataset specifically for the diagnosed gap: contemporary explanations, accurate distinctions, multi-turn behavior, structured task contracts, and questions with supplied evidence plus grounded answers/appropriate uncertainty. Include general instruction-following preservation examples. Audit completeness, role structure, label correctness, truncation, and source provenance.
2. Do not automatically train on the retrieval corpus or revive raw Summa continuation data. Prior [Qwen experiments](QWEN-TESTING-CASE-STUDY.md) exposed problems with fragmented articles and answer-format learning. Fine-tuning changes behavior; it does not establish reliable factual recall.
3. Freeze train/dev/test separation and a maximum trial count, steps/tokens, wall time and cost. Start with one candidate and one declared configuration. Evaluate held-out development behavior at fixed intervals; stop on non-finite state, excessive resource use, failure to save/resume, or declared general/grounding regressions.
4. Compare **untouched QAT → adapted training checkpoint → final exported artifact**. Run the same task suite at each runnable stage; quantify export loss separately. A smaller training loss is not acceptance evidence.
5. Select by development results only. Restore the best validated checkpoint if later training regresses; export and reload it afresh. Keep the stock candidate as a fallback. Do not spend the sealed final set selecting among training checkpoints.

**Output and exit gate:** adapted candidate demonstrably improves its intended behavior without unacceptable quality/export/device regressions, or a bounded negative result. If stock remains better, select stock.

### QAT-08 — Freeze and validate one finalist

Dependencies: QAT-05 plus QAT-06/07 if used. Freeze the artifact, runtime, prompt/template, retrieval snapshot, sampling/reasoning configuration, context and task budgets before the sealed test.

Complete Sections 4–5, review implementation independently, and link every criterion to evidence. Any material fix after final testing requires revalidation; tuning on final-test feedback consumes that holdout and requires a new one.

### QAT-09 — Decision and handoff

Deliver `decision.md` with one of: **adopt stock candidate**, **adopt adapted candidate**, **continue bounded testing**, or **stop this route**. Adoption means a recommended versioned integration candidate, not an automatic production switch.

Report per-candidate quality/device results, limits and failures, exact reproducibility commands, artifact/runtime hashes, resource cost, and the smallest next step. For adoption, prepare deployment/rollback and download/storage requirements and update `MODEL-INTEGRATION.md` when an actual integration decision is made. For a negative result, state whether failures were capability, deployment, retrieval, or resources and whether R12 evidence justifies revisiting the ternary plan.

## 4. Quality criteria and evaluation integrity

Freeze these proposed thresholds in QAT-02 before candidate selection. Revisions require a dated rationale and a new experiment configuration; do not loosen them after seeing final results.

- Use anchored 0–4 ratings for reasoning/correct distinctions, usefulness/directness, evidence/citation faithfulness, and instruction/task adherence; convert the declared weighted average to 0–100. Default equal weights; mark non-applicable dimensions and predeclare aggregation. Report critical categories separately, not only an average.
- Blind candidate identities and randomize paired answer order. Use two independent reviewers for source-sensitive/critical cases and a shared subset, record agreement, and adjudicate disagreements against source-backed keys. Automated judges may assist but cannot alone establish theological accuracy or promotion.
- Final paired preference against B0 must be at least **55%**, counting ties as half, with a conversation-clustered bootstrap 95% confidence interval above 50%. If another viable candidate is available, report its comparison too; use development evidence to choose the finalist, not repeated final-set trials.
- Grounding and task-adherence category scores must have a lower paired 95% confidence bound no worse than **−5 points** versus B0 on the 0–100 scale. A candidate with better prose but worse evidence handling does not qualify. Uncertain evidence is inconclusive.
- No new confirmed critical regression may be waived by average scores: fabricated citations on evidence tests, unbounded repetition, script/token corruption, hidden-reasoning leakage, or invalid required structured output after the allowed repair policy. Report raw failure and repair rates separately; backend rescue never counts as local success.
- For an adapted finalist, additionally require a measurable development improvement in the declared target behavior versus its untouched QAT base and no critical regression; report the stock/adapted comparison in final review without using it to retune.
- Document actual inference settings and token counts. Compare the same evidence and semantic request, not identical token IDs across tokenizers. No forced verbosity or extra reasoning budget may be hidden as a model-quality gain.

The 60-case screen selects what merits further work; it does not support a release claim. Keep training/calibration, development, and final sets disjoint by passage and question family. Record all trials and include failures, cancellations, and timeouts in results.

## 5. Physical-device acceptance

Provisional target: base iPhone 17 / 8 GB, text-only, **4,096 total tokens** including all system text, retrieval, history, reasoning and output, with at least 512 output tokens reserved. A 2,048-token configuration may be a separately evaluated fallback, not a silent pass for 4K. Determine exact OS/build and minimum supported hardware in QAT-00.

| Metric | Proposed requirement | Measurement |
| --- | --- | --- |
| Peak footprint | Below a numerical operating cap fixed from device/app constraints, with ≥15% headroom | Actual process physical footprint during load, full-context prefill, decode and teardown; account for Metal/shared buffers without double-counting |
| Warm first visible token | p95 ≤5 seconds at a 512-token prompt | Real app path; report whether the runtime buffers the whole answer |
| Cold load | p95 ≤15 seconds | At least 10 fresh-process loads; distinguish process-cold from cold file cache |
| Decode | p10 ≥8 tokens/s | At least 30 warm measurements across three sessions; include 128-token and longer outputs |
| Sustained behavior | Remains within declared memory and speed targets | At least one 30-minute workload; record thermal/power state and slowdown over time |
| Reliability | Zero fatal failures in the declared pilot and no confirmed critical regressions | At least 300 short requests spanning local task types plus lifecycle cases; report counts and uncertainty |

These are proposals, not measured capabilities. There is no assumed universal iOS per-app memory allowance. A 300-request zero-failure result only roughly bounds an independent failure rate below 1% at 95% confidence; correlated workloads give weaker evidence.

Record Release-equivalent build settings, debugger attachment, device/OS, power mode/charging, ambient/thermal conditions, runtime/backend, tokenization and batch sizes. Measure prefill at 128, 512, 2,048 and near-limit prompt lengths with output reserve. Include metadata generation, structured repair, full answer time, and energy measurements where supported; do not fabricate battery estimates from model size.

Test multiple-turn growth, context compaction/eviction, abrupt topic changes, repeated engine creation/destruction, cancellation during load/prefill/decode, background/foreground, memory warnings, interrupted model install, invalid hash/truncation, insufficient space, failed initialization, and rollback. Confirm a stable post-teardown memory plateau and inspect crash/termination diagnostics. No requirement to generate in the background is implied.

Use a disposable probe bundle and verified production-data backup. Never remove or overwrite the production app container. Simulator tests are useful for contracts and debugging; only physical hardware clears performance/fit gates. Local acceptance runs must succeed without a network/backend dependency.

## 6. Reproducibility and agent handoff

Each run must record the following, using machine-readable schemas with a validator:

| Record | Required fields |
| --- | --- |
| `manifest.json` | Schema/run/task/candidate IDs, parent run, timestamps/status; source repository/revision/files/hashes; artifact byte count/hash; tokenizer/template; runtime/converter/compiler/SDK; repo commits/dirty diff; device; dataset versions |
| `config.json` | Task/prompt/retrieval revisions, context/output/reasoning/decoder policy, seeds, precision/cache/backend; numerical tolerances and quality/resource limits; any training/adapter/export parameters |
| `requests.jsonl` / `responses.jsonl` | Case/conversation IDs, supplied evidence IDs, expected contract, presented answer, relevant raw visible output, stop reason, errors, retries, generated counts and timing; exclude private reasoning and personal data |
| `metrics.json` | Quality rubric/category scores, paired preference and intervals, task validity/failure rates, latency distributions, physical footprint/allocations, thermal conditions, costs, sample counts and units |
| `commands.jsonl` / `logs/` | Exact working directory/executable/arguments, dependency versions and sanitized environment, timestamps, exit codes, log references; no credentials |
| `gate.json` / `report.md` | Thresholds/config hash, prerequisite evidence, per-criterion pass/fail/inconclusive/blocked, reviewer, limitations, failures, and next action |

Unknown fields must be `null` / `not_measured`, not invented values. A gate depending on missing evidence cannot pass. Make artifact installation/export atomic, use fresh run IDs, reject partial outputs, and validate model identity before allocating/loading. Do not commit model weights, private evaluation data, access tokens, caches, or raw private reasoning.

Implementation responsibilities are: artifact registry/capability checks; a shared semantic request schema with model-specific template adapters; baseline/candidate runners; output/task/citation validators; device metrics ingestion; optional training/export adapter; and gate/report generation. Reuse existing evaluators where suitable rather than copying entire harnesses. Add focused checks for prompt rendering, result attribution, failure accounting, dataset separation, hash/schema rejection, and export parity; do not create tests that merely repeat configuration values.

At every stop or handoff, update Section 0 and `research/gemma-4-qat/STATUS.md` with:

```text
Current task and objective:
Repositories/worktrees/commits and dirty changes:
Current source/artifact/runtime/config identities:
Last passed gate and evidence paths/hashes:
Last command, working directory, result, and logs:
Active processes, device state, remaining cleanup:
Open decisions/blockers and affected tasks:
Next smallest action, required inputs and expected output:
Verified commands / commands still to discover:
Time/compute/storage budget used and remaining:
```

A new agent starts by reading this plan, applicable repository instructions, and the handoff evidence; it verifies current state before continuing the first eligible incomplete task. Missing phone access blocks phone gates, not dataset/reference preparation. Paid compute, unsupported export work, or a new compression project requires an explicit scoped decision; routine reversible research within existing authorization does not require repeated permission requests.

## 7. References and evidence limits

Primary sources checked September 18, 2026; pin exact repository/tool versions during QAT-01:

- [Google model overview](https://ai.google.dev/gemma/docs/core): QAT families and format routing.
- [Google QAT announcement](https://blog.google/innovation-and-ai/technology/developers-tools/quantization-aware-training-gemma-4/): published checkpoints and fine-tuning availability; not proof of our export fidelity.
- Official candidate repositories linked in Section 1: metadata discovery points, not validated local artifacts.
- [Unsloth QAT documentation](https://unsloth.ai/docs/models/gemma-4/qat): vendor-reported conversion behavior and mobile-derived GGUFs; verify exact current files and source lineage independently.
- [Model integration](MODEL-INTEGRATION.md), [Qwen case study](QWEN-TESTING-CASE-STUDY.md), and [prior QAT/DWQ work](Aquinas-QAT-DWQ-Writeup.md): existing app contracts and historical failures. Do not reuse their results as current candidate measurements.

**This plan is complete as an execution specification, not as evidence that a candidate works.** Its first deliverable is a verified inventory; its final deliverable is a reproducible model-selection decision. Training and production deployment are conditional outcomes.

## 8. Dependency map, decisions, and completion evidence

Task IDs are stable. Keep state per candidate/runtime pair (for example `QAT-01/M2/LiteRT` and `QAT-01/M2-U/llama.cpp`) as well as the Section 0 summary. A blocked M4 export must not stop independent M2 evaluation. A working Q4_0 route must not conceal an untested mobile route. An unsuccessful experiment can be fully documented without passing its acceptance gate.

| Task | Prerequisites | Required evidence to pass / permitted next action |
| --- | --- | --- |
| QAT-00 | None | Verified workspace, B0 identity, source/binary provenance, device/resource inventory and isolation map; unlock metadata and evaluator work |
| QAT-01 | QAT-00 | Candidate registry, tensor/format map, supported runtime/export route, pinned build and successful smoke test for the advancing candidate |
| QAT-02 | QAT-00; B0 runnable | Versioned split manifests, evaluator contract checks, reproducible baseline, frozen quality/resource thresholds and trial policy; dataset preparation may precede candidate downloads |
| QAT-03 | QAT-01 for selected E2B route; QAT-02 | Complete stock E2B development report and phone/app smoke evidence; record all rejection causes |
| QAT-04 | QAT-01 for selected E4B route; QAT-02 | Same evidence for E4B; normally run after E2B to reuse the verified harness, but E2B success is not a prerequisite |
| QAT-05 | Documented outcomes for QAT-03/04, including explicit blocked/rejected routes | Selection report compares available evidence and states its limitations; finalist must have passed its own required gates; adaptation additionally needs a testable gap and bounded budget |
| QAT-06 | QAT-05 chooses adaptation | Same-variant source identity, unmodified export round trip, tiny training/export pilot, fresh-process continuation, phone reload, and resource estimates |
| QAT-07 | QAT-06 | Budgeted adaptation, best-checkpoint selection using development data, final fresh export/load, stock-versus-adapted-versus-export report |
| QAT-08 | QAT-05; QAT-06/07 if adapted | Frozen candidate, all final quality/device/app criteria, artifact integrity, independent review; record each criterion separately |
| QAT-09 | QAT-08 for adoption, or earlier documented stop | Reproducible adopt/continue/stop report and exact next action; early stop cannot produce an adoption recommendation |

Only QAT-06/07 and R12 are optional. They need a recorded skip reason. Device fit, stock identity, task contracts, and final quality cannot be skipped for adoption. Missing B1/source-reference access limits diagnosis and must be disclosed; it does not fabricate a parity result. Stock published-artifact evaluation can proceed without a source reference, but a newly converted/adapted artifact requires the applicable round-trip/export evidence.

### Decision register

Create `decisions.json` with `id`, question, current choice, status (`provisional`, `frozen`, `superseded`), evidence, owner, timestamp, and affected task IDs. Link its human-readable summary from Section 0.

| Decision | Initial position | Freeze by |
| --- | --- | --- |
| D01 — Minimum device/OS and context | Base iPhone 17 / 8 GB, 4K total context provisional; confirm actual access/support requirement | QAT-02, before comparative phone measurements |
| D02 — Baseline model/pipeline | Resolve actual installed artifact and offline behavior; no inferred current hash | QAT-00 |
| D03 — Mobile artifact/runtime route | Prefer a validated supported route; record every source/export/runtime edge | QAT-01 per candidate |
| D04 — Time/trials/compute/privacy | Local bounded inventory permitted; long runs and paid/remote execution need numerical limits and authorization | Before affected downloads/experiments or training |
| D05 — Retrieval and task scope | Snapshot current corpus/pipeline and all tasks proposed for model replacement | QAT-02 |
| D06 — Evaluation/decoder policy | Frozen rubric, datasets, seeds, reasoning budgets, trial allowance and failure rules | QAT-02; finalist policy locked at QAT-05 |
| D07 — Phone resource envelope | Measured operating cap, 15% headroom, context/cache allocation and latency targets | QAT-02 before phone load expansion |
| D08 — Adaptation | None by default; one candidate only for a measured gap | QAT-05 |
| D09 — Finalist and release scope | Exact artifact/runtime/task coverage; no automatic production replacement | QAT-08/09 |

Technical choices may be resolved through evidence within authorized scope. Ask the owner for missing product preferences, access, spending, or a material change of scope. Do not request repeated approval for choices already settled in the session.

### Default bounds for candidate search

Start with one deployment route per size. Allow at most one fallback route per size when it answers a documented compatibility/quality question. Start with one declared decoding policy; allow at most two additional development policies per size with equal evaluation effort. These are proposed trial limits to freeze in D04/D06, not authorization for unlimited runtime or spending. Stop route troubleshooting at the predeclared time limit and report a blocker rather than starting custom kernels.

R12 is a separate diagnostic and may be skipped if a smaller finalist satisfies the product need. Fine-tuning is not a prerequisite to claiming useful Aquinas behavior. If more experimentation is justified, record a new bounded round and preserve earlier negative results.

## 9. Component and data contracts

The following names specify responsibilities, not installed APIs or existing scripts. Reuse existing repository modules where their contracts match; otherwise implement small adapters with focused tests. Keep model identification, prompt rendering, inference, scoring, and reporting separate so a runtime switch cannot silently change the benchmark.

| Component | Inputs → outputs | Required invariant / test |
| --- | --- | --- |
| Artifact registry | Pinned source metadata and local paths → immutable candidate identity and dependency graph | Source, derivative, adapted and exported candidates get distinct IDs; missing hashes cannot masquerade as verified artifacts |
| Capability resolver | Candidate tensor/quantizer inventory + runtime build → supported route or explicit gaps | Every required operation/dtype has a known path; silent CPU fallback, dequantization or omitted tensors must be surfaced |
| Request renderer | Semantic case, task, evidence, history, template/tokenizer → model input and token accounting | Correct role/EOS/channel boundaries; deterministic truncation; no candidate-specific evidence advantage |
| Inference adapter | Model/input/configuration → visible tokens, terminal result, stop/error metadata, timings | Exactly one terminal event per request; cancellation distinguishable from failure; no backend substitution during local scoring |
| Task validators | Raw visible output + expected task schema + evidence IDs → contract/grounding findings | Validation cannot rewrite the answer into correctness; record any allowed repair as a separate attempt |
| Device runner | Frozen workload + probe identity → per-request metrics and lifecycle events | Serialized engine ownership, reproducible reset/cleanup, app state retained only when the case requires it |
| Training/export adapter | Matching source, dataset/config, adapter state → checkpoint, export and parity report | Base identity and quantizer behavior preserved or changes declared; bounded memory; fresh-process reload |
| Evaluator | Blind paired responses + rubric and reviewer results → category scores and intervals | Conversation is the resampling unit; failed/missing outputs cannot be silently dropped |
| Gate/reporter | Versioned criteria + evidence → pass/fail/inconclusive/blocked and next action | Does not infer a pass from absent metrics, a completed process, or a model's fluent example |

### Candidate identity and tensor treatment

Extend `candidate-registry.json` with `candidate_id`, `family`, `size_label`, `source_revision`, `parent_candidate_id`, `artifact_revision`, `format`, `quantization_scheme`, `quantizer_config_hash`, `tokenizer_hash`, `template_hash`, `modality_policy`, `runtime_commit`, `artifact_sha256`, `artifact_bytes`, and provenance evidence. Do not identify a model only by a filename suffix such as “Q4” or “mobile.”

Create `tensor-map.json` for converted/adapted candidates: source/runtime names, shapes, dtype/packing, group axis/size, scale/zero-point dtype, tied/alias owner, embedding/PLE treatment, static activation parameters, cache requirements, and any excluded tensor plus its reason. Validate expected counts and shape/alias relationships. A source that already stores quantized or dequantized weights must retain its original-precision history in the manifest.

A supported format conversion must account for every required text tensor and quantization parameter. Reject duplicate names, unexpected omissions, dimension mismatches, non-finite scales/weights, invalid offsets/lengths, and unsupported types/versions. Avoid full-weight hashing repeatedly inside timed inference; validate once before the measured loading boundary and record whether validation time is included in the user-facing install/load metric.

### Request, response, and error schemas

`requests.jsonl` must include `case_id`, `conversation_id`, `split`, `category_tags`, `task_kind`, semantic messages, evidence passage/source IDs and hashes, expected schema, source-backed rubric/key reference, context/output budgets, reasoning policy, decoder policy, seed, and whether state carries across turns. A final-set runner must not expose answer keys to inference or training components.

`responses.jsonl` must include the request/config/artifact hashes, candidate/runtime IDs, attempt index, raw visible and presented answers, validator findings, input/output/reasoning token counts where available, truncation flag, stop reason, timestamps, retries, and device/session ID. Distinguish `eos`, `token_limit`, `cancelled`, `timeout`, `load_error`, `runtime_error`, `invalid_output`, and `completed`; a token-limit stop may yield an incomplete answer and must be scored accordingly. Unknown counts remain unknown.

Use a declared per-task timeout solely for bounded experiments; report it as a failure/timeout and include elapsed cost. Do not silently alter production cancellation behavior. Record infrastructure-invalid runs separately and rerun the paired block under the original policy; do not relabel model failures as infrastructure problems to remove them from scoring.

### Agent-facing command contract

The implementation README must provide verified invocations for: inspect/register artifact, validate configuration, render a prompt fixture, run one case, run a development suite, run a device workload, validate an export, score blinded results, and generate a gate report. Conditional training adds pilot, resume, export, and re-evaluate commands.

Each command must document its working directory/environment, required arguments, outputs, resource expectations, resume behavior, and exit codes. Support a metadata-only validation/dry-run mode where useful. Reject source/output path collisions and unknown candidate IDs. No command should download unrelated variants, mutate the source model, change the production bundle, or launch a training job as an implicit side effect.

## 10. Detailed comparison, workload, and export validation

### Controlled experiment matrix

| Comparison | Hold fixed | What it can establish |
| --- | --- | --- |
| B0 versus B1 | Same verified tuned lineage, semantic request/evidence, compatible budgets | Potential export/runtime degradation; if multiple settings differ, report confounding |
| B0 versus stock M2/M4 or Q2/Q4 | Case/evidence/task and predeclared decoder/resource policy | Product improvement of the entire candidate pipeline, not QAT's isolated causal effect |
| Source versus its own export | Same weights/variant, tokenizer/input IDs where compatible, precision policy and template | Conversion/runtime numerical and behavioral loss |
| Actual retrieval versus gold evidence | Same model/request with only the supplied passages varied | Retrieval gap versus evidence-use gap |
| Untouched versus adapted QAT | Same source lineage and evaluation/deployment policy | Adaptation benefit or regression |
| Adapted checkpoint versus adapted export | Same adapter merge state, source identity, request and quantization configuration | Loss introduced by merging/export/deployment |

Comparing E2B with E4B or Q4_0 with mobile QAT does not isolate quantization quality. An optional matched non-QAT stock control is allowed only to answer a specific causal question within budget; it is not required to determine whether a phone candidate meets the product goal. Higher-bit conversion of a QAT checkpoint is not automatically the original pre-QAT teacher.

### Numerical and export validation ladder

1. **Static validity:** hashes, schema, tokenizer vocabulary/special tokens, tensor inventory, metadata, quantization configuration, shapes and aliases.
2. **Input equivalence:** golden fixtures for a definition, evidence-grounded response, multi-turn exchange, structured task, and near-limit context. Check role/template rendering, BOS/EOS, control channels, padding and truncation.
3. **Teacher-forced comparison:** on fixed token sequences from development fixtures, compare finite logits, top-token agreement, and distribution divergence where accessible. Specify KL direction and stable FP32 computation, masking and treatment of padding. Record normal numerical drift from repeated same-artifact runs before setting tolerances. Do not use cross-tokenizer perplexity as a fidelity measure.
4. **Behavioral comparison:** deterministic diagnostic decoding plus the frozen user-facing policy. Report changed answers and task failures, including a long response and topic shift; free-generation token divergence alone is not corruption.
5. **Fresh-process export reload:** verify exported hashes and load in a new process; repeat key fixtures and phone smoke tests. For training, resume from saved state before accepting checkpoint continuation.

Numerical tolerances must be frozen per dtype/runtime comparison in `config.json`, including absolute/relative criteria and stable behavior near zero. Missing logits access limits diagnosis: use visible-output/contract checks and identify the limitation; never invent a numeric pass. New scale changes, adapter merges, cache precision changes, tokenizer patches, or converter/runtime updates require the relevant ladder to be rerun.

### Dataset construction and rubric anchors

Allocate the initial 60 cases as six groups of ten: definitions/distinctions; applied multi-step reasoning; correct supplied evidence/citations; missing/irrelevant/conflicting evidence; multi-turn/topic transitions; and structured/local-task behavior. Tags may overlap. Each case must have an expected success description and severity rationale, not necessarily one exact prose answer. Include explicit false-premise and evidence-instruction-conflict cases; retrieved text is data, not authority to change the task instructions.

For each 0–4 rubric dimension, use these anchors: **0** unusable/incorrect; **1** major error or failure to follow the task; **2** partly useful with a substantive omission; **3** correct and useful with minor issues; **4** correct, complete for the request, well-grounded where required, and clear. Define dimension-specific examples before blind review. Verbosity, scholastic style, and citation count are not substitutes for correctness. Use only applicable dimensions with normalized declared weights; separately report denominator/category coverage so a model cannot gain by avoiding difficult tasks.

Freeze a case-level critical-failure list and distinguish legitimate multilingual names/quotations from unintended script corruption. Citations must identify a real supplied/retrieved source and accurately support the claim; syntactically valid references alone are insufficient. For missing evidence, reward appropriate uncertainty and useful bounded explanation, not invented authority.

Bootstrap whole conversations with fixed seed and declared resample count; keep paired candidate/baseline units together. Predeclare aggregation across sampled repeats within each case. Publish wins/ties/losses, scored and missing units, rubric/category intervals, and raw failure/repair counts. Final-set exposure for candidate selection invalidates its sealed status. Multiple failures from one conversation are not independent samples.

### Local application workload inventory

During QAT-00/02, resolve every model-backed operation in the intended release scope. The following is a checklist, not a claim that each currently runs locally:

| Task | Checks |
| --- | --- |
| Ordinary conversation | Relevant answer, source distinctions, requested depth, visible-only history, topic changes and cancellation |
| Key-term metadata / highlights | Valid schema, exact term/excerpt occurrence, no fabricated spans; metadata never replaces prose |
| Definition / contextual definition | Correct term/sense, follows supplied context and the existing response schema |
| Compaction | Preserves entities, user intent, unresolved questions and supported facts without inserting new claims |
| Question of the Day / labels / child concepts / Midpoints | Existing per-task schema, neutrality, required counts and task boundaries where applicable |
| Repair and queue interactions | Fixed repair allowance, no repair loops, foreground priority, one engine owner and safe cancellation |

Write `task-coverage.json` with each operation's current provider, candidate provider, prompt/schema revision, tests, local/remote status, and in/out-of-release scope reason. A conversation-only success must be labeled conversation-only. Do not claim replacement of the common model provider unless all affected operations pass; keep excluded operations on the known-good path through an explicitly reviewed routing design.

### Memory and latency accounting

Record decimal model bytes and system GiB explicitly. Estimate packed weights from actual file/tensor payloads, scale/zero-point overhead, alignment and unique aliases. Embeddings and per-layer embeddings count regardless of whether only a subset is accessed each token.

For cache planning, use actual allocated layer/token/head dimensions and dtype: `sum_l(allocated_tokens_l × kv_heads_l × (key_dim_l × key_bytes_l + value_dim_l × value_bytes_l))`, adjusted for real K/V sharing, block metadata and padding. Verify the runtime allocation; a sliding attention window does not prove the cache allocation shrinks. Include graph/preallocation, prefill activations, staging copies, app/retrieval buffers and the largest individual Metal buffer.

Require `measured_peak <= 0.85 × frozen_operating_cap` for the proposed headroom rule. Set the cap from observed known-good app behavior and applicable device/runtime constraints, not total RAM or a deliberate production jetsam test. Record supported and fallback context settings as different configurations.

Define timing boundaries: request accepted; retrieval complete; prefill start/end; first generated token; first visible token delivered to UI; visible answer complete; metadata/repair complete. Report time to first visible output honestly if the runtime buffers generation. Stock and candidate probes must report the same boundaries. Cold-load percentiles from ten trials are coarse estimates; preserve every sample and report the maximum as well as p95. Interleave or alternate baseline/candidate sessions when practical to reduce thermal/order bias.

## 11. Stop conditions, recovery, and final deliverables

### Resource and failure policy

Before each long run, compute remaining worst-case writes plus the 25 GiB reserve, including source download, unpacking, export, optimizer/checkpoint state and failed-output cleanup. Plan training memory separately from inference: base/adapter tensors, trainable gradients, optimizer/master state, activation/checkpoint buffers, and export copies. A model fitting inference does not establish training feasibility on the 24 GB Mac.

Freeze a per-phase wall-time/trial/cost cap and memory-pressure/swap-growth thresholds after a bounded pilot. A missing numerical threshold blocks the dependent long job, not read-only work. Avoid a dense teacher and trainable student co-residing unless a measured resource budget supports them. Memory monitoring and graceful-stop checkpoints need their own disk/time reserve.

| Trigger | Required action | Conditions for resuming |
| --- | --- | --- |
| Wrong hash, incomplete download or invalid tensor/format metadata | Reject/quarantine that research artifact before load | Verified complete artifact and fresh validation |
| Unsupported quantization/operation or unintended dense expansion | Stop that route; record capability gap and bounded supported fallback | Pinned supported route plus repeated smoke/parity checks |
| Phone allocation failure, jetsam, or repeated engine crash | Stop that configuration; preserve diagnostics; do not repeatedly retry near the same limit | Revised measured ledger or supported configuration, reviewed failure hypothesis and new run ID |
| Numerical corruption, template mismatch or substantial export loss | Localize with Section 10 ladder before scoring/training | Fixed cause with relevant fixture and behavioral rechecks |
| Non-finite training state or failed resume | Reject current checkpoint as a candidate | Last validated checkpoint or fresh bounded pilot; no unverified partial-state recovery |
| Storage/swap/thermal/resource limit or monitor failure | Graceful stop where feasible; label partial output non-promotable | Restored resources, validated state and updated run budget |
| Better training loss but worse grounded/task behavior | Reject the regression; keep best validated stock/adapted candidate | A distinct bounded hypothesis and development evidence |
| Budget exhausted or repeated inconclusive results | Stop the round and write QAT-09 report | Explicit new scoped budget/hypothesis; no automatic extension |

Separate compatibility failure from model-quality failure in every report. A stock model that cannot run through the available mobile toolchain has not been shown to be intellectually inadequate. A capable source checkpoint with a bad export has not produced a deployable success.

### Adaptation-specific safeguards and checkpoints

Use assistant-answer loss masking and include the exact role/evidence structure required by production; check that truncation retains intended answer targets. Verify the actual trainable parameter list/count before the pilot. Record frozen-versus-updated embeddings, normalization and quantizer parameters; a LoRA implementation unexpectedly training the base is a configuration failure.

Full checkpoint identity includes adapter state, base/source hashes, quantization config, optimizer/scheduler, RNG, data permutation/cursor, precision settings and exporter versions. Preserve immutable frozen state by hash/reference only when it is actually recoverable. Test interrupted-save handling and fresh-process resumption early. Validation/evaluation must not mutate training state; compare a fixed validation fingerprint before save and after reload under the declared tolerance.

No changes to vocabulary/tokenizer are included in the first adaptation experiment. New special tokens or altered embeddings require a separate compatibility/export decision. If preserving mobile quantization requires unavailable QAT tooling, report that limitation and consider the stock model or a supported-format variant; do not promise that generic adapter training will preserve the mobile footprint and quality.

### Final review package and adoption boundary

QAT-09 must contain:

1. A candidate comparison table with exact identities, tested task/context scope, quality intervals, critical failures, retries, footprint, latency/throughput, thermal behavior, and measured versus estimated labels.
2. The selected outcome, why it beats the feasible alternatives or why testing stopped, and the outstanding limitations. Separate product gain from any unproven causal claim about QAT.
3. Reproducibility instructions: verified commands/environments, source/runtime/export provenance, dataset/config hashes, artifact checksums, gate records, failed-run references, and resource costs.
4. For recommended adoption: independent code/config review; model/license/distribution verification; task-routing map; storage/install/hash-validation and cancellation behavior; versioned artifact/runtime pairing; known-good rollback artifact and proof the disposable probe can recover from failed installation/load.
5. A concrete integration proposal describing which provider/tasks change, how offline behavior is preserved, necessary download/storage UI or delivery work, test coverage, and how `MODEL-INTEGRATION.md` will be updated when the change is implemented.

Meeting this plan's gates makes a candidate eligible for a controlled integration decision. It does not make an untested hosted download, production rollout, or replacement of existing user data part of this evaluation. If a model is accepted only for a subset of tasks/context, state that scope explicitly.

## 12. Agent start/resume checklist

1. Read Section 0, the dependency table, and applicable repository instructions. Verify source-of-truth integration facts against the selected checkout and actual artifacts.
2. Inspect current Git/worktree status and active model/build processes. Preserve unrelated changes and do not repurpose another experiment's checkout, virtual environment, device container, or outputs.
3. Load `STATUS.md`, candidate registry, decision register and last gate report. If they are missing, start QAT-00. If records disagree, reconcile with artifact hashes and measured evidence before dependent work.
4. Select the first eligible incomplete candidate/task. Record current scope and the smallest next experiment; missing hardware or approval blocks only work that depends on it.
5. Freeze inputs/configuration, prerequisites, expected output, pass criteria, resource bounds and command before launching. Use new run IDs and distinguish repeat measurement from a changed experiment.
6. Inspect outputs and exit status, record failures, validate artifacts in a fresh process when required, then write the gate result. A successful command exit does not establish model correctness.
7. Update Section 0 and the detailed handoff record together, including evidence links, active processes, cleanup, remaining budget and the exact next command or discovery action. Keep historical reports immutable.

Do not launch parallel agents or heavyweight jobs merely because tasks are separable. Do not repeat completed experiments unless a changed input, failed check, or missing evidence justifies repetition. When a task is blocked, name the missing input and continue eligible independent work; when a gate fails, choose a bounded corrective hypothesis or a documented stop rather than relaxing the criterion.

**Definition of completion:** a fresh agent can locate the model and runtime identities, reproduce the relevant measurements, understand why the selected route passed or failed, and perform the next authorized action without reconstructing this conversation. All experimental task states remain `not_started` until implementation evidence exists.
