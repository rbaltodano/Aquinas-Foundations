# Aquinas Foundations

The shared product, design, and architecture documentation for
[Aquinas](https://github.com/rbaltodano/Aquinas-iOS), a private, local-first iOS study and
conversation environment.

This repository contains no code. It is the source of truth that the
[iOS app](https://github.com/rbaltodano/Aquinas-iOS) and the
[development backend](https://github.com/rbaltodano/Aquinas-Backend) are built against. When
architecture or behavior changes, these documents are updated alongside the corresponding
application or backend work.

## Product and design

| Document | Purpose |
| --- | --- |
| [`MISSION.md`](MISSION.md) | Why Aquinas exists and the principles used to judge whether a feature fits |
| [`DESIGN.md`](DESIGN.md) | "The Scriptorium" visual and interaction design system |
| [`FUNCTIONALITY.md`](FUNCTIONALITY.md) | Current reference for user-visible behavior |
| [`POST-LAUNCH-FEATURES.md`](POST-LAUNCH-FEATURES.md) | Ideas intentionally held back from the launch surface |

## Architecture and contracts

| Document | Purpose |
| --- | --- |
| [`SPECIFICATION.md`](SPECIFICATION.md) | Technical specification and core data model |
| [`MODEL-INTEGRATION.md`](MODEL-INTEGRATION.md) | Language model, embeddings, retrieval, backend, and iOS integration contracts |
| [`INSIGHT-TREE.md`](INSIGHT-TREE.md) | Design and data contract for the model-assisted Insight Tree |
| [`PERSISTENT_MEMORY_IMPLEMENTATION_PLAN.md`](PERSISTENT_MEMORY_IMPLEMENTATION_PLAN.md) | Planned SwiftData persistence migration (not yet implemented) |

## Model research

On-device model quality is the project's hardest constraint. These documents record the
experiments, including the ones that failed, so decisions stay grounded in measured results.

| Document | Purpose |
| --- | --- |
| [`research/QWEN-TESTING-CASE-STUDY.md`](research/QWEN-TESTING-CASE-STUDY.md) | Case study of the llama.cpp / Qwen3-4B fine-tuning investigation |
| [`research/Aquinas-QAT-DWQ-Writeup.md`](research/Aquinas-QAT-DWQ-Writeup.md) | Quantization-aware training (DWQ) session writeup |
| [`research/GEMMA-4-QAT-EVALUATION-PLAN.md`](research/GEMMA-4-QAT-EVALUATION-PLAN.md) | Gemma 4 E2B / E4B QAT evaluation plan, with working notes in [`research/gemma-4-qat/`](research/gemma-4-qat/) |
| [`research/ROTATED-TERNARY-COMPRESSION-PLAN.md`](research/ROTATED-TERNARY-COMPRESSION-PLAN.md) | Rotated ternary compression research plan |
| [`research/LLAMA-CPP-MIGRATION-SCOPING.md`](research/LLAMA-CPP-MIGRATION-SCOPING.md) | Scoping llama.cpp as an alternative to LiteRT-LM |
| [`research/WI8AFP32-REQUANTIZATION-SCOPING.md`](research/WI8AFP32-REQUANTIZATION-SCOPING.md) | `wi8_afp32` requantization scoping, with its [blind evaluation set](research/wi8afp32-eval-questions.txt) |

## For coding agents

Agents should begin with [`CLAUDE.md`](CLAUDE.md), which summarizes the current architecture and
routes work to the relevant document. Implementation happens in the sibling repositories, which
are expected to be checked out next to this one.

## License

Copyright © 2026 Ryan Baltodano. All rights reserved. The source is public for reference and
review; see [`LICENSE`](LICENSE) for details.
