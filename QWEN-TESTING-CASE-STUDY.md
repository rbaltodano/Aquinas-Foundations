# Case Study: The llama.cpp / Qwen3-4B Fine-Tuning Investigation

**Branch:** `codex/llama-cpp-12b-research` (Aquinas-iOS, Aquinas_Backend, Aquinas-Foundations —
research-only, never merged to `main`). **Timeframe:** late August 2026. **Status:** concluded;
branch parked, not deleted.

## The Question

Production Aquinas runs a fine-tuned Gemma 4 checkpoint via LiteRT-LM. During ordinary use, the
model was observed confusing **John 14** with **John 4** — a citation error, not a reasoning error.
That raised an obvious question: is this a *Gemma* problem, fixable by swapping in a different base
model, or something more fundamental? `llama.cpp`'s Metal backend had separately been confirmed to
run a text-only Qwen3-4B GGUF on a physical iPhone at usable speed, so this became the test vehicle:
fine-tune Qwen3-4B on the same Aquinas voice/behavior data, wire it into the real conversation UI
(not just a benchmark harness), and see whether the same failure mode reappears — and if it does,
figure out why, rather than assuming "wrong model" and moving on.

## Setup

- Official `Qwen3-4B-Q4_K_M.gguf` (`ggml-org/Qwen3-4B-GGUF`), confirmed to load and generate on a
  physical iPhone 17 through a locally built Metal `llama.cpp` framework (~18 tok/s sustained,
  no jetsam at 2K–4K context).
- A disposable `LlamaCPPAquinasModel` conformance let this run inside the **actual app UI** —
  same conversation view, same input flow — gated behind a `--llama-cpp-chat` launch flag and a
  separate research bundle ID (`com.ryanbaltodano.Aquinas-iOS.ModelProbe`) so it could never reach
  the production runtime or container.
- Fine-tuning via `mlx_lm lora` (LoRA, all layers, batch size 1, `max-seq-length 2048`), fused and
  converted to GGUF via `llama.cpp`'s own converter (`mlx_lm`'s built-in GGUF export doesn't support
  the Qwen3 architecture).

## Round 1 — Raw continuation training on the Summa Theologica

The first attempt trained directly on ~3,100 Summa Theologica articles (full text: Objections, "On
the contrary," "I answer that," Replies) against one fixed generic prompt ("Explain this Scholastic
concept using the method of the Summa"). The result was bad in a specific way: answers skipped the
actual thesis and jumped straight into a stray "Reply to Objection N," sometimes drawing the
opposite conclusion, with severe verbatim repetition loops on longer generations.

**Root cause, found by data analysis, not guesswork:** 32% of training examples ended mid-sentence,
clustering tightly around ~3,200 characters — one PDF page. The raw source repeats the current
article's title as a running page header, and the split regex (`Article. N -`) treated every
occurrence as a new article boundary, including the repeated headers — so a third of "articles"
were random mid-article fragments mislabeled as complete ones. An ellipsis-based first fix only
caught headers with long/truncated titles (69% fixed); the real fix compared each heading's article
*number* to the immediately preceding heading's number — a repeated running header always cites the
same in-progress article, so this reduced corruption to 0.5%. Retrained on the corrected data (v2):
val loss 1.808 → 1.579, and the "skips the thesis" failure mode was gone.

## Round 2 — v2 vs. real production Gemma, same prompt

With clean data, the next test was a direct comparison: the same John 14:6 prompt against both the
v2 fine-tune and real production Gemma via the Mac backend. The fine-tune still fabricated citations
confidently, and one long-generation response contained a literal token-corruption artifact — a real
Scripture quote (1 Cor. 2:9) with the nonsense string **"Obedience 3056478"** injected mid-sentence.
Repetition-penalty sampling (`penalty_last_n=64, penalty_repeat=1.15`) tamed short-range loops but
missed a case where two "Reply to Objection" blocks repeated verbatim more than 64 tokens apart —
outside that window's reach.

A separate experiment tried steering register via a system prompt ("answer in plain contemporary
English, not archaic scholastic phrasing"). It had **zero measurable effect** — the fine-tuned model
never saw a system-role turn in training, so it likely never learned to treat one as a real
instruction; register stayed fully archaic regardless.

## Round 3 — Fixing what fine-tuning can actually fix

Two real, separable problems had emerged: **repetition** (a sampling/decoding issue) and
**fabrication** (a training-data-format issue). Each got its own fix:

- **DRY sampler** (`llama_sampler_init_dry`, full-context penalty window) added ahead of the
  short-range penalty, specifically to catch repeats spaced further apart than 64 tokens. This
  eliminated the exact-repeat loops in later testing.
- **Reasoning-transfer reformatting**, the more substantial change: raw-continuation training taught
  the model to *produce Summa-shaped prose on demand*, not to *apply a specific answer to a specific
  question* — so at inference time it fell back on completing the learned shape (Objection →
  citation → Reply) with invented content, since nothing in training mapped a real question to a
  real answer. Fix: extract each article's own `"Whether ...?"` heading as the question and just its
  `"I answer that,"` determination as the answer — dropping the Objection/citation/Reply apparatus
  entirely rather than training on it. 99.4% of articles (3,093/3,113) matched this extraction
  cleanly. A second source — Rickaby's 1905 public-domain abridged translation of the *Summa Contra
  Gentiles* ("Of God and His Creatures," Internet Archive `ofgodhiscreature00thom_0`) — was added
  unreformatted, specifically for its continuous argumentative prose without a citation-heavy
  scaffold, giving the model exposure to answering well *without* falling into that format at all.

The resulting v3 dataset (2,833 train / 311 valid / 311 test) retrained cleanly (final val loss
2.063, ~3.3 hours on an M-series Mac once back on stable AC power — throughput measurably drops
under battery-critical throttling, a real operational gotcha worth remembering for future runs).

## Round 4 — Did it work?

Retested v3 on the same John 14:6 question. **Genuine improvement:** no Objection/Reply scaffold, no
repetition loop, no token-corruption artifact — coherent, conversational prose. **The core problem
persisted anyway:** it confidently cited John 13:32, 14:5, 16:7, and 14:8 with specific quoted text
attributed to each — none of which matched the real verses (verified against the Douay-Rheims text).
The fabrication wasn't fixed; it just stopped happening inside a citation-apparatus scaffold.

A final experiment tested the actual hypothesis directly: **if given the real verse text as context,
does the model use it accurately?** Real Douay-Rheims John 14:1–9 was supplied in-prompt with "using
only this passage, tell me about John 14." The model didn't fabricate — but it also didn't
substantively answer, producing a short, vague, true-but-unhelpful meta-comment instead. It had never
seen a training example shaped like "context passage + question → answer grounded in that passage,"
so it had no learned behavior for using supplied context at all.

## Conclusions

1. **Fine-tuning changes voice and format, not factual reliability** — confirmed three independent
   ways in one investigation: raw continuation training caused format-locked fabrication; reasoning-
   pair reformatting fixed the format lock but not the underlying fabrication; and grounding text
   without a matching training shape didn't get used correctly at all.
2. **This was never really a Qwen-vs-Gemma question.** The fine-tuned failure mode — confident,
   fluent, wrong citations — was structurally identical to the original Gemma complaint that started
   this whole investigation. Switching base models didn't address the root cause, because the root
   cause isn't in the base model.
3. **Reliability requires retrieval, and retrieval requires matching training data.** A small model
   cannot be fine-tuned into having reliable parametric recall of arbitrary source text. Closing this
   gap needs real passages supplied at inference time *and* training examples that teach the model
   what to do with them — neither alone is sufficient.
4. **An unplanned but valuable side-finding:** investigating this surfaced that
   `Aquinas_Backend`'s retrieval-first grounding (`corpus/sources.yaml`, `ingest_corpus.py`,
   `grounding_retrieval.py`) is already a real, general-purpose system — dozens of public-domain
   sources including the full Bible, already embedded into a live Chroma index, already wired into
   backend conversation generation. The on-device iOS path (`MiniLMGroundingProvider`,
   `OnDeviceGroundingStore`) is fully coded to consume an export of that exact corpus, but the export
   itself, and a bundled Core ML MiniLM model, don't exist yet. `CLAUDE.md` in both the Aquinas-iOS
   main and research repos described only the old small hardcoded stopgap and has been corrected to
   reflect this. Closing the on-device grounding gap looks like a bounded engineering task (an export
   script + a Core ML model conversion), not a new corpus-building project — and is a more promising
   lever for factual reliability than anything on this branch.

## Outcome

The `codex/llama-cpp-12b-research` branch is concluded and parked — kept (not deleted, since the
DRY-sampler fix, the SCG cleaning script, and the reasoning-pair reformatting script are reusable
later), but not merged to `main` and not under active iteration. Recommendation going forward:
continue with Gemma as the production model, and prioritize on-device grounding parity over any
further fine-tuning experimentation — that is the lever that can actually move factual reliability,
on either model.
