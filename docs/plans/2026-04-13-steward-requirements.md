# Steward: Requirements

*Functional and non-functional requirements for a local-first personal memory and retrieval system.*

---

## Scope

**v1** delivers a working CLI-based retrieval system over three sources with hybrid search and LLM-assisted answers. The user can type `recall "query"` and get cited results from their prospector log, one LLM chat export, and iMessage history.

**Deferred** means explicitly out of scope for v1 but designed-for in the architecture (no decisions that block future addition).

---

## User Stories

1. **US-1: Cross-channel recall.** As a prospector, I want to search across all my channels with one query, so that I stop wasting mental energy remembering which channel something was in.

2. **US-2: Historical retrieval.** As a prospector, I want to ask "what did I figure out about X back in 2019?" and get an answer with sources, so that I stop re-deriving things I already solved.

3. **US-3: Channel-agnostic lookup.** As a prospector, I want to find "that thing about battery chemistry" without knowing whether it was in Claude, ChatGPT, iMessage, or email, so that routing metadata stops consuming my working memory.

4. **US-4: Actor + topic search.** As a prospector, I want to ask "what did Dan send me about Y?" and get results filtered by person and topic, so that I can trace conversations across channels.

5. **US-5: Frictionless capture.** As a prospector, I want to log "this matters" in under 10 seconds from anywhere (desktop, mobile), so that my prospector log grows without feeling like work.

6. **US-6: Non-destructive trove ingestion.** As a prospector, when I discover an old account (Dropbox, email archive), I want to index the valuable parts without migrating or reorganizing anything, so that newfound troves become searchable immediately.

7. **US-7: Cited answers.** As a prospector, I want every answer to cite its sources with a link back to the original, so that I can verify anything the system tells me.

8. **US-8: Morning brief.** As a prospector, I want a daily digest of items worth remembering, surfaced without me asking, so that valuable context arrives proactively. *(Deferred to Phase 7.)*

9. **US-9: Context-aware surfacing.** As a prospector, I want the system to notice what I'm working on and surface related past items, so that forgotten context appears at the right moment. *(Deferred to Phase 7.)*

10. **US-10: Semantic recall.** As a prospector, I want to search by meaning ("that thing about feeling burned out last fall"), not just keywords, so that I can find things I can't phrase precisely.

11. **US-11: Quick pointer logging.** As a prospector, when I find something valuable in any channel, I want to drop a one-line pointer into my prospector log without leaving my current context.

12. **US-12: Incremental source addition.** As a prospector, I want to add new sources (Apple Notes, email, Dropbox) one at a time without rebuilding the system, so that the system grows with my needs.

---

## Functional Requirements

### Ingestion

- **FR-1**: Ingest data from source-native formats (SQLite databases, JSON exports, markdown files, IMAP, filesystem) without copying or modifying source data.
- **FR-2**: Normalize all ingested items into a common item shape (id, source, timestamp, actors, content, context, metadata, source_pointer).
- **FR-3**: Support incremental ingestion — only process items newer than the last ingestion run.
- **FR-4**: Deduplicate items by stable ID (hash of source + timestamp + content).
- **FR-5**: Source adapters must implement a healthcheck that verifies the source is reachable and the schema is as expected.
- **FR-6**: Adapter failures must fail loudly (logged, surfaced in health endpoint) and must not block other adapters.

### Indexing

- **FR-7**: Maintain a keyword index (SQLite FTS5) over all item content with Porter stemming.
- **FR-8**: Maintain a semantic index (sqlite-vec) over chunked item content using local embeddings.
- **FR-9**: Chunk items at ~500 tokens with ~50 token overlap for embedding.
- **FR-10**: Both indices must live in a single SQLite database file.

### Retrieval

- **FR-11**: Support keyword-only, semantic-only, and hybrid retrieval modes.
- **FR-12**: Hybrid retrieval must query both indices in parallel, merge results with score-weighted union, and rank items appearing in both indices higher.
- **FR-13**: Support optional LLM re-ranking of top candidates for precision.
- **FR-14**: Support LLM-assisted answer synthesis using only retrieved items as context.
- **FR-15**: Every answer must include citations with source pointers to the original items.
- **FR-16**: The LLM must never answer without injected retrieved context (no hallucinated memories).

### Capture

- **FR-17**: Support manual capture via CLI command that appends a line to prospector.md and indexes it immediately.
- **FR-18**: Capture must complete end-to-end in under 10 seconds.
- **FR-19**: Support capture from desktop (CLI/hotkey), mobile (iOS Shortcuts), and browser (bookmarklet).

### Integration

- **FR-20**: Expose all retrieval and ingestion capabilities via a local HTTP API on `127.0.0.1`.
- **FR-21**: Thin clients (CLI, Raycast, iOS Shortcuts, Obsidian plugin, browser extension) consume only the HTTP API — no direct database access.
- **FR-22**: Support context filtering (work/personal/all) at query time.

### Write-back

- **FR-23**: When background radar finds high-confidence matches, auto-append pointer lines to prospector.md with `[radar]` tag.
- **FR-24**: Write-back must be configurable (on/off, confidence threshold).

---

## Non-Functional Requirements

### Privacy

- **NF-1**: All components run locally on the user's machine by default.
- **NF-2**: Cloud LLM calls are opt-in per query, never default.
- **NF-3**: Cloud LLM calls must strip actor identifiers and source pointers before transmitting.
- **NF-4**: Exclusion rules (per-source, per-folder, per-actor, per-time-window) must be enforced at both index time and query time.
- **NF-5**: No telemetry, no analytics, no phone-home.

### Performance

- **NF-6**: Retrieval queries must return results in under 2 seconds for corpora under 500K items.
- **NF-7**: Keyword search (FTS5) must return in under 100ms.
- **NF-8**: Full pipeline (hybrid + re-rank + synthesis) may take up to 10 seconds when using local LLM.

### Resources

- **NF-9**: Background daemon must use less than 1% CPU average and under 300MB RAM.
- **NF-10**: Storage growth must be predictable and documented per source type.

### Format Stability

- **NF-11**: All machine-facing state stored in SQLite.
- **NF-12**: All human-facing artifacts stored in plain text / markdown.
- **NF-13**: No proprietary formats anywhere in the system.

### Trust

- **NF-14**: System never modifies, moves, or deletes source data.
- **NF-15**: Deleting the entire Steward index must leave user data untouched.
- **NF-16**: All indices, embeddings, and retrieval traces must be inspectable as plain files.

### UX

- **NF-17**: Capture must complete in under 10 seconds end-to-end.
- **NF-18**: Mac-first for v1; cross-platform desirable but not required.
- **NF-19**: Offline-capable for retrieval (cloud LLM is an enhancement, not a requirement).

---

## Source Matrix

| Source | Format | Ingestion Method | v1 Priority |
|--------|--------|-----------------|-------------|
| prospector.md | Markdown | File read, line parse | **v1** |
| Claude/ChatGPT export | JSON | Data export parse | **v1** |
| iMessage | SQLite | Direct DB read | **v1** |
| Apple Notes | SQLite | NoteStore.sqlite read | v2 |
| Browser history | SQLite | Chrome/Safari DB read | v2 |
| Email (Gmail) | IMAP | imaplib or .mbox export | v2 |
| Old Dropbox | Files | Filesystem walk + text extraction | v2 |
| Slack/Discord | JSON | Workspace export parse | v3 |
| ScreenPipe | SQLite | ScreenPipe DB read | v3 (optional) |

---

## Out of Scope for v1

- Multi-machine sync
- GUI / desktop app (CLI + API only)
- Real-time streaming ingestion (scheduled batch only)
- Audio/video transcription (defer to ScreenPipe/Whisper integration)
- Multi-user or shared corpora
- Mobile-native retrieval (capture via Shortcuts is in scope; full retrieval is not)
- Ambient / proactive surfacing (morning brief, radar passes)
- ScreenPipe integration
- Browser extension or Obsidian plugin

---

## Acceptance Criteria for v1

v1 is "done" when all of the following are true:

1. `recall "query"` returns keyword hits from prospector.md, one LLM chat export, and iMessage — in a single result set.
2. `recall "query"` supports hybrid mode (keyword + semantic) with merged ranking.
3. `recall "query" --answer` returns an LLM-synthesized answer with citations to specific source items.
4. Every citation includes a source pointer that identifies the original location.
5. `capture "one line of text"` appends to prospector.md and the item is immediately searchable.
6. The entire system runs locally with no cloud calls by default.
7. Exclusion rules prevent specified sources/actors/folders from appearing in results.
8. The index is a single SQLite file that can be deleted and rebuilt from sources.
9. `/health` endpoint reports adapter status, index size, and last ingestion time.
10. A new source adapter can be added by implementing the adapter contract (~100 lines) without modifying core code.
