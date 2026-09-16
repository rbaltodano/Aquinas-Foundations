# Model Integration — System Plan

> Status: source of truth for connecting the Aquinas language model, semantic embeddings, backend,
> and iOS app. This document covers the whole model pipeline. Feature-specific behavior remains in
> the relevant feature spec, especially `INSIGHT-TREE.md`.

### Implementation status

- **Implemented:** MiniLM dependency and local model cache; reusable backend relatedness provider;
  normalized 384-value embeddings; cosine comparison; normalized Node centroids; health and
  similarity API routes; lazy Insight-to-Node assignment; SQLite persistence for per-conversation
  Insights, Nodes, embeddings, ownership, and relatedness; validated Aquinas contextual-definition
  and conversation-response generation with one JSON repair attempt; conversation/source-scoped
  definition lookup and caching; durable hidden context compaction; validated key-term metadata;
  filtered NDJSON streaming of user-facing approach summaries and response deltas; automatic
  fast/deep conversation routing; direct-JSON fast generation; live iOS delta rendering;
  foreground/background generation priority and tree-analysis preemption; local-first iOS
  generation through a process-scoped LiteRT-LM engine with native streaming, cancellation, and
  adaptive lifecycle ownership; prose-only on-device answers with inline-annotated, exact key
  terms; persisted public approach summaries; a replaceable local factual-grounding provider
  with initial council and Didache reference notes; backend recovery; a priority-aware visible iOS Model Task queue; automatic
  response-analysis extraction into Node Concepts; atomic, idempotent tree mutations with
  provenance and deletion tombstones; sparse persisted Node edges; affected-Node budding; iOS
  save/load/remove/analyze consumption with a durable retry queue; bounded multimodal image
  attachments from iOS through Gemma 4's vision tower; focused provider, generation, tree engine,
  and persistence tests; a backend grounding corpus (`Aquinas_Backend/corpus/sources.yaml`) with
  per-source license-status tracking, an ingestion pipeline (`ingest_corpus.py`), and a
  MiniLM/Chroma passage retriever (`grounding_retrieval.py`) wired into
  `POST /conversation/respond` and `POST /conversation/respond/stream`.
- **Not wired yet:** midpoint bridge persistence and explicit reorganize/merge operations remain.
  Release model delivery still needs a hosted versioned artifact, resumable/background transfer,
  storage/settings UI, and removal of the bundled development seed. The local structured adapters
  validate output and recover through the backend; parity work still includes a local one-repair
  pass for malformed structured output and sustained memory, thermal, lifecycle, cancellation,
  multimodal, and multi-turn testing. On-device factual retrieval still uses the small lexical
  bootstrap catalog described below; the backend's corpus/Chroma retriever does not yet reach the
  on-device LiteRT path, so only backend-recovery conversations benefit from it today. The
  retrieval corpus is retrieval-only supporting evidence for prompts, not a fine-tuning input —
  fine-tuning remains scoped to Aquinas's behavior and voice, per §8.

### On-device LiteRT-LM feasibility checkpoint — July 30, 2026

The iOS app now links a locally vendored Swift package wrapper for Google's LiteRT-LM framework.
An isolated launch-argument path, `--litert-probe --litert-probe-auto`, loads the stock
`gemma-4-E2B-it.litertlm` mobile package with the Metal backend and generates one response without
contacting the Mac backend. The 2,588,147,712-byte model remains gitignored and is diagnostic input,
not a checked-in application resource.

The corrected one-shot probe passed on a base iPhone 17 with 8 GB RAM:

| Measurement | Observed result |
| --- | --- |
| First uncached initialization | approximately 9.1 seconds |
| Subsequent cached initialization | 0.54 seconds |
| Subsequent one-sentence generation | 1.10 seconds |
| Model size shown on device | 2.59 GB |
| Backend | LiteRT-LM GPU path with a successfully created Metal device |
| Stability check | remained alive for more than 25 seconds after completion |

The verified response to “What is prudence?” was: “Prudence is the prudent and judicious
application of practical wisdom to make sound judgments and decisions.”

The first diagnostic implementation exposed an important lifecycle constraint. SwiftUI restarted
the automatic task after completion, the probe replaced its live `Conversation`, and LiteRT
crashed in `SessionBasic` destruction. Making the diagnostic strictly one-shot eliminated the
second initialization and crash. Production integration must therefore give the engine and its
conversations explicit long-lived ownership, serialize creation and teardown, and test
conversation replacement, cancellation, backgrounding, memory pressure, and app termination
before switching the live `AquinasModel` implementation.

This checkpoint answers the hardware-feasibility question: the target 8 GB phone can load and run
the stock mobile model locally with useful latency.

### Fine-tuned Aquinas checkpoint

The fused MLX language tensors were mapped back into the canonical Hugging Face Gemma 4 tensor
layout and exported with the base vision tower. The resulting multimodal LiteRT-LM package uses a
4-bit dynamic-weight decoder, externalized embeddings, an 8-bit vision encoder, a 4,096-token
cache, a 128-token prefill signature, and FP16 activation preference. It is 2,722,385,120 bytes
with SHA-256 `5cb26c8e29d52ecf3e2b651e590761fe593dddcab0cbcdac7dc0692605ee5569`.

The exact fine-tuned package passed on the base iPhone 17:

| Measurement | Observed result |
| --- | --- |
| Cold initialization | 4.33 seconds |
| One-sentence generation | 1.27 seconds |
| Model size shown on device | 2.72 GB |
| Runtime activation type | FLOAT16 |
| Backend | LiteRT-LM GPU path with Metal device/environment created |
| Stability check | remained alive for more than 25 seconds after completion |

The response to “What is prudence?” was: “Prudence is the virtue of the right course of action in
matters of morality and religion.”

An earlier package omitted the FP16 activation preference. Although it selected the GPU backend,
the runtime resolved FLOAT32 activations and required 8.20 seconds to load plus 114 seconds to
generate the same response. Marking the decoder `prefer_activation_type: fp16` made the runtime
resolve FLOAT16, reduced load time by about 47%, and made generation about 90 times faster. This
metadata is therefore a required performance contract, not an optional packaging hint.

This checkpoint answers both hardware feasibility and whether the fine-tuned Aquinas weights can
run locally with useful latency. The app now has a process-scoped local driver behind
`AquinasModel`, native streaming/cancellation, neutral structured-action adapters, adaptive
lifecycle ownership shared with the task queue, integrity-checked model storage/install
primitives, and backend recovery.

### On-device reliability checkpoint — July 31, 2026

The production runtime now treats model initialization as a real result: it retries a failed load
once and then reports failure instead of silently admitting a generation task with no usable
engine. Ordinary conversation remains deterministic because even low-temperature sampling can
corrupt this 4-bit checkpoint. It detects substantial phrase repetition even when repeated spans
are separated, and also rejects mixed-script token corruption. LiteRT-LM currently delivers the
generated text only after native decoding finishes, despite exposing a stream API. Generation
therefore has no automatic wall-clock cutoff: a slow answer is allowed to finish, while the user
can still stop it through the Model Task control.

The exact 2,722,385,120-byte fine-tuned package also runs in the arm64 iOS Simulator on Apple
silicon, so a cable is not required for local quality testing. On an Apple-silicon Mac with 24 GB
RAM, the simulator cold-loaded the package in 13.07 seconds and generated the 17-word prudence
probe in 1.32 seconds. That probe reproduced the shallow-answer behavior independently of the
phone and backend and is the baseline for the depth safeguards. A sampled production-path probe at
temperature 0.2 finished after 68.30 seconds with unrelated multilingual token garbage; this is why
ordinary conversation must stay deterministic. Simulator latency is not a substitute for final
physical-device thermal and memory testing.

Explicit 120–320-word prompt targets and an automatic short-answer expansion retry were also
tested, then removed: the deterministic package entered a rejected loop on the developed justice
and mercy prompt and fell through to recovery after about 41 seconds. The current package therefore
cannot be made reliably deeper by demanding more words at runtime. Answer-depth work must compare
the pre-LiteRT checkpoint against this exact package, then correct the fine-tune or export rather
than shipping a prompt workaround that increases latency and failure rates.

### On-device structured and grounding checkpoint — August 1, 2026

Physical-device testing exposed two separate quality gaps. First, the 4-bit checkpoint can corrupt
proper names and invent historical categories; the full Mac checkpoint can also make factual
errors, so quantization is not the only cause. Fine-tuning supplies behavior and voice, not a
reliable reference library. Second, the original local conversation adapter generated plain prose
and selected highlights with a word-scoring heuristic, bypassing the backend's validated
`key_terms` contract.

Local ordinary conversation now generates the visible answer as prose only. After the answer is
complete, a separate short structured pass selects `key_terms`. The adapter validates exact
term/excerpt occurrence, rejects generic one-word highlights, and falls back to deliberately
conservative deterministic highlighting when metadata fails. Metadata can never replace, truncate,
or leak into the visible answer; legacy JSON answers are still recovered during migration. Public
approach summaries are generated by the app, persisted with their response, and remain
available through **Show Thinking** after navigation or relaunch. They describe the high-level
approach only; raw hidden chain-of-thought is never displayed or stored.

`AquinasGroundingProviding` is the local factual-retrieval seam. Its bootstrap implementation uses
a small offline lexical index with trusted notes for Nicaea (325), Constantinople (381), Nicaea II
(787), and Didache authorship. Retrieved notes are authoritative within their narrow scope and are
inserted before generation. Direct, high-confidence factual questions covered by these notes bypass
free-form generation and use a verified response contract; the local checkpoint is not allowed to
override a retrieved fact. Broader questions still use grounded generation and deterministic output
validation. This fixes known regressions and establishes the prompt/data contract;
that checkpoint was not yet a general knowledge system. The current on-device path runs
MiniLM retrieval for every question, requires corpus evidence for source-dependent claims, and
allows ordinary definitions, reflections, practical discussion, and hypotheticals to use model
knowledge when no passage applies. Those answers are labeled **General knowledge** and do not
seed the Insight Tree. Production replaces ranking with `all-MiniLM-L6-v2`, expands a versioned
licensed corpus, and keeps deterministic validation and explicit uncertainty when retrieval does
not establish a fact.

The first comparison confirms the boundary: the pre-LiteRT Mac checkpoint answered the same
justice-and-mercy prompt coherently in 338 words, with a 4.40-second first visible response and
21.60-second total generation. Debug simulator builds may use `--force-backend-model` to bypass
the bundled mobile package and exercise this Mac path at `127.0.0.1`. Release builds ignore the
flag. This is the working quality-test topology while the mobile conversion is corrected.

The visible Thinking summary is not private chain-of-thought and must never be described as such.
It is a concise public explanation of the approach, distinctions, and trusted facts used for the
answer. Specialized summaries
must apply only to questions they actually fit (for example, act/intention/circumstances is for
judging an action, not every question containing “moral”). Fallback summaries must remain specific
without echoing the user's question or claiming to reveal hidden scratch work.

Conversation prompts must preserve named entities and negation exactly. An explicit authorship
correction receives a focused instruction that keeps the disputed work and person separate and
forbids drifting into the person's other writings without blindly assuming the user is correct.
If a completed local draft both positively asserts an attribution and says the authorship is
unknown, iOS rejects that internally inconsistent draft and performs one bounded corrective retry.
Attribution, source, and date answers must state uncertainty instead of guessing.

Backend recovery is device-aware. The simulator may use `127.0.0.1` to reach the Mac, but that
address means the iPhone itself on physical hardware. A physical-device build therefore never
attempts loopback backend recovery; if its local retry also fails, it returns an honest on-device
retry message. The bundled local model does not require `uvicorn` for ordinary iPhone use.

The remaining production work is:

1. Host and version the package, then connect resumable/background transfer and storage/settings
   UI to the verified installer instead of shipping the 2.72 GB development seed.
2. Add a local one-repair attempt for malformed structured outputs rather than relying on backend
   recovery for repair.
3. Run sustained memory, thermal, background/foreground, cancellation, multimodal, and multi-turn
   quality tests on the base supported phone.

LiteRT-LM does not provide a standard 6-bit Gemma export recipe in the current toolchain. The
higher-precision mobile experiment therefore uses `dynamic_wi8_emb4_afp32`: 8-bit fully connected
decoder weights with 4-bit embedding tables. It is exported beside the working 4-bit artifact and
must pass package-size, cold-load, sustained-memory, latency, and blind answer-quality gates before
the app manifest or bundled development seed is changed.

The first candidate package was produced on August 1, 2026 at 3,862,121,696 bytes with SHA-256
`9a6345f1a6cd39283f957977c84d31cc63b8dd56f2b8fffeb784940f63365282`. Its decoder is 2.14 GiB,
versus roughly 1.1 GiB in the working 4-bit package. The first simulator load failed during LiteRT
GPU engine creation with `Failed to initialize kernel` and delegate rollback. The app stayed alive,
but no generation began. This candidate therefore failed its compatibility gate and must not be
promoted or installed on the phone. The verified 4-bit package remains the app default.

### Topic isolation and mobile-tree checkpoint — August 1, 2026

Physical-device testing showed that the LiteRT session could return an earlier answer verbatim
after a substantial change of subject. iOS now compares the latest two user questions. A clear
topic shift starts generation without prior-turn history, while follow-ups that share the subject
retain context. Completed output is also compared with prior assistant answers; an exact repeat
forces one bounded engine reload and fresh-context retry. These are session-integrity protections,
not factual verification.

Direct-definition cards no longer trigger from the words `what is` alone. Explicit definition
language still qualifies, and a short standalone concept question such as “What is natural law?”
qualifies, but broad questions, ordinal/list questions, requests for methods, and questions ending
in `about` remain ordinary conversation. This intent gate is deterministic so it adds no second
model call to every question.

A focused simulator probe also established a model-quality boundary. The fine-tuned package ran
but incorrectly described the Peloponnesian War as part of the Greco-Persian Wars. The available
stock package failed Metal engine creation before generation, so it is not a deployable replacement
in the current runtime. General factual quality still requires a compatible improved checkpoint
and broader trusted retrieval; the duplicate/session fixes do not make the fine-tune authoritative.

Conversation-tree analysis and MiniLM topology remain backend services. A physical iPhone pointed
at loopback cannot reach them because `127.0.0.1` is the phone itself. In that configuration iOS
now skips and clears unreachable analysis jobs, does not remain on `Mapping…`, and releases the
persisted-tree entrance gate so already available local Insights/Nodes can render. This is an
offline fallback, not an on-device MiniLM implementation. Full local-first tree parity still
requires porting `all-MiniLM-L6-v2`, assignment, and persistence to iOS (or configuring a reachable
Mac LAN backend during development).

### Replacement-model and factual-reliability checkpoint — August 1, 2026

The current fine-tuned Gemma 4 E2B package is compatible with the target phone, but its raw factual
and general-reasoning quality is not sufficient to treat model memory as authoritative. Product
direction is to evaluate the strongest practical raw text model first; multimodal input may be
deferred until a better efficient phone-compatible vision-language model is available. Do not
fine-tune a replacement before measuring its untouched instruction checkpoint, because a narrow
behavior fine-tune can damage general capabilities.

`litert-community/Qwen3-4B` mixed INT4 was evaluated as the first replacement candidate. The exact
artifact is 2,659,057,664 bytes with SHA-256
`f0794bc77efeaaf4f7af815f04c483b19b8f2ae4a102cef1b7b760a25848a18e`. It failed the simulator
Metal gate before generation because one 388,956,160-byte tensor exceeded the simulator GPU's
268,435,456-byte maximum allocation. On the base physical iPhone 17, LiteRT reached Metal
initialization and iOS terminated the process with signal 9 before load completion or generation.
No Qwen answer tokens were produced. This package is rejected and must not replace the bundled
Gemma package. Its isolated backend-workspace copy is diagnostic input only.

A stronger raw model still cannot guarantee factual reliability outside its training memory.
The production reliability design is retrieval-first: ingest approved public-domain/openly
licensed sources automatically, retain source/license/version metadata, chunk and embed passages
with `all-MiniLM-L6-v2`, answer from retrieved evidence, validate claims against that evidence,
and show citations. When the curated library has insufficient evidence, the app must state
uncertainty or—with explicit product support—use current online sources. Fine-tuning controls
voice and task behavior; it is not the factual database.

Physical model-candidate tests must use a disposable probe app/bundle identifier or a complete
verified app-data backup. Never use `xcrun devicectl device copy to` with
`--domain-type appDataContainer --remove-existing-content true` against the production bundle,
even when a nested destination is supplied. A Git commit or Xcode build does not back up
`UserDefaults`. Record Home data counts before and after any install, export the container before
testing, and do not promote a model until simulator and physical-device gates both pass.

### Backend retrieval-first grounding checkpoint — August 2026

The retrieval-first design described above is now implemented on the backend, though it is a
retrieval pipeline only — it is not a fine-tuning input, and fine-tuning remains scoped to
Aquinas's behavior and voice per §8. `Aquinas_Backend/corpus/sources.yaml` is a manifest of
candidate public-domain and openly-licensed sources (Aquinas's own works, Scripture, patristic and
conciliar texts, the primary philosophical and historical sources he cites, and several
Reformation-era confessions), each tagged `confirmed_pd`, `permission_required`, or
`pending_verification`; `ingest_corpus.py` hard-fails on anything other than `confirmed_pd` unless
explicitly overridden. As of this checkpoint, ingestion has fetched 35 sources into
`data/raw_data` and embedded them with MiniLM into a persisted Chroma collection at
`data/corpus/index` (`grounding_retrieval.py`, `GroundingRetriever`). `POST /conversation/respond`
and `POST /conversation/respond/stream` now retrieve nearest passages for the latest user message
and pass them to generation as `grounding_passages`, replacing the earlier small hardcoded
bootstrap notes for those routes.

This closes a real gap in the backend-recovery path's factual grounding, but it does not by itself
fix the product's core factual-reliability problem: the on-device LiteRT path is what ships, and
its `AquinasGroundingProviding` implementation is still the small offline lexical bootstrap
described in the checkpoint above. Porting corpus retrieval on-device, or using the corpus to
improve the next fine-tune/replacement checkpoint's raw factual recall, remains open product
direction, not yet decided as of this checkpoint.

### On-device inline key-term annotation checkpoint — August 8, 2026

The local conversation path's key-term highlighting was reworked twice this checkpoint after
production use exposed problems with each earlier design, and now stands on different footing
than the backend's `key_terms` JSON contract described in §3.

The original local design (`generatePresentationKeyTerms`) ran a **second full LiteRT generation
call** after the prose answer completed: the finished answer was fed back in as literal input to
a separate structured JSON prompt asking the model to copy out exact `display_text`/
`context_excerpt` pairs. This doubled on-device generation cost and GPU/thermal load per
conversational turn (two full `Conversation` creations and prefills instead of one) for a
correctness benefit — the model was handed the exact final text to copy from, so its excerpts
reliably validated. That cost was judged not worth it and the two-pass design was retired in
favor of merging key-term selection into the same generation call as the answer, either as a
trailing JSON field or (the design that shipped) inline markers — see below.

The first single-pass attempt asked the model to return
`{"response": "...", "key_terms": [...]}` as one JSON object. This removed the extra generation
call, but introduced a different failure: while writing `key_terms`, the model had to recall its
own already-written `response` text verbatim from its own recent context rather than copying from
literal input handed back to it, and a quantized, greedy-decoded checkpoint does this unreliably —
`context_excerpt`/`display_text` pairs intermittently failed the required exact-substring
validation, so real answers sometimes had zero surviving key terms for no principled reason. A
deterministic on-device `NLTagger`-based extractor (named-entity tagging plus a small curated
philosophical/conciliar vocabulary) was tried as a complete replacement for model judgment, but
was rejected on selection quality: `NLTagger`'s general-purpose named-entity model has no
awareness of domain-specific vocabulary (e.g. book-of-the-Bible names), so its coverage was
inconsistent in a way with "no apparent rhyme or reason" from the user's perspective, and the
curated lists it fell back to could never generalize to arbitrary topics.

The shipped design keeps single-pass generation but has the model mark terms **inline, in the
prose itself**, wrapping just that word or short term in `{{double curly braces}}` exactly where
it already occurs while writing the answer (`"Aquinas distinguishes {{essence}} from
{{existence}}."`). `LiteRTAquinasModel.inlineAnnotatedResponse` strips the markers back out to
produce the visible text and turns each marker's position into a `KeyTerm`. Because the marker is
removed in place — surrounding prose never moves — `display_text` is an exact substring of the
final answer *by construction*, not by a separate validation step the model can fail. This
eliminates the recall-accuracy problem the two-field JSON design had, without paying for a second
generation call.

This checkpoint's real-device testing also established that the model does not comply with the
inline-marker instruction on every turn — some answers come back with zero `{{markers}}` even
where marking would have been reasonable. Per explicit product direction, **no heuristic fallback
fills this in**: an answer legitimately renders with zero highlighted terms when the model chose
not to mark any, rather than forcing highlights from `NLTagger` or a curated vocabulary just to
avoid an empty state. Highlighting quality is therefore currently bounded by how reliably this
specific quantized checkpoint follows the inline-marking instruction; improving that is a
prompt/fine-tune quality question, not an extraction-mechanism question, and the deterministic
`NLTagger`/curated-phrase machinery was deleted from the codebase rather than kept as a dead
fallback path.

The August 17 prompt revision keeps that architecture but broadens the selection policy:
substantive answers should annotate recognizable article-worthy subjects at Wikipedia-like
frequency (typically 3-5 terms when short and 6-10 when concept-rich, with a ceiling of 12), while
zero is reserved for trivial or purely conversational prose. The same policy is repeated in the
factual accuracy audit so its rewrite does not shed the first draft's links. This is still a
generation instruction rather than a deterministic guarantee; no annotation-only second pass or
heuristic fallback was restored.

## 1. The simple mental model

The system has three different jobs:

1. **Aquinas writes and explains.** It produces conversation answers, contextual definitions,
   Node labels, blended Insights, and generated child Insights.
2. **MiniLM compares meaning.** `sentence-transformers/all-MiniLM-L6-v2` converts short pieces of
   text into vectors so the system can calculate which ideas are related.
3. **Application code makes decisions.** The backend and iOS app use the generated content and
   similarity scores to build the Insight Tree, save state, and render the interface.

Aquinas should not be asked to invent numeric relatedness scores or manage persistent IDs.
MiniLM should not write user-facing content. The application should not depend on parsing
unstructured prose when it needs structured data.

## 2. Current decisions

- **Language model:** Gemma 4 E2B loaded through MLX-VLM with the Aquinas LoRA adapter applied to
  the language tower. This preserves the fine-tuned Aquinas behavior while retaining the base
  checkpoint's vision tower for image understanding. This remains the deployed model, not the
  final quality target; the Qwen3-4B LiteRT candidate was rejected before generation.
- **Embedding model:** `sentence-transformers/all-MiniLM-L6-v2`.
- **Embedding size:** 384 values.
- **Similarity calculation:** cosine similarity on normalized embeddings.
- **Backend:** FastAPI in `../Aquinas_Backend`.
- **Client:** SwiftUI app in `../Aquinas-iOS`.
- **Product direction:** local-first and private. The current backend is a local development
  service; production packaging must preserve the product's local-only requirement.
- **Model output:** structured JSON for application tasks, validated by Pydantic on the backend.
- **Persistent IDs:** generated by application code, never by either model.

### Permanent reasoning constitution

Ordinary conversation has a stable reasoning layer beneath any configurable personality. Aquinas
treats its earlier claims as revisable positions rather than commitments to defend, evaluates the
strongest reasonable version of the user's argument, and applies the same intellectual standard to
both sides. It distinguishes contradictions and invalid inferences from disputed premises, missing
empirical evidence, and differences in definition.

When the user's reasoning defeats a premise, exposes a contradiction, introduces decisive
evidence, or supplies a better distinction, Aquinas explicitly revises the affected conclusion,
explains what changed, and carries the revision through dependent conclusions. Revision extends
only as far as the argument warrants. Aquinas does not change position merely because the user
disagrees, insists, or sounds confident, and does not defend an earlier answer merely for
consistency or authority. Confidence remains proportionate to the available reasons and evidence.

Conversation compaction preserves consequential revisions by recording the earlier claim, the
revised position, and the decisive reason. The revised position governs later turns while material
unresolved disagreement remains visible. Personality settings may change expression and tone, but
not this reasoning standard. Structured application actions continue to use their neutral,
persona-independent instructions.

The Balanced personality joins the intellectual habits of Aquinas to the relaxed voice of a loving
older brother. It reasons through natures, distinctions, causes, ends, objections, and synthesis,
but weaves that depth into contemporary conversational language rather than a formal disputation.
It remains friendly, personal, candid, and emotionally attentive without claiming a relationship
that does not exist. Natural reactions, direct address, occasional inclusive phrasing, gentle
lightness, and companionable endings give the prose character; they remain selective and earned
rather than becoming praise, canned empathy, pet names, or repeated verbal tics.

The global runtime instruction contains this reasoning constitution plus Aquinas's identity,
hidden-reasoning protection, and structured-task precedence; it does not contain a conversational
personality. Ordinary conversation injects the currently selected personality in its
conversation-specific prompt. This keeps definitions, Insights, Nodes, Midpoints, daily questions,
compaction, extraction, and repair neutral regardless of the user's personality setting.
Tokenizer configuration contains no embedded identity or conversational prompt; runtime prompt
assembly in `Aquinas_Backend/main.py` is the sole authority for Aquinas's identity and behavior.
Legacy standalone chat entry points are not retained because they would bypass structured
generation, personality selection, validation, task priority, and durable context handling.

### Prompt authority and action contracts

`Aquinas_Backend/main.py` is the only production code that assembles tokenizer chat messages. It
places the stable runtime instruction in a system message and the operation-specific prompt in a
separate user message. Tokenizer configuration must not contain an embedded identity, persona, or
custom chat-template override. A regression test enforces the single prompt-assembly entry point,
and runtime startup rejects known legacy identity overrides.

| Operation | Core contract |
| --- | --- |
| Ordinary conversation | Apply the selected conversational personality on top of the permanent reasoning constitution; match answer length to the question; return validated response, optional public approach summary, key terms, and any directly requested definition Insight. |
| Contextual definition | Explain the term specifically in its source context using only title, context label, and definition; omit pronunciation, part of speech, and examples. |
| Question of the Day | Ask one grounded, open-ended question from unresolved conversation material; consult at most four relevant Insights and cite one only when it materially contributes. |
| Node label | Name the most elementary concept organizing the supplied Insights in one to five words, using as few words as precision permits. |
| Response-driven tree extraction | Add at most one pivotal, answer-grounded Node Concept; highlighted terms are evidence aids, not automatic Insights. |
| Midpoint | Generate five substantive candidates from all selected sources and their normalized weights; application code selects the candidate mathematically nearest the weighted embedding centroid. |
| Make Node | Generate exactly three distinct, non-overlapping child Insights adapted to the promoted concept. |

When the user quotes an Insight into a conversation turn, application state keeps the quote
separate from the visible question. The local runtime and backend prompt assembler serialize its
title and contextual definition as escaped XML immediately before the question:

```xml
<insight_quote>
<title>Insight title</title>
<definition>Contextual definition</definition>
</insight_quote>

User question:
How does this apply here?
```

The model treats that block as deliberately selected context for resolving references such as
“this” or “that idea.” Classifiers and routing continue to inspect the unwrapped question so quote
markup does not alter direct-definition detection or fast/deep routing.
| Context compaction | Preserve attribution, epistemic status, revisions, unresolved disagreement, preferences, and commitments while removing obsolete wording and repetition. |
| Repair | Change only the invalid or missing contract surface, preserve valid substance, prefer schema-permitted `null` or empty optional values over invention, and fail after one unsuccessful repair. |

All operations except ordinary conversation use neutral, persona-independent editorial language.
The model never owns persistent IDs, vector scores, weights, graph topology, task scheduling, or
retry decisions.

## 3. What Aquinas must generate

### Conversation response

The backend returns the visible answer and presentation metadata in the validated payload below.
The constrained local LiteRT path does not use this JSON contract for key terms at all: it asks
the model for plain prose with important concepts, subjects, named ideas, and specialized words
marked inline at Wikipedia-like editorial frequency —
`{{double curly braces}}` around just that word or short term, exactly where it occurs — in the
same single generation call as the answer. `LiteRTAquinasModel.inlineAnnotatedResponse` strips
the markers back out to produce the visible text and turns their positions into key terms; see
the on-device inline key-term annotation checkpoint above for why (a prior two-pass design and a
single-pass JSON `key_terms` field were both tried and retired first). Its public approach summary
is generated by the iOS adapter.

```json
{
  "response": "Aquinas distinguishes essence from existence...",
  "thinking_summary": [
    "Distinguished the two concepts before explaining their relationship.",
    "Applied the distinction to the question's specific context."
  ],
  "key_terms": [
    {
      "display_text": "essence",
      "canonical_term": "Essence",
      "context_excerpt": "distinguishes essence from existence"
    }
  ],
  "insight": null
}
```

The backend validates that each `display_text` actually occurs in `response` and requires the
exact `context_excerpt` to contain it; the local path gets the same guarantee for free, since
`inlineAnnotatedResponse` derives `display_text` directly from where the marker was in the final
text rather than validating a separately-recalled copy. The client may turn validated terms into
its current `aq://` links, but the model should not be trusted to produce correct application URLs
itself. Key terms now follow the editorial rhythm of useful links in a good Wikipedia article. The
model marks the first meaningful occurrence of important concepts, subjects, people, works,
doctrines, events, and specialized vocabulary that a curious reader may reasonably want defined or
explore further. Central concepts come first, followed by worthwhile adjacent subjects. A short
substantive answer will often contain 3-5 annotations and a concept-rich or multi-paragraph answer
6-10, with a validated ceiling of 12; a substantive answer containing meaningful concepts should
not intentionally return none.
Ordinary connective language, incidental details, indiscriminate proper nouns, and repeat occurrences
remain unmarked. The local path still has no heuristic fallback, so a checkpoint that disobeys the
inline-marker instruction can technically yield zero despite this stronger generation contract.

When the latest user message directly asks for a term to be defined, `insight` is required to
contain that term's complete contextual definition and the iOS
client renders it as an in-text Insight card. An otherwise valid response that omits the required
card uses the existing one-repair attempt. `thinking_summary` is a short, user-facing account of the
answer's approach; it is generated by Aquinas for the response UI and is not raw hidden
chain-of-thought or model scratch work. The API retains `thinking_enabled` (default `false`) for
the public summary and accepts `generation_mode` (`automatic`, `fast`, or `deep`). The iOS product
sends `thinking_enabled: true` and `generation_mode: automatic`: routine questions use a
direct-JSON fast path, while explicit depth requests, analytical/comparative prompts, objections,
proofs, derivations, and multi-part questions retain hidden deep reasoning. The old Thinking toggle
remains removed. The backend stream emits `start` only once generation actually begins.

Every structured repair prompt follows one shared minimal-repair policy: preserve valid substance,
wording, attribution, uncertainty, and qualifications; change only the invalid or missing contract
surface; and never introduce unsupported claims, conclusions, citations, evidence, concepts, or
examples. Invalid key-term metadata is removed rather than forcing a rewrite of valid prose.
Missing required content may be reconstructed only from the original task material supplied again
to the repair prompt. Each operation retains its single repair attempt and fails safely if the
repaired output still does not validate.

Approved response deltas update the visible response card immediately. The final validated payload
atomically adds key-term links. User questions and definitions are foreground work. Automatic
Insight Tree analysis waits for a five-second idle window, runs as background work, and yields at a
generation-token boundary when a foreground request arrives; its durable job is then retried.

Conversation messages may include bounded JPEG, PNG, or WebP image attachments. The iOS client
normalizes uploaded images to JPEG with a 1,536-pixel maximum dimension before base64 encoding
them. The backend validates encoding, file type, decoded size, and pixel count, labels the images
in transcript order, and passes at most the eight most recent images to Gemma 4 in that same order.
Images are request context only; the backend does not persist their bytes.

Conversation prompts define the Tree controls authoritatively so the model can answer interface
questions accurately. **Inquire Connection** analyzes the strongest meaningful relationship among
two to eight selected Insights or Node Concepts without forcing a synthesis; for larger sets it
may identify the organizing pattern. If the user asks what the chip means, Aquinas first gives a
brief plain-language explanation and then performs the analysis. Aquinas can likewise explain
Midpoint and Make Node, but it does not claim that their vector math or topology is language-model
judgment.

### Contextual definition

Input includes the selected term, the sentence it came from, and relevant conversation context.
The definition must explain the term **as used in this conversation**, not merely quote a
dictionary.

```json
{
  "title": "Essence",
  "context": "Essence and existence",
  "definition": "Essence names what a thing is, considered distinctly from the act by which it exists."
}
```

This maps to one context-definition entry on the Swift `ConceptDefinition`. Saved Insights are
unique by normalized title. Saving the same term from a different source context appends a second
context-definition entry to the existing Insight rather than creating a duplicate card or
overwriting its first meaning. Legacy saved definitions migrate as an unlabeled first entry.

For persisted conversations, definitions are cached by conversation ID, normalized term, and a
hash of the source assistant response. The client performs a lookup before enqueueing generation.
This is important for interaction semantics: an already-defined term opens immediately even while
other model work is occupied.

### Question of the Day

Input is recent conversation context plus at most four relevant saved Insights. Aquinas returns
one stand-alone, open-ended question grounded in a specific unresolved idea, assumption,
distinction, tension, consequence, or application. It should invite reflection or judgment rather
than test recall, ask only one thing, avoid leading and yes/no framing, and optionally cite exactly
one provided Insight only when its definition materially contributes. The accompanying
`reason_for_asking` is short editorial rationale, not private chain-of-thought.

The iOS app removes the card once answered. When a question is answered or expired into a new day,
an eligible non-conversation page waits for 15 seconds of model idleness and queues
`Consolidate information`; active status reads `Consolidating...`. Opening an active conversation
does not cancel already queued consolidation. Failure persists no placeholder question.
Backend generation uses a preemptible direct-JSON background path so this small structured action
does not spend its budget on unnecessary hidden reasoning.

### Node label

Input is a cohesive group of Insight titles and definitions.

```json
{
  "label": "Being and Existence",
  "summary": "The distinction between what a thing is and that it is."
}
```

Labels contain one to five words and use as few words as possible. The summary can be used as the
Node definition.

### Response-driven tree extraction

Input is the completed question-answer pair plus the response's already-validated highlighted
terms. Aquinas returns a question-led subject label/summary and zero or one pivotal conceptual
Node Concept seed containing a label, contextual summary, and exact answer evidence. Highlighted
terms guide this check but do not automatically become Insights or Node Concepts. The backend
rejects ungrounded, incidental, non-durable, and semantically duplicate candidates. Aquinas never
assigns IDs, ownership, similarity, or mutation decisions.

### Midpoint blend

Input contains two to eight concepts and the user's normalized blend weight for each one. Aquinas
returns five distinct candidate titles and contextual definitions. Every candidate must integrate
the two dominant nonzero sources; across the pool, lower-weight sources must contribute
substantively. For five or more sources, a separate center pass first produces an integrated
weighted semantic anchor so the result does not become a checklist. MiniLM compares the candidates
with the weighted source-vector centroid; the application selects the mathematically nearest
candidate and subsequently determines where it belongs in the tree.

Midpoint uses its own larger structured-generation token budget. The five-candidate contract
proved unreliable under the smaller budget shared by definitions and Node labels; this separation
keeps lighter actions tight without truncating Midpoint generation or its single repair attempt.

```json
{
  "title": "Prudent Application of Natural Law",
  "definition": "...",
  "example": "..."
}
```

### Make Node children

When an Insight is promoted into a Node Concept, Aquinas returns exactly three useful child
Insights. They must be novel relative to the parent, mutually non-overlapping, and adapted to the
concept rather than drawn from a fixed generic taxonomy:

```json
{
  "children": [
    { "title": "...", "definition": "..." },
    { "title": "...", "definition": "..." },
    { "title": "...", "definition": "..." }
  ]
}
```

The application assigns stable child IDs before saving them.

## 4. What MiniLM must calculate

MiniLM embeds a concise semantic description of each Insight:

```text
Essence. In this conversation, essence means what a thing is, considered distinctly from the act
by which it exists.
```

Do not embed only a short title, and do not embed an entire conversation. The model is intended for
sentences and short paragraphs and truncates long input.

Generate normalized embeddings:

```python
embedding = embedding_model.encode(
    f"{insight.title}. {insight.definition}",
    normalize_embeddings=True,
)
```

With normalized embeddings:

```python
similarity = dot(embedding_a, embedding_b)
distance = 1.0 - similarity
```

Store the raw similarity and model version. A cosine score is a ranking signal, not a percentage
or probability.

### Node representation

A Node's semantic embedding is the normalized average (centroid) of its member Insight embeddings.
Its Aquinas-generated label is for people; its centroid is for comparison.

MiniLM supplies:

- **Insight → Node similarity:** used for ownership and Insight line length.
- **Node → Node similarity:** used for the sparse Node graph and Node line length.
- **Insight → Insight similarity:** used to decide whether loose Insights form a cohesive bud.
- **Blend → source similarity:** used to select ownership and secondary bridge lines.

## 5. Insight Tree decision flow

### Automatic response update

1. The visible answer finishes streaming and is persisted.
2. iOS enqueues its stable response, branch, and conversation identifiers. Reopening retries this
   durable queue but does not synthesize jobs for older transcript history.
3. `POST /insight-tree/{conversation_id}/responses/{response_id}/analyze` extracts candidates.
4. Exact evidence and durable-content validation run before embedding.
5. MiniLM skips similarity to an existing Insight or Node at **0.86** or above.
6. A remaining seed is inserted directly as a Node Concept, never as an automatic Insight.
7. If the Tree is empty and Aquinas returns no specific seed, the first substantive pair can still
   create its question-led subject Node.
8. The transaction rebuilds sparse Node edges and persists the result under the response ID.

The same response ID always returns the stored original result. A failed transaction leaves the
tree unchanged.

### Saving a bookmarked Insight

1. Aquinas generates a contextual definition.
2. The user chooses Save.
3. Application code creates the Insight ID.
4. MiniLM embeds its title and definition.
5. The tree engine compares it with existing Node centroids.
6. It joins the most-related Node if the score is above the membership threshold.
7. Otherwise, it creates a Node and asks Aquinas for the Node label.
8. Only affected centroids, edges, and layout targets are updated.

### Node graph

Application code builds a minimum spanning tree using `distance = 1 - similarity`, then adds
additional Node pairs above the strong-relatedness threshold. This keeps the graph connected
without creating an all-pairs hairball.

### Automatic budding

1. Find a Node's loosely related Insights.
2. If there are more than three, compare those Insights with one another.
3. Bud only a subgroup of at least three that are mutually cohesive.
4. Ask Aquinas to label the new Node.
5. Persist the moved membership and keep the new Node linked to its origin.
6. Do not automatically undo the bud later.

## 6. Backend boundaries

The backend should expose task-specific operations rather than one endpoint that returns arbitrary
text. Exact routes may change, but the responsibilities should remain:

- `POST /conversation/respond` with `generation_mode`
- `POST /conversation/respond/stream` for filtered `thinking_summary`, `response_delta`, and
  validated `complete` NDJSON events; `start` and `complete` include the resolved mode
- `POST /conversation/compact` for a durable hidden checkpoint that replaces older model context
  while leaving the client-visible transcript unchanged
- `POST /concept/define`
- `POST /conversation/{conversation_id}/concept/lookup`
- `POST /conversation/{conversation_id}/concept/define`
- `POST /home/question-of-the-day`
- `POST /insight-tree/label-node`
- `POST /concept/blend`
- `POST /concept/children`
- `POST /insight-tree/{conversation_id}/responses/{response_id}/analyze`

`POST /insight-tree/assign` remains a stateless diagnostic route. The persistent application flow
is `POST /insight-tree/{conversation_id}/insights`; it loads the conversation's existing tree,
assigns and embeds the new Insight, updates its owning Node, and commits the result to SQLite.
`GET /insight-tree/{conversation_id}` returns the stored tree without exposing raw embeddings.
`DELETE /insight-tree/{conversation_id}/insights/{insight_id}` removes an Insight, repairs the
owning Node's centroid and member scores, and removes the Node if it becomes empty. Automatic
Insights also leave a source-response tombstone.

Suggested services:

```text
AquinasGenerationService
├── respond
├── define_term
├── label_node
├── blend_concepts
└── generate_children

MiniLMRelatednessProvider
├── embed_insight
├── node_centroid
├── insight_node_similarity
├── insight_insight_similarity
└── node_node_similarity

InsightTreeEngine
├── assign_new_insight
├── evaluate_bud
├── build_sparse_node_edges
└── produce_graph_mutation
```

Load both models once at backend startup. Do not reload either model for each request.

## 7. Persistence contract

The persistent tree is per conversation. At minimum, save:

- Conversation, Insight, and Node IDs.
- Insight title and contextual definition.
- Insight embedding and embedding-model version.
- Exactly one owning Node ID per Insight.
- Node label, summary, and member Insight IDs.
- Settled/budded membership state.
- Node origin link for automatic buds.
- Midpoint source IDs and bridge relationships.
- Raw relatedness scores used by visible edges.
- Stable positions or layout anchors.
- Response-analysis jobs/results keyed by response and branch ID.
- Generated Node Concept seeds, with model-returned label and contextual summary.

iOS persists only pending identifiers, never duplicate transcript text. It retries with bounded
backoff while active and whenever the conversation is reopened; the stored conversation supplies
the question and answer.

Conversation-scoped dynamic definitions are also stored in backend SQLite. Their cache identity is
the conversation ID plus normalized term plus source hash, so the same visible term can have
different definitions in different response contexts.

Raw embeddings can remain on the backend. The iOS app normally needs only content, topology,
scores, and positions.

When the embedding model or semantic text format changes, increment the embedding version and
re-embed saved Insights in a controlled migration.

## 8. Fine-tuning policy

The existing Aquinas fine-tune teaches domain knowledge, voice, and response style. First try
task-specific prompts plus strict JSON validation; do not retrain merely to connect the app.

If structured tasks remain unreliable, add training examples with explicit task markers:

```text
<TASK:CONVERSATION>
<TASK:CONTEXTUAL_DEFINITION>
<TASK:NODE_LABEL>
<TASK:BLEND>
<TASK:GENERATE_CHILDREN>
```

Do not train Aquinas to invent similarity numbers. If MiniLM struggles with fine theological
distinctions, improve it separately with a labeled contrastive dataset.

## 9. Evaluation and threshold tuning

`Aquinas_Backend/evaluation/prompt_quality_cases.json` is the shared end-to-end prompt-quality
catalog. Its real-model runner covers conversation, contextual definitions, Question of the Day,
Node labels, response-driven extraction, Midpoint, Make Node, and compaction. Every operation has
representative and adversarial cases plus a forced malformed-output repair probe.

Objective checks validate contracts without asserting exact prose. Subjective accuracy,
philosophical depth, contextual fit, neutrality, restraint, diversity, and repair preservation use
explicit 1–5 human rubrics in a readable Markdown report. Reviewed scores live in the versioned
`prompt_quality_reviews.json` catalog and merge by case and rubric ID, so focused reruns do not
erase the audit. Runs are filterable and resumable, and Midpoint cases optionally load MiniLM to
record which candidate is mathematically nearest the weighted source-vector centroid. The
generating checkpoint must not grade its own substantive quality.

The separate relatedness/threshold set should include:

- Related and unrelated theological concept pairs.
- Insights that should share a Node.
- Insights that should remain in separate Nodes.
- Loose Insights that should and should not bud together.
- Expected Node labels and blends.

Tune membership, budding, cohesion, and strong-edge thresholds separately. Do not assume one
number works for every decision, and do not treat cosine similarity as a probability.

## 10. Implementation order

1. ~~Add shared schemas and structured Aquinas task methods.~~ Implemented for conversation,
   compaction, contextual definitions/caching, Question of the Day, response-driven Node Concept
   extraction, Node labels, Midpoint candidates, and Make Node children.
2. ~~Add a single, startup-loaded MiniLM relatedness provider.~~ Implemented.
3. Implement contextual definition and validated key-term output end to end. The backend
   contextual-definition route is implemented, verified against the real local model, and wired
   into iOS. The conversation route and validated key-term output are also implemented, verified
   against the real model, and wired into iOS.
4. Persist Insights, Nodes, membership, embeddings, and model version per conversation.
   Implemented for the current attach-or-create topology; edges, positions, and later origin/source
   metadata will be added with their respective features.
5. ~~Replace the Swift chain placeholder with lazy membership from the tree engine.~~ Implemented
   for per-conversation Canvas Mode: saving a contextual definition persists it through the
   backend, and iOS renders stored ownership and Insight-to-Node distances. The global Insight
   Library canvas intentionally remains an in-memory view.
6. ~~Implement sparse MST Node edges and relatedness-based line lengths.~~ Implemented.
7. ~~Wire Node labels and Make Node child generation.~~ Implemented, including the stable-ID
   reveal flow and exactly-three-child validation.
8. Midpoint candidate generation and client-side weighted-centroid selection are implemented.
   Persist midpoint ownership and bridge lines.
9. ~~Implement cohesive automatic budding.~~ Implemented for response mutations and affected
   Nodes only.
10. ~~Build the structured prompt-quality evaluation set.~~ Implemented with 27 initial cases,
    objective contract checks, human-review rubrics, incremental result persistence, and Markdown
    reporting. Continue adding real regressions, tune graph thresholds against a separate labeled
    relatedness set, and verify migrations/offline packaging.

## 11. Current code seams

### Backend

- `../Aquinas_Backend/server.py` exposes the structured generation, relatedness, and Insight Tree
  routes described in §6; `POST /ask` remains available as the older unstructured route.
- `../Aquinas_Backend/main.py` is the sole tokenizer prompt assembler. It supplies the permanent
  reasoning constitution and structured-task precedence as the system message; operation prompts
  remain separate user messages. It loads the full Gemma 4 E2B multimodal base through MLX-VLM and
  applies `models/aquinas_adapters` to the language tower.
- `../Aquinas_Backend/structured_generation.py` provides validated contextual-definition,
  conversation-response, Question of the Day, compaction, Node labeling, response-driven Node
  Concept extraction, Midpoint, and Make Node tasks, including JSON extraction and one automatic
  repair attempt per operation. The real Aquinas model requires generous task budgets because it
  may use several hundred
  hidden-reasoning tokens before producing its final JSON.
- `POST /conversation/respond` accepts up to 20 recent user/assistant messages plus an optional
  `thinking_enabled` flag, `generation_mode`, and selected `personality`. Each message may include
  up to eight validated image objects containing `name`, `media_type`, and `data_base64`; the
  generation path uses at most eight images across the recent transcript. The route returns prose,
  an enabled-only approach summary, validated key terms whose `display_text` occurs in that prose,
  and a required in-text Insight for direct definition requests.
- `POST /conversation/respond/stream` filters raw generation into approved NDJSON fields. Its
  `start` event occurs after the MLX generation lock begins producing output, not merely when the
  HTTP request arrives.
- `POST /conversation/compact` merges a previous checkpoint and turns since that checkpoint into a
  validated replacement summary.
- `POST /concept/define` returns an uncached structured contextual definition. Conversation-scoped
  lookup/define routes persist and reuse results by normalized term plus source hash.
- `POST /home/question-of-the-day` generates a validated neutral daily question from recent
  conversation context and at most four Insights.
- `POST /insight-tree/label-node`, `POST /concept/blend`, and `POST /concept/children` provide
  elementary Node labels, five weighted Midpoint candidates, and exactly three Make Node children.
- `../Aquinas_Backend/relatedness.py` loads MiniLM once from the local cache and provides normalized
  embedding, comparison, and centroid operations.
- `../Aquinas_Backend/insight_tree.py` uses MiniLM to attach a new Insight to its best Node or
  create a new Node. The `0.40` membership threshold is a tunable placeholder.
- `../Aquinas_Backend/tree_store.py` stores per-conversation Insights, Nodes, normalized
  embeddings, owning membership, model version, provenance, response-analysis results, tombstones,
  and sparse Node edges in
  `data/insight_tree.sqlite3`.
- `POST /insight-tree/{conversation_id}/insights` performs persistent assignment,
  the response-analysis route performs atomic automatic mutation, `GET
  /insight-tree/{conversation_id}` reads the saved tree, and the corresponding `DELETE` route
  removes an Insight and repairs its former Node. Raw embeddings remain backend-only.

### iOS

Every generative call funnels through `AquinasModel`
(`Aquinas-iOS/Services/AquinasModel.swift`). `AquinasApplicationRuntime` is the composition root.
When `LiteRTModelStore` resolves a verified package, it creates one process-scoped
`LiteRTAquinasRuntime` and shares that exact instance between `LiteRTAquinasModel` and
`ModelTaskQueue`. The actor owns one long-lived LiteRT engine, serializes conversation creation
and teardown, streams accumulated visible text, and forwards cancellation to the native session.
`LiteRTAquinasModel` runs ordinary conversation locally as a single generation call: prose answer
plus inline `{{term}}` markers for important explorable terms, stripped back out by
`inlineAnnotatedResponse` into key terms with no separate structured pass and no heuristic
fallback. Substantive answers are prompted toward 3-5 markers when short and 6-10 when concept-rich,
with a validated ceiling of 12;
trivial or purely conversational answers may carry zero. While generation
is incomplete the UI reports work in progress without presenting canned text as model reasoning;
the app-generated public approach summary is stored in `ChatBranch.responsePresentations` so its disclosure
survives view recreation and persistence. Its deterministic sampler matches the original MLX
runtime so 4-bit quantization noise is not amplified by a broad high-temperature sample. The local
grounding provider supplies narrowly relevant trusted reference notes before generation. It also returns
direct-definition Insight metadata and adapts compaction, contextual definitions,
Node labels, weighted Midpoint candidates, Make Node children, and Questions of the Day to
validated local structured prompts. `BackendAquinasModel` is used when the package is absent and
as recovery for non-cancellation local failures. `MockAquinasModel` remains previews/tests only.

`ModelTaskQueue` serializes user questions, contextual definitions, and tree updates. Its state
drives the `Idle` / `Thinking` status button, `n/total` progress, popup rows, cancellation, removal,
and upcoming-task reordering. Completed task rows clear after a short idle delay. Cancelling a job
must invoke its UI cleanup closure.

Every queued model operation also acquires a lease from the actor-isolated
`ModelRuntimeLifecycleManager`. The backend driver is explicitly always-resident. The live
LiteRT driver releases its engine, conversation/KV cache, and compute resources on unload. Its
adaptive policy keeps weights warm for five foreground-idle minutes,
shortens that window to 60 seconds under serious thermal pressure, and unloads as soon as active
leases finish after backgrounding, critical thermal pressure, or a memory warning. Background
queue work is preempted without running its destructive cancellation callback, remains queued,
and resumes after the app becomes active. MiniLM is outside this lifecycle and stays resident.
A cold task shows `Loading...` before returning to its task-specific status.

Model runtime transitions, load/unload intervals, thermal state, time between tasks, and resident
memory are recorded with local OS signposts. The iOS test target uses a fake unloadable driver and
short timeouts to verify warm retention, idle/thermal unloads, lease safety, cold reload, the
always-resident backend, and preservation of background work.

For a tapped term, iOS first calls `cachedDefinition`; only a miss enqueues `defineTerm`. The
conversation-scoped routes use the selected term, source response, and transcript to return a
contextual `ConceptDefinition`. Persistent IDs (`stableUUID(from:)`,
`ConceptDefinition.stableID(forTerm:)`) remain application-assigned.

`compact` sends the existing checkpoint plus only the uncompacted transcript. On success,
`ChatBranch.compactedContext` and `compactedThroughBlockCount` are persisted while the visible
transcript is left untouched. `/clear` is a client-side reset and does not call the model.

`InsightTreeService` is the separate deterministic-tree client boundary. Completed responses enter
an identifiers-only durable queue after persistence; `BackendInsightTreeService` analyzes them in
the background and refreshes only on an actual mutation. Saving a contextual definition writes it
to both the global Insight Library and the active conversation tree; a durable client-side ID
association supports later reconciliation. Bookmark removal and conversation-tree deletion remain
distinct actions. Canvas Mode renders backend-owned membership,
Insight-to-Node distances, and sparse Node edges. A failed request leaves the current tree intact.

Study Topic trees reuse this boundary with the topic UUID as a separate persisted scope. After the
user accepts a topic update, iOS derives the desired Insight set from the durable memberships of
all conversations assigned to that topic. Opening the topic canvas reconciles saved-definition
additions and removals before applying the backend MiniLM snapshot. The accepted aggregate is also
persisted on device, providing the local fallback when the backend cannot be reached.

The simulator uses `http://127.0.0.1:8000`, configured by `AquinasBackendURL` in `Info.plist`.
An Xcode scheme can override it with `AQUINAS_BACKEND_URL`. If a development task intentionally
uses the backend from a physical device, it must use the Mac's reachable LAN address and the
backend must listen on `0.0.0.0`. Physical-device builds reject loopback as a recovery target.
The Info plist declares local network usage and a local-network ATS exception; this HTTP
arrangement is for private local development, not remote production traffic.

Live availability behavior:

- Local conversation and structured-action failures recover through the backend unless the user
  cancelled or a physical device is configured with a loopback URL. If both paths fail, the
  action exposes its normal explicit availability failure.
- Definitions, Node labels, Midpoint candidates, Make Node children, and Questions of the Day use
  throwing model operations. A failure never substitutes `MockAquinasModel` output, and mock
  generation is restricted to previews and tests.
- Definition and canvas actions expose a retryable error. Failed Midpoint and Make Node generation
  remove their temporary identity-bearing loading state and restore the prior Tree. A failed daily
  question is not persisted and can be retried through the normal idle queue.
- Cached real definitions remain immediately usable without new generation. Durable conversation
  membership and tree-analysis work remain available for later reconciliation.
- `EmbeddingProvider`'s only iOS implementation is `NLEmbeddingProvider` (on-device
  `NLEmbedding`), used by legacy in-memory/global-library canvas features. Persisted conversation
  trees use backend MiniLM membership and distances.
- The global Insight Library canvas clusters bookmarks in the local embedding space and builds a
  sparse connected Node graph; conversation Canvas Mode consumes the stored backend MiniLM
  topology.
- Midpoint bridge persistence and explicit reorganize/merge operations are not implemented.

## 12. Guidance for future agents

- Read this file before changing model/backend integration.
- Read `INSIGHT-TREE.md` before changing tree behavior or layout.
- Preserve the distinction between the iOS priority-aware Model Task queue and the backend
  generation coordinator. Both serialize access, but foreground work may preempt background tree
  generation.
- Keep deep-think output limited to an approved user-facing approach summary. Never expose raw
  chain-of-thought.
- Look up a conversation-scoped definition before enqueueing generation.
- Keep generation, relatedness, graph decisions, and rendering as separate layers.
- Use structured, validated data across backend/client boundaries.
- Preserve stable application-generated IDs.
- Never silently replace MiniLM similarity with Aquinas-generated numeric scores.
- Update this file when an integration decision, schema, provider, or implementation phase changes.
- Update the relevant feature spec when product behavior changes.
- Test external multi-gigabyte models only in a disposable app container. Never use
  `--remove-existing-content true` with the production app-data domain.
