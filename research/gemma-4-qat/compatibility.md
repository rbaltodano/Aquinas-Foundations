# QAT-01 compatibility result

**Outcome: M2-L has a verified prebuilt LiteRT-LM route; a fresh isolated rerun is still required.** M2/M4 source-export compatibility remains blocked. This is a deployment finding, not an assessment of candidate quality.

| Candidate | Source / revision | Available local route | Result |
| --- | --- | --- | --- |
| M2-L | `litert-community/gemma-4-E2B-it-litert-lm` / `b3ca0d2f076785a8f4b2219ddbd2bdb99954eae1` | Vendored LiteRT-LM `v0.14.0` iOS CPU/GPU | **Route verified.** The local `LiteRT-Stock/gemma-4-E2B-it.litertlm` exactly matches the published filename, byte count (`2,588,147,712`), and SHA-256 (`181938105e0eefd105961417e8da75903eacda102c4fce9ce90f50b97139a63c`). The maintainer identifies this published package as mixed int2/int4/int8 QAT. Existing physical base-iPhone evidence documents GPU load and generation (3.89-second cold load, 0.78-second one-shot generation); repeat it under this workstream's run record before promotion. |
| M2 | `google/gemma-4-E2B-it-qat-mobile-transformers` / `dd693ff40353f057ca5f07e945ad867f4afbf2ec` | Current LiteRT exporter | Blocked. Official metadata describes the source as `wNa8o8` mobile QAT with static activations, targeted 2-bit decode layers, and optimized KV caches. The installed exporter (`litert_torch 0.10.0.dev20260730`) accepts only `dynamic_wi4_afp32`, `dynamic_wi8_emb4_afp32`, or `dynamic_wi8_afp32`; it has no declared preservation/import path for the mobile-QAT schema. |
| M4 | `google/gemma-4-E4B-it-qat-mobile-transformers` / `8e3ac78f1a4a97c19f5f194ab362a2206f71d961` | Current LiteRT exporter | Same block as M2. No conversion, model load, or smoke test was attempted. |
| Q2 | `google/gemma-4-E2B-it-qat-q4_0-gguf` / `675cff42a74c774d6cb76f76d8eacb49b48c9b93` | Research-only llama.cpp iOS reference | Potential fallback only. The existing dirty reference checkout has a `LlamaCPPChatSession` that loads a `.gguf`, requests 4,096 context tokens, and enables Metal context creation. It does not establish that this exact QAT GGUF loads, fits, or matches the active app contracts. |
| Q4 | `google/gemma-4-E4B-it-qat-q4_0-gguf` / `4b4a2c1d584be7264f87aac328a1bc739ce81b6c` | Research-only llama.cpp iOS reference | Potential fallback only; same unresolved smoke, memory, prompt, and application-contract checks as Q2. |

## Runtime provenance captured

- The active app wrapper is LiteRT-LM. Its local `Package.swift` refers to LiteRT-LM `v0.14.0` for the macOS binary; the iOS framework contains arm64 and arm64-simulator slices. The arm64 framework binary SHA-256 is `32c2576cddd50542934289b850ce2061115f90f63de2857ff6f7c5f8f2425607`.
- The installed converter environment reports `litert_torch 0.10.0.dev20260730` and `transformers 5.14.1`.
- The present E2B baseline exporter requires a normal Gemma4 Transformers layout and exports a newly selected dynamic quantization recipe. It does not validate original QAT scales, static activations, mixed-width tensors, or mobile-QAT KV treatment. Re-exporting M2/M4 through it would be a different candidate and cannot be labeled mobile-QAT evaluation.

## LiteRT-LM route follow-up and correction

LiteRT-LM's current public release is `v0.16.0`; it documents Gemma 4 E2B/E4B LiteRT-LM packages under `litert-community`, including an E4B command-line example. The current E2B maintainer clarification establishes that the published E2B package is QAT, despite the filename. It does **not** establish a public Transformer-source import/conversion route for M2/M4 or package provenance for E4B. The project’s public QAT-support clarification issue remains open. Consequently, replacing the local `v0.14.0` wrapper with `v0.16.0` is not required to evaluate M2-L and is not evidence that M2/M4 source weights will load.

M2-L is a separately identified prebuilt deployment candidate, not a claim that the M2 Transformers source-export route passed. The existing local 2,588,147,712-byte stock package matches M2-L exactly. It is not the active B0 app package and has not yet been rerun in this workstream.

## Required decision before QAT-01 can advance

Choose one bounded route:

1. Run the existing M2-L artifact through a fresh disposable LiteRT-LM phone/probe smoke test, recording the package/runtime identity and exact metrics; then
2. Independently verify an E4B LiteRT-LM package's QAT provenance before considering M4-L; or
3. Authorize a single Q2 GGUF download into a clean llama.cpp research worktree to prove the distinct fallback route; or
4. Stop after the M2-L result.

No candidate file was downloaded, converted, loaded, or evaluated. The official source pages are the metadata evidence: [M2](https://huggingface.co/google/gemma-4-E2B-it-qat-mobile-transformers), [M4](https://huggingface.co/google/gemma-4-E4B-it-qat-mobile-transformers), [Q2](https://huggingface.co/google/gemma-4-E2B-it-qat-q4_0-gguf), and [Q4](https://huggingface.co/google/gemma-4-E4B-it-qat-q4_0-gguf).
