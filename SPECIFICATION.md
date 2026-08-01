# Aquinas Technical Specification

## Core Objective
To provide a local-first, privacy-centric environment for high-fidelity intellectual inquiry, utilizing a branching logic structure to map the evolution of thought.

## Architecture: The Scholastic Method
The software architecture is modeled after the *Disputatio*:
1. **The Propositio (The Node):** Every entry is a discrete unit of thought (a Node) containing text, metadata, and links.
2. **The Obiectio (The Branch):** Users can create divergent branches from any node to explore counter-arguments or alternative perspectives without destroying the original premise.
3. **The Adiectio (The Connection):** A system of semantic and explicit links that create a web of interconnected ideas, forming a "Personal Corpus."

## Data Integrity & Privacy
- **Local-Only Execution:** The application operates entirely on-device. There is no network-level telemetry.
- **Zero-Knowledge Architecture:** Since no data leaves the device, there is no encryption key management required for the user; the security of the data is synonymous with the security of the device itself.
- **Structured Permanence:** Data is stored in a human-readable, non-proprietary format to ensure that even if the software becomes obsolete, the user's "Corpus" remains accessible.

The current development topology runs the model and tree database in a local FastAPI process on
the user's Mac and connects over local HTTP. This is not a relaxation of the production
local-only requirement; it is an integration stage. The current iOS conversation snapshot uses
`UserDefaults`, and a structured SwiftData/file-backed migration remains planned.

## Functional Requirements
- **Branching Logic:** Ability to fork a conversation thread into a new "argumentative branch."
- **Insight Extraction:** Capability to link nodes via semantic similarity (local LLM-driven).
- **The Corpus View:** A visualization of the interconnected web of nodes, representing the totality of the user's knowledge.
- **Context Checkpointing:** `/compact` condenses older model context without modifying the
  transcript the user sees. `/clear` resets the active conversation.
- **Visible Model Work:** Questions, contextual definitions, and tree updates are serialized,
  cancellable tasks with an Idle/Thinking status and inspectable queue.
- **Safe Thinking Display:** The model may show a concise explanation of its approach, but raw
  chain-of-thought and provider scratch work must never be exposed.
- **Reasoning Integrity:** The model treats earlier conclusions as revisable, evaluates the
  strongest reasonable form of the user's argument, and changes its position only when the
  reasoning or evidence warrants it.
- **Persona Isolation:** Personality settings affect ordinary conversational expression only.
  Definitions, Questions of the Day, Insights, Nodes, Midpoints, Make Node children, extraction,
  compaction, and structured repair remain neutral.
- **Explicit Availability:** Live model actions never substitute mock or placeholder knowledge
  when generation fails. They preserve or restore user state and expose retry where appropriate.
- **Daily Inquiry:** Home may present one grounded, open-ended Question of the Day. Once answered
  or expired, its replacement is generated through the visible model-task queue after an eligible
  idle interval.

## Canvas Mode: Insight Tree Semantics
Canvas Mode is the conversation's explorable semantic map. It is entered from the active conversation view and shows the concepts discussed in that conversation as a web of related ideas.

The graph has two primary semantic objects:

1. **Insights:** Smaller, concrete contextual definitions the user explicitly saves. These are the
   granular intellectual units the user can inspect, save, quote, or use as the seed for further
   inquiry.
2. **Node Concepts:** Higher-level subjects that group related Insights or preserve a pivotal
   concept extracted from a completed response. Automatic response analysis creates Node Concepts,
   never automatic Insights.

Relationships are represented spatially and with connecting lines:

- **Insight-to-Node Concept edges:** Each Insight is connected to the Node Concept it belongs to. Edge length communicates semantic closeness: shorter edges mean the Insight is strongly related to the Node Concept, while longer edges mean the relationship is weaker or more peripheral.
- **Node Concept-to-Node Concept edges:** Node Concepts are also connected to each other when their subject clusters are semantically related. Their placement and edge lengths should communicate conceptual proximity between broader subjects.

Conversation Node Concept connectors currently have a 360px minimum length. Insight-to-Node
connectors retain a separate shorter floor and relatedness mapping.

The purpose of Canvas Mode is not decoration or analytics. It is an inquiry interface: the user should be able to move from a linear conversation into a structured web, see how the ideas they are discussing relate to one another, and continue learning by following those relationships.
