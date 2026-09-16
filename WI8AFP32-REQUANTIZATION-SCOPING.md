# wi8_afp32 requantization — scoping and game plan

**Status: Phase 1 done, Phase 2 fully done (backend: all 17 questions; on-device: all 17
questions), Phase 3/4/5 not started — Phase 3 (non-functional gates) is next.**
This is a planning document, not a final record — delete it once the initiative ships and fold its
lasting facts into `MODEL-INTEGRATION.md` and `Aquinas-iOS/CLAUDE.md` permanently, the same way
prior model transitions (4-bit → `dynamic_wi8_emb4_afp32`) are documented there today.

**Phase 1 result (2026-08-20):** full multimodal `dynamic_wi8_afp32` export succeeded.
`Aquinas_Backend/models/Aquinas-Candidate-wi8afp32/model.litertlm` — 5,237,853,408 bytes (4.9 GiB),
sha256 `f4d719188bc3a487615bbd5fc3fdb2306b16b340203c37faa1bc717c1f51c04e`. Export took ~12.5 minutes
end to end (vision encoder export/quantization alone was ~4.5 min of that). This is bigger than
the shipped 3.86 GB `dynamic_wi8_emb4_afp32` package — expected, since the embedding table is no
longer compressed to 4-bit.

**Phase 2 backend-half result (2026-08-20):** ran the full 17-question eval set
(`wi8afp32-eval-questions.txt`, sits alongside this file) against both full-precision configs (base
and fine-tuned, via `Aquinas_Backend`'s MLX path, not LiteRT) — full transcript in
`/tmp/wi8afp32_backend_eval_results.txt` on this Mac (copy it out before it gets cleaned up if this
needs to survive a reboot). This *isolates fine-tune vs. base* but does **not** test the actual
quantized mobile export — that's the on-device half of Phase 2, still pending (physical device was
`unavailable` both times this was checked; retry when reachable).

**Honest read of the backend results — mixed, not a clean win:**

- **All 6 Scripture chapter questions correct in both configs.** No chapter-number confusion at
  all (the original John 14/John 4 failure mode) — base config even quoted real verse text
  correctly. This directly supports the original hypothesis for *that* failure mode.
- **Doxology, Council of Trent, Peloponnesian War:** correct in both configs. Peloponnesian War in
  particular no longer conflated with the Greco-Persian Wars (a previously documented failure in
  `MODEL-INTEGRATION.md:226-230`).
- **Didache authorship — fine-tuned config regressed, base config was correct.** Fine-tuned
  answered "Who wrote the Didache?" with a fabricated attribution to "Simon the Stylite" (unrelated
  5th-century figure, not connected to the Didache at all), and on "Was the Didache written by
  Paul?" it answered "No" and then **directly contradicted itself** two sentences later ("generally
  considered to have been written by Paul"). Base config answered both correctly and consistently
  (authorship unknown/anonymous, not Paul). This is a real counter-example to "it's not the
  fine-tune" — for this specific fact, the fine-tune measurably hurt, full precision or not.
- **Nicaea I and II — both configs hallucinated, differently.** Fine-tuned claimed Nicaea II
  (787 AD, correct date) was presided over by "Athanasius of Alexandria" — anachronistic, since
  Athanasius died in 373 AD, over 400 years earlier. Base config got the *date* wrong instead,
  claiming the Second Council of Nicaea was "held in 325 AD" (that's Nicaea I's date), while
  separately also crediting "Athanasius of Alexandria" as presiding bishop of Nicaea I (also wrong
  — Athanasius attended Nicaea I as a young deacon, not as presiding bishop).
- **Conversational questions** (friendship/patience, faith and reason, natural law) — reasonable
  and accurate in both, no red flags.

**What this means:** the embedding-precision fix is real and targeted — it fixes the specific
chapter/number-recall failure mode this initiative started from. It does **not** make the model
generally hallucination-free; authorship and presiding-figure details still get fabricated at full
precision, in both configs, just differently. This reinforces rather than undercuts the existing
hard-grounded bypass entries in `AquinasGrounding.swift` (Nicaea I/II, Didache authorship) — those
are catching exactly the category of error that persists here regardless of quantization or
fine-tuning. Don't treat this quantization fix as a reason to remove or de-prioritize that
grounding. Phase 3's gates and the on-device Phase 2 pass are still required before any Phase 4
decision — and given this mixed result, the on-device pass should specifically include an Nicaea/
Didache authorship recheck, not just the scripture-chapter questions that started this.

**Phase 2 on-device result (2026-08-20), priority questions only:** ran 5 highest-risk questions
(the original "Tell me about John Chapter 14" bug, plus Nicaea I, Nicaea II, and both Didache
authorship questions from backend testing) through the real `LiteRTAquinasModel.respond()`
production pipeline on a physical iPhone 17, using the full multimodal candidate
(`wi8afp32-full-candidate.litertlm` in the app's Documents folder). All 5 correct — no
hallucinations reproduced. Most notable: **"Tell me about John Chapter 14" — the exact original
bug this initiative started from — is fixed** (8.62s load, 12.46s real generation): correctly
covers the Farewell Discourse, the Advocate, "the way, the truth, and the life," and explicitly
distinguishes it from John 4/the Samaritan woman. This is the second independent confirmation (the
first was the text-only skip-vision candidate) — the fix holds with vision included too.

For the other 4, **read this carefully before treating it as validation**, because it isn't a
clean re-test of the backend findings:

- Nicaea I (12.11s load, 48.61s generation) and Nicaea II (17.36s load, 20.11s generation): both
  correct, matching the existing curated grounding text almost verbatim (including the exact
  Catechism §242 citation). These likely benefited from the app's existing soft-grounding
  injection or the `verifiedGroundedResponse` bypass regardless of which model is loaded — they
  don't isolate the new quantization's own contribution.
- "Who wrote the Didache?" (6.80s load, 11.17s generation — a real generation, not the bypass):
  correct, "author is unknown... no reliable evidence." This is the more meaningful data point,
  since it didn't hit the near-instant bypass and still avoided the backend fine-tuned config's
  fabricated "Simon the Stylite" attribution. Not a fully clean comparison though — this run had
  the app's grounding context in its prompt; the backend eval script had none at all.
- "Was the Didache written by Paul?" (10.12s load, **0.01s generation**): correct and consistent,
  no self-contradiction — but the 0.01s generation time means this hit the deterministic
  `verifiedGroundedResponse` hard bypass, not real model generation. This result mainly confirms
  the existing safety net works, not that the new quantization fixed the self-contradiction bug
  seen in the raw backend test.

**Net read:** nothing bad reproduced on-device, which is a genuinely good sign, and the original
John 14 bug is confirmed fixed via real generation (not a bypass) on two independent builds. Of the
other 4, though, 3 were substantially aided by the app's existing grounding safety nets rather than
being a clean test of the new model's raw knowledge. The remaining eval questions (doxology,
Council of Trent, Peloponnesian War, and a fresh on-device pass at the Scripture chapters) haven't
been run on-device yet and would give a cleaner read, since fewer of them are covered by existing
hard-grounded entries. **Latency is a real, separate concern regardless of correctness** — cold
loads ranged 6.8–17.4s and real generations ranged 11–49s on physical hardware, well above what the
shipped 4-bit-embedding package does today. This needs a proper Phase 3 pass, not just a vibe check
from these 4 runs.

**Phase 2 on-device result (2026-08-21), remaining 12 questions — full 17/17 now complete:**

| # | Question | On-device result | Load / Gen | Backend result (for comparison) |
|---|---|---|---|---|
| 1 | Tell me about John Chapter 14 | ✅ Correct, distinguishes from John 4 | 8.6s / 12.5s | ✅ Correct (both configs) |
| 2 | Tell me about John Chapter 3 | ✅ Correct — Nicodemus, born again, John 3:16 | 6.2s / 16.6s | ✅ Correct (both configs) |
| 3 | Tell me about Matthew Chapter 5 | ✅ Correct — Beatitudes, Mosaic law deepened | 12.8s / 18.1s | ✅ Correct (both configs) |
| 4 | Tell me about Romans Chapter 8 | ✅ Correct — no condemnation, adoption, Spirit | 6.3s / 23.7s | ✅ Correct (both configs) |
| 5 | Tell me about 1 Corinthians Chapter 13 | ✅ Correct content, **but leaks internal grounding-note text into the visible answer** | 6.1s / 27.8s | ✅ Correct (both configs, no leakage — backend has no grounding at all) |
| 6 | Tell me about Psalm 23 | ✅ Correct — shepherd, valley of the shadow of death | 6.2s / 18.9s | ✅ Correct (both configs) |
| 7 | What does doxology mean? | ✅ Correct, concise | 15.7s / 2.4s | ✅ Correct (fine-tuned had an odd unrelated "doxxing" tangent; base was clean) |
| 8 | What was the Council of Trent? | ✅ Correct | 9.0s / 30.9s | ✅ Correct (both configs) |
| 9 | What was the Peloponnesian War? | ✅ Correct, no Greco-Persian conflation | 6.8s / 25.9s | ✅ Correct (both configs) |
| 10 | Who wrote the Didache? | ✅ Correct — "unknown," real generation | 6.8s / 11.2s | ❌ Fine-tuned fabricated "Simon the Stylite"; base correct |
| 11 | Was the Didache written by Paul? | ✅ Correct, consistent — hit the hard bypass (0.01s gen) | 10.1s / 0.01s | ❌ Fine-tuned self-contradicted ("No" then "generally considered... Paul"); base correct |
| 12 | What was the First Council of Nicaea? | ✅ Correct, matches curated grounding text | 12.1s / 48.6s | ⚠️ Fine-tuned wrongly credited "Athanasius" as presiding bishop; base got the *date* wrong instead |
| 13 | What was the Second Council of Nicaea? | ✅ Correct — 787 AD, icon veneration | 17.4s / 20.1s | ⚠️ Fine-tuned anachronistically credited Athanasius (d. 373) as presiding in 787; base wrongly dated it 325 |
| 14 | What is the meaning of the word prudence? | ✅ Correct, concise | 9.0s / 4.2s | ✅ Correct (both configs) |
| 15 | How can friendship help a person become more patient? | ✅ Reasonable, no factual claims at risk | 7.4s / 19.2s | ✅ Reasonable (both configs) |
| 16 | What is the difference between faith and reason according to Aquinas? | ✅ Mostly excellent, **but fabricates "the Philosopher's Stone"** (alchemy term, likely garbled "the Philosopher" = Aristotle) | 10.9s / 53.5s | ✅ Reasonable (both configs, no similar fabrication seen) |
| 17 | Can you explain what natural law is? | ✅ Correct, well-reasoned | 10.0s / 27.0s | ✅ Reasonable (both configs) |

**17/17 on-device runs avoided the category of hallucination this initiative set out to fix**
(chapter/reference confusion) and 15/17 had no issues of any kind. Two new, different problems
surfaced that are worth separate follow-up, neither of them the original bug:

1. **Grounding-note leakage** (Q5): the model's visible answer included what reads like raw
   internal grounding-reference text ("...which is referenced in the note as..."). This is a
   prompt-leakage bug in how grounding context is presented, not a hallucination — worth a
   dedicated look at `conversationSystemInstruction`'s grounding-injection wording in
   `LiteRTAquinasModel.swift`, independent of this quantization initiative.
2. **"The Philosopher's Stone"** (Q16): a real, if minor and isolated, fabrication — conflating an
   alchemical term with Aquinas's actual habit of calling Aristotle "the Philosopher." One
   occurrence in 17 questions; not remotely at the rate of the original chapter-confusion bug, but
   proof this candidate isn't hallucination-free either.

**Latency across the full 17-question on-device run:** cold loads ranged 6.1–17.4s, generations
ranged 0.01s (the one hard-bypass hit) to 53.5s, with most real generations landing in the
15-30s range. This is the headline concern for Phase 3 — a 15-50s wait per turn is a real UX
cost regardless of correctness, and needs a proper measurement pass (not just these incidental
timings) before any promotion decision.

## Why

The shipped production package (`dynamic_wi8_emb4_afp32` — 8-bit decoder weights, **4-bit
embeddings**) hallucinates unpredictably on basic chapter/reference questions. Confirmed on a
physical iPhone 17 across three separate runs of "Tell me about John Chapter 14": it answered
"John chapter 4" (Samaritan woman at the well), then a response about John the Baptist, then a
completely fabricated "First Apology, Chapter 4" citation with an invented quote — three different
wrong answers to the same question, non-deterministic-looking despite deterministic decoding,
because each run's context/retry state differed slightly.

A controlled 3-way comparison (see "Evidence" below) isolated the cause: it is **not** the LoRA
fine-tune, and quantization matters more than this project's own docs previously concluded for a
different pair of questions (doxology/Council of Trent). Specifically, it is the **4-bit embedding
table** — where token/number identity such as chapter numbers lives — that's destroying recall.
Both the fine-tuned model and the un-fine-tuned base model, run at full precision, answered
correctly. A `dynamic_wi8_afp32` (8-bit weights **and** 8-bit embeddings, no 4-bit anywhere) export
of the same fine-tuned checkpoint also answered correctly, on-device, through the real production
`LiteRTAquinasModel.respond()` pipeline.

Decision made 2026-08-18: fix this properly (requantize and promote a real candidate) rather than
patch around it, even if that costs significant time. Soft grounding-in-prompt (the Scripture
stopgap added to `AquinasGrounding.swift` in a separate, already-shipped change) was tested and
confirmed **not sufficient** — the model ignored the injected reference outright in the
"First Apology" failure above.

## Evidence already gathered (2026-08-18, do not re-derive)

Three-way comparison on "Tell me about John Chapter 14," via `Aquinas_Backend`'s MLX/`mlx_vlm`
path (not LiteRT-LM — this was a precision/fine-tune isolation test, not a mobile-runtime test):

| Config | Precision | Result |
|---|---|---|
| Fine-tuned + quantized (shipped LiteRT package, `dynamic_wi8_emb4_afp32`) | 8-bit decoder / 4-bit embeddings | Hallucinates ("John chapter 4", etc. — varies by run) |
| Fine-tuned (`google/gemma-4-E2B-it` + `models/aquinas_adapters`) + full precision | fp16 | Correct — identifies Farewell Discourse, Advocate, etc. |
| Base (`google/gemma-4-E2B-it`, no adapter) + full precision | fp16 | Correct, most accurate — quotes actual verse text |

Then, a real on-device test of the actual mobile export recipe: `scripts/export_litert_aquinas.py
--source models/Aquinas-Final-HF --output models/Aquinas-Test-wi8afp32 --quantization-recipe
dynamic_wi8_afp32 --skip-vision` produced a 5,071,574,992-byte (5.07 GB) text-only `.litertlm`
package. Tested on a physical iPhone 17 (device "Ry") through the real
`LiteRTAquinasModel.respond()` pipeline (see "How to test on-device" below for the exact working
invocation — it took several failed attempts to get right):

> "John chapter 14 is part of Jesus's Farewell Discourse... He also speaks about the way, the
> truth, and life... Jesus promises to send the Holy Spirit, which he calls the Advocate or
> Helper... **This passage contrasts with the account of Jesus and the Samaritan woman at the
> well, which is found in John chapter 4**"

Correct, and explicitly distinguishes the two chapters that were previously being confused. Cold
load 7.79–10.00s, generation 3.36–11.70s across runs (physical device, GPU).

This confirms the mechanism (embedding precision) but is **one question on a text-only candidate
that hasn't been through any promotion gate.** Everything below is what's still needed before this
is a real production decision, not a demo.

## How to test on-device (hard-won — don't rediscover this)

Debug builds support `--litert-probe` (routes the app's root view to the probe screen) and
`--litert-probe-auto` (auto-runs on launch). But the probe view's default `run()` path uses a
**hardcoded** smoke-test question (`"What is prudence?"`, literally hardcoded in
`LiteRTDeviceProbe.swift`) and completely ignores `--litert-probe-question`. That flag is only
read inside `runConversationQualityProbe()`, which requires the **separate**
`--litert-quality-probe` flag to trigger — and that function is the one that actually goes through
the real `LiteRTAquinasModel.respond()` production pipeline (grounding, audit guard, everything).
The working invocation, once the app is installed and the candidate `.litertlm` file has been
copied into the app's on-device Documents folder (`xcrun devicectl device copy to --domain-type
appDataContainer --domain-identifier com.ryanbaltodano.Aquinas-iOS --source <path> --destination
Documents/<name>.litertlm` — do **not** add `--remove-existing-content`, that flag wipes
`UserDefaults` per the standing warning in `Aquinas-iOS/CLAUDE.md`):

```sh
xcrun devicectl device process launch --device <device-udid> com.ryanbaltodano.Aquinas-iOS \
  --litert-probe --litert-probe-auto --litert-quality-probe \
  --litert-model-document <name>.litertlm \
  --litert-probe-question "Your question here"
```

**Also non-obvious:** `devicectl device process launch` does not restart an already-running
instance of the app — it just foregrounds it, silently reusing whatever probe result is already
on screen (confirmed by identical cold-load/generation timings across "different" runs). Always
check `xcrun devicectl device info processes --device <udid> | grep -i aquinas` for an existing
PID and `xcrun devicectl device process terminate --device <udid> --pid <pid>` it before each
relaunch, or you'll silently re-read stale results.

To see the phone's screen without physically holding it: `xcrun devicectl device capture
screenshot --device <udid> --destination <path>.png` — real screenshot capture, works over the
network once paired, no separate app needed.

## Game plan

### Phase 1 — Produce the real candidate (not yet run)

The tested 5.07 GB package was `--skip-vision` (text-only) for speed. Production needs the vision
tower (the app supports image uploads per `Aquinas-iOS/CLAUDE.md`). Re-run without `--skip-vision`:

```sh
cd Aquinas_Backend
litert_conversion_env/bin/python scripts/export_litert_aquinas.py \
  --source models/Aquinas-Final-HF \
  --output models/Aquinas-Candidate-wi8afp32 \
  --quantization-recipe dynamic_wi8_afp32
# needs 40+ GiB free disk (script enforces this); expect longer than the ~8 min skip-vision run —
# vision_encoder export/quantization is called out in export_litert_aquinas.py's own --skip-vision
# help text as "the single largest memory/disk consumer in the export"
```

Record the resulting byte count and `shasum -a 256` — needed for Phase 4's manifest update and for
folding into permanent docs at the end.

### Phase 2 — Real quality testing (not yet run)

One correct answer is not validation. Build a blind eval set (~15-25 questions) covering
categories already known fragile in this project's own history — pull from
`MODEL-INTEGRATION.md` and `LLAMA-CPP-MIGRATION-SCOPING.md`:

- Scripture chapters: John 14 (done, correct), John 3, Matthew 5, Romans 8, 1 Corinthians 13,
  Psalm 23 (the same set curated in `AquinasGrounding.swift`'s Scripture stopgap — good chance to
  cross-check whether the better-quantized model needs the soft grounding injection at all, or
  makes it redundant)
- Known prior hallucinations: "What does doxology mean?", "What was the Council of Trent?" (both
  documented hallucinating on the shipped package in `LLAMA-CPP-MIGRATION-SCOPING.md:39-46`)
- Known prior confusion: something touching the Peloponnesian War (previously conflated with the
  Greco-Persian Wars per `MODEL-INTEGRATION.md:226-230`)
- Authorship/attribution questions (Didache, etc. — the existing hard-grounded set, as a
  regression check)
- A few ordinary conversational questions with no "gotcha," to confirm no regression in normal
  answer quality/voice

Run each through both the shipped package and the new candidate via the `--litert-quality-probe`
invocation above, on the physical device, and log both answers side by side for review. This is
the actual go/no-go evidence — not a vibe check on one question.

### Phase 3 — Non-functional gates (not yet run)

`Aquinas-iOS/CLAUDE.md` already states the rule: "Never promote a candidate before base-iPhone
load, latency, memory, stability, and blind answer-quality gates pass." Concretely, on the base
supported iPhone (17, 8 GB RAM):

- Cold load time
- Generation latency (single-turn and multi-turn/follow-up)
- Memory usage and thermal behavior under sustained use (multiple turns, not one probe call)
- The existing corruption/repetition guards (`generationGuardRejectsRepetitiveLoops`,
  `generationGuardRejectsSeparatedPhraseLoops`, `generationGuardRejectsMixedScriptCorruption` in
  `LiteRTProductionRuntimeTests.swift`) still hold against the new package — these were written
  against the 4-bit package's failure modes; confirm they still make sense for this one

### Phase 4 — Wire in as production, and a decision that can't be deferred

Swapping `LiteRTModelManifest.aquinas` (byte count, sha256) in
`Aquinas-iOS/Services/LiteRTModelStore.swift` is mechanically trivial. **How the 5.24 GB package
actually reaches the phone is the real decision**, and it's now fully scoped (2026-08-21 research
pass) rather than a vague TODO:

**What already exists** (don't rebuild these):
- `LiteRTModelManifest` + size/SHA-256 verification, and a 3-tier resolution order (dev override →
  verified Application Support download → bundled seed) — `LiteRTModelStore.swift:62-90, 160-172`.
- `LiteRTModelInstaller.install(from:)` — a working one-shot download-verify-atomic-install: HTTP
  status check, size check, chunked SHA-256, move-into-place with a `.verified.json` receipt
  (`LiteRTModelInstaller.swift:34-89`).

**What's genuinely missing, confirmed by code search across all three repos, not inferred:**
- **A hosted URL.** Nowhere — not in `Aquinas-iOS`, `Aquinas-Foundations`, or `Aquinas_Backend`.
  `install(from:)` accepts any caller-supplied URL but nothing supplies a real one.
  `Aquinas_Backend/CLAUDE.md:44-53` explicitly keeps model artifacts gitignored, and the backend
  has no S3/GCS/CDN/static-file-serving code anywhere. **Where this would actually live (S3, GCS,
  GitHub Releases, a route on the existing FastAPI backend, Firebase) is a completely open
  decision** — not something already chosen that just needs wiring up.
- **Resumable/background transfer.** Current installer uses a single foreground
  `URLSession.shared.download(from:)` — no `URLSessionConfiguration.background`, no resume-data,
  no retry/backoff. A 5+ GB transfer over real-world mobile networks without this will fail
  regularly.
- **Progress reporting.** The installer returns only a final `URL` on completion — no `Progress`,
  delegate, or byte-count stream for a UI to show.
- **Any UI at all.** Nothing in the app currently calls `install(from:)` outside developer/test
  code. No download button, no progress bar, no storage-used display, no cancel/delete control.
  `Aquinas-iOS/Features/Settings/PrivacyAndDataSettingsView.swift` (453-561) is the closest
  structural precedent — it already has a "Data Controls" subsection and even an unimplemented
  placeholder row pattern (`SettingsUnavailableActionRow`) — but nothing model-storage-specific
  exists; this would be new screens built on existing scaffold components
  (`SettingsDetailScaffold`, `SettingsSubsection`, `SettingsControlCard`).
- **Removal of the bundled seed.** It remains a permanent fallback in `installedModelURL()`;
  nothing currently plans to strip it from release builds.
- **App size / App Store constraints.** Not discussed anywhere in any repo's docs — Apple's
  per-app-thinning-variant and cellular-download-warning thresholds haven't been checked against a
  5+ GB bundle. Needs fresh research before deciding to bundle, not after.

**The actual decision, now concrete:**

| Option | What it needs | Cost |
|---|---|---|
| **A. Bundle the 5.24 GB candidate** (defer download infra again) | Nothing new to build — same pattern as today, just a bigger file | Cheap now, but grows the exact debt this doc's Phase 4 already flagged as overdue; every dev build and every install gets heavier |
| **B. Build real download delivery** | Pick a host, add resumable `URLSession` background transfer + progress API to `LiteRTModelInstaller`, build a real settings screen, strip the bundled seed from release builds, verify against App Store size limits | Real scope — a multi-day effort on its own, not a Phase-4 afternoon task |

Given the "do it right, timeline is irrelevant" decision that started this doc, **B is the
directionally correct choice**, but it is large enough that it may deserve to be its own
initiative/scoping doc rather than a subsection of this one. Decide explicitly with the user before
starting Phase 4 code — don't default into A by momentum just because it's the path of least
resistance in the moment.

### Phase 5 — Documentation (fold in, then delete this file)

Update `Aquinas-iOS/CLAUDE.md`'s "Model integration is live" section and
`../Aquinas-Foundations/MODEL-INTEGRATION.md` with: the new recipe (`dynamic_wi8_afp32`), final
byte count and sha256, the retirement of `dynamic_wi8_emb4_afp32` and why (embedding precision
caused unreliable chapter/number recall — link back to the evidence above, condensed), and updated
promotion-gate results. Same pattern already used for the 4-bit → 8-bit-decoder transition
documented in `Aquinas-iOS/CLAUDE.md` today. Once that's done, this scoping file has served its
purpose — delete it.

## What's separate / already shipped, don't re-bundle into this initiative

- The meta-commentary/hedging guard and narrowed accuracy-audit trigger (in
  `LiteRTAquinasModel.swift`) fix a *different* failure mode (confused clarification-seeking
  responses) and are already shipped. Keep them regardless of this initiative's outcome — they're
  orthogonal and cheap.
- The Scripture grounding stopgap (`AquinasGrounding.swift`) is already shipped too. It's
  confirmed insufficient on its own (soft injection, model can ignore it) but may still be worth
  keeping alongside a better-quantized model as defense in depth — re-evaluate in Phase 2 whether
  it's still pulling weight once the base model is more reliable.
- Conversation sampling is back to fully deterministic (`topK: 1, temperature: 0`) — do not
  reintroduce sampling as a workaround for hallucination; that was tried, didn't fix the root
  cause, and just made failures non-reproducible. This requantization is the real fix path.
