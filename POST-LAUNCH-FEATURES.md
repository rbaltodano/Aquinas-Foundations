# Post-Launch Features

This document tracks product ideas intentionally excluded from the launch settings surface. These
features should return only when their behavior is implemented, testable, and accurately described
in the app. A stored preference without a runtime consumer is not a finished feature.

## Custom Instructions

Allow a user to provide durable guidance that applies to ordinary conversations.

Before shipping:

- Define precedence relative to Aquinas's stable system identity and conversation personality.
- Apply instructions only to ordinary conversation. Structured tasks such as definitions,
  compaction, Insight Tree analysis, labels, and retrieval should remain neutral.
- Set a clear size limit and show the user exactly where the instructions apply.
- Defend against instructions that request fabricated quotations, citations, or hidden reasoning.
- Add prompt-assembly tests for empty, conflicting, oversized, and adversarial instructions.
- Keep the instructions on device with the rest of the user's private conversation data.

## Conversation Memory

Cross-conversation memory needs a typed data model, an explicit review/delete surface, and a clear
privacy contract before it appears in Settings. It must distinguish user-approved personal facts
from inferred themes and from ordinary conversation history. No silent memory extraction.

## Fine-Grained Model Behavior

Deferred controls:

- Conversational initiative
- Knowledge level
- Intellectual challenge
- Theological framing
- Response format

The existing Personality control covers the useful launch-level variation. Additional controls
should be introduced only with stable prompt contracts and evaluation cases proving that each
choice causes a distinct, predictable behavior without weakening factual safeguards.

## Research and Citations

Deferred controls:

- Citation preference
- Citation format
- Link handling

These depend on a licensed, versioned local source corpus and a response contract that can tie
claims to retrieved passages. Citation formatting should not launch before citation correctness.

## Insight and Study Preferences

Deferred controls:

- Automatic Insight Mapping modes
- Definition Highlighting modes
- Question of the Day focus

Launch behavior should remain one understandable automatic policy. Mapping and highlighting
preferences can return after semantic thresholds and definition-intent behavior are calibrated
against real conversations. Question of the Day focus can return when local generation reliably
supports subject selection.

## Local-First Follow-Up

Production Aquinas should treat the iPhone as the canonical source of truth. The Mac backend may
remain a development and recovery tool, but the following should move on device where practical:

- Automatic conversation-tree analysis and persisted topology
- Home loose-thread selection
- Glossed-term selection
- Today in History from a bundled, versioned dataset
- Quote-notability evaluation
- Retrieval, citations, and claim support over the local source corpus

Work that depends on the final text model or licensed source library should wait for those inputs.
Deterministic storage, selection, and MiniLM relatedness work can proceed independently.

## Quantization-Aware Training (QAT) Model Candidate

**Closed for the current shipping checkpoint as of 2026-08-13.** The 4-bit DWQ mechanism worked,
but the retry against the actual production 8-bit quantization scheme did not produce a deployable
checkpoint: attempts encountered non-finite gradients, Metal out-of-memory failures, worse
validation, and nondeterministic validation that prevented exact checkpoint/resume verification.
The shipping 8-bit model already has very little quantization error, so the likely product benefit
does not justify more research or hardware time. Production remains on the verified 3.86 GB
`dynamic_wi8_emb4_afp32` package.

Retain the tooling, but do not schedule another retry for this checkpoint. Revisit DWQ only for a
future checkpoint with a demonstrated quantization problem, an intentional lower-bit package with
a meaningful size/latency payoff, or after deterministic validation is proven and stronger hardware
is already available. Prioritize checkpoint/data quality, production-path evaluation, and runtime
optimization instead.

See [Aquinas-QAT-DWQ-Writeup.md](Aquinas-QAT-DWQ-Writeup.md) for the full experiment history,
memory analysis, retained tooling and patches, controlled-retry rejection, and final recommendation.
