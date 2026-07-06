# Steward: Implementation Plan

*Phased build sequence for a local-first personal memory and retrieval system. Each phase is useful on its own and earns the right to the next.*

---

## Guiding Principles

1. **Earned progression.** Do not build Phase N+1 until Phase N has stuck as a habit. Beautiful tools that go unused are worse than no tools — they add guilt to the problem.
2. **Useful at every step.** Stopping after any phase must leave the user better off than not starting. No phase is scaffolding for a future phase.
3. **Validation gates.** Every phase has a specific, testable criterion. If the gate is not met, the phase is not done — and the next phase is not started.
4. **Plain text and SQLite only.** No proprietary formats at any phase. Everything the system produces is readable in 2050.
5. **Don't pre-build.** Add sources and integrations when the pain of their absence becomes acute, not before.

---

## Phase 0: The Prospector Log

**Goal:** Establish the foundational capture habit before writing any code.

**Deliverables:**
- A single file: `prospector.md` in an iCloud/Dropbox-synced folder or Obsidian vault
- Format: `date | source/location | one sentence about what's there and why future-self cares`
- No tags, no folders, no categories

**Effort:** 10 minutes

**Tech stack:** A text editor. Nothing else.

**Validation gate:** 5+ entries per week for 2 consecutive weeks, without it feeling like work.

**Risks:**
- The habit doesn't stick because the file is hard to reach → put it on the home screen / dock
- Over-engineering the format from day one → resist; three fields is the ceiling

---

## Phase 1: Frictionless Capture

**Goal:** Reduce capture friction to under 10 seconds from anywhere.

**Deliverables:**
- [ ] One-hotkey capture on the main machine (Raycast script, Alfred workflow, or shell alias)
- [ ] iOS Shortcut that prompts for text and appends to the synced prospector.md
- [ ] Browser bookmarklet that grabs current URL + title and appends

**Effort:** 1 weekend

**Tech stack:** Raycast/Alfred (Mac), iOS Shortcuts, a bash one-liner

**Validation gate:** The hotkey is used reflexively — the log grows without conscious effort.

**Risks:**
- Sync conflicts on the file if edited from multiple devices simultaneously → use iCloud or Dropbox (both handle append-only single-file sync well enough)
- Over-building the capture tool instead of keeping it to one hotkey → stop at one hotkey

---

## Phase 2: Ingestion Pipeline

**Goal:** Make three sources searchable from one CLI command.

**Deliverables:**
- [ ] Python script with three source adapters: prospector.md, one LLM chat export (Claude or ChatGPT JSON), iMessage SQLite
- [ ] Normalization into the common item shape
- [ ] SQLite database with FTS5 keyword index
- [ ] CLI command: `recall "query"` returns keyword hits across all three sources with source pointers

**Effort:** 1 weekend

**Tech stack:** Python 3.11+, sqlite3 (stdlib), click or argparse for CLI

**Validation gate:** User can find something they remembered seeing in iMessage by typing one keyword command. The moment this works, the system is real.

**Risks:**
- iMessage database requires Full Disk Access permission on macOS → document the permission grant in setup
- LLM chat export format varies between providers → start with one, add the other later
- Scope creep into "let me also add email and notes" → resist; three sources only

---

## Phase 3: Semantic Search

**Goal:** Find things by meaning, not just keywords.

**Deliverables:**
- [ ] Ollama installed with nomic-embed-text model
- [ ] Chunking pipeline (~500 token chunks, ~50 token overlap)
- [ ] sqlite-vec extension integrated for vector storage
- [ ] CLI: `recall "query"` now runs hybrid search (keyword + semantic, score-weighted merge)

**Effort:** 1 weekend

**Tech stack:** Ollama, nomic-embed-text, sqlite-vec Python bindings

**Validation gate:** A query that would fail via keywords alone (e.g., "that thing about feeling burned out last fall") returns the right items via semantic similarity.

**Risks:**
- Ollama installation friction on some machines → document prerequisites clearly
- Embedding all existing items takes time on first run → show progress, make it resumable
- sqlite-vec version compatibility with system SQLite → pin versions, test on fresh machine

---

## Phase 4: LLM-Assisted Retrieval

**Goal:** Get prose answers with citations, not just ranked item lists.

**Deliverables:**
- [ ] Re-rank step: LLM judges top-20 hybrid results for relevance
- [ ] Synthesis step: LLM answers in prose using only the top items as context
- [ ] Citations in every answer linking back to source_pointer
- [ ] `recall "query" --answer` flag for full pipeline; default remains hybrid without LLM
- [ ] Optional `--cloud` flag for higher-quality answers via Claude API (opt-in, never default)

**Effort:** 1 weekend

**Tech stack:** Ollama with Llama 3.x for local inference; anthropic Python SDK for optional cloud path

**Validation gate:** User asks a real question they care about, gets a trustworthy answer, and can verify it via the citation.

**Risks:**
- Local LLM quality may disappoint on complex queries → cloud fallback exists but must remain opt-in
- Re-rank + synthesis adds latency (~5-10s with local LLM) → keep the fast non-LLM path as default
- Citations can be wrong if the LLM hallucinates source references → structured prompting with source IDs injected, validate citations post-synthesis

---

## Phase 5: More Source Adapters

**Goal:** Expand the corpus as specific gaps become painful.

**Deliverables:** One new adapter per pain point, conforming to the source adapter contract.

**Rollout order** (based on likely pain frequency):
1. Apple Notes (NoteStore.sqlite) — high volume of personal notes
2. Browser history (Chrome/Safari SQLite) — "where did I read that?"
3. Email via IMAP (Gmail first) — work and personal communication
4. Old Dropbox (filesystem walk + text extraction) — the "treasure trove" use case
5. Slack/Discord exports — team communication history
6. Additional LLM chat exports — other AI chat tools

**Effort:** ~half a day per adapter (~100 lines each)

**Tech stack:** Same Python + SQLite stack; source-specific libraries as needed (imaplib for email, etc.)

**Validation gate per adapter:** A query that should hit the new source returns correct results with proper source pointers.

**Risks:**
- Source schema changes between OS versions (especially Apple Notes, iMessage) → healthcheck in every adapter; fail loudly
- Temptation to add all adapters at once → add one, use it for a week, then add the next

---

## Phase 6: Integration Points

**Goal:** Make retrieval accessible from everywhere the user works, not just the terminal.

**Deliverables:** Thin clients consuming the local HTTP API.

**Rollout order:**
1. Local HTTP API (FastAPI on 127.0.0.1:7777) — foundation for all clients
2. Raycast/Alfred command — quick lookup from anywhere on Mac
3. iOS Shortcut for mobile recall — extends capture to retrieval on phone
4. Obsidian plugin — recall from inside any note
5. Browser extension — "find related" on any web page

**Effort:** ~half a day per integration point

**Tech stack:** FastAPI (Python), Raycast script commands, iOS Shortcuts HTTP actions, Obsidian plugin API (JS), browser extension (JS)

**Validation gate per client:** User reaches for the new client instead of opening a terminal.

**Risks:**
- Local HTTP API requires the daemon to be running → launchd / systemd service registration
- iOS recall requires the phone to reach localhost → Tailscale or local network; document the options
- Too many integration points become their own fragmentation → only build the ones you actually use

---

## Phase 7: Ambient / Proactive Layer

**Goal:** The system surfaces valuable context without being asked.

**Deliverables:**
- [ ] Background daemon with scheduled ingestion (configurable per source: hourly, daily, weekly)
- [ ] Radar passes: pre-configured queries that run on schedule ("items from this week worth remembering", "old notes matching current projects")
- [ ] Morning brief: daily digest accessible via `GET /brief` or written to a file
- [ ] Write-back: radar results auto-appended to prospector.md with `[radar]` tag
- [ ] Optional ScreenPipe integration for context-aware surfacing

**Effort:** 1-2 weekends for core; ongoing tuning

**Tech stack:** APScheduler or cron, same Python stack, optional ScreenPipe adapter

**Validation gate:** The morning brief contains at least one item the user didn't remember but finds valuable, in the first week of use.

**Risks:**
- Over-presenting → high precision threshold; pull mode (digest) not push mode (notifications)
- The daemon becomes a resource hog → strict CPU/memory budget; throttle by battery state
- Write-back pollutes the prospector log with low-quality entries → tunable confidence threshold; separate `[radar]` tag for filtering

---

## Optional Track: Passive Capture via ScreenPipe

**When:** Anytime after Phase 4, in parallel with other work.

**What:** Install ScreenPipe, add a source adapter that reads its local SQLite database (OCR'd screen text + audio transcriptions). Now "everything you saw on screen" is part of the searchable corpus.

**Why optional:** Passive capture without good retrieval creates the exact volume problem Steward is designed to solve. The retrieval pipeline (Phases 2-4) must work first.

**Effort:** 1 weekend (ScreenPipe install + adapter)

---

## Anti-Patterns to Avoid

1. **Don't skip validation gates.** The prospector log habit is more important than any code. If Phase 0 doesn't stick, nothing else matters.
2. **Don't pre-build adapters.** Add sources when their absence hurts, not before.
3. **Don't use proprietary formats.** Not for "just this one thing." Not for "temporary" storage. Plain text and SQLite, always.
4. **Don't add passive capture before deliberate capture is a habit.** Automated logging without a working retrieval layer creates a pile, not a memory.
5. **Don't skip citations.** Trust collapses without provenance. Every answer cites its sources, even in the CLI.
6. **Don't build a GUI.** The product is the API and the thin clients. If you're designing screens, you've drifted.
7. **Don't merge work and personal without thinking.** Tag items with context at ingestion time; filter at query time. Contamination in front of colleagues is a trust-breaker.

---

## Summary Table

| Phase | Goal | Effort | Key Deliverable | Gate |
|-------|------|--------|----------------|------|
| 0 | Capture habit | 10 min | prospector.md | 5+ entries/week for 2 weeks |
| 1 | Frictionless capture | 1 weekend | One-hotkey append | Reflexive use |
| 2 | Searchable corpus | 1 weekend | `recall` CLI + 3 sources + FTS5 | Keyword lookup works |
| 3 | Semantic search | 1 weekend | Embeddings + hybrid retrieval | Meaning-based query works |
| 4 | LLM answers | 1 weekend | Re-rank + synthesis + citations | Trusted cited answer |
| 5 | More sources | Ongoing | One adapter at a time | Each new source returns results |
| 6 | Everywhere access | Ongoing | HTTP API + thin clients | Clients used instead of terminal |
| 7 | Proactive | 1-2 weekends | Daemon + radar + morning brief | Brief surfaces forgotten value |
