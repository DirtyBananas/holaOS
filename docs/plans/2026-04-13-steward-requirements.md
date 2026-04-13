# Steward — Requirements

**Status:** Draft v1
**Date:** 2026-04-13
**Owner:** DirtyBananas / Steward
**Companion documents:** Vision, Architecture, Implementation Plan, Risks

---

## 1. Purpose

Steward is a personal memory and retrieval system for a "prospector" — a developer who has accumulated years of high-value information across dozens of channels and can no longer find what they know they have. This document specifies the functional and non-functional requirements for v1. It is the contract between the product vision and the implementation plan: every requirement here must be testable, and anything not listed is explicitly deferred.

---

## 2. Scope

### 2.1 In scope for v1
- Ingestion adapters for a prioritized subset of sources (see Section 7).
- Normalized item store in SQLite with append-only semantics.
- Dual index: SQLite FTS5 (keyword) plus a local vector index (semantic).
- Hybrid retrieval with re-ranking and citation-based answers.
- One-hotkey manual capture from desktop and iOS.
- Background "radar" pre-computed query runner.
- Three thin clients: terminal CLI, macOS launcher action (Raycast), and Obsidian plugin.
- Context tagging (work / personal / sub-contexts) and exclusion rules.
- Local-first execution; cloud LLM calls are opt-in per query.

### 2.2 Deferred to later versions
- Full cross-platform desktop parity (Linux / Windows clients).
- Mobile-native app beyond iOS Shortcuts.
- Real-time OCR of screen capture streams (ScreenPipe/Rewind ingest is deferred; hooks reserved).
- Multi-user or shared memory.
- Cloud sync / multi-device state replication.
- Automatic knowledge graph extraction beyond actor/topic tags.

---

## 3. User Stories

Each story is framed "as a / I want / so that" and is traceable to one or more functional requirements (FR-*).

1. **Historical semantic recall.** As a prospector, I want to ask "What did I figure out about X back in 2019?" so that knowledge from earlier life eras is recoverable without remembering where I wrote it. (FR-R1, FR-R2, FR-I2)
2. **Channel-agnostic recall.** As a user, I want to ask "Where did I see that thing about battery chemistry?" and get the matching item regardless of whether it was an iMessage, a PDF, or a Claude chat. (FR-R1, FR-N1)
3. **Source-routing amnesia.** As a user, I want to ask "Was that in Claude or ChatGPT?" and have Steward answer from unified memory so I never have to remember which AI I used. (FR-I3, FR-R4)
4. **Actor plus topic.** As a user, I want to ask "What did Dan send me about Y?" and get items filtered by actor and topic across channels. (FR-R3)
5. **Proactive context surfacing.** As a user writing a new doc, I want Steward to surface relevant past notes about the current project without me composing a query. (FR-W2, FR-INT3)
6. **Non-destructive trove ingest.** As a user who just rediscovered an old Dropbox, I want to point Steward at it and have the valuable parts indexed in place without migration or modification. (FR-IN1, FR-IN4, NFR-T1)
7. **Frictionless capture.** As a user, I want a single hotkey to log a one-line thought from anywhere in under 10 seconds so that I actually do it. (FR-C1, NFR-UX1)
8. **Morning brief.** As a user, I want a pre-computed "radar" digest surfaced every morning showing candidates from my archive that relate to today's active threads. (FR-W1)
9. **Editor context.** As a user writing in Obsidian, I want a side panel showing anything Steward knows that relates to what I'm currently writing. (FR-INT3)
10. **Citation click-through.** As a user, I want every answer or search result to link back to the original item in its native location (Messages, Dropbox, Obsidian, etc.). (FR-R5)
11. **Exclusion guarantee.** As a user, I want to mark a folder, a contact, or a time window as excluded and trust absolutely that Steward will never index it. (FR-IN5, NFR-P1)
12. **Inspectability.** As a developer-user, I want to open any index file or embedding trace with standard tools so I can debug or audit. (NFR-F1, NFR-F2)

---

## 4. Functional Requirements

### 4.1 Ingestion (FR-IN*)
- **FR-IN1. Native-format ingestion.** Each adapter must read the source in its native format (SQLite, JSON export, IMAP, OS API, markdown, mbox, plist). No pre-conversion by the user.
- **FR-IN2. Incremental ingestion.** Adapters must support resumable syncs keyed off a per-source cursor (timestamp, message id, file mtime). Re-running an adapter on an unchanged source must be a no-op.
- **FR-IN3. Append-only.** Ingestion must never write to, rename, or delete any source file or database. Violations must fail closed.
- **FR-IN4. In-place indexing of newly discovered troves.** A user must be able to register a new directory or archive and have it indexed without files being moved or copied out.
- **FR-IN5. Exclusion rules.** Adapters must honor a per-source rules file with exclusions by: path prefix, glob, contact identifier, thread id, and time range. Excluded items must never be read past the identifier stage.
- **FR-IN6. Contextual tagging at ingest.** Each item must be assigned zero or more context tags (e.g., `work`, `personal`, `side-project-foo`) by rules defined per source.

### 4.2 Normalization and Storage (FR-N*)
- **FR-N1. Common item shape.** Every ingested item must conform to the schema: `{id, source, timestamp, actors[], content, context[], metadata{}, source_pointer}`. `id` is a content hash. `source_pointer` must be sufficient to open the original.
- **FR-N2. Stable ids.** Re-ingesting the same source item must produce the same `id` so duplicates collapse.
- **FR-N3. Item store.** Items are persisted in a single SQLite database. Schema migrations must be forward-only and versioned.
- **FR-N4. Provenance integrity.** Every item must retain enough of its original metadata to reconstruct the source_pointer even if the original is later moved.

### 4.3 Indexing (FR-I*)
- **FR-I1. Keyword index.** All item `content` must be indexed in SQLite FTS5 with snippet and rank support.
- **FR-I2. Semantic index.** All item `content` must be embedded into a local vector index (e.g., sqlite-vec, LanceDB, or equivalent local store). Embeddings must be computed by a locally runnable model by default.
- **FR-I3. Index parity.** Both indices must stay consistent with the item store. A reconciliation task must detect and repair drift.
- **FR-I4. Rebuildability.** Deleting any or all index files must leave the item store and the source data untouched. Re-running indexing from the item store must fully reconstruct them.

### 4.4 Retrieval (FR-R*)
- **FR-R1. Hybrid query.** A single query function must fan out to FTS5 and the vector index, merge candidates, and re-rank them into a unified result list.
- **FR-R2. Temporal filters.** Queries must accept time-range filters (absolute dates and relative like "in 2019", "last month") and must apply them as hard constraints.
- **FR-R3. Actor and context filters.** Queries must accept actor filters (e.g., `from:Dan`) and context filters (e.g., `context:work`). These must compose with keyword and semantic terms.
- **FR-R4. Natural-language answering.** Steward must support a QA mode that synthesizes an answer from retrieved items with inline citations to each supporting item id.
- **FR-R5. Provenance links.** Every result — whether raw or synthesized — must expose a working link or command that opens the original in its native app or location.
- **FR-R6. Deterministic replay.** A given query plus a given snapshot of the item store must return the same candidate set (ranking may vary only if the LLM reranker is invoked).

### 4.5 Capture (FR-C*)
- **FR-C1. One-hotkey desktop capture.** A global hotkey must open a minimal prompt, accept one line of text, and commit it to the item store as `source: "prospector-log"` in under 10 seconds end-to-end (NFR-UX1).
- **FR-C2. iOS capture.** An iOS Shortcut must accept a one-line input (text or voice-dictated) and post it to the local Steward endpoint (via Tailscale, local network, or queued sync).
- **FR-C3. Offline capture queue.** If Steward is unreachable, captures must buffer locally and flush on reconnect, without loss.

### 4.6 Integration and Clients (FR-INT*)
- **FR-INT1. Terminal CLI.** A CLI must support: `steward ingest <source>`, `steward query <text>`, `steward ask <text>`, `steward log <text>`, `steward radar`.
- **FR-INT2. Launcher action.** A Raycast (or equivalent) action must invoke query/ask/log flows without opening a separate window.
- **FR-INT3. Obsidian plugin.** A plugin must render a side panel that, on demand or on document-change debounce, queries Steward for items related to the active note's content.
- **FR-INT4. Adapter pluggability.** New source adapters must be addable as a single module implementing a documented `SourceAdapter` interface, without core changes.

### 4.7 Write-back and Radar (FR-W*)
- **FR-W1. Radar passes.** A scheduled background runner must execute a user-defined set of saved queries on a cadence (e.g., daily) and write ranked candidates to a `radar/` output directory as markdown.
- **FR-W2. Optional prospector.md write-back.** On successful retrieval, the user may opt to append a pointer (item id + one-line summary) to a designated `prospector.md`. Write-back must be append-only and must never modify existing lines.
- **FR-W3. Dry-run.** Every write-back operation must support a dry-run mode that prints what would be written.

---

## 5. Non-Functional Requirements

### 5.1 Privacy (NFR-P*)
- **NFR-P1. Exclusion absolutism.** Exclusion rules must be enforced at the ingestion layer. A unit test suite must verify that excluded items never reach the item store or either index.
- **NFR-P2. Local-first by default.** All indices, embeddings, and item data must live on the user's machine. No network call is made for retrieval unless the user explicitly invokes a cloud-LLM mode.
- **NFR-P3. Per-query network opt-in.** Cloud LLM use must be a per-query flag, never an ambient default. The system must log each cloud call in an auditable trace.

### 5.2 Performance (NFR-PE*)
- **NFR-PE1. Retrieval latency.** 95th percentile query latency for the common case (hybrid retrieve + local rerank, no cloud LLM) must be under 2 seconds on the reference machine (Apple Silicon laptop).
- **NFR-PE2. Ingestion throughput.** Initial bulk ingest of 100k items must complete in under 60 minutes on the reference machine.
- **NFR-PE3. Incremental sync latency.** Incremental sync for an active source (e.g., iMessage) must surface new items in retrieval within 5 minutes of them appearing in the source.

### 5.3 Resources (NFR-R*)
- **NFR-R1. CPU budget.** Background components (radar runner, incremental syncers) must average under 1% CPU over a one-hour window when idle-ish.
- **NFR-R2. Memory budget.** Resident memory of the background daemon must stay under 300 MB.
- **NFR-R3. Storage growth predictability.** Per-source storage growth must be reported, and retention/compaction policies must be configurable per source (e.g., drop embeddings for items older than N years while keeping FTS + item row).

### 5.4 Format and Inspectability (NFR-F*)
- **NFR-F1. Plain formats.** All persistent state must be in SQLite, plain text, markdown, or JSON. No proprietary binary blobs except for the vector index file, which must be a documented open format.
- **NFR-F2. File-level inspectability.** Every index, embedding store, log, and trace must be a file on disk the user can open with standard tools.
- **NFR-F3. No lock-in.** An export command must dump the item store to JSONL in a documented schema.

### 5.5 Trust (NFR-T*)
- **NFR-T1. No destructive operations.** Steward must never modify, rename, or delete source data. This is enforced by code review and a test that mounts sources read-only.
- **NFR-T2. Safe uninstall.** Deleting the Steward data directory must leave all source data intact and the user's machine in its prior state.
- **NFR-T3. Auditable ingest.** Every ingest run must produce a log entry with source, items added, items skipped (with reason), and duration.

### 5.6 UX (NFR-UX*)
- **NFR-UX1. Capture friction.** Hotkey to committed item must take under 10 seconds end-to-end for a one-line log, measured from first keystroke.
- **NFR-UX2. Empty-state clarity.** First-run experience must surface the list of available adapters and their current status (connected / not connected / excluded).
- **NFR-UX3. Offline retrieval.** All core retrieval flows must work fully offline. Cloud features must degrade gracefully with a visible banner.
- **NFR-UX4. Platform baseline.** v1 must run on macOS 14+. Linux is best-effort; Windows is out of scope.

---

## 6. Source Matrix

| Source | Native format | Ingestion approach | v1 priority |
|---|---|---|---|
| iMessage | SQLite (`chat.db`) | Direct read-only SQLite, copy to shadow | P0 |
| Apple Notes | SQLite + protobuf | Read-only SQLite + decoder | P0 |
| Obsidian vault | Markdown files | Filesystem walk + frontmatter parse | P0 |
| Claude chats | JSON export | Import of export files; watch directory | P0 |
| ChatGPT history | JSON export | Import of export files | P0 |
| Email (IMAP) | IMAP | Read-only IMAP fetch, store headers + body | P1 |
| Gmail (API) | Google API | OAuth, incremental history id | P1 |
| Dropbox (local mount) | Files | Filesystem walk of mounted folder | P1 |
| Google Drive | Google API | OAuth, changes feed | P1 |
| Slack | Export ZIP | Parse export JSON | P1 |
| Browser bookmarks | SQLite / JSON | Browser-specific readers | P1 |
| Screenshots folder | Image files | Filename + EXIF; OCR deferred | P2 |
| WhatsApp | Export | Parse chat export | P2 |
| Discord | Export | Data package parse | P2 |
| Calendar | `.ics` / EventKit | Read-only | P2 |
| Reminders / Todos | EventKit / app-specific | Read-only | P2 |
| ScreenPipe / Rewind | SQLite / API | Hooks only, no ingest | Deferred |

---

## 7. Out of Scope for v1

- Writing back into any source system (no sending a Slack reply, no editing a note).
- Server-hosted multi-device deployment.
- Automatic extraction of structured knowledge graphs beyond tags.
- Full-fidelity OCR of screenshots and screen recordings.
- Fine-tuning embedding or LLM models on personal data.
- Sharing memory with other users.
- Native Android or Windows clients.

---

## 8. Acceptance Criteria for v1

Steward v1 is considered "working" when all of the following can be demonstrated on the reference machine against a real personal corpus of at least 50k items:

1. **End-to-end query.** The user runs `steward ask "what did I figure out about X in 2019"` from the CLI and gets a cited answer in under 2 seconds (local mode).
2. **Channel-agnostic recall.** A single query returns results from at least three distinct source types in one merged ranked list.
3. **Hotkey capture.** The user triggers the desktop hotkey, types one line, and the item is queryable within 5 seconds, with full round-trip under 10 seconds.
4. **Non-destructive trove ingest.** The user points Steward at a newly mounted old Dropbox folder; files are indexed in place; an `md5sum` of the folder before and after matches.
5. **Exclusion honored.** A folder and a contact marked excluded produce zero hits in either index, verified by a query that would otherwise match.
6. **Radar digest.** A scheduled radar pass writes a dated markdown digest to the configured output directory.
7. **Obsidian side panel.** Editing a note in Obsidian surfaces at least one relevant historical item in the Steward panel within 1 second of debounce.
8. **Uninstall safety.** Deleting the Steward data directory removes all Steward state and leaves every source system unchanged, verified by pre/post hashes.
9. **Resource budget.** Over a 1-hour idle window, the background daemon stays under 1% average CPU and 300 MB RAM.
10. **Offline retrieval.** With the network disabled, all queries in criteria 1 and 2 still succeed.

Any failure of the above blocks the v1 release.
