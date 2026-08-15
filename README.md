# Linus

> **The model is a component. Linus is the system around it.**

Linus is a **browser-resident, local-first AI middleware architecture** that separates the reasoning engine from durable state, memory, governance, provenance, verification, tools, and execution policy.

The configured LLM is a replaceable **reasoning socket**. It generates language and inference, but it does not own Linus's durable memory, belief state, provenance, governance, storage, scheduling, export/import, or execution policy.

LinusDB provides the local state authority. Around it, Linus coordinates semantic memory, belief graphs, quarantine, provenance, verification, frame governance, drift measurement, tool execution, workspace state, bounded reasoning rounds, serialized model transport, background maintenance, and governed state transitions.

Instead of treating an AI agent as a monolithic cloud service whose model output implicitly owns the conversation, Linus places the durable authority of the system in a local browser runtime built around **IndexedDB / LinusDB**. The configured large language model — external or local — is a replaceable reasoning socket. It generates reasoning and language; the surrounding system controls what context reaches it, how execution is coordinated, what gets remembered, what gets challenged, what can supersede an existing belief, and what is retained as evidence.

The result is an AI architecture designed around **operational continuity rather than a static chat transcript**: persistent local state can influence future retrieval and routing, background maintenance can reorganize that state, and governance mechanisms can prevent a single fluent generation from becoming durable truth automatically.

---

## The Central Architectural Thesis

The architecture deliberately separates three dimensions of behavior:

- **Model-dependent** — raw generation capability, world knowledge, calibration, and reasoning behavior supplied by the configured LLM.
- **Architecture-dependent** — orchestration, memory persistence, governance, provenance, concurrency, verification boundaries, tool execution, and execution policy supplied by Linus.
- **History-dependent** — the accumulated memories, beliefs, rejected evidence, provenance, and prior state transitions carried by a particular Linus instance.

This separation is the central architectural thesis:

> **Changing the reasoning engine does not require surrendering ownership of the surrounding cognitive state machine.**

Linus is therefore not simply a model wrapper, a chat history, or a conventional RAG layer. It is a governed execution architecture around a reasoning engine.

The model generates. Linus determines what context reaches it, how generation is orchestrated, what gets remembered, what gets challenged, what gets verified, what can become durable state, what remains unresolved, and how state changes are preserved over time.

---

## Core Philosophy

### The intelligence is in the system

Linus separates two different kinds of state:

- **Operational Vector** — the real-time token-generation process executed through the configured reasoning socket.
- **Structural Map** — the persistent client-side state containing memory, beliefs, provenance, governance records, telemetry, workspace state, and synchronization metadata.

The model is therefore **not the owner of system memory, beliefs, provenance, storage, scheduling, export/import, or policy**. History-dependent behavior comes from persistent local state; architecture-dependent behavior comes from the middleware; model-dependent behavior comes from the configured reasoning engine.

This boundary is explicit in the implementation: local IndexedDB state is authoritative for durable Linus state, while live model calls and external research remain network operations.

---

## Architecture at a Glance

```text
                           ┌──────────────────────────────┐
                           │         WORLD INPUTS         │
                           │                              │
                           │ Text · Voice · Vision · APIs│
                           │ Files · Sensors · Agents    │
                           └──────────────┬───────────────┘
                                          │
                                          ▼
                    ┌────────────────────────────────────────┐
                    │       AGENTIC MIDDLEWARE / RUNTIME     │
                    │                                        │
                    │ identity · session · context assembly │
                    │ tools · bounded rounds · stop rules   │
                    │ streaming · API serialization         │
                    └────────────────────┬───────────────────┘
                                         │
                        ┌────────────────┴────────────────┐
                        │                                 │
                        ▼                                 ▼
             ┌─────────────────────┐          ┌─────────────────────┐
             │  REASONING SOCKET   │          │      LINUSDB         │
             │                     │          │  local authority     │
             │ Claude / GPT /       │          │                     │
             │ local Llama / other │          │ IndexedDB state      │
             └──────────┬──────────┘          │ memory · beliefs     │
                        │                     │ provenance · threads │
                        │                     │ review · telemetry  │
                        │                     └──────────┬──────────┘
                        │                                │
                        │                                ▼
                        │                     ┌─────────────────────┐
                        │                     │  GOVERNED MEMORY   │
                        │                     │                     │
                        │                     │ retrieve → stage    │
                        │                     │ verify → promote    │
                        │                     │ quarantine → review │
                        │                     └──────────┬──────────┘
                        │                                │
                        │                                ▼
                        │                     ┌─────────────────────┐
                        │                     │ BELIEF / PROVENANCE │
                        │                     │                     │
                        │                     │ consistency          │
                        │                     │ ratification         │
                        │                     │ lineage / audit      │
                        │                     └──────────┬──────────┘
                        │                                │
                        │                                ▼
                        │                     ┌─────────────────────┐
                        │                     │ VERIFICATION LAYER  │
                        │                     │                     │
                        │                     │ Witness Engine       │
                        │                     │ Pre-Stream Gate      │
                        │                     │ Drift / coherence    │
                        │                     └──────────┬──────────┘
                        │                                │
                        └────────────────┬───────────────┘
                                         ▼
                           ┌──────────────────────────────┐
                           │   LOCAL STATE PORTABILITY    │
                           │                              │
                           │ signed operations · export  │
                           │ WebRTC peer replication     │
                           └──────────────────────────────┘
```

The implementation describes this as a **hybrid client-side middleware stack** in which orchestration, memory, verification, provenance, and governance are coordinated around a replaceable LLM.

---

# Architectural Pillars

## 1. Agentic Orchestration & Bounded Execution

A user turn is not treated as a blind one-shot model call.

The runtime coordinates:

- the configured reasoning endpoint
- local memory retrieval
- live-source context and tools
- bounded tool/reasoning rounds
- stopping conditions
- streaming output
- persona/context routing
- a serialized network request queue
- memory writes and downstream maintenance

The surrounding runtime therefore controls execution rather than delegating the entire system to the model.

### Serialized network execution

Calls to the configured underlying LLM API are serialized through a single promise-chain queue, one request at a time. Read-only third-party lookups may use a separate parallel lane, but they do not share the model queue. This keeps background memory and maintenance activity from colliding with an active live response.

---

## 2. LinusDB: Local State as the Authority

**LinusDB is the local memory authority in the browser.** IndexedDB stores the durable state of the application instead of relying on a hosted memory database.

The implemented storage layer includes distinct stores for different semantic responsibilities:

| Store / subsystem | Role |
|---|---|
| `beads` / `idx` | Fast-path fuzzy-match cache for previously answered questions and lookup history |
| `ghost` | Durable long-term memory pool of embedded fragments |
| `quarantine` | Staging area before synthesized material can become durable memory |
| `linus_db` | Distilled/reasoning archive |
| `belief_nodes` / `belief_edges` / `belief_log` | Structured belief graph and belief history |
| `witness_log` / `drift_log` | Verification and drift telemetry |
| `raw_friction_vault` / `structural_review_queue` | Retention path for rejected or high-friction material |
| `ratification_queue` | Explicit review path for proposed supersession of protected beliefs |
| `workspace_files` | Code/workspace state |
| `threads` | Saved conversation threads |
| `answer_hashes` | Bounded duplicate-answer detection |
| `generation_dependency_log` | Review-only claim-to-premise audit ledger |
| `linus_discoveries` | Private scratch record of unprompted hypotheses/anomalies; never surfaced in UI |
| `frame_artifacts` | QUERY→FRAME formalization objects, written once per frame transition, never reconstructed |
| `frame_drift_log` | Output-boundary exclusion-match findings against a written frame (append-only) |
| `escalation_events` | Append-only ledger for invariant-floor breaches |
| `rejection_log` / `provenance_chains` | Append-only evidence ledgers for epistemic provenance |
| `sovereign_meta` / `sovereign_ops` / `sovereign_devices` / `sovereign_peers` | Identity, signing, portability, and device synchronization metadata |

The source explicitly states that these structures are client-side and that Linus does not use an external memory server as the authority for durable state.

---

## 3. Governed Local Memory

Linus treats memory as **governed state**, not a passive cache.

A typical durable-memory path is conceptually:

```text
Generated response
      │
      ▼
Chunk + embed
      │
      ▼
Quarantine
      │
      ▼
Independent convergence
      │
      ▼
Belief consistency / governance
      │
 ┌────┴────┐
 │         │
 ▼         ▼
ACCEPT   CONFLICT / REJECT
 │         │
 ▼         ▼
Ghost    Review / Ratification
```

The implementation describes new material as entering quarantine rather than going directly from a single turn into durable memory. Promotion depends on independent later convergence, lineage separation, and belief consistency. Conflicting or high-friction material can remain in review instead of being silently discarded.

### Promotion pipeline

A generated response is chunked and embedded, then staged into quarantine rather than written straight to ghost. It stays there until independent convergence — the same content, embedded again from an unrelated later turn, has to land above a similarity threshold, from a different session, without overlapping the retrieval lineage that produced the first copy (so a fact retrieved and simply restated doesn't count as confirming itself). Normally two independent sessions are required; if either contributing turn came from a round where memory search returned nothing to work from, the bar rises by one session, since a fluent but ungrounded synthesis is more likely in that case. Only once that convergence bar is met does the fragment get checked against the belief graph and, if it doesn't conflict, written into ghost as durable memory. A verdict that rejects or conflicts blocks promotion outright and leaves the item in quarantine rather than deleting it, since the underlying disagreement is unresolved, not just unconfirmed. Refusals are filtered out before any of this starts — a declined request is not a claim about the world and is never eligible to become memory.

### Memory retrieval

Ghost fragments are short embedded text fragments with confidence/weight and timestamps. Retrieval considers semantic similarity, confidence weighting, and recency decay before selecting relevant material. GhostPipeline handles memory interception, validation, and reconciliation paths.

### Memory is not self-certifying

A retrieved memory is evidence/context for the runtime; it is not automatically equivalent to truth. The architecture intentionally keeps free-text memory, belief state, provenance, verification, and audit concepts separate.

---

## 4. Belief Governance

Linus includes a structured belief graph separate from the free-text ghost pool.

The governance layer can distinguish between:

- accepted claims
- protected beliefs
- schema/validation failures
- foundational axiom conflicts
- hierarchy/authority conflicts
- quarantined claims
- claims awaiting explicit ratification

The implementation uses explicit structural rejection types, including `VALIDATION_ERROR`, `AXIOM_CONFLICT`, and `HIERARCHY_OUTRANKED`, so orchestration does not have to infer a routing decision from free-form error text.

A protected belief is therefore not silently overwritten just because a later model response says something different. A conflicting proposal can remain visible as a proposed supersession until the relevant governance path resolves it.

### Source hierarchy

When two learned (non-axiom) belief claims conflict, a fixed ranking decides it: live web search outranks an explicit user statement, which outranks the model's own prior assertion, which outranks ambient recalled memory. A claim from a higher-ranked source supersedes (subject to ratification if the target is protected); a claim from a lower-ranked source is rejected; equal rank or an unranked source falls through to quarantine rather than guessing.

---

## 5. Witness Engine

The **Witness Engine** is a verification mechanism, not a truth oracle.

It can obtain an independent context-free answer on a background cadence and compare semantic divergence against the primary generation. The result can produce nominal, advisory, or intervention signals that influence later reasoning/context handling.

The critical boundary is deliberate:

> Verification can produce a warning about divergence. It does not retroactively turn a generated sentence into objective truth.

The architecture explicitly distinguishes the Witness Engine from the Pre-Stream Gate even though both use related cold-call infrastructure.

---

## 6. Pre-Stream Gate

The **Pre-Stream Gate** is positioned at the memory boundary.

It independently evaluates the same user message using a stripped context, embeds the independent answer and primary answer, and calculates semantic divergence. A divergence above the configured threshold blocks the durable-memory write path for that turn.

In the current implementation:

- divergence threshold: `0.35`
- gate deadline: `900 ms`
- a blocked turn keeps the already-rendered answer visible
- the answer is not retracted
- the memory write is blocked instead
- gate failures fail open rather than turning a transient verification outage into a global memory-write outage

### What this means

```text
Primary answer ───────────────► User sees answer
      │
      └──► Independent cold evaluation
                    │
                    ▼
              divergence check
                    │
              ┌─────┴─────┐
              │           │
            PASS        BLOCK
              │           │
              ▼           ▼
         memory may      memory does not
         be promoted     enter durable path
```

This is an important architectural choice: **verification governs persistence, not the user's already-visible output.**

---

## 7. Self-Correcting Code Sandbox

The Linus architecture includes a self-modification workflow around a browser-local code/data sandbox.

Conceptually:

```text
Architectural bottleneck / ingestion constraint
                  │
                  ▼
              Draft patch
                  │
                  ▼
        Isolated sandbox_store
                  │
                  ▼
       Syntax + behavioral tests
                  │
                  ▼
      Historical / mock validation
                  │
            ┌─────┴─────┐
            │           │
          PASS        FAIL
            │           │
            ▼           ▼
     secure promotion  reject patch
     to runtime
```

The architectural idea is that correction should happen inside a bounded environment before a change is promoted into the operational runtime. A live sandboxed execution tool sits ready for anything checkable — a non-trivial calculation, an algorithm's real output, whether written code actually runs. The runtime surfaces actual stdout, stderr, and exit status instead of assuming code worked. File execution is separate from durable memory and separate from the model's text generation.

---

## 8. Memory Compaction Daemon

Long-term memory cannot grow indefinitely without a maintenance strategy. Linus uses an opportunistic **compaction daemon** around the ghost pool.

The current implementation avoids a blunt "delete everything older than N" approach. Instead it clusters sufficiently similar, sufficiently old fragments and collapses a cluster into a denser summary while preserving provenance boundaries.

### Priority scheduling

Ghost mutations use a two-lane `PriorityWriteQueue`:

- **HIGH** — latency-sensitive live/user mutations
- **LOW** — background maintenance

After 10 consecutive HIGH operations while LOW work is waiting, one LOW operation is forced through. This is scheduling control, not a table lock.

### Optimistic concurrency

Compaction uses monotonic fragment versions:

1. snapshot source IDs and versions
2. perform slow summarization/embedding without holding the active write slot
3. reacquire the LOW lane
4. reopen a single read/write transaction
5. re-read current records
6. compare versions against the snapshot
7. commit only if all versions still match
8. otherwise abort stale work and retry from current state

There is deliberately **no cluster-level mutex**. High-priority writes do not wait for a compaction read.

### Summary coverage is bounded

A compaction summary is not an eternal or exhaustive replacement for its source cluster. Each summary carries snapshot metadata describing the exact covered fragment IDs and snapshot time (`snapshot_timestamp` and `snapshot_fragment_ids`). Newer fragments remain independently retrievable. A summary never licenses deletion or suppression of fragments that fall outside its snapshot.

### Retry and deferral

A cluster that repeatedly loses the version-check race may occupy no more than the retry cap (`COMPACT_RETRY_CAP`, currently 5 attempts including the initial attempt) before it is explicitly deferred. Retry exhaustion is not permission to keep consuming LOW slots; the exhausted cluster moves to a lower-frequency deferral path.

---

## 9. Provenance & Generation Dependencies

Linus records the path by which system state was produced without pretending that an audit record is itself proof of truth.

The architecture contains provenance and generation-dependency records that capture relationships such as:

```text
claim ─────depends on─────► premise
  │                           │
  └──── frame / query ────────┘
             │
             ▼
        audit record
```

Generation-dependency records are intentionally **review instrumentation**, not hidden chain-of-thought. The belief graph remains authoritative for live dependency structure, while generation-time artifacts preserve enough information for auditing, replay, and reconciliation.

### Epistemic Provenance

EpistemicProvenance records structural fingerprints, rejection history, inoculation checks, provenance chains, rehydration/hold state, and explicit override flows. `recordRejection()` records rejected claims; `inoculationCheck()` compares incoming claims against prior structural rejection history; `recordAcceptance()` can route structurally similar claims into ratification instead of direct acceptance.

---

## 10. Frame Governance, Drift & Coherence

The runtime also models how a query can be framed and how that framing changes over time.

Implemented frame concepts include descriptive, causal, temporal, structural, counterfactual, comparative, predictive, and normative frames, together with assumptions, exclusions, frame drift, and reconciliation state.

The frame is written once, at the QUERY→FRAME transition, as a discrete object keyed by `frame_id` — never reconstructed from conversational context afterward, since reconstruction would reintroduce self-report through the back door.

The system also exposes diagnostics such as:

- `DriftEngine` — measures embedding-level pre/post state drift around designated generation transitions and persists tracked drift to `drift_log`
- `CoherenceTrajectory` — tracks accepted, ambiguous, and blocked post-gate exchanges, specificity, prompt-version segment boundaries, recency-weighted closing coherence, and volatility
- `MemoryMetrics` — rolling measurements for retrieval-latency variance, confidence/accuracy correlation, write-on-read rate, interference index, and compression-artifact frequency

These are **calibration and system-behavior signals**, not measurements of consciousness or universal truth.

---

## 11. Invariant Floor

Four structural rules are enforced at the storage layer itself, independent of any single feature. Each is a named axiom:

- **Axiom 1 (raw immutability):** Append-only stores — `raw_friction_vault`, `structural_review_queue`, `witness_log`, `drift_log`, `belief_log`, `escalation_events`, `linus_discoveries` — reject every delete outright and reject any overwrite of an existing record. These are logs and audit trails; they only ever grow.
- **Axiom 2 (non-fungibility of identity):** Belief nodes in a protected tier (`axiom` or `anchored`) cannot have their tier changed by a plain write. The only legal path is the explicit supersession flow described in Ratification.
- **Axiom 3 (zero-loss state conservation):** A write that overwrites an existing record cannot silently drop a top-level field that record already had. Partial writes must read-merge-write, not blind-overwrite.
- **Axiom 4 (metaprogramming firewall):** A proposed change targeting a protected runtime path (the invariant floor itself, the belief evaluator, or core runtime logic) is captured, the affected pipeline segment is suspended, and the attempt is written to `escalation_events` as an append-only record — rather than either silently allowed or silently dropped.

A breach of any of these throws before the write reaches IndexedDB; nothing partially lands.

---

## 12. Ratification

When a new claim outranks an existing node by hierarchy but that node sits in a protected tier, the old node is not retired on the spot. The pending replacement is staged in `ratification_queue` with both the old and new text attached, and the old node keeps its current tier and stays fully in force until someone acts on the entry. Nothing times out or auto-resolves.

Exactly two outcomes exist:

- `beliefRatify()` — confirms the correction. The old node's tier flips to `superseded`, a supersedes edge is written linking the new node to it, and the queue entry is marked `ratified`.
- `beliefDismiss()` — cancels it. The old node is untouched, and the entry is marked `dismissed`.

Both actions are explicit and external to the automatic write path; nothing in the belief-check logic itself can complete a supersession. Protection blocks silent, autonomous overwriting, not correction — it just moves correction from automatic to confirmed.

---

## 13. Answer Deduplication

Every response given in a session is SHA-256 hashed and checked against the last 500 hashes before being treated as fresh. A match means: don't repeat that exact answer verbatim — vary the angle instead. This is a same-session repetition guard, not a durable-memory mechanism, and it prunes its oldest entries once past the 500 cap.

---

## 14. Epistemic Control State

- **strainIndex** — a persisted decaying cross-turn signal based on thin/failed rounds
- **epistemicBudget** — a per-turn effort budget that adjusts round/temperature ceilings
- **imaginaryVision** — a cheap silent pre-search step that proposes search angles and a sparse-search signal

These shape effort allocation and retrieval strategy, not factual validity.

---

# Sovereign State

## Local credentials

The configured model credential is handled separately from ordinary memory.

The implementation uses dedicated IndexedDB stores for model-key state plus an origin-scoped, non-extractable cryptographic key for AES-GCM encryption of the credential payload. Credential stores are deliberately excluded from Sovereign peer replication.

This is a **browser/application boundary**, not a claim that the external model provider cannot receive data when a live model request is made.

---

## Sovereign Export / Import

Backup/export is a local persistence operation.

Exported state can contain information required to restore Linus state, including signed operations, dependency order, hashes, and device identity material. Import is verified locally, and foreign append-only state is subject to evidence/quarantine rules instead of automatically acquiring authority.

A rejected import does not get a partial memory write. Fork divergence is not auto-resolved by timestamp or heuristic winner selection — forks preserve the local and foreign spans, identify their common ancestor, classify divergence, and require explicit ratification or dismissal.

---

## WebRTC Peer Replication

Linus does not use a hosted memory database as the replication authority.

Trusted browsers can establish a direct **WebRTC** data channel and exchange signed Linus state operations. Pairing uses Linus identity information, and the implementation explicitly reports that NAT/firewall conditions can prevent a direct peer connection. Credentials are not included in the replication path.

Conceptually:

```text
 Browser A                                  Browser B
 ┌──────────────┐                          ┌──────────────┐
 │ Local LinusDB│                          │ Local LinusDB│
 │ + identity   │                          │ + identity   │
 └──────┬───────┘                          └──────┬───────┘
        │                                          │
        └──────────── WebRTC / signed ops ─────────┘
                           │
                           ▼
                  deterministic merge /
                     local verification
```

The goal is **state portability and peer replication**, not centralized memory storage and not replication of the underlying LLM weights.

---

# Semantic Memory Engine

The application includes a browser-side semantic embedding layer using **Transformers.js** with `Xenova/all-MiniLM-L6-v2` and 384-dimensional sentence embeddings. The model is lazy-loaded and cached client-side after first use.

When the page is opened through `file://`, the runtime detects the environment and uses a hash-embedding fallback because browser module/CDN loading is restricted for that protocol. Serving the app over HTTP(S) enables the full semantic embedding path.

---

# Failure & Degraded Modes

Linus is designed so that optional maintenance and verification failures do not automatically corrupt or block the primary response path.

### Live model unavailable

When the live model cannot provide an answer, the runtime can use a **validated local-memory fallback** when the governing pipeline permits it. The fallback is explicitly marked as degraded and is not rendered as if it were a live model answer. If no valid local fallback exists, the application reports the live failure rather than inventing an answer from arbitrary fragments.

### Verification unavailable

Witness and Pre-Stream verification paths fail open under defined failure conditions. A verification failure is not silently converted into a false claim of clean verification.

### Background maintenance failure

Compaction and maintenance work are intentionally isolated from the live response critical path. Cluster failures can be retried or deferred without preventing later maintenance work from continuing.

---

# What Makes Linus Different

Traditional assistant architectures often collapse these concerns into a single model-centric loop:

```text
user → model → answer → chat history
```

Linus separates them:

```text
                        ┌──────────────┐
                        │   User turn  │
                        └──────┬───────┘
                               ▼
                    ┌────────────────────┐
                    │ Agentic middleware │
                    └──────┬───────┬─────┘
                           │       │
                 ┌─────────┘       └──────────┐
                 ▼                            ▼
          ┌──────────────┐             ┌──────────────┐
          │ Reasoning    │             │ Local state  │
          │ socket       │             │ + memory     │
          └──────┬───────┘             └──────┬───────┘
                 │                            │
                 └────────────┬───────────────┘
                              ▼
                     ┌──────────────────┐
                     │ Governance       │
                     │ + verification   │
                     │ + provenance     │
                     └────────┬─────────┘
                              ▼
                     ┌──────────────────┐
                     │ Durable state    │
                     │ only when rules  │
                     │ allow it         │
                     └──────────────────┘
```

The underlying idea is simple:

> **The intelligence of the system is not only the text the model emits.**

The surrounding architecture determines what context is admitted, what gets remembered, what gets challenged, what can supersede an existing belief, what gets audited, and how much work a turn deserves.

---

# Design Invariants

Several rules are intentionally load-bearing across the architecture:

1. **The LLM is a replaceable reasoning component.** It is not the authority for local memory, governance, provenance, or policy.
2. **Local IndexedDB state is authoritative for durable Linus state.**
3. **Ghost writes are serialized.** Untracked competing write streams are treated as an architectural bug.
4. **Compaction is opportunistic and optimistic.** High-priority writes do not wait behind background compaction.
5. **Summaries have bounded coverage.** A summary never becomes authority to hide or delete newer fragments merely because they share a cluster.
6. **Verification is evidence, not an oracle.** Witness and Pre-Stream signals govern persistence and diagnostics; they do not certify universal truth.
7. **Belief conflicts do not silently overwrite protected state.** Conflicting proposals can enter quarantine or ratification.
8. **Credential state stays outside peer replication.**
9. **Fallback output is explicitly marked as degraded.**
10. **Background autonomy is software autonomy, not a claim of consciousness or execution after the browser is gone.** The strongest defensible continuity claim is operational continuity through persistent state and subsequent retrieval/routing.
11. **Indexes are not authorities.** Inoculation, fingerprint, record-key, protected-key, device, status, and similar indexes are operational lookup structures; they do not establish truth.
12. **Source disposition post-commit is total.** Every ID in `snapshot_fragment_ids` must, after a successful commit, either be absent from the ghost store or carry an explicit non-active disposition.
13. **Single-transaction atomicity.** Summary creation and source deletion/marking must occur in exactly one IndexedDB readwrite transaction.

---

# Running the Current Browser Application

The supplied implementation is a browser HTML application rather than a conventional Node package.

For the full semantic embedding path, serve the application from an HTTP(S) origin. The implementation specifically detects `file://` and falls back to hash embeddings because module/CDN loading is restricted there.

A simple local server is sufficient for development:

```bash
python3 -m http.server 8000
```

Then open the application through the local HTTP origin in your browser.

> Exact deployment, provider endpoint, and API-key configuration are runtime-specific. The README does not assume a cloud backend that the current application does not use.

---

# Security & Privacy Boundary

Linus is **local-first**, not "network-free."

The following distinctions matter:

| Path | Local? | Network? |
|---|---:|---:|
| Durable Linus state | Yes | No hosted memory authority |
| IndexedDB memory / belief state | Yes | No |
| Model credential storage | Yes, browser-local | Credential store excluded from peer replication |
| Live model reasoning | Local orchestration, remote endpoint may be used | Yes |
| Live research | Runtime-dependent | May contact external sources |
| First-load semantic model over HTTP(S) | Browser runtime | May fetch model/package from CDN |
| Sovereign browser-to-browser sync | Local state + direct peer transport | WebRTC peer path |

The implementation itself states that requests to a configured model endpoint cross the network and that external research may also cross the network. Therefore **local persistence should not be confused with end-to-end network isolation**.

This README is a technical architecture document, not a legal privacy policy.

---

# Current Architecture Boundaries

Linus deliberately avoids several stronger claims that are not justified by the runtime:

- It does **not** claim the underlying LLM weights are local merely because memory is local.
- It does **not** use a hosted memory database as the authority for Linus state.
- It does **not** claim computation continues after the browser/process is gone.
- It does **not** treat drift metrics, belief graphs, or verification scores as universal truth meters.
- It does **not** treat generation-dependency records as hidden chain-of-thought.
- It does **not** treat every fluent model output as durable memory.
- It does **not** replicate model credentials through Sovereign peer sync.
- It does **not** use timestamp heuristics to silently decide which branch of a fork is authoritative.
- It does **not** treat a failed repair scan as a substitute for transactional atomicity.

These boundaries are part of the architecture, not disclaimers added after the fact.

---

# The Bigger Idea

Linus is built around a different unit of AI engineering.

The question is not only:

> **"Which model should answer?"**

It is also:

> **"What system should surround the model so that reasoning can accumulate state, challenge itself, preserve provenance, govern belief, recover from failure, and remain portable without giving the model ownership of the system?"**

That is the role of Linus.

The model can change.

The browser can change.

The memory can be reorganized.

A belief can be challenged.

A verification signal can block persistence.

A later piece of evidence can supersede an earlier state through governance.

And the durable architecture remains the system around the reasoning socket.

---

## Status

**Architecture:** browser-resident, client-side middleware

**Primary persistence:** IndexedDB / LinusDB (v28)

**Reasoning:** replaceable configured LLM endpoint / reasoning socket

**Semantic memory:** Transformers.js + `Xenova/all-MiniLM-L6-v2` (384-dimensional embeddings)

**Memory governance:** quarantine, convergence, belief consistency, ratification, provenance

**Verification:** Witness Engine + Pre-Stream Gate + drift/coherence instrumentation

**Maintenance:** opportunistic ghost compaction with serialized priority writes and optimistic version checks

**State portability:** local export/import with signed state operations and verification

**Peer synchronization:** direct WebRTC replication between trusted browser instances

**Credential boundary:** browser-local encrypted credential stores, excluded from Sovereign peer replication

**Invariant enforcement:** four named axioms enforced at the storage layer before any IndexedDB write

**Answer deduplication:** SHA-256 session hash-log (last 500 responses)

**Frame governance:** QUERY→FRAME artifact system with drift tracking and coherence trajectory

**Generation audit:** append-only generation dependency log with provenance tagging

---

## Source Basis

This README was written from the provided Linus architecture description and the uploaded `LinusDBnew_prompt.html` implementation. Where the implementation provides exact behavior or boundaries, those details are documented as implementation facts; where the source does not establish a stronger guarantee, this README intentionally avoids inventing one.

Prompt-level cognitive guidance is intentionally described as guidance, not as a runtime guarantee unless the code enforces it.
