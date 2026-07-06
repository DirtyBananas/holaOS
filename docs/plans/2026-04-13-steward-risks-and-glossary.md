# Steward: Risks, Open Questions, and Glossary

*The honest reckoning. What could go wrong, what is undecided, and what the words mean.*

---

## Risks

### 1. Automation Creates More Noise Than It Removes

**Description:** The original problem is artifact overload. Naive passive capture (ScreenPipe, automated logging) without good retrieval produces more data without making any of it findable — a strictly worse version of the status quo.

**Likelihood:** High (if sequencing is wrong)
**Impact:** High — the user abandons the system because it recreated the problem it was supposed to solve.
**Mitigation:** Enforce earned progression. Deliberate capture (prospector log) first, automated capture only after retrieval pipeline (Phases 2-4) is working and trusted.

### 2. Assembly Itself Becomes a New Burden

**Description:** Assembling 5+ tools (Ollama, sqlite-vec, FastAPI, ScreenPipe, adapters) into a working pipeline is itself a project. If the build takes months and the system is not usable until everything is wired up, the builder gets demoralized and quits.

**Likelihood:** Medium
**Impact:** High — the system is never finished.
**Mitigation:** Phased rollout with each phase useful on its own. Phase 2 alone (3 sources + keyword search) is a functional product. Stop-points are real stop-points.

### 3. Privacy / Trust Failure

**Description:** Indexing iMessages, emails, and screen captures requires deep trust. A misconfigured exclusion rule, a leaked index file, or an accidental cloud LLM call with sensitive content could destroy that trust irreversibly.

**Likelihood:** Low (with discipline)
**Impact:** Critical — once trust is broken, the user will not re-enable sensitive sources.
**Mitigation:** Local-first by default. Exclusions enforced at both index and query time. Cloud calls strip actor/pointer fields. No telemetry. Index is disposable; source data is untouched.

### 4. Retrieval Quality Gap

**Description:** Naive RAG over personal data returns mediocre results without significant tuning. Embeddings may miss context. Re-ranking may be noisy. The user asks a real question, gets a bad answer, and stops trusting the system.

**Likelihood:** Medium
**Impact:** High — bad retrieval makes the system useless despite good ingestion.
**Mitigation:** Hybrid search (keyword catches what semantic misses and vice versa). LLM re-ranking for precision. Citations so the user can always verify. Iterative tuning over time.

### 5. Source Format Breakage

**Description:** Apple Notes schema changes between macOS versions. iMessage database schema evolves. IMAP quirks vary by provider. LLM export formats change without notice. An adapter that worked on macOS 14 may silently fail on macOS 15.

**Likelihood:** High (over a multi-year horizon)
**Impact:** Medium — missing data from one source, not total system failure.
**Mitigation:** Healthcheck in every adapter. Adapters fail loudly (logged, surfaced in `/health`) rather than silently returning empty results. Schema expectations documented per adapter.

### 6. Format Lock-In Over Time

**Description:** If any component uses a proprietary storage format, future migration becomes painful. A format that can't be read in 5 years is a trap.

**Likelihood:** Low (if discipline holds)
**Impact:** High — data loss or expensive migration.
**Mitigation:** SQLite and plain markdown only. No proprietary blobs. This is a load-bearing principle, not a preference.

### 7. Resource Budget Violations

**Description:** Background indexing, embedding computation, or screen capture can drain battery, consume RAM, and spin up the laptop fan. Users uninstall tools that make their machine hot.

**Likelihood:** Medium
**Impact:** High — the system gets disabled and stays disabled.
**Mitigation:** Strict resource budget (<1% CPU, <300MB RAM for daemon). Throttle by battery state. Lazy processing (queue work during idle). Monitor and alert if budget is exceeded.

### 8. Citations Hallucinated or Missing

**Description:** LLM-synthesized answers may cite items that don't exist or misattribute content to the wrong source. Without verifiable citations, the system's answers are no more trustworthy than a bare LLM.

**Likelihood:** Medium
**Impact:** High — trust erosion.
**Mitigation:** Structured prompting that injects source IDs. Post-synthesis citation validation (verify every cited ID exists in the result set). Never let the LLM answer without injected context.

### 9. Capture Habits Don't Stick

**Description:** The prospector log is the highest-quality signal in the system — human judgment about what matters. If the user stops adding entries, the system loses its most valuable input.

**Likelihood:** Medium
**Impact:** Medium — the system still works on other sources but loses its sharpest signal.
**Mitigation:** Capture friction must be under 10 seconds. One hotkey from anywhere. Track habit health (entries per week). If the habit fades, diagnose friction rather than adding automation.

### 10. Ambient Layer Over-Presents

**Description:** Proactive surfacing (morning briefs, radar passes) sounds great in theory. In practice, frequent low-quality suggestions train the user to ignore the system entirely.

**Likelihood:** Medium
**Impact:** Medium — the most valuable layer becomes shelfware.
**Mitigation:** High precision threshold. Pull mode (daily digest the user checks) not push mode (notifications). User controls cadence. Start with one radar query and add more only if the first one proves useful.

### 11. Work / Personal Contamination

**Description:** A unified corpus means a work query might surface personal messages, or personal content might appear on a shared screen during a presentation.

**Likelihood:** Medium
**Impact:** High — embarrassment, or worse.
**Mitigation:** Context tagging at ingestion time (work/personal per source). Context filter at query time (default: all; switchable to work-only or personal-only). Future: "presentation mode" that hides personal results.

### 12. Multi-Machine Sync Is Undesigned

**Description:** The current architecture is single-machine. If the user uses a work laptop and a personal machine, each has its own partial corpus and neither knows about the other.

**Likelihood:** High (most people use 2+ machines)
**Impact:** Medium — fragmentation at the machine level, which is ironic.
**Mitigation:** Defer for now. The SQLite file is portable (copy to sync). Future options: git-based sync, iCloud/Dropbox sync of the DB file, Syncthing. Design decision deferred to post-v1.

---

## Open Questions

### OQ-1: Which embedding model?

**Why it matters:** Embedding quality directly affects semantic search precision. Model size affects disk and RAM.
**Options:** nomic-embed-text (768-dim, ~500MB, strong quality-to-size ratio) | bge-small-en (384-dim, smaller) | e5-small-v2 (384-dim)
**Recommendation:** Start with nomic-embed-text. Reassess if resource budget is tight.
**Decide by:** Phase 3.

### OQ-2: Local LLM vs cloud LLM for re-rank and synthesis?

**Why it matters:** Local is private but lower quality; cloud is higher quality but sends content off-machine.
**Options:** Ollama + Llama 3.x (local default) | Claude API (opt-in per query) | Hybrid (local default, cloud flag)
**Recommendation:** Hybrid. Local default, cloud opt-in with `--cloud` flag.
**Decide by:** Phase 4.

### OQ-3: Mobile retrieval parity?

**Why it matters:** The user is on their phone half the day. Capture-only on mobile is useful but incomplete.
**Options:** Capture-only via iOS Shortcuts | Full retrieval via Tailscale to localhost | Companion mobile app
**Recommendation:** Capture-only for v1. Retrieval via Tailscale as a stretch goal.
**Decide by:** Phase 6.

### OQ-4: Multi-machine sync strategy?

**Why it matters:** Most users have 2+ machines; a single-machine tool eventually frustrates.
**Options:** Defer entirely | iCloud/Dropbox sync of SQLite file | git-based sync | Syncthing | Server mode
**Recommendation:** Defer. Design the DB to be copyable. Revisit post-v1.
**Decide by:** Post-v1.

### OQ-5: Work/personal isolation model?

**Why it matters:** Contamination risk (Risk 11). Separate databases are clean but double the maintenance; unified with tagging is simpler but riskier.
**Options:** Separate SQLite databases per context | Unified database with context tags + query-time filter
**Recommendation:** Unified with tagging. Simpler, and the filter is cheap.
**Decide by:** Phase 2.

### OQ-6: ScreenPipe from day 1 or opt-in later?

**Why it matters:** Passive capture is valuable but can overwhelm before retrieval is mature.
**Options:** Install with Steward from day 1 | Explicitly deferred to Phase 7+
**Recommendation:** Deferred. Install only after Phases 2-4 are working.
**Decide by:** Phase 4 completion.

### OQ-7: Long-lived daemon vs on-demand CLI?

**Why it matters:** A daemon enables scheduled ingestion and radar passes. An on-demand CLI is simpler and uses zero resources when idle.
**Options:** Always-on daemon (launchd/systemd) | On-demand CLI + cron for scheduled tasks | Hybrid (daemon for Phase 7, CLI-only before)
**Recommendation:** Hybrid. CLI-only through Phase 6; daemon in Phase 7.
**Decide by:** Phase 6 completion.

### OQ-8: How is prospector.md distinguished from other markdown?

**Why it matters:** The system ingests markdown files; it needs to know which one is the prospector log (human-judgment signal) vs. regular notes.
**Options:** Hardcoded path in config | Special frontmatter marker | File naming convention
**Recommendation:** Hardcoded path in config (`prospector_path` in config.yaml). Simplest, most reliable.
**Decide by:** Phase 2.

### OQ-9: Index behavior when source data moves?

**Why it matters:** iMessage databases rotate. Dropbox folders get renamed. Source pointers become stale.
**Options:** Accept stale pointers (log warning on click) | Re-ingest periodically and update pointers | Track source paths and detect changes
**Recommendation:** Accept stale pointers with warning. Re-ingestion handles most cases naturally.
**Decide by:** Phase 5.

### OQ-10: How are embeddings refreshed for updated content?

**Why it matters:** Some sources have mutable content (edited notes, updated docs). Stale embeddings mean semantic search misses the current version.
**Options:** Re-embed on each ingestion run (expensive) | Track content hash, re-embed only on change | Never re-embed (accept staleness)
**Recommendation:** Track content hash. Re-embed only when hash changes. Amortizes cost.
**Decide by:** Phase 3.

### OQ-11: Where does configuration live?

**Why it matters:** Needs to be human-readable, editable, and version-controllable.
**Options:** YAML file at `~/.steward/config.yaml` | SQLite table | Both (YAML is source of truth, loaded into SQLite at startup)
**Recommendation:** YAML file. Simple, readable, diffable.
**Decide by:** Phase 2.

### OQ-12: Does the system need a GUI?

**Why it matters:** A GUI could make the system more accessible but risks scope creep and the "another app" trap.
**Options:** No GUI, CLI + thin clients only | Minimal web UI served by the daemon | Full desktop app (Tauri)
**Recommendation:** No GUI for v1. The product is the API. A web UI can be added as a thin client later if desired.
**Decide by:** Post-v1.

### OQ-13: Schema versioning across releases?

**Why it matters:** As the system evolves, the SQLite schema will change. Users need to upgrade without losing their index.
**Options:** Manual migration scripts | Alembic or similar migration tool | Version table + auto-migration on startup
**Recommendation:** Version table in SQLite + auto-migration on startup. Keep migrations simple and reversible.
**Decide by:** Phase 3.

---

## Glossary

**Ambient retrieval.** Retrieval that happens without the user asking — the system proactively surfaces relevant items based on context (current activity, calendar, time of day). Distinct from on-demand retrieval where the user types a query.

**Append-only.** A design principle: Steward never modifies or deletes source data. It only reads and indexes. The index itself is append-only in the sense that new items are added but existing items are not removed unless an exclusion rule applies.

**Capture layer.** The part of the system responsible for getting information into the corpus. Includes both deliberate capture (prospector log entries) and passive capture (ScreenPipe screen recording). In Steward's architecture, capture is considered a solved problem.

**Chunking.** The process of splitting a long text item into smaller overlapping segments (~500 tokens each) for embedding. Necessary because embedding models have token limits and because fine-grained chunks produce more precise semantic search results.

**Citation.** A reference from a retrieval answer back to the specific source item(s) that support it. Every Steward answer must include citations with source pointers. Citations are the mechanism by which trust is maintained.

**Context.** The work/personal (or other) classification of an item. Used for filtering at query time to prevent contamination between contexts.

**Cross-encoder.** A neural model that scores the relevance of a (query, document) pair. More accurate than embedding similarity but too expensive to run on all items — used only for re-ranking the top candidates.

**Earned progression.** The principle that each phase of the build must validate as a working habit before the next phase is started. Prevents building tools that go unused.

**Embedding.** A fixed-length numeric vector (e.g., 768 floats) that represents the semantic meaning of a text chunk. Similar meanings produce similar vectors, enabling search by meaning rather than keyword.

**Exclusion rule.** A user-defined rule that prevents specific data from being indexed or returned. Scoped by source, folder, actor, or time window. Enforced at both index time and query time.

**Fragmentation axes.** The three dimensions along which a user's information is scattered: channel (which app/tool), context (work/personal), and time (current/historical). The multiplicative combination of these axes produces the ~60-location lookup problem Steward is designed to solve.

**FTS5.** SQLite's built-in full-text search extension. Provides fast keyword search with Porter stemming, phrase matching, prefix matching, and Boolean operators. Zero dependencies beyond SQLite itself.

**Guest in your data.** The design principle that Steward reads data in place without copying, moving, or modifying it. The system is a guest in the user's existing data landscape, not a landlord demanding migration.

**Hybrid retrieval.** A search strategy that queries both the keyword index (FTS5) and the semantic index (sqlite-vec) in parallel, then merges the results. Items appearing in both result sets are ranked higher. Catches cases that either index alone would miss.

**Integration layer.** The part of the system that exposes retrieval to the user's actual workflow — CLI, Raycast, iOS Shortcuts, Obsidian, browser. This is Steward's primary contribution; capture and retrieval primitives are solved problems, but integration is not.

**Item shape.** The normalized data structure that every source adapter produces. Fields: id, source, timestamp, actors, content, context, metadata, source_pointer. The common shape enables source-agnostic indexing and retrieval.

**Keyword index.** The FTS5-based search index over item content. Handles exact matches, phrases, prefixes, and Boolean queries. Complements the semantic index.

**Local-first.** The principle that all processing and storage happens on the user's machine by default. Cloud services (LLM APIs) are opt-in per query and never the default path.

**Normalized item.** An ingested piece of information that has been converted from its source-native format into the common item shape. Source-specific details are preserved in the metadata field.

**Ollama.** An open-source tool for running LLMs locally. Steward uses it for both embedding (nomic-embed-text) and inference (Llama 3.x). Runs on CPU or Apple Silicon without requiring a GPU.

**Prospector.** A user archetype: someone who accumulates information because their past work has genuine operational value. When old information resurfaces, a prospector feels relief ("oh wow, valuable"), not dread ("oh no, more debt"). Steward is built for prospectors.

**Prospector log.** The foundational artifact: a single markdown file (`prospector.md`) with one line per entry recording date, source/location, and a sentence about what's there and why it matters. Represents human judgment about what is valuable. The signal that all automation serves.

**Provenance.** The traceable origin of a retrieved item. Every Steward answer includes provenance: which source, which item, what timestamp, and a pointer to the original location.

**Radar pass.** A scheduled background query that runs without user input. Examples: "items from this week worth remembering," "old notes matching current projects." Results feed the morning brief or write back to the prospector log.

**RAG (Retrieval-Augmented Generation).** The pattern of retrieving relevant items from a corpus and injecting them into an LLM prompt as context before generating an answer. Steward's answer mode is a RAG pipeline.

**Re-ranking.** The stage in the retrieval pipeline where an LLM or cross-encoder scores the top candidates for relevance to the original query. More expensive than initial retrieval but dramatically improves precision.

**Routing metadata.** The mental overhead of remembering which channel, context, or time period a piece of information lives in. The core tax that fragmentation imposes. Steward exists to eliminate this tax.

**ScreenPipe.** An open-source tool that continuously captures screen content (via OCR) and audio (via Whisper). Stores everything in a local SQLite database. Steward consumes ScreenPipe as a source adapter — it does not rebuild screen capture.

**Semantic search.** Search by meaning rather than exact keywords. Implemented via embedding similarity: the query is embedded and compared against pre-computed chunk embeddings using cosine similarity.

**Source adapter.** A module that reads from one specific data source (iMessage, Apple Notes, Dropbox, etc.) and produces normalized items. Each adapter implements the same contract: healthcheck, ingest, normalize, resolve_pointer.

**sqlite-vec.** A SQLite extension for storing and querying vector embeddings. Enables cosine similarity search within the same database file as the keyword index. Chosen for one-file portability.

**Storage without retrieval.** The failure mode of most existing tools: they preserve information (storage) but cannot surface the right piece at the right moment (retrieval). They are "landfills with a search bar."

**Trust layer.** The cross-cutting set of invariants that ensure Steward earns and keeps user trust: citations always, exclusions enforced absolutely, local by default, non-destructive, inspectable. Not a component but a set of properties enforced everywhere.

**Validation gate.** A specific, testable criterion that must be met before moving to the next build phase. Examples: "5+ entries per week for 2 weeks" (Phase 0), "keyword lookup works across 3 sources" (Phase 2). Prevents building on top of unvalidated assumptions.

**Vector search.** See *Semantic search*.

**Whisper.** OpenAI's open-source speech recognition model. Used (via whisper.cpp) for transcribing audio in ScreenPipe and potentially for voice-memo ingestion. Free, local, state-of-the-art quality.

**Write-back loop.** The mechanism by which the system feeds itself: when a radar pass or retrieval query surfaces something valuable, a pointer is automatically appended to the prospector log. Closes the loop between automated discovery and human-judged value.
