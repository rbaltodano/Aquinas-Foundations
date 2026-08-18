# Aquinas-Foundations

Docs-only repo: the product source of truth for the Aquinas project. No code lives here.

## Model integration is live

Before changing model behavior, conversation responses, dynamic definitions, embeddings, or the
Insight Tree, read [MODEL-INTEGRATION.md](MODEL-INTEGRATION.md). It is the cross-repo source of
truth for what is already implemented and what remains.

Current checkpoint: the local fine-tuned Aquinas MLX model is connected end to end through
`../Aquinas_Backend` and `../Aquinas-iOS`. Filtered response streaming, model-generated
user-facing approach summaries, validated tappable key terms, context compaction, and
conversation-cached contextual definitions are live. The iOS UI uses automatic fast/deep routing
and a priority-aware visible task queue with live answer deltas. MiniLM
(`all-MiniLM-L6-v2`, 384 dimensions)
handles persistent conversation-tree relatedness. Automatic response extraction creates Node
Concepts, while manual definition saves create Insights. Do not replace these working paths with
mocks or assume they still need their first implementation.

The finalized prompt architecture has one runtime identity assembler in
`../Aquinas_Backend/main.py`. The global layer contains Aquinas's stable reasoning constitution,
hidden-reasoning protection, and structured-task precedence. Conversation personalities are
injected only into ordinary conversation; definitions, daily questions, extraction, labels,
Midpoints, Make Node children, compaction, and repair remain neutral.

The exact fine-tuned 2.72 GB LiteRT package can be tested without a phone cable through the arm64
iOS Simulator on Apple silicon. Ordinary local conversation stays deterministic because sampled
decoding corrupts this 4-bit checkpoint. It uses separated phrase-loop recovery and mixed-script
corruption rejection. Prompt-forced depth and short-answer retries were tested and removed because
they caused slow local failure; depth work now requires a pre-LiteRT-versus-LiteRT quality comparison.
That comparison now shows a coherent 338-word Mac-checkpoint answer versus mobile-package failure;
Debug simulator builds can use `--force-backend-model` while the Mac backend is running.

- [MISSION.md](MISSION.md) — why the product exists; use it to judge whether a feature fits.
- [DESIGN.md](DESIGN.md) — visual/interaction principles; the reference for design reviews.
- [SPECIFICATION.md](SPECIFICATION.md) / [FUNCTIONALITY.md](FUNCTIONALITY.md) — what the app should do.
- [MODEL-INTEGRATION.md](MODEL-INTEGRATION.md) — how the Aquinas model, MiniLM embeddings, backend,
  persistence, and iOS client work together; includes structured contracts and implementation order.
- [INSIGHT-TREE.md](INSIGHT-TREE.md) — current design spec for the model-driven Insight Tree:
  automatic Node Concept extraction, manual Insights, layout, relatedness, budding, Midpoint, and
  model seams.
- [PERSISTENT_MEMORY_IMPLEMENTATION_PLAN.md](PERSISTENT_MEMORY_IMPLEMENTATION_PLAN.md) — the
  SwiftData migration away from whole-snapshot `UserDefaults`, now implemented; its status section
  distinguishes what's shipped from what remains (the dedicated Insights popup UI, in-place
  field-level diffing, backend deletion-tombstone coordination).

Implementation happens in the sibling repos: `../Aquinas-iOS` (SwiftUI app — has its own CLAUDE.md with build instructions) and `../Aquinas_Backend` (FastAPI + MLX model server). If a session here turns into code changes, prefer starting/continuing it from the repo being changed.

Keep these docs in sync with reality: when a session decides something that contradicts or extends
a doc, update the doc as part of the work. Product requirements and implementation status must be
labeled separately—the local HTTP backend is a development topology, while private local-only
operation remains the production requirement.
