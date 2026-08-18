# Persistent Memory Implementation Plan

> Status: implemented, with noted deviations from this plan's original shape and gaps called out
> below. iOS is now SwiftData-backed (`Aquinas-iOS/Persistence/SwiftData/`) with file-backed
> attachments and a one-time migration from the legacy `UserDefaults` snapshot. This document's
> designed shape (an `InquiryRepository` type, `.modelContainer(for:)` in `Aquinas_iOSApp.swift`,
> a dedicated Insights popup UI) does not match 1:1 what shipped — see "Implementation status"
> below for what changed and why, before assuming any specific type/file name in the rest of this
> document exists verbatim.

## Implementation status

**Implemented:** `PersistedConversation`/`PersistedBranch`/`PersistedChatBlock`/`PersistedAttachment`/
`PersistedInsightLibraryEntry` SwiftData models (`Persistence/SwiftData/PersistedModels.swift`);
a `ModelContainer` singleton and all read/write/sync/mapping logic
(`Persistence/SwiftData/AquinasPersistence.swift`); file-backed attachment storage under
Application Support (`Persistence/SwiftData/AttachmentFileStore.swift`); a one-time migration from
the legacy `UserDefaults` snapshot, run from `Aquinas_iOSApp.init()`; a last-active-conversation
`UserDefaults` preference; a SwiftData-backed global Insight Library
(`PersistedInsightLibraryEntry`); automated round-trip/deletion/sibling-independence tests
(`Aquinas-iOSTests/ConversationPersistenceTests.swift`).

**Deliberately built differently than this plan describes:** there is no separate
`InquiryRepository` type or `.modelContainer(for:)` environment injection in
`Aquinas_iOSApp.swift`. Nearly every call site in `ContentView.swift`/`CurrentConversation.swift`
already funneled through exactly two static stores — `CurrentConversationsStore` (the real,
live-read conversation store; **not** `InquiryPersistenceStore`, which this plan originally
assumed was the live store but turned out to be dead code writing to a separate, never-read
`UserDefaults` key) and `InsightLibraryStore`. Both kept their original `load()`/`save(_:)`
signatures and now delegate internally to `AquinasPersistence`, so the large majority of call
sites in those two files needed zero edits — only `ContentView.swift`'s `renameConversation`
changed, to fix it using the dead store. This was a deliberate choice to minimize edit surface in
a ~4,500-line pair of files this pass could not compile-check (see "Verification" below), not an
oversight of steps 3/6/8's literal wording.

Per-conversation saves sync each branch/chat-block/attachment row by id against what's already
persisted rather than deleting and recreating the conversation's whole row tree, so SwiftData's
own change tracking skips writing any row whose content didn't change — this is what step 8's
"replace whole-snapshot saves with targeted writes" resolves to under the preserved-signature
design above, rather than literal per-call-site rewrites.

**Not implemented / deferred:**
- The dedicated cross-conversation Insights popup UI ("Insights Popup Integration" section below)
  is a new UI feature, not a persistence change, and is separate follow-up work.
- `ConversationInsightMembershipStore`, `InsightTreeAnalysisQueue`, and `InsightDiscoveryStore`
  remain `UserDefaults`-backed by design, not migrated to SwiftData tables. They store only ID
  strings and retry counters, never transcript text or image bytes — they were never the
  whole-snapshot-blob growth problem this plan exists to fix, so moving them was deprioritized
  rather than silently skipped. Branch-level compaction checkpoints
  (`compactedContext`/`compactedThroughBlockCount`) *are* migrated, since those are plain fields
  on `ChatBranch` that flow through the SwiftData branch row like every other branch field.
- True field-level diffing within one changed branch is not implemented: a branch with any change
  re-syncs its whole row tree (still per-row change-tracked, so unchanged individual chat
  blocks/attachments within it still cost nothing extra) rather than diffing at, say, the single
  edited field.
- Deletion coordination with the backend Insight Tree store (step 14) remains open.

**Verification:** this implementation was written and reviewed but not compiled or run — the
session that wrote it had no macOS/Xcode toolchain available. Build against a concrete arm64
simulator destination and run `Aquinas-iOSTests` (including the new
`ConversationPersistenceTests` suite) before relying on it; also run the manual scenarios in "Test
Plan" below on a real device/simulator, since a passing build and test suite verify compilation
and the tested behaviors, not full end-to-end product correctness (e.g. Make Node / Midpoint /
canvas-physics interaction with the new store hasn't been exercised in a running app at all).

## Summary

Replace the current prototype persistence system with a production-ready local storage layer. The current implementation JSON-encodes the entire conversation canvas into `UserDefaults`, including uploaded image data. That is fine for early prototyping, but it will get slower and heavier as conversations, branches, attachments, and canvases grow.

The new system should use structured local persistence for conversation data and file-system storage for attachments. The goal is for Aquinas to feel durable, fast, and native: users can quit and reopen the app without losing work, switch between conversations instantly, attach media without bloating app state, and eventually support search, pinning, deletion, export, and sync.

## Current persistence inventory

This section describes the pre-migration baseline this plan replaced — see "Implementation
status" above for what actually exists now.

The production migration had to account for all storage that existed:

- `CurrentConversationsStore` stored conversations, the active conversation ID, branches, chat
  blocks, uploads (including image bytes), promoted Insight IDs, and per-branch hidden compaction
  state (`compactedContext` and `compactedThroughBlockCount`) in one `UserDefaults` snapshot. (An
  earlier version of this document named `InquiryPersistenceStore` here — that store used a
  separate, never-read `UserDefaults` key and was already dead code, not the live store.)
- `InsightLibraryStore` owns globally bookmarked contextual definitions on the client.
- `ConversationInsightMembershipStore` stores only the stable IDs that associate global saved
  Insights with a conversation.
- `InsightTreeAnalysisQueue` stores stable response/conversation/branch identifiers and retry
  counts, not duplicate transcript text.
- `InsightDiscoveryStore` tracks which conversation-tree additions remain undiscovered.
- The backend SQLite database (`Aquinas_Backend/data/insight_tree.sqlite3`) separately stores
  conversation-scoped definition cache records and persistent Insight Tree content: Insights,
  Node Concepts, embeddings/model version, membership, sparse edges, response-analysis results,
  provenance, and tombstones.

SwiftData migration must preserve stable IDs shared with the backend. Moving iOS records must not
silently mint new conversation, branch, response, Insight, or Node identifiers.

## Key Changes

### Storage Architecture

Use **SwiftData** as the main persistence layer.

Persist these records as SwiftData models:

- `PersistedConversation`
  - `id: UUID`
  - `title: String`
  - `createdAt: Date`
  - `updatedAt: Date`
  - `isPinned: Bool`
  - ordered relationship to branches

- `PersistedBranch`
  - `id: UUID`
  - `conversationID: UUID`
  - `parentBranchID: UUID?`
  - `startingConceptID: UUID?`
  - `duplicatedResponse: String?`
  - `yOffset: Double`
  - `generatedBranchTitle: String?`
  - `compactedContext: String?`
  - `compactedThroughBlockCount: Int?`
  - top/bottom question state
  - ordered relationship to chat blocks

- `PersistedChatBlock`
  - `id: UUID`
  - `branchID: UUID`
  - `sortIndex: Int`
  - `kind: user | model`
  - `text: String`
  - `conceptID: UUID?`
  - ordered relationship to attachments

- `PersistedConcept`
  - `id: UUID`
  - `sourceConversationID: UUID?`
  - `word: String`
  - `partOfSpeech: String`
  - `pronunciation: String`
  - `meaning: String`
  - `example: String`
  - `isSavedToLibrary: Bool`
  - `createdAt: Date`
  - `updatedAt: Date`
  - stable canonical-term identity compatible with `ConceptDefinition.stableID(forTerm:)`

- `PersistedInsightLibraryEntry`
  - `id: UUID`
  - `conceptID: UUID`
  - `sourceConversationID: UUID?`
  - `sourceBranchID: UUID?`
  - `createdAt: Date`
  - used to power the cross-conversation Insights popup

- `PersistedAttachment`
  - `id: UUID`
  - `chatBlockID: UUID?`
  - `pendingBranchID: UUID?`
  - `name: String`
  - `localRelativePath: String`
  - `contentType: String`
  - `rotationDegrees: Double`
  - `createdAt: Date`

Store uploaded file/image bytes separately in Application Support, not inside SwiftData:

```text
Application Support/
  Aquinas/
    Attachments/
      {attachmentID}.jpg
      {attachmentID}.png
      {attachmentID}.pdf
```

SwiftData stores only the metadata and relative file path.

Saved insights should be first-class persisted records, not embedded inside any single conversation. Conversations can reference insights through concepts, but the user's saved insight library must remain available across conversations.

### App State Shape

Keep the existing lightweight UI structs:

- `InquiryConversation`
- `ChatBranch`
- `ChatBlock`
- `UploadedFile`
- `ConceptDefinition`

Add mapper helpers between UI state and SwiftData models:

- SwiftData to UI state when opening a conversation
- UI state to SwiftData when saving a conversation
- Attachment file path to display image thumbnail
- Pending uploaded file to saved attachment file after submit
- Saved insight records to the Insights popup's current-conversation and all-conversations tabs

Do not bind SwiftUI directly to SwiftData models inside the canvas. The canvas is interactive and animation-heavy, so it should continue using local `@State` structs while active. SwiftData should be the durable backing store, not the live rendering model.

### Save Behavior

Replace whole-snapshot `UserDefaults` saves with targeted writes.

Save immediately after durable events:

- creating a new conversation
- switching conversations
- submitting a user question
- receiving a completed model response
- saving, unsaving, quoting, or forking an insight
- creating or deleting a branch
- renaming, pinning, or deleting a conversation
- adding or removing attachments before submit
- app moving to background

Avoid saving on every keystroke unless autosave is later desired. Pending typed text can remain transient for now.

Add a small debounced save helper for non-critical UI updates:

```swift
final class InquiryAutosaveCoordinator {
    func scheduleSave(reason: InquirySaveReason)
    func flushImmediately()
}
```

Use immediate saves for submitted content and debounced saves for layout-only changes like branch `yOffset`.

### Attachment Flow

When a user selects an attachment:

1. Keep it in pending UI state as today.
2. Show thumbnail from in-memory data.
3. On submit, write the file/image data to Application Support.
4. Create `PersistedAttachment`.
5. Store attachment metadata on the submitted user `ChatBlock`.
6. Clear pending uploads.

When loading a conversation:

1. Read attachment metadata from SwiftData.
2. Resolve file paths from Application Support.
3. Load thumbnails lazily for visible attachment strips.
4. If a file is missing, show a safe placeholder and keep the conversation readable.

### Migration From Current Prototype Memory

Keep `InquiryPersistenceStore.load()` temporarily as a one-time migration source.

On first launch after the new system ships:

1. Check whether SwiftData already has conversations.
2. If SwiftData is empty, attempt to load the old `UserDefaults` snapshot.
3. Convert old conversations, branches, chat blocks, concepts, and attachments into SwiftData records.
4. Write image data from old `UploadedFile.imageData` into Application Support.
5. After successful migration, remove the old `UserDefaults` key.
6. If migration fails, leave the old data intact and log the failure.

Add a marker key:

```swift
aquinas.inquiry.persistence.migratedToSwiftData.v1
```

### Conversation Loading

At app launch:

1. Fetch conversations sorted by `isPinned DESC`, then `updatedAt DESC`.
2. Restore the last active conversation ID from a small `UserDefaults` preference.
3. If the last active conversation still exists, load it.
4. Otherwise load the first conversation.
5. If no conversations exist, create a blank `New Conversation`.

Keep only the active conversation's full branch canvas in memory. The side menu should fetch conversation summaries only.

### Side Menu Integration

The side menu should read from lightweight conversation summaries:

- ID
- title
- pinned state
- updated date

Selecting a conversation should:

1. Save or flush the current active conversation.
2. Load the selected conversation from SwiftData.
3. Convert it to UI structs.
4. Reset canvas mode and zoom state.
5. Close the menu.

Deleting a conversation should:

1. Delete related branches, chat blocks, concepts if no longer referenced, and attachment metadata.
2. Delete attachment files from disk.
3. If the deleted conversation was active, load the next available conversation or create a blank one.

### Insights Popup Integration

The `+` menu should expose an `Insights` action that opens a dedicated saved-insight popup.

The popup should read from the persisted insight library and show two scopes:

- `This Conversation`: saved insights that were created in, quoted into, forked from, or referenced by the active conversation.
- `All`: every saved insight across conversations.

The popup should support:

- horizontal swipe between insight cards
- left/right arrow navigation
- page count display, for example `1/4`
- quote insight into the current thread
- fork insight into a new branch
- save/unsave insight from the global library

When a user saves an insight from any conversation, it should become available in the `All` tab immediately and persist across app launches.

## Implementation Steps

1. ~~Add SwiftData model classes for conversations, branches, chat blocks, concepts, and
   attachments.~~ Implemented, with embedded `ConceptDefinition`/`ResponsePresentationMetadata`
   content stored as an encoded JSON blob per row rather than exploded into further relationships
   — see "Implementation status" above.
2. ~~Add a model container.~~ Implemented as `AquinasPersistence.container`, a dedicated
   singleton rather than `.modelContainer(for:)` in `Aquinas_iOSApp.swift` — every read/write call
   site is a static enum function outside SwiftUI's environment, not a `View`.
3. Not implemented as its own `InquiryRepository` type. `AquinasPersistence` is the sync/mapping
   layer; `CurrentConversationsStore`/`InsightLibraryStore` keep their original names and
   signatures as the call-site-facing boundary instead — see "Implementation status" above.
4. ~~Create an attachment store for writing, reading, and deleting files in Application
   Support.~~ Implemented as `AttachmentFileStore`.
5. ~~Add mappers between current UI structs and persisted models.~~ Implemented, inside
   `AquinasPersistence.swift` rather than a separate mapper file.
6. Not applicable as written — there is no `ActiveInquiryView`; `CurrentConversationsStore.save/load`
   usage across `ContentView.swift`/`CurrentConversation.swift` needed no call-site changes, since
   that store's signature didn't change (see "Implementation status" above). The one broken call
   site (`ContentView.swift`'s `renameConversation`, which wrote to the dead
   `InquiryPersistenceStore`) is fixed to use `CurrentConversationsStore`.
7. `InquiryPersistenceStore` is marked `@available(*, deprecated)` and left otherwise unused —
   it turned out to already be dead code (a separate, never-read `UserDefaults` key), not the
   live store this plan originally assumed, so it was never a migration source.
8. ~~Update conversation switching, new chat, delete, rename, pin, branch creation, and submit
   flows.~~ Implemented — via the preserved-signature store, not individual call-site rewrites;
   see "Implementation status" above for what "targeted writes" resolves to under that design.
9. ~~Add a last-active-conversation preference.~~ Implemented
   (`aquinas.persistence.active-conversation-id` in `AquinasPersistence`).
10. ~~Add migration from the current `UserDefaults` JSON snapshot.~~ Implemented
    (`AquinasPersistence.runMigrationIfNeeded()`, called from `Aquinas_iOSApp.init()`).
11. ~~Add reads/writes for the saved insight library.~~ Implemented
    (`PersistedInsightLibraryEntry`). Active-conversation insight *filtering* is unchanged —
    `ConversationInsightMembershipStore` remains `UserDefaults`-backed by design; see
    "Implementation status" above.
12. Partially implemented: automated round-trip/sibling-independence/deletion/Insight-Library
    tests exist (`Aquinas-iOSTests/ConversationPersistenceTests.swift`). The manual scenarios
    below are not yet run — this pass had no macOS/Xcode toolchain to run them on.
13. Partially implemented: branch compaction checkpoints migrated (they're plain `ChatBranch`
    fields, now SwiftData columns). Insight membership, pending response-analysis jobs, and
    discovery state remain on their existing `UserDefaults` stores, deliberately not moved into
    SwiftData tables — see "Implementation status" above. No stable IDs changed anywhere in this
    work.
14. Not implemented — deletion coordination with the backend tree store remains open.

## Test Plan

### Automated Tests

- Create a conversation with one branch, save it, reload it, and verify title/text survives.
- Create multiple branches with parent relationships and verify branch IDs, parent IDs, and `yOffset` values survive reload.
- Submit user questions with concepts and verify chips are restored correctly.
- Save model responses and verify ordering is preserved.
- Save insights and verify they appear in the global insight library across conversations.
- Verify the Insights popup correctly separates active-conversation insights from all saved insights.
- Save attachments and verify metadata persists while file bytes are written to disk.
- Delete a conversation and verify related attachment files are removed.
- Migrate a legacy `UserDefaults` snapshot into SwiftData.
- Verify compacted context and its represented block count survive migration without altering the
  visible transcript.
- Verify pending Insight Tree analysis jobs still resolve their persisted conversation, branch,
  response index, and stable response ID.
- Verify conversation-scoped cached definitions and tree records remain addressable after iOS
  migration.
- Handle corrupt legacy JSON by ignoring it without crashing.

### Manual Scenarios

- Start a conversation, submit a question, force quit, reopen, and confirm the same canvas appears.
- Create two conversations, switch between them, force quit, reopen, and confirm the last active one returns.
- Add image uploads, submit, restart, and confirm thumbnails still display.
- Save several insights, open the `+` menu Insights popup, and confirm tabs, arrows, page count, and horizontal swipe work.
- Fork from an insight and from a response, restart, and confirm branches and connector positions remain correct.
- Rename, pin, and delete conversations from the side menu.
- Confirm app launch remains fast with several long conversations.
- Confirm old prototype conversations migrate once and do not duplicate.

## Assumptions

- iOS target supports SwiftData.
- Cloud sync is out of scope for this pass.
- Search can be built later on top of SwiftData.
- Pending unsent text does not need to survive app restarts unless explicitly added later.
- Attachments should be private app-local files, not Photos-library references.
- The existing UI structs remain the best shape for canvas rendering and animation.
