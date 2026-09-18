# QAT-00 workspace map

Inventory time: 2026-09-18. This record is evidence, not a claim that any QAT candidate runs.

| Area | Verified path / revision | State and intended use |
| --- | --- | --- |
| Foundations evidence | `Aquinas-Foundations` at `d07f4eb` | Documentation-only. The evaluation plan and rotated-ternary plan are untracked user files; QAT evidence lives under `research/gemma-4-qat/`. |
| Backend source/reference | `Aquinas_Backend` at `460d6ff` | Dirty; contains the current MLX/adapter implementation, reusable evaluators, exporter scripts, and local models. Do not use this checkout for QAT edits. |
| Canonical iOS source | `Aquinas-iOS-main` at `3d8d33e` | Dirty; owns the active `LiteRTModelStore`, process-scoped runtime, and device probe. |
| llama.cpp reference | `Aquinas-iOS-llama-cpp-12b` at `fe2529f` | Dirty; reference only, not the active iOS baseline. |
| Backend B1 source | `Aquinas_Backend/models/Aquinas-Final` | 8.7 GiB fused Hugging Face checkpoint. Its config declares Gemma 4 text, 35 layers, 131,072 maximum positions, and bf16 source dtype. It has not been measured in this workstream. |
| Adapter provenance | `Aquinas_Backend/models/aquinas_adapters` | 195 MiB; records LoRA for `google/gemma-4-E2B-it`, rank 8, scale 20, 600 iterations, seed 0, 2,048 max sequence length. |
| B0 package | `Aquinas-iOS-main/Aquinas-iOS/LocalModels/gemma-4-E2B-it.litertlm` | Present; exact manifest-compatible 3,862,121,696-byte package. See `baseline-identity.json`. |
| Other local LiteRT artifacts | `Aquinas_Backend/models/LiteRT-Stock/gemma-4-E2B-it.litertlm`; `Aquinas_Backend/models/Aquinas-Final-LiteRT/model.litertlm` | 2,588,147,712-byte stock diagnostic and 2,556,106,704-byte older Aquinas package, respectively. Both differ from B0 and cannot be substituted as B0. |
| Runtime wrapper | `Aquinas-iOS-main/Vendor/LiteRTLM` | Local Swift package; its `Package.swift` names LiteRT-LM and pins the macOS binary dependency to release `v0.14.0`. iOS framework provenance/capabilities require QAT-01 verification. |

## Baseline reconciliation

The active app source is authoritative for B0. `LiteRTModelStore.aquinas` requires `gemma-4-E2B-it.litertlm` at `3,862,121,696` bytes and the B0 SHA-256. The exact bundled development file exists and hashes correctly. This is the `dynamic_wi8_emb4_afp32` candidate described in the current model-integration record; it was previously reported to fail its simulator GPU compatibility gate. Therefore **B0 is identifiable but not a current phone-fit pass**.

Older documentation describes a 2,722,385,120-byte 4-bit package, while the available corresponding backend package is 2,556,106,704 bytes with a different hash. Neither matches B0. QAT-02 must use the B0 manifest identity, and any B0/B1 parity work must label differences in template/export/runtime settings rather than infer equivalence.

## Active-work and resource inventory

- The backend has modifications to evaluation, retrieval, export, server, structured-generation, tests, `runs/`, and DWQ scripts. The iOS checkout has modifications to conversation, Insight Tree, and `LiteRTAquinasModel`; the llama.cpp reference has modifications to project/runtime files. They are preserved as unrelated work.
- `df` reported 104.18 GiB free on the data volume. This exceeds the plan's 25 GiB reserve but is not a complete large-job budget: source, staging, cache, export, and checkpoint requirements remain unaccounted.
- A sandboxed `sysctl` memory query and `ps` process query were denied. Refresh these read-only measurements in an authorized local session before a large operation.
- `xcrun simctl` and `xcrun devicectl` both failed because CoreSimulatorService/CoreDeviceService were unavailable. No simulator or physical device was verified. No device container was touched.

## Isolation decision

This Foundations repository is the only modified location for QAT-00 records. A new, clean descendant worktree of `Aquinas_Backend` is required for any QAT evaluator/export implementation. No model artifacts, caches, private datasets, or run output will be committed.
