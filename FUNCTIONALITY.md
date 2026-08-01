# Aquinas Functional Blueprint

> Current implementation reference for user-visible behavior. Product intent comes from
> `MISSION.md`; technical contracts live in `MODEL-INTEGRATION.md`; detailed tree behavior lives in
> `INSIGHT-TREE.md`.

## 1. Product mandate

Aquinas is a sovereign cognitive interface: a private, local-first place for deep, uninterrupted
philosophical and theological inquiry. Its pillars are sovereignty, permanence, and rigorous
inquiry. The UI should communicate the structure of thought, not merely display model output.

## 2. Main application areas

- **Home** — Question of the Day, recent work, saved Insights, and entry points into new questions.
- **Conversation** — the active branching dialogue, uploads, contextual definitions, model
  activity, and entry into the conversation's Insight Tree.
- **Open Conversations** — search, selection, rename, pin/unpin, study-topic assignment, and
  deletion.
- **Study Topics** — topic containers and conversations assigned to them.
- **Insight Library** — globally bookmarked contextual definitions and its in-memory semantic
  canvas. Bookmark changes are applied to the Global Insight Tree only after the user accepts the
  update prompt shown above its model controls.
- **Settings** — appearance, user name, response typography/alignment, and related preferences.

## 3. Conversation and branching

The default view is a vertically scrolling dialogue. Each conversation can contain horizontally
arranged branches. A branch can begin from a quoted concept or duplicated model response without
destroying its parent line of inquiry.

The model returns plain response text plus validated key-term metadata. Client code turns those
terms into underlined `aq://` links. Tapping one opens a contextual definition flow:

1. Check the active conversation's cache for the normalized term and source response.
2. If defined already, open it immediately even when another model task is active.
3. If absent, enqueue a `Define “…”` model task.
4. The completed definition can be quoted, forked, or saved.
5. Saving adds it to the global Insight Library and the current conversation's persistent tree.

Definitions contain a title, source-context label, and concise contextual meaning—not
pronunciation, part of speech, or an example. Saved Insights deduplicate by normalized title. If
the same term is saved again from a materially different context, the new meaning is appended to
the existing card as a secondary context-definition entry.

The conversation model also recognizes broad direct-definition wording such as “Define X,” “What
does X mean?”, and “What is X?” when X is a concise term rather than a broader inquiry or interface
control. Those answers include an in-text Insight card for the requested definition.

Queued questions display `Question queued` with the shared breathing animation. Queued definition
affordances breathe as well. Cancelling their model task clears the corresponding pending state.

## 4. Model behavior and task queue

The old Thinking toggle has been removed. Conversation responses use automatic routing: routine
questions take the direct-JSON fast path, while complex questions retain deep reasoning. Both may
provide a short, user-facing summary of the approach taken. This is not raw chain-of-thought.
The summary contains only the few inquiry-specific approach notes the question warrants; it does
not expose scratch work, repeat the answer, or claim tools or evidence that were not used.

Ordinary conversation follows a permanent reasoning constitution: represent the strongest
reasonable version of the user's argument, distinguish invalid inference from disputed premises
or missing evidence, revise conclusions when the user's reasoning warrants it, propagate the
revision through dependent conclusions, and keep confidence proportional to the evidence.
Conversation personalities may change expression but not this standard. Structured application
actions remain neutral regardless of personality.

Current conversation model work is scheduled through one priority-aware `ModelTaskQueue`:

- `User Question`
- `Define “<term>”`
- `Update Insight Tree`
- `Consolidate information`

Questions and definitions are foreground work. Tree updates wait for a five-second idle window and
yield when foreground work arrives.

The dock's Model Status reads `Idle` when empty and `Thinking` while active. With multiple jobs it
shows progress such as `1/3 Thinking`; only `Thinking` shimmers and active copy is light green.
The status is available in Branch and conversation Insight Tree modes, except while an Insight is
hovered.

Tapping status opens the Model Tasks card. Completed tasks show a checkmark, the current task shows
a spinner and stop action, and upcoming tasks show a dotted circle and remove action. Upcoming
tasks can be reordered by drag with haptic feedback. Task/icon changes fade, and completed rows
clear shortly after the queue becomes idle. The empty popup remains 345px wide.

Structured generation receives one narrow repair attempt when required output is malformed. Repair
preserves valid substance and fixes only the contract surface; optional content is omitted rather
than invented. If the backend remains unavailable or output remains invalid, live actions fail
explicitly instead of substituting preview/mock knowledge. Definitions and canvas actions offer
retry; failed Midpoint and Make Node work rolls back temporary Tree state.

## 5. Question of the Day

Home shows one open-ended Question of the Day grounded in an unresolved idea, distinction,
assumption, tension, consequence, or application from prior conversation. Generation consults
recent conversation context and at most four relevant saved Insights. It asks one thing, supports
reflection rather than recall, avoids yes/no or leading framing, and remains neutral.

Answering the question opens its conversation and removes the card from Home. Once the current
question has been answered or has expired into the next day, an eligible non-conversation page
schedules a new `Consolidate information` task after the model has been idle for 15 seconds. The
active status reads `Consolidating...`. Opening an active conversation does not cancel an already
queued consolidation task. Failed generation does not persist a placeholder question and remains
retryable.

## 6. Context controls and slash commands

The context control is a non-spinning usage gauge without a visible `Context` label. Tapping it
opens the context card, which can clear the active conversation.

Only two slash commands are supported:

- `/compact` asks the backend to combine the previous compacted checkpoint and turns since that
  checkpoint into a new hidden model context. The visible transcript remains unchanged.
- `/clear` cancels applicable model work and resets the active conversation in place, retaining
  the conversation identity/study-topic relationship.

Compaction state is persisted per branch as `compactedContext` plus
`compactedThroughBlockCount`. Future requests send that checkpoint and only later transcript
blocks.

## 7. Insight Tree

Canvas Mode is the current conversation's zoomable semantic map:

- **Insights** are manually saved contextual definitions.
- **Node Concepts** are broader subjects that organize Insights or pivotal ideas extracted from a
  response.

After a completed answer is persisted, iOS queues an idempotent background analysis keyed by a
stable response ID. The backend may add zero or one pivotal Node Concept. It never creates an
automatic Insight. Reopening retries durable pending jobs but does not backfill old transcript
history.

Insight-to-Node and Node-to-Node line lengths communicate semantic distance. Conversation
Node-to-Node connections have a 360px minimum so broader subjects remain visually separated.
The graph uses sparse persisted topology rather than all-pairs connections.

Users can inspect and hover cards, quote concepts into conversation, select multiple targets,
create Midpoints, and promote an Insight with Make Node.

Midpoint accepts two through eight selected Insights or Node Concepts. The selection’s weights are
derived from handle geometry, normalized by application code, and used to calculate a weighted
embedding centroid. Aquinas supplies five substantive candidates; MiniLM selects the candidate
nearest that centroid. This is a mathematical vector-space operation, not a prompt asking for a
verbal “50/50 mixture.” Every candidate integrates the two dominant sources; larger selections use
an integrated weighted-center pass and require the remaining sources to contribute across the
pool. At eight selected items, add-to-selection remains hidden. The selection also retains a
visible cancel control.

Make Node promotes the selected Insight and generates exactly three distinct, non-overlapping
children adapted to that concept. New child bonds maximize angular separation from one another and
from existing parent/Node connections.

## 8. Current persistence boundaries

- iOS conversation/branch state is still stored as a whole JSON snapshot in `UserDefaults`.
- Global Insight Library content is client-side.
- Conversation-to-global-Insight membership and pending tree-analysis IDs use dedicated
  `UserDefaults` stores.
- The backend SQLite database persists conversation-scoped definitions and Insight Tree content,
  embeddings, ownership, sparse edges, analyses, provenance, and tombstones.

The SwiftData/file-backed production persistence migration remains planned in
`PERSISTENT_MEMORY_IMPLEMENTATION_PLAN.md`.
