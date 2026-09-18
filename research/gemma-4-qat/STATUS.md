# Gemma 4 QAT research status

**Updated:** 2026-09-18

## Current task and objective

`QAT-01` is **in progress**: M2-L, the published E2B QAT LiteRT-LM artifact, now has verified artifact provenance and historical physical-device smoke evidence. A fresh isolated smoke run is next. The M2/M4 Transformers source-export route remains blocked; the distinct Q4_0 GGUF fallback has only an unproven llama.cpp reference route. `QAT-00` passed on 2026-09-18; its inventory is in [workspace.md](workspace.md) and [baseline-identity.json](baseline-identity.json).

## Current identities

- **B0 — current iOS baseline:** `dynamic_wi8_emb4_afp32` LiteRT-LM package, `3,862,121,696` bytes, SHA-256 `9a6345f1a6cd39283f957977c84d31cc63b8dd56f2b8fffeb784940f63365282`. The manifest and bundled development artifact agree.
- **B1 — diagnostic pre-export lineage:** `Aquinas-Final`, a fused Gemma 4 E2B checkpoint (8.7 GiB locally) derived from `google/gemma-4-E2B-it` plus the recorded Aquinas LoRA configuration. It is available locally, but has not been run or compared in this workstream.
- **Runtime:** iOS wrapper at `Aquinas-iOS-main/Vendor/LiteRTLM`; `Package.swift` pins the macOS binary URL to LiteRT-LM `v0.14.0`. The iOS binary framework hash/revision remains to be captured in QAT-01.

## Last gate

`QAT-00: passed` — current production identity, intended research locations, available artifacts, dirty worktrees, host disk state, and device-probe state were verified. Evidence: [workspace.md](workspace.md), [baseline-identity.json](baseline-identity.json), and [decisions.md](decisions.md).

`QAT-01/M2-L: route verified, fresh smoke pending` — candidate revisions and formats are registered in [candidate-registry.json](candidate-registry.json). [compatibility.md](compatibility.md) records exact artifact identity, prior physical-device evidence, and the remaining fresh-run requirement. M2/M4 source-export compatibility remains blocked.

## Important limits and blockers

- The physical-device and simulator services could not be enumerated: CoreSimulatorService/CoreDeviceService were unavailable. This blocks phone gates only; it does not block QAT-01 metadata work or QAT-02 evaluator preparation.
- Host resource reads are incomplete: sandboxed `sysctl` and `ps` were denied. Disk free space was measured as 104.18 GiB, satisfying the 25 GiB reserve for metadata work, but memory and active-process evidence must be refreshed before any large load/download/export.
- M2-L is supported as a prebuilt QAT LiteRT-LM runtime route, based on exact artifact identity and prior device evidence. No fresh QAT evaluation results exist in this workstream; do not promote the candidate.
- The backend, iOS, and llama.cpp reference worktrees are dirty. Do not edit or run heavyweight work in them; use a dedicated backend research worktree only after the route is selected.

## Last command and result

From `Aquinas-Foundations`, SHA-256 verification of `Aquinas-iOS-main/Aquinas-iOS/LocalModels/gemma-4-E2B-it.litertlm` succeeded and matched the app manifest. It also confirmed two older, distinct backend packages and the local LiteRT-LM package manifest.

The currently active command is a resumable, background download of **M4-L only** from `litert-community/gemma-4-E4B-it-litert-lm` revision `2eee7ac325f20eb8c9ac1d0e972f7c84663062da`. It writes only `../Models/gemma4-qat/gemma-4-E4B-it.litertlm.partial`; it will not be renamed or tested until it matches the expected 3,659,530,240 bytes and SHA-256 `0b2a8980ce155fd97673d8e820b4d29d9c7d99b8fa6806f425d969b145bd52e0`.

## Next smallest action

Complete and hash-verify M4-L, then independently verify its QAT provenance before labeling it QAT. Run M2-L and M4-L only through a disposable phone probe, preserving B0 and production data. QAT-02 cannot pass until B0 is runnable through a controlled baseline evaluator.

## Budget and cleanup

M4-L staging is the sole active download: 3,659,530,240 bytes expected, one partial file, a 45-minute transfer budget, and a start-state free-disk measurement of 104.18 GiB. It retains the 25 GiB reserve and allocates no model runtime memory. If it fails or exceeds the budget, retain the partial file for resumable download and do not treat it as an artifact. No training, export, iOS build, device test, or model mutation was performed.
