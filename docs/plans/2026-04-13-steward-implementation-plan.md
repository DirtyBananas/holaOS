# Steward Implementation Plan

*A phased build sequence for Steward, a local-first personal memory and retrieval system.*

**Date:** 2026-04-13
**Owner:** DirtyBananas/Steward (solo developer)
**Status:** Planning

---

## Guiding Principles

Steward is being built by one person, for one person. The plan below is organized around three ideas that override any particular technology choice.

### 1. Earned progression

Each phase must earn the next. Do not start Phase N+1 until Phase N has stuck as a habit or actually demonstrated value in daily use. The ordering of phases matters more than the specific tools inside each phase; skipping ahead is how beautiful unused systems get built.

### 2. Validation gates

Every phase ends with a concrete validation criterion, not a time budget. If the gate is not met, the correct response is to either iterate on that phase or stop. Continuing upward on an unvalidated foundation is the single biggest failure mode for projects like this.

### 3. Useful at every step

Stopping after any phase must leave the user meaningfully better off than not starting. Phase 0 alone (a plain text log) is already a productivity win. The system should never require completion to deliver value.

---

## Phase 0: The Prospector Log

**Goal:** Prove that the underlying habit — noticing and noting things worth remembering — can be sustained before any code is written.

**Deliverables:**

- A single file: `prospector.md` in an iCloud/Dropbox folder, or inside an Obsidian vault.
- A personal working line format, committed to memory:
  `date | source/location | one sentence about what's there and why future-self cares`
- No tags, no folders, no categories. Categories are a trap at this stage.

**Effort estimate:** ~10 minutes to set up.

**Tech stack:** None. A text editor and a sync folder.

**Validation gate:**

- [ ] At least 5 entries per week for 2 consecutive weeks.
- [ ] Logging feels like jotting, not like bookkeeping.
- [ ] At least one entry per week that the user returned to and found useful.

**Risks specific to this phase:**

- Over-formatting: adding tags/categories/YAML front-matter before the habit has proven itself.
- Tool-shopping: evaluating note apps instead of writing lines.
- Premature automation: scripting anything at all during the validation window.

If the gate is not met after 2 weeks, do not proceed. The whole system rests on this habit being real.

---

## Phase 1: Frictionless Capture

**Goal:** Reduce capture latency to under 10 seconds end-to-end, from anywhere the user already works. Don't add new surfaces; wire up the ones already in use.

**Deliverables:**

- One-hotkey capture path on the main machine:
  - Mac: Raycast or Alfred script command that appends a line to `prospector.md`.
  - Cross-platform fallback: shell alias (e.g. `pl "<note>"`) that appends with timestamp.
- iOS Shortcut triggered from the share sheet that appends to the same iCloud-synced file.
- Browser bookmarklet that grabs the current tab's URL + title and appends a line.
- Hard cap: any capture path must complete in under 10 seconds, measured.

**Effort estimate:** 1 weekend.

**Tech stack:** Raycast or Alfred (existing installs preferred), iOS Shortcuts, a one-line bookmarklet, a shell alias. No databases, no servers.

**Validation gate:**

- [ ] The hotkey gets used reflexively (not consciously-decided) for at least 1 week.
- [ ] The log grows without the user thinking about "logging."
- [ ] At least one capture per day happens from a non-desktop context (phone or browser bookmarklet).

**Risks specific to this phase:**

- Building a capture UI before validating that the existing surfaces are the bottleneck.
- Letting the iOS path drift to a different file format than the desktop path. Enforce one format.
- Sync conflicts in iCloud from simultaneous writes. Acceptable at this volume; revisit only if it bites.

---

## Phase 2: Ingestion Pipeline

**Goal:** Prove that retrieval across multiple sources is achievable and useful on a small, honest footprint before scaling sources or sophistication.

**Deliverables:**

- A Python ingestion script (`steward_ingest.py`) that reads from exactly three sources:
  1. `prospector.md` — parse lines into items.
  2. One exported LLM chat history (Claude or ChatGPT JSON export) — parse messages into items.
  3. iMessage local SQLite (`~/Library/Messages/chat.db`) — read messages via SQLite directly.
- A normalized item shape shared by every source:
  `{id, source, timestamp, author, body, uri, raw}`
- A single SQLite database (`steward.db`) with an FTS5 virtual table over `body`.
- A CLI: `recall "<query>"` that returns ranked hits across sources with source, timestamp, and a snippet.

**Effort estimate:** 1 weekend.

**Tech stack:** Python 3, `sqlite3` stdlib, FTS5 (built into SQLite), `click` or `typer` for the CLI. No servers, no embeddings yet.

**Validation gate:**

- [ ] The user can, from the terminal, recall something they remember seeing in iMessage by typing one keyword.
- [ ] `recall` returns results in under 1 second for a realistic DB.
- [ ] Re-running ingestion is idempotent (no duplicates).

**Risks specific to this phase:**

- Scope creep: adding a fourth source "since we're already here." Don't.
- Schema thrash: the normalized shape must be stable or every later phase re-migrates. Write it down once, commit, and defend it.
- iMessage schema drift: Apple has changed `chat.db` internals historically. Pin a parsing function and test on one real export.

---

## Phase 3: Semantic Search

**Goal:** Handle queries that keyword search cannot — fuzzy recall, conceptual queries, rephrasings.

**Deliverables:**

- Ollama installed locally with `nomic-embed-text` model pulled.
- `sqlite-vec` extension installed and loaded in Steward's DB.
- Chunking: items split into ~500-token chunks with modest overlap (~50 tokens). Chunk rows linked back to parent items.
- Embedding step in ingestion: each new chunk embedded and stored in a vec-indexed table.
- New CLI: `recall-semantic "<query>"` returning top-K by cosine similarity.
- Hybrid `recall`: merges FTS5 hits and semantic hits into a single score-weighted union, de-duplicated by item id.

**Effort estimate:** 1 weekend.

**Tech stack:** Ollama, `nomic-embed-text`, `sqlite-vec`, a small chunker in Python. No new data stores.

**Validation gate:**

- [ ] A query that obviously wouldn't work via keyword — e.g. *"that thing about feeling burned out last fall"* — returns the right items in the top 5.
- [ ] Hybrid `recall` is strictly >= keyword-only on a handful of saved "benchmark" queries maintained by the user.
- [ ] Embedding the full current corpus completes in a reasonable time (minutes, not hours) on the user's machine.

**Risks specific to this phase:**

- Chunk boundary bugs (losing context at splits). Keep the chunker dumb and overlapping; fancy boundary detection is not the win here.
- Score fusion that silently degrades keyword precision. Sanity-check hybrid against keyword on queries that should be exact matches.
- Pulling heavier embedding models "because they're better" and blowing the local inference budget.

---

## Phase 4: LLM-Assisted Retrieval

**Goal:** Turn Steward from a search box into something the user can ask questions of — with citations so trust is auditable.

**Deliverables:**

- Re-rank step: pass the top-20 hybrid candidates to a local LLM which scores each for relevance to the query; take the top-5.
- Answer synthesis step: a local LLM answers the query in prose, strictly using only the top-5 re-ranked items as context.
- **Mandatory citations**: every factual claim in the answer includes a source pointer (item id, source name, timestamp).
- Default model: local Ollama LLM (e.g. Llama 3.1 8B or similar). Opt-in per query to a cloud LLM (Claude API) via a `--cloud` flag for harder questions.
- A simple local HTTP API (FastAPI or similar) that exposes `search`, `ask`, and `ingest` endpoints — groundwork for Phase 6 integration clients.

**Effort estimate:** 1 weekend.

**Tech stack:** Ollama (with a chat model), optional Anthropic API, FastAPI, `httpx`. No new storage.

**Validation gate:**

- [ ] The user asks a real question they actually care about.
- [ ] The answer is one they trust.
- [ ] Every claim is independently verifiable via the included citations.

**Risks specific to this phase:**

- Hallucination without citation discipline. Citations must be generated by code from the retrieved set, not written by the LLM.
- Letting the LLM rewrite the item content in ways that drift from the source. Answer synthesis should quote, not paraphrase without markers.
- Cloud-LLM creep: defaulting to cloud for "quality." Keep the local default honest.

---

## Phase 5: More Source Adapters

**Goal:** Extend coverage of the user's channel inventory. Incremental. Ongoing.

**Deliverables:**

- A documented **source adapter contract**: a Python function taking `(config, since_timestamp)` and yielding normalized items.
- New adapters, added one at a time, only when the absence of that source is an actual pain point.
- Each adapter: target ~100 lines; each gets a small fixture test with a sample row.

**Effort estimate:** Ongoing. ~half a day per source.

**Tech stack:** Whatever each source requires (SQLite, IMAP, HTTP APIs, filesystem walks). Nothing new for Steward core.

**Validation gate (per adapter):**

- [ ] At least one real successful recall hit originating from that source within a week of adding it.
- [ ] The adapter is idempotent and incremental (uses `since_timestamp`).

**Risks specific to this phase:**

- Pre-building adapters for sources the user doesn't actually use. Don't.
- Leaking source-specific fields into the normalized shape. Source-specific data goes into `raw`.
- Letting one big adapter (Dropbox walk) derail the weekly rhythm. Box each adapter inside one sitting.

### Source adapter rollout order

The ordering below reflects the user's stated channel inventory and the gradient from low-effort to high-effort.

1. **Apple Notes (`NoteStore.sqlite`)** — already indexed by the OS, local, high-signal, low-effort.
2. **Browser history (Chrome/Safari local DB)** — massive corpus of things the user actually looked at; pure SQLite read.
3. **Email via IMAP (Gmail first)** — high-value channel for work context; IMAP is a known quantity.
4. **Old Dropbox (API + filesystem walk)** — the "treasure trove" case; higher effort but biggest payoff once the system is trustworthy.
5. **Slack/Discord exports** — useful but conversational noise is high; wait until re-rank is good.
6. **Other LLM chat exports** — lowest urgency because current LLM chats are increasingly captured inline already.

---

## Phase 6: Integration Points

**Goal:** Multiply the value of everything underneath by putting Steward at the user's fingertips wherever they already work.

**Deliverables:**

- Each client is a thin caller of the local HTTP API from Phase 4.

**Effort estimate:** Ongoing. ~half a day per integration.

**Tech stack:** Whatever the target surface requires. Steward core does not change.

**Validation gate (per client):**

- [ ] That client becomes the user's primary way to invoke Steward in its context within a week.
- [ ] If it doesn't, remove it — clutter is a cost.

### Integration client rollout order

1. **Terminal alias** — already in place from Phases 2–4. Baseline.
2. **Raycast/Alfred command** — biggest daily-usage unlock on the Mac because the launcher is already a reflex.
3. **iOS Shortcut for mobile recall** — pairs with the capture Shortcut from Phase 1; unlocks on-the-go recall.
4. **Obsidian plugin** — recall from inside any note; closes the loop between note-taking and memory.
5. **Browser extension** — right-click "find related on this page"; highest potential surprise value, highest build cost, so last.

---

## Phase 7: Ambient / Proactive Layer

**Goal:** Have Steward surface things without being asked. Highest leverage, also highest risk of over-engineering, so it comes last.

**Deliverables:**

- A background daemon running scheduled queries:
  - **Morning brief:** top 5 items Steward thinks are relevant to today (based on calendar and recent activity).
  - **Weekly digest:** a retrospective across the past 7 days with notable items.
  - **Calendar cross-reference:** before a meeting, surface items about the attendees or topic.
- Auto write-back: when the daemon finds something genuinely valuable, it appends a pointer line to `prospector.md` so the loop stays coherent.
- Optional ScreenPipe integration for context-aware surfacing based on what is currently on screen.

**Effort estimate:** 1–2 weekends for the first cut, then ongoing tuning.

**Tech stack:** A scheduler (`launchd` on macOS, cron fallback), the existing local HTTP API, email/notification delivery.

**Validation gate:**

- [ ] At least one proactive surfacing per week is judged "glad I saw that" by the user.
- [ ] Noise rate (surfacings judged not useful) is below 50%.
- [ ] The daemon can be silenced with one command and the rest of Steward still works normally.

**Risks specific to this phase:**

- Notification fatigue. The daemon must degrade gracefully and stay quiet by default.
- Over-fitting on early feedback into tuning knobs that nobody wants to manage.
- Coupling the daemon so tightly that Phase 2–6 break if it does. Keep it a pure client of the HTTP API.

---

## Optional / Parallel Tracks

### Passive capture via ScreenPipe

Runs in parallel to Phase 4 and beyond, not instead of it.

- Install ScreenPipe as an optional capture layer on the main machine.
- Add a Steward source adapter (per the Phase 5 contract) that reads from ScreenPipe's local SQLite.
- Treat passive capture as a **luxury safety net** underneath deliberate capture, not as a replacement for it. Deliberate capture is what proves the habit; passive capture mops up what slipped through.
- Do not start this track until Phase 1 is validated, and do not use it to excuse skipping Phase 1.

### Ambient daemon track

Technically the same as Phase 7; listed here only as a reminder that it is separable from Phases 0–6 and should never block them.

---

## Anti-patterns to Avoid

- **Skipping validation gates.** The prospector log habit is more important than any code you can write. If the habit isn't there, more code won't summon it.
- **Ingesting everything before retrieval works.** Three sources in Phase 2 is enough. Breadth is earned with retrieval quality, not assumed.
- **Proprietary formats.** Plain text (`prospector.md`) plus SQLite (`steward.db`). If the user can't `cat` or `sqlite3` their memory, trust evaporates.
- **Automated capture before deliberate capture is a habit.** Passive pipes feeding a dead habit create dead data.
- **Skipping citations.** Every LLM-generated answer must include source pointers. The first uncited claim the user can't verify collapses trust in the whole system.
- **Pre-building adapters.** Add sources when a gap hurts, not when a list exists.
- **Rebuilding existing tools.** Steward integrates with what the user already uses. If a phase starts feeling like "a better Obsidian," stop.
- **Tuning over using.** Until Phase 7, the correct reaction to "the ranking isn't perfect" is to use it more, not to tune harder.
- **Cloud-first drift.** Local by default, cloud opt-in, never the reverse.

---

## Summary Table

| Phase | What it is | Effort | Validation gate |
|------|------------|--------|------------------|
| 0 | Prospector log (no code) | 10 min | 5+/week habit for 2 weeks |
| 1 | Frictionless capture | 1 weekend | Reflexive hotkey use |
| 2 | Ingestion + FTS5 `recall` | 1 weekend | Find an iMessage thing by keyword |
| 3 | Hybrid semantic search | 1 weekend | Fuzzy query returns the right item |
| 4 | LLM re-rank + answer + API | 1 weekend | Trusted, cited answer to a real question |
| 5 | Source adapters (ongoing) | ~0.5 day each | Real recall hit from new source within a week |
| 6 | Integration clients (ongoing) | ~0.5 day each | Client becomes primary in its context |
| 7 | Ambient/proactive layer | 1–2 weekends + tuning | Weekly "glad I saw that" surfacing |

---

## Closing Note

The point of this plan is not to be finished. The point is that at every pause — after a weekend, after a month, after a year — Steward is a useful thing and not a half-built thing. If the developer stops at Phase 2, they still have fast keyword recall across three sources. If they stop at Phase 4, they have a trustworthy local question-answering memory. Every additional phase is a multiplier on a foundation that is already carrying its own weight.

Pin this document to the wall. Advance one phase at a time. Earn the next.
