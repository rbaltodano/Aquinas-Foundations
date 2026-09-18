# LiteRT-LM QAT support clarification

Checked September 18, 2026. This is documentary evidence, not a local runtime or phone test.

## Finding

The statement that LiteRT-LM cannot run Gemma 4 QAT is too broad. In the official E2B repository discussion, LiteRT Community organization member `marissaw` states that the published `.litertlm` models already use the QAT discussed in Google's announcement, and identifies `gemma-4-E2B-it.litertlm` as mixed int2/int4/int8. [Maintainer clarification](https://huggingface.co/litert-community/gemma-4-E2B-it-litert-lm/discussions/30)

This corrects the categorical description of the stock E2B LiteRT artifact as “non-QAT” in `compatibility.md`. Its filename lacking `qat` is not evidence of its training provenance. Verify the exact local artifact's revision/hash against the published package before assigning the same provenance to a cached file.

The [E2B model card](https://huggingface.co/litert-community/gemma-4-E2B-it-litert-lm) and [E4B model card](https://huggingface.co/litert-community/gemma-4-E4B-it-litert-lm) publish iOS CPU/GPU measurements. These establish published deployment evidence, not fit on our base phone or support by our particular vendored framework. The explicit QAT provenance statement retrieved here is for E2B; verify E4B's exact artifact provenance independently rather than extrapolating that statement.

## Separate questions

| Question | Evidence / status |
| --- | --- |
| Can LiteRT-LM execute a published Gemma 4 QAT package? | Yes, supported by the E2B maintainer clarification and model card. |
| Can our installed exporter import the public mobile-QAT Transformers checkpoint? | The local compatibility report identifies a missing import/preservation path. This is a separate exporter question. |
| Can we fine-tune that source and reproduce Google's mobile package faithfully? | Not established; still requires the planned source/export pilot. |
| Does the pinned package work in our vendored iOS runtime on the base phone? | Not yet measured in this workstream; requires artifact/runtime pinning and a disposable probe. |

The public [litert-torch issue #1044](https://github.com/google-ai-edge/litert-torch/issues/1044) reports failure during source checkpoint import before runtime loading. Its suggested unsupported-message wording comes from the issue author, not a runtime capability verdict from a maintainer. The open [LiteRT-LM issue #2497](https://github.com/google-ai-edge/LiteRT-LM/issues/2497) asks for package/conversion clarification; an open question is not proof that published packages are non-QAT.

## Next implementation action

Register the prebuilt E2B package as a distinct candidate, such as `M2-L`, with exact repository revision, filename, SHA-256, runtime requirements and the provenance citation. Check whether the verified cached stock artifact matches before downloading another copy. Resolve the pinned framework's compatibility and run the planned disposable smoke test within the declared resource budget. Do not require a successful Transformers-to-LiteRT export to evaluate an already compiled package.

Keep M2/M4 source-export blockers scoped to those routes. Do not mark any local smoke/phone gate passed based on this note. Reconcile the candidate registry, compatibility report and handoff status when the implementing agent resumes. Record exact support evidence for each artifact/runtime pair; do not guess a minimum runtime version from a different model's release notes.
