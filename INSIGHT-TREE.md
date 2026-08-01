# Insight Tree — Design Spec

> Status: active spec for the response-driven, model-assisted Insight Tree.
> Conversation trees use persisted backend MiniLM topology; the global Insight Library canvas
> remains a client-side graph using `NLEmbedding`. Its accepted bookmark snapshot is persisted
> separately from the live Insight Library.
> See `MODEL-INTEGRATION.md` for the system-wide model contracts, backend boundaries, persistence,
> and implementation order. This file remains the source of truth for Insight Tree behavior.

## 1. Product model

The Insight Tree is the Canvas Mode view of a single conversation's conceptual structure. Two
levels of concept:

- **Insight** — a specific idea pulled from the conversation. Concrete; can be inspected, saved,
  quoted, or used to start new inquiry.
- **Node Concept (Node)** — a higher-level subject that groups related Insights or preserves a
  pivotal concept extracted from a response. Its label names the subject represented by the Node.
  Dynamically generated.

Edges encode relationship strength via **length**:

- **Insight → Node** line length ∝ relatedness to that Node. Shorter = more related. Floor **~48px**.
- **Node → Node** line length ∝ relatedness between the subjects. Floor **360px**.

Everything is **per-conversation**.

## 2. Lifecycle: response → automatic Node Concepts

After each newly completed assistant response is persisted, iOS adds an identifiers-only durable
analysis job and exposes it as `Update Insight Tree` in the shared Model Task queue. Reopening a
conversation retries only already queued jobs; it never backfills every historical response.
Aquinas returns the broad subject of the question plus zero or one pivotal conceptual seed. It
does not choose IDs, membership, similarity, or layout. Highlighted terms inform the seed check
but remain definition affordances, not automatic Insights. The backend validates exact answer
evidence, rejects non-durable candidates, then inserts an accepted seed directly as a Node
Concept:

- similarity **≥ 0.86** to an existing Insight or Node: skip as a duplicate;
- otherwise: create a Node Concept and connect it through the sparse Node graph.

The first substantive question-answer pair creates one subject Node as the conversation's seed,
even when Aquinas returns no more-specific Node Concept seed. A trivial first exchange leaves the
tree empty, so the first later substantive pair becomes the seed. Later answers normally add no
Node Concept; one is added only when the turn introduces a central definition, distinction,
principle, causal relationship, or conclusion that materially extends the inquiry. Examples,
applications, restatements, and merely useful facts stay out of the Tree unless the user bookmarks
them as Insights.

The response-analysis operation is idempotent by conversation and response ID. Its job and result
are persisted, so retries return the original mutation. iOS stores a durable identifiers-only queue
and reconstructs the question and answer from conversation persistence when retrying.

### Manual save remains separate

1. A model response renders certain key terms/phrases **highlighted + underlined**. Foundational
   or prerequisite concepts receive priority, while genuinely interesting discovery terms may
   also be included to encourage useful rabbit holes. There is no quota.
2. Tapping one first checks the active conversation/source cache. A cached definition opens
   immediately, even while the model is occupied. Only a cache miss queues the model to
   **generate a definition of that term as it relates to this conversation** (contextual, not a
   dictionary lookup).
3. Choosing to **save** bookmarks that definition in the global Insight Library **and** attaches
   it to the active conversation's Insight Tree. The client persists that conversation association
   by Insight ID so an offline backend write can be reconciled later. It is not the trigger for
   automatic response analysis.
   If that term is already saved with a different contextual meaning, saving adds the new
   context-definition entry to the existing Insight card rather than creating a second card.

### Global Insight Tree updates

Saving or removing a global bookmark updates the Insight Library immediately but does not
silently regroup the Global Insight Tree. When Global Insights opens with bookmark changes that
are not in its last accepted snapshot, a confirmation pill appears above the model controls.
Choosing **Yes** replaces the snapshot with the current deduplicated bookmark library and rebuilds
the semantic canvas; choosing **No** preserves the current tree. The prompt may appear again when
the page is reopened or the bookmark library changes.

Seam: `openDynamicDefinition` first calls `AquinasModel.cachedDefinition`; on a miss,
`requestDynamicDefinition(for:)` calls the conversation-scoped `defineTerm` overload.
`BackendAquinasModel` is the live environment default and the definition path has no mock
availability fallback. The response's id is re-stamped via
`ConceptDefinition.stableID(forTerm:)` rather than kept as the model's own id, so re-saving the
same term always dedups (`MODEL-INTEGRATION.md`: "Persistent IDs generated by application code,
never by either model").

Backend/client status: the uncached `POST /concept/define` route and the conversation-scoped
`/concept/lookup` + `/concept/define` routes are implemented. The conversation flow consumes
filtered `/conversation/respond/stream` output, whose validated key-term metadata creates the
highlighted, tappable terms that open this definition flow.

## 3. Relatedness (the core signal)

Every Insight has a relatedness to its Node, and every Node pair has a relatedness. Both come from
a single **relatedness provider**:

- **Decided provider:** `sentence-transformers/all-MiniLM-L6-v2` supplies normalized embeddings;
  application code calculates cosine similarity. Node embeddings are normalized centroids of their
  member Insight embeddings.
- **Current wiring:** conversation-scoped trees use backend MiniLM assignment and persisted
  Insight→Node distances through `InsightTreeService`. `NLEmbeddingProvider` clusters the global
  Insight Library canvas and remains the local provider for previews and client-only features.
  These vector spaces are kept separate; persisted MiniLM topology is never recomputed with
  `NLEmbedding`.

The fine-tuned Aquinas language model generates contextual definitions, Node labels, blended
Insights, and Make Node children. It does **not** invent relatedness numbers.

Relatedness is a **soft target**, never an exact measurement — a set of pairwise distances almost
never embeds exactly in 2D, so line length reads as "roughly proportional." All thresholds/floors
live as **named constants** in one place.

## 4. Node membership & creation (lazy)

- If the tree has no existing Node, its first manually saved Insight creates one. A response-driven
  subject/seed Node may already exist before the first manual Insight.
- Each **new** Insight joins its most-related existing Node if relatedness is above threshold;
  otherwise it starts its own Node.
- An Insight has exactly one **owning** Node (it may still have secondary "bridge" lines — see §8).

## 5. Layout rules

- Insight→Node and Node→Node lengths follow §1's relatedness-with-floor mapping. Reuse
  `insightBondLength` / `mapDistanceToLength` (already min-floored) in `InsightTreeViewModel.swift`.
- **Anchor & nudge (stability):** existing nodes keep their positions as **anchors** and are only
  weakly pulled toward updated targets (the existing anchor spring, `anchorSpringK = 0.04` in
  `InsightTreeCanvasView.swift`). Only **new / budded** items are freshly placed. A full re-solve
  runs **only** on an explicit "reorganize" action or a topology change (bud / merge). Adding one
  thing never reshuffles the map.
- **Crowding:** the floor is a *minimum*, not a target. The chip ring grows with count
  (`insightOrbitRadius`, `InsightTreeModels.swift`). Every visible Insight label participates in
  one global footprint collision pass, including Insights owned by different Node Concepts;
  contact rotates both free Insight bonds and separates their parent nodes. Within a Node Concept,
  Insight bonds repel one another more strongly than visible node-to-node edges, and new bonds
  seed into the largest open angular gap. When a Node is dense, the shorter=closer ordering is
  locally approximate — accepted.
- **Exact title match:** when an Insight's normalized title (ignoring case, punctuation, and a
  leading article) matches its owning Node label, the canvas draws only the Node. The Insight
  remains stored and available in the Node card for its contextual definition; only the redundant
  orbiting chip and connector are collapsed. Placed Midpoints are exempt because their Insight
  chip is their only visible representation.
- **Node graph is sparse:** connect Nodes with a **minimum spanning tree** over relatedness, plus
  any extra pairs above a *strong* threshold. Never an all-pairs hairball.
- **Nearest-neighbor fallback:** anything with no above-threshold neighbor still attaches to its
  single nearest neighbor, so the graph stays one connected structure. (This generalizes the
  current placeholder chain — `chainExtensionPosition` / `maximizedGapAngle` become the
  nearest-neighbor placement primitive.)
- **Hover-card replacement:** switching directly between hovered Insights animates the current
  card out and the new card in; do not mutate only the text of one persistent card.

## 6. Automatic budding

When a Node accumulates **more than 3** Insights whose relatedness to it is **below threshold**
(the loose ones), they can split into a new Node:

- **Cohesion gate:** only bud a subgroup of those loose Insights that is *also* **mutually
  related** (3+ tight with each other). "Far from the parent" ≠ "belongs together" — with no
  cohesive subgroup, **no bud**.
- **Only the loose, cohesive Insights move.** Well-related Insights stay on the original Node.
- The new Node's **label is generated from the shared subject** of the moved Insights (model).
- The new Node **stays linked to its origin Node** (parent line) plus any other related Nodes — the
  tree stays connected.
- **Sticky + hysteresis:** budding does **not** auto-reverse. Re-evaluate membership only for
  **newly-added** Insights, not settled ones. Reversal/merge happens only by explicit user action.
  This prevents Insights ping-ponging between Nodes on successive rebuilds.

## 7. Make Node (manual budding)

Promotes one Insight in a Node into its **own** Node Concept; that Insight becomes the Node and
**3 child Insights** are generated under it. Already implemented (`appendPromotedNodes` +
`chainExtensionPosition` placement, the docked-card link-growth animation, `makeNodeChildIDs`).
Children are distinct, non-overlapping dimensions adapted to the promoted concept. Their bonds
maximize angular separation from one another and from visible lines connecting the parent Node
Concept to other Nodes.

## 8. Midpoint tool

Select **2–8** targets (Insights and/or Nodes) → a draggable **handle** appears at the selection's
center → its position sets **blend weights** per target (linear along the segment for 2; normalized
inverse-distance for 3+) → committing spawns a new **blended Insight**.

The model generates five shared-region candidates rather than verbally estimating a blend.
Every candidate integrates the two dominant sources; with five or more inputs, an integrated
weighted-center pass keeps lower-weight contributions coherent across the pool. MiniLM then selects
the candidate mathematically nearest the weighted source-vector centroid.

At eight selected targets, add-to-current-selection affordances remain hidden. The current
selection retains a visible cancel control.

- **Relax into the layout:** the drop position is only the initial seed. Afterward the blended
  Insight **unpins**, is **owned by its most-related source**, and its line length then follows
  relatedness like every other line — so all lines mean the same thing. (Changes today's behavior,
  where the midpoint stays pinned.)
- **Cross-Node blends:** owned by the most-related source; the other connectors are secondary
  **bridge** lines. A **Node + Node** blend seeds a new **bridge Node** rather than an Insight.
- `InsightTreeView` places an identity-only loading item, requests real candidates, chooses the
  nearest candidate in vector space, and replaces that item without changing its identity. On
  generation failure it removes the loading item and leaves the Tree unchanged.

## 9. Deletion & edge cases

- Remove an Insight → repair its Node centroid (sticky; no auto-un-bud). Legacy
  `automatic_response` Insight rows still use source-response tombstones so old/reconciled data
  cannot be recreated; the current response-analysis flow creates Node Concepts instead.
- Remove a Midpoint's source → drop that one connector, keep the Insight.
- Remove a Node → reparent its Insights to the nearest Node.
- Nothing related to anything (early, or weak stub relatedness) → the nearest-neighbor fallback (§5)
  keeps the map connected and anchored.

## 10. Model seams

Every generative call funnels through the `AquinasModel` protocol (`Aquinas-iOS/Services/`);
swapping in the real model is one conforming type plus overriding
`EnvironmentValues.aquinasModel` — see `MODEL-INTEGRATION.md` §3/§6 for the backend JSON/route
contract each method mirrors, and `AquinasModel.swift`'s doc comments for the current wiring
detail per method.

| Seam | `AquinasModel` method | Called from | Wired into the live flow? |
| --- | --- | --- | --- |
| Conversation response + key terms | `respond(to:)` | `ConversationComponents.appendSimulatedResponse` | Yes |
| Conversation compaction | `compact(_:)` | `CurrentConversation.compactContext` | Yes |
| Cached contextual-definition lookup | `cachedDefinition(for:in:conversationID:)` | `CurrentConversation.openDynamicDefinition` | Yes |
| Term → contextual definition | `defineTerm(_:in:conversationID:)` | `CurrentConversation.requestDynamicDefinition` | Yes |
| Node subject / label | `labelSubject(forTitles:)` | `InsightTreeViewModel.generateSuggestedNode` | Yes |
| Midpoint blend | `blendConceptCandidates(_:weights:)` | `InsightTreeView.requestMidpointDefinition` | Yes — generated candidates are selected by vector distance and replace an identity-only loading item |
| Make Node children | `generateChildren(for:)` | `InsightTreeViewModel.generateMakeNodeChildren` | Yes — exactly three validated children replace identity-only reveal state |

Relatedness has two explicit client boundaries in the `Services/` folder:
`InsightTreeService` consumes backend MiniLM topology for conversation trees, while
`EmbeddingProvider`/`NLEmbeddingProvider` supports the global in-memory canvas and client-only
features. Persisted MiniLM distances are rendered directly rather than recomputed on device.

Persistent IDs (Insight/Node/term ids) are always assigned by application code, never returned by
the model — see `stableUUID(from:)` (`InsightTreeViewModel.swift`) and
`ConceptDefinition.stableID(forTerm:)` (`ContentView.swift`), which re-stamps a saved term's id
from its canonical text so re-saving the same term always dedups instead of minting a duplicate.

## 11. Code map

- `Aquinas-iOS/Services/AquinasModel.swift` — the model boundary protocol, `ConversationContext`,
  `KeyTerm`/`ModelResponse` (see §10), and the `EnvironmentValues` injection points.
- `Aquinas-iOS/Services/BackendAquinasModel.swift` — live conversation and definition client.
- `Aquinas-iOS/Services/MockAquinasModel.swift` — previews and tests only.
- `Aquinas-iOS/Services/EmbeddingProvider.swift` — the relatedness-vector boundary protocol +
  `NLEmbeddingProvider`.
- `Aquinas-iOS/Services/InsightTreeService.swift` — persistent conversation-tree
  analyze/save/load/remove boundary and backend snapshot decoding.
- `Aquinas-iOS/Features/Conversation/ModelTaskQueue.swift` — serialized question, definition, and
  tree-update jobs, including stop/remove/reorder state.
- `Aquinas-iOS/Persistence/ConversationInsightMembershipStore.swift` — client-side association
  between global saved Insights and a conversation.
- `Aquinas-iOS/Persistence/InsightTreeAnalysisQueue.swift` — durable identifiers-only retry queue.
- `InsightTreeViewModel.swift` — membership, layout targets, budding, promotion, midpoint placement,
  `ensureEmbeddings`/`refreshEmbeddingsIfNeeded` (the versioned embedding pipeline).
- `InsightTreeCanvasView.swift` — rendering + physics (anchor spring, VSEPR chip-angle spread,
  simulated-annealing cooling), gestures, Make Node / Midpoint interaction.
- `InsightTreeView.swift` — SwiftUI shell, docked Insight / Node cards, orchestration.
- `SemanticLayout.swift` — MDS + Procrustes engine, retained for future "advanced folding."
- `InsightTreeModels.swift` — `InsightModel` / `NodeModel` / `EdgeModel`, `insightOrbitRadius`.
- Reusable primitives: `insightBondLength`, `mapDistanceToLength`, `semanticDistance`,
  `chainExtensionPosition` / `maximizedGapAngle`, `stableUUID(from:)`.

## 12. Tunable constants (one home)

`insightNodeMinLength ≈ 48` · `nodeNodeMinLength = 360` ·
`insightDuplicateThreshold = 0.86` · `membershipThreshold = 0.40` ·
`budTriggerCount = 3` (fires at 4+) · `budCohesionThreshold = 0.55` ·
`nodeEdgeStrongThreshold` ·
`anchorSpringK = 0.04`. All placeholders until re-tuned against the real model's relatedness.
