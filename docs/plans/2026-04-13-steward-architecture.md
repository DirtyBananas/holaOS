# Steward: Architecture

*Technical architecture for a local-first personal memory and retrieval system.*

---

## High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                        THIN CLIENTS                                 │
│  CLI  │  Raycast/Alfred  │  iOS Shortcut  │  Obsidian  │  Browser  │
└───────────────┬─────────────────────────────────────────────────────┘
                │ HTTP (127.0.0.1:PORT)
┌───────────────▼─────────────────────────────────────────────────────┐
│                     LAYER 4: INTEGRATION                            │
│  Local HTTP API  │  Background Daemon  │  Radar / Ambient Engine    │
└───────────────┬─────────────────────────────────────────────────────┘
                │
┌───────────────▼─────────────────────────────────────────────────────┐
│                     LAYER 3: RETRIEVAL PIPELINE                     │
│  Query Rewrite → Parallel Search → Merge → Re-rank → Synthesize    │
└───────────────┬─────────────────────────────────────────────────────┘
                │
┌───────────────▼─────────────────────────────────────────────────────┐
│                     LAYER 2: INDEX                                  │
│  SQLite FTS5 (keyword)  │  sqlite-vec (semantic)  │  Embeddings DB  │
└───────────────┬─────────────────────────────────────────────────────┘
                │
┌───────────────▼─────────────────────────────────────────────────────┐
│                     LAYER 1: UNIFIED SOURCE                         │
│  Source Adapters: prospector.md │ iMessage │ Apple Notes │ Email │   │
│  Browser History │ ChatGPT/Claude exports │ Dropbox │ ScreenPipe    │
└───────────────┬─────────────────────────────────────────────────────┘
                │ reads in place
┌───────────────▼─────────────────────────────────────────────────────┐
│                     USER'S EXISTING DATA                            │
│  (untouched — Steward is a guest, not a landlord)                   │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│                     LAYER 5: TRUST & CONTROL                        │
│  (cross-cutting: exclusions, citations, inspectability, local-first)│
└─────────────────────────────────────────────────────────────────────┘
```

---

## Layer 1: Unified Source Layer

### Purpose

Pull data from every channel the user operates in, each in its native format and native location. Never copy, move, or modify source data. Normalize everything into a common item shape for downstream indexing.

### Source Adapter Contract

Every adapter is a Python module that implements one interface:

```python
class SourceAdapter:
    name: str                    # e.g. "imessage", "dropbox-old"
    
    def healthcheck(self) -> HealthStatus:
        """Verify the source is reachable and the schema is as expected.
        Return HEALTHY, DEGRADED (with reason), or UNREACHABLE."""
    
    def ingest(self, since: datetime | None) -> Iterator[RawItem]:
        """Yield items from the source, optionally filtered to items
        newer than `since` for incremental ingestion. Must not modify
        the source in any way."""
    
    def normalize(self, raw: RawItem) -> NormalizedItem:
        """Convert a source-native item into the normalized shape."""
    
    def resolve_pointer(self, pointer: str) -> str:
        """Given a source_pointer, return a clickable deep link or
        file path that navigates the user back to the original."""
```

### Source Inventory

| Source | Native Format | Access Method | Notes |
|--------|--------------|---------------|-------|
| prospector.md | Markdown | File read | One line per entry; parse `date \| source \| description` |
| iMessage | SQLite | `~/Library/Messages/chat.db` | Join `message`, `handle`, `chat` tables; full-disk-access permission required on macOS |
| Apple Notes | SQLite | `~/Library/Group Containers/group.com.apple.notes/NoteStore.sqlite` | Schema changes between macOS versions; alternatively use AppleScript export |
| Claude/ChatGPT | JSON | Data export downloads | Both offer "export my data"; parse conversation → message arrays |
| Email (Gmail) | IMAP | `imaplib` or local `.mbox` export | Thread reconstruction via `In-Reply-To` / `References` headers |
| Browser history | SQLite | Chrome: `~/Library/Application Support/Google/Chrome/Default/History`; Safari: `~/Library/Safari/History.db` | Schema is simple (url, title, visit_time) |
| Dropbox | Files | Local sync folder or Dropbox API | Walk filesystem, extract text from known file types (md, txt, pdf, docx) |
| Slack/Discord | JSON | Workspace export tools | Parse channel → message arrays |
| ScreenPipe | SQLite | ScreenPipe's local database | OCR'd text + transcriptions with timestamps |

### Normalized Item Shape

Every source adapter produces items conforming to this shape:

```json
{
  "id": "sha256:a1b2c3d4...",
  "source": "imessage",
  "timestamp": "2024-11-03T14:22:00Z",
  "actors": [
    {"role": "sender", "name": "Dan", "identifier": "+15551234567"},
    {"role": "recipient", "name": "Me", "identifier": "+15559876543"}
  ],
  "content": "Check out this article about solid-state batteries...",
  "context": {
    "thread_id": "chat789",
    "conversation_name": "Dan & Me"
  },
  "metadata": {
    "has_attachment": true,
    "attachment_type": "link"
  },
  "source_pointer": "imessage://chat789?msg=12345"
}
```

The `id` is a stable hash of `(source + timestamp + content)` so re-ingestion produces the same IDs and deduplication is trivial.

---

## Layer 2: Index Layer

### Purpose

Make the normalized corpus searchable via two complementary methods: exact keyword matching and semantic similarity. Both indices live in a single SQLite database file for portability.

### SQLite Schema

```sql
-- Core item storage
CREATE TABLE items (
    id           TEXT PRIMARY KEY,   -- stable hash
    source       TEXT NOT NULL,      -- adapter name
    timestamp    TEXT NOT NULL,      -- ISO 8601
    actors       TEXT,               -- JSON array
    content      TEXT NOT NULL,      -- searchable text
    context      TEXT,               -- JSON object
    metadata     TEXT,               -- JSON object
    source_pointer TEXT NOT NULL,    -- deep link back to original
    ingested_at  TEXT NOT NULL       -- when Steward indexed this
);

-- Full-text search (keyword)
CREATE VIRTUAL TABLE items_fts USING fts5(
    content,
    content=items,
    content_rowid=rowid,
    tokenize='porter unicode61'
);

-- Chunk storage for embeddings
CREATE TABLE chunks (
    id           INTEGER PRIMARY KEY AUTOINCREMENT,
    item_id      TEXT NOT NULL REFERENCES items(id),
    chunk_index  INTEGER NOT NULL,   -- position within the item
    chunk_text   TEXT NOT NULL,      -- the actual chunk content
    token_count  INTEGER
);

-- Vector embeddings (via sqlite-vec extension)
CREATE VIRTUAL TABLE chunk_embeddings USING vec0(
    chunk_id INTEGER PRIMARY KEY,
    embedding FLOAT[768]            -- nomic-embed-text dimension
);

-- Source registry and health
CREATE TABLE sources (
    name         TEXT PRIMARY KEY,
    adapter_type TEXT NOT NULL,
    config       TEXT,               -- JSON: paths, credentials ref, etc.
    last_ingest  TEXT,
    last_health  TEXT,               -- HEALTHY / DEGRADED / UNREACHABLE
    health_detail TEXT
);

-- Exclusion rules
CREATE TABLE exclusions (
    id           INTEGER PRIMARY KEY AUTOINCREMENT,
    scope        TEXT NOT NULL,      -- "source", "folder", "actor", "time"
    pattern      TEXT NOT NULL,      -- what to exclude
    reason       TEXT,
    created_at   TEXT NOT NULL
);

-- Indexes
CREATE INDEX idx_items_source ON items(source);
CREATE INDEX idx_items_timestamp ON items(timestamp);
CREATE INDEX idx_chunks_item ON chunks(item_id);
```

### Keyword Index

SQLite FTS5 with Porter stemming and Unicode tokenization. Handles phrase queries, prefix queries, Boolean operators. Built-in to SQLite — zero dependencies.

### Semantic Index

Items are chunked at ~500 tokens with ~50 token overlap. Each chunk is embedded using `nomic-embed-text` via Ollama (768-dimensional vectors, ~500MB model, runs on CPU or Apple Silicon). Vectors stored via `sqlite-vec` extension in the same database file.

### Why One Database File

- **Portability**: copy one file to back up or move the entire index.
- **Atomicity**: SQLite transactions keep keyword and vector indices consistent.
- **Format stability**: SQLite is the most deployed database format on earth; it will be readable in 2050.
- **Simplicity**: no database server, no configuration, no ports, no processes.

---

## Layer 3: Retrieval Pipeline

### Purpose

Turn a user's question into a cited, trustworthy answer by searching both indices, merging results, and synthesizing with an LLM.

### Pipeline Stages

```
User Query
    │
    ▼
┌──────────────────┐
│ 1. Query Rewrite │  Optional LLM call to expand/clarify
│    "Dan battery"  │  → "messages from Dan about batteries
│                   │     or solid-state battery chemistry"
└────────┬─────────┘
         │
    ┌────▼────┐
    │ PARALLEL │
    ├─────────┤
    │         │
    ▼         ▼
┌────────┐ ┌────────┐
│Keyword │ │Semantic│  Top 50 from each
│ FTS5   │ │Vec sim │
└───┬────┘ └───┬────┘
    │          │
    ▼          ▼
┌──────────────────┐
│ 3. Merge & Dedup │  Score-weighted union; items in both rank higher
│    Top ~30       │
└────────┬─────────┘
         │
         ▼
┌──────────────────┐
│ 4. Re-rank       │  LLM judges top 20 for relevance to original query
│    Top ~10       │
└────────┬─────────┘
         │
         ▼
┌──────────────────┐
│ 5. Synthesize    │  LLM answers using ONLY the top items
│    + Citations   │  Every claim cites [source_pointer]
└────────┬─────────┘
         │
         ▼
┌──────────────────┐
│ 6. Present       │  Answer + ranked source list with click-through
└──────────────────┘
```

### Retrieval Modes

| Mode | Stages Used | Use Case |
|------|-------------|----------|
| `keyword` | FTS5 only, no LLM | Fast exact lookups, names, dates, codes |
| `semantic` | Vec only, no LLM | Meaning-based recall ("that thing about burnout") |
| `hybrid` | Both + merge | Default mode — best precision |
| `answer` | Full pipeline | Natural-language Q&A with citations |

### Technology

- **Query rewrite**: single LLM call, optional, skipped for simple queries
- **FTS5 search**: SQLite built-in, sub-millisecond on typical corpus sizes
- **Vector search**: `sqlite-vec` cosine similarity, fast for <1M vectors
- **Re-rank**: LLM call (local Ollama or cloud API) judging relevance
- **Synthesis**: LLM call with retrieved items injected as context
- **Local LLM**: Ollama running Llama 3.x (8B parameter model, runs on laptop)
- **Cloud LLM**: Claude API, opt-in per query, indicated in the response

---

## Layer 4: Integration Layer

### Purpose

This is Steward's novel contribution. Expose retrieval everywhere the user actually works, via a local HTTP API consumed by thin clients. Plus a background daemon for proactive surfacing.

### Local HTTP API

All endpoints on `127.0.0.1:{PORT}` (default 7777). JSON request/response.

```
POST /recall
  Body: { "query": "...", "mode": "hybrid|keyword|semantic|answer",
          "context_filter": "work|personal|all", "limit": 10 }
  Returns: { "answer": "...", "items": [...], "citations": [...] }

POST /capture
  Body: { "content": "...", "source": "manual", "metadata": {} }
  Appends a line to prospector.md and indexes it immediately.

GET /sources
  Returns: list of registered source adapters with health status.

POST /ingest
  Body: { "source": "imessage", "since": "2024-01-01" }
  Triggers incremental ingestion for a specific source.

POST /ingest/all
  Triggers incremental ingestion across all registered sources.

GET /health
  Returns: system health (index size, last ingest times, daemon status).

GET /exclusions
  Returns: current exclusion rules.

POST /exclusions
  Body: { "scope": "actor", "pattern": "boss@work.com", "reason": "..." }
  Adds an exclusion rule; affected items are removed from index.

DELETE /exclusions/{id}
  Removes an exclusion rule; affected items are re-indexed.

GET /brief
  Returns: today's pre-computed radar results (morning brief).

GET /stats
  Returns: item counts by source, index size, embedding coverage.
```

### Thin Clients

Each client is a minimal wrapper around the HTTP API:

| Client | Implementation | Primary Use |
|--------|---------------|-------------|
| `recall` CLI | Python script, ~100 lines | Terminal power users; scripting |
| Raycast/Alfred | Script command hitting `/recall` | Quick lookup from anywhere on Mac |
| iOS Shortcut | HTTP action to Steward API (requires local network or Tailscale) | Mobile capture via share sheet + recall |
| Obsidian plugin | JS plugin calling `/recall`, renders results in sidebar | Context while writing |
| Browser extension | Content script + popup calling `/recall` | "Find related" on any web page |

### Background Daemon

A long-running process (or scheduled cron) that:

1. **Scheduled ingestion**: re-ingests sources on a configurable cadence (e.g., iMessage every hour, browser history daily, Dropbox weekly).
2. **Radar passes**: runs pre-configured queries against the corpus:
   - "Items from this week worth remembering" → appends pointers to prospector.md
   - "Items matching today's calendar events" → populates the morning brief
   - "Old notes related to recently active projects" → surfaces forgotten context
3. **Write-back**: when a radar pass finds high-confidence matches, auto-appends a pointer line to `prospector.md` with `[radar]` tag.
4. **Health monitoring**: runs adapter healthchecks, logs warnings when a source degrades.

---

## Layer 5: Trust and Control

### Purpose

Cross-cutting layer that ensures the system earns and keeps the user's trust. Not a component — a set of invariants enforced everywhere.

### Invariants

1. **Citations always.** Every answer from the retrieval pipeline includes `source_pointer` references. The LLM is never allowed to answer without injected context. No hallucinated memories.

2. **Exclusion enforcement.** Exclusion rules are checked at index time (items matching an exclusion are never stored) AND at query time (belt and suspenders). Exclusions are absolute — no "unless the query is really relevant" overrides.

3. **Local by default.** The system runs entirely on the user's machine. Cloud LLM calls (for re-rank or synthesis) are:
   - Opt-in per query (flag in the API request)
   - Visually indicated in the response ("answered via cloud model")
   - Never the default path

4. **Non-destructive.** Steward never writes to, modifies, moves, or deletes source data. It only reads. The entire Steward index can be deleted and the user's data is exactly as it was. The index is disposable; the sources are sacred.

5. **Inspectable.** The SQLite database is a file anyone can open with `sqlite3` or DB Browser. Embeddings are queryable. Retrieval traces (which items were considered, how they scored, why they were ranked) are logged to a `traces/` directory as JSON. No black boxes.

---

## Technology Choices

| Component | Choice | Rationale |
|-----------|--------|-----------|
| Language | Python | Fastest iteration for a solo developer; rich ecosystem for NLP/ML; acceptable performance for a personal tool |
| Database | SQLite | One-file portability, decades-stable format, FTS5 + sqlite-vec in one process, no server |
| Keyword search | FTS5 | Built into SQLite, Porter stemming, phrase/prefix/Boolean, zero dependencies |
| Vector search | sqlite-vec | SQLite extension, same file as FTS5, cosine similarity, good enough for <1M vectors |
| Embeddings | nomic-embed-text via Ollama | Free, local, ~500MB, 768-dim, strong quality for its size |
| Local LLM | Llama 3.x via Ollama | Free, local, good enough for re-rank and synthesis on a laptop |
| Cloud LLM | Claude API (opt-in) | Higher quality for hard queries; pay-per-use; never default |
| Audio transcription | whisper.cpp | Free, local, state-of-the-art quality, C++ so fast on CPU |
| Screen capture | ScreenPipe (external) | Open source, cross-platform, SQLite-based, designed as an engine to build on; we consume it as a source adapter, we do NOT rebuild it |
| HTTP framework | FastAPI | Lightweight, async, auto-generated OpenAPI docs, Python-native |
| Config | YAML file | Human-readable, editable, version-controllable; lives at `~/.steward/config.yaml` |
| Artifacts | Markdown + SQLite | Plain text for human-facing outputs (prospector.md, briefs); SQLite for machine-facing state (index, embeddings) |

---

## Data Flow

```
Source Data (untouched)
    │
    │  Source Adapter reads in place
    ▼
Normalized Items
    │
    ├──► items table (SQLite)
    ├──► items_fts (FTS5 keyword index)
    └──► chunks → chunk_embeddings (sqlite-vec semantic index)
              │
              │  User or daemon issues query
              ▼
         Retrieval Pipeline
              │
              │  Hybrid search → merge → re-rank → synthesize
              ▼
         Cited Answer
              │
              ├──► Thin Client (CLI, Raycast, Obsidian, etc.)
              └──► Write-back to prospector.md (if radar pass)
```

---

## Cross-Cutting Concerns

### Logging

All ingestion runs, retrieval queries, and daemon actions are logged to `~/.steward/logs/` as structured JSON (one file per day, rotated). Logs include timing, item counts, errors, and health status. Logs never contain the full content of items — only IDs and metadata.

### Error Handling

- **Adapter failures**: logged, surfaced via `/health`, do not block other adapters. An adapter that fails 3 consecutive healthchecks is marked UNREACHABLE and skipped until manually re-enabled.
- **Embedding failures**: individual items that fail to embed are logged and skipped; they remain keyword-searchable but not semantically indexed.
- **LLM failures**: if the local LLM is unavailable, retrieval degrades gracefully to hybrid search without re-rank/synthesis (returns raw items instead of a prose answer).

### Privacy

- Exclusion rules enforced at both index and query time.
- No telemetry, no analytics, no phone-home.
- Cloud LLM calls strip `actors` and `source_pointer` fields before sending, transmitting only the `content` needed for synthesis.
- The SQLite database should live in an encrypted volume (FileVault on macOS, LUKS on Linux) — Steward assumes disk encryption but does not implement it.

### Configuration

```yaml
# ~/.steward/config.yaml
port: 7777
db_path: ~/.steward/steward.db
log_dir: ~/.steward/logs
prospector_path: ~/Documents/prospector.md

sources:
  imessage:
    enabled: true
    path: ~/Library/Messages/chat.db
    schedule: hourly
  apple_notes:
    enabled: true
    schedule: daily
  browser_history:
    enabled: true
    browsers: [chrome, safari]
    schedule: daily

embeddings:
  model: nomic-embed-text
  chunk_size: 500
  chunk_overlap: 50

llm:
  local_model: llama3.2
  cloud_provider: anthropic  # opt-in per query
  cloud_model: claude-sonnet-4-20250514

radar:
  enabled: true
  schedule: "0 7 * * *"  # 7am daily
  queries:
    - "items from this week worth remembering"
    - "old notes related to active projects"
  write_back: true
```

---

## Explicit Non-Goals

Things this architecture does **not** attempt to solve:

1. **Cross-platform screen capture.** ScreenPipe handles this. We consume its output via a source adapter.
2. **Multi-machine sync.** Deferred. The current design is single-machine. Sync strategies (git, iCloud, Syncthing) are an open question for a later phase.
3. **Multi-user / shared corpora.** Steward is a personal tool for one human. No auth, no permissions model, no sharing.
4. **Real-time streaming ingestion.** Adapters run on a schedule (hourly, daily). Sub-minute freshness is not a goal.
5. **A polished GUI.** The product is the API and the thin clients. If a GUI emerges, it's a thin client like the others.
6. **Replacing any existing tool.** Steward does not want to be your notes app, your email client, or your chat app. It reads from all of them and owns none of them.
7. **Production-grade security hardening.** This runs on localhost for one user. TLS, auth tokens, and rate limiting are unnecessary overhead.
