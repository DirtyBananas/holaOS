# Repo Triage & Artifact Strategy

Generated 2026-06-27. A prioritized plan for taming a large set of scattered
repositories (GitHub + local + other hosts) that target mixed platforms
(Android, PC/desktop, web), where it has become hard to remember what each repo
is, whether it is worth keeping, and what its build actually produces.

## The core insight

This looks like one problem ("my repos are a mess") but it is three problems
wearing one coat, and they must be solved **in order** because each one gates
the next:

1. **Triage / memory** — you can't decide *what* to invest in until you can
   *see* what you have. Cheap. Do it first, across everything.
2. **Artifact visibility** — a repo that builds an APK/EXE you can't see is a
   repo you can't evaluate. Medium effort. Do it for the keepers.
3. **Live sandbox / emulator** — the "fire it up and poke it" wish. Expensive
   and platform-specific. Do it last, only for the few survivors that earn it.

> The failure mode to avoid: starting at #3. Building an emulator sandbox for
> *every* repo means pouring weeks into repos you will end up deleting. You
> cannot prioritize what you cannot see.

```
  Phase 0          Phase 1               Phase 2                 Phase 3
  Inventory   →    Triage decisions  →   Standardize build  →    Live sandbox
  (all repos)      (keep/kill/merge)     + artifacts (keepers)   (survivors only)
   cheap            cheap                 medium                  expensive
```

---

## Phase 0 — Inventory (cheap, do across everything)

**Problem:** Repos are scattered across GitHub, local disk, and possibly other
hosts. There is no single list, so "how many do I even have" is unanswered.

**Desired behavior:** One command produces a single normalized inventory of
every repo from every source, de-duplicated (a repo cloned locally *and* on
GitHub is one entry, not two).

**Scope:**
- A collector with pluggable sources:
  - **GitHub** — list repos via API for the account/org.
  - **Local disk** — walk a set of root directories, find every `.git`, read
    its `origin` remote to reconcile against the GitHub list.
  - **Other hosts** — any extra remotes discovered on local clones get recorded
    even if we can't crawl them.
- Output: a machine-readable `inventory.json` + a human `INVENTORY.md` table.
- Reconciliation key: normalized remote URL; fall back to repo name when no
  remote exists (local-only repos).

**Per-repo signals to capture (the cheap, automatable ones):**

| Signal | Why it matters |
|--------|----------------|
| Last commit date | Dead vs. active is the #1 triage axis |
| Primary language(s) | Groups repos; hints at build system |
| Build system detected | `gradle`/`package.json`/`Cargo.toml`/`*.csproj`/Makefile |
| Target platform guess | Android / desktop / web / library / unknown |
| Has CI? | Already-automated vs. needs work |
| Produces artifacts? | Releases present, or build outputs in `.gitignore` |
| LOC / file count | Rough size/effort signal |
| README present + first paragraph | Auto-draft of "what is this" |
| Open issues / PRs (GitHub) | Abandoned-but-had-users signal |
| Uncommitted local changes | Local-only work at risk of loss |

Nothing here requires *running* the repo — it's all static inspection, so it's
fast and safe to run over hundreds of repos.

---

## Phase 1 — Triage decisions (cheap, but needs you)

**Problem:** Even with an inventory, "why should I keep this / what's unique
about it" is a judgment only you can make — but right now there's nowhere to
*record* that judgment, so you re-derive it every time.

**Desired behavior:** Every repo gets a committed `STATUS.md` that captures the
decision once, so future-you never re-investigates.

**Scope — a standard `STATUS.md` template:**

```markdown
# STATUS

- **What this is:** <one sentence>
- **Why it exists / what's unique:** <the thing only this repo does>
- **Decision:** KEEP | ARCHIVE | MERGE-INTO(<repo>) | KILL
- **Reason for decision:** <one sentence>
- **Target platform & artifact:** <e.g. Android APK / Windows EXE / web>
- **Build status:** builds clean | broken | unknown
- **Last reviewed:** 2026-06-27
```

**The decision rubric (so triage is fast and consistent):**

| Decision | When |
|----------|------|
| **KEEP** | Unique purpose, builds (or fixably), you'd miss it |
| **ARCHIVE** | Done/finished or historically interesting, but inactive — set GitHub to archived, stop worrying about it |
| **MERGE** | Overlaps heavily with another repo — fold the unique bits in, then kill |
| **KILL** | Duplicate, experiment that taught you the lesson already, or superseded |

**Throughput tip:** triage in *batches by cluster* (the inventory groups repos
by language/platform), not one-by-one. Most repos are an obvious KILL/ARCHIVE in
seconds; spend your real attention on the ambiguous middle.

This phase converts an overwhelming "100 repos" into a short "12 keepers" list,
which is the only list that matters for Phases 2 and 3.

---

## Phase 2 — Standardize build & artifact visibility (medium, keepers only)

**Problem:** A repo builds an Android APK or a PC executable, but you can't see
the result without checking out, installing toolchains, and building locally —
so you never do, and the repo stays opaque.

**Desired behavior:** Every keeper, on every push to its main branch, builds its
platform artifact and publishes it where you can grab it in two clicks — plus a
screenshot/recording so you can eyeball the result without installing anything.

**Scope — a small library of reusable CI workflow templates**, one per platform
family, each doing the same three things:

1. **Build** the platform artifact (APK / AAB / EXE / app bundle / web dist).
2. **Publish** it — attach to a GitHub Release (tagged) and as a workflow
   artifact (every run), so there's always a downloadable result.
3. **Capture a preview** — a screenshot or short recording of the thing
   running, committed to the run summary.

| Platform | Build | Artifact | Cheap preview |
|----------|-------|----------|---------------|
| Android | Gradle `assembleRelease` | APK/AAB on Release | Screenshot from headless emulator boot |
| Desktop (Electron) | `electron-builder` | Installer per OS | Playwright screenshot of launched app |
| Desktop (native) | platform compiler | EXE/app bundle | Screenshot of windowed run under xvfb |
| Web | framework build | Static dist / preview deploy | Live preview URL (this is nearly free) |
| Library | build + test | Package | Test/coverage badge (no UI to show) |

**Why templates, not per-repo bespoke CI:** scattered + mixed means consistency
is the whole win. Drop the matching template into a keeper, fill in 2–3
variables, done. One place to fix when something breaks.

**holaOS leverage:** `holaOS` already has a desktop build pipeline
(`desktop/electron-builder.config.cjs`, e2e via Playwright) and a runtime
release flow (`RELEASING.md`, `runtime/deploy`). The desktop/web templates can
be lifted almost directly from what already works here rather than invented.

---

## Phase 3 — Live sandbox / emulator (expensive, survivors only)

**Problem:** Even with a downloadable artifact and a screenshot, some repos you
want to actually *use* — boot it, click around — without a local install.

**Desired behavior:** A keeper exposes a "▶ Launch sandbox" path that boots the
app in an emulator/preview you can interact with.

**Scope — and the honest cost, because this is the expensive phase:**

| Platform | Feasible approach | Cost / caveat |
|----------|-------------------|---------------|
| **Web** | Ephemeral preview deploy per branch | Cheap — do this freely |
| **Android** | Cloud emulator (managed device farm) or browser-streamed AVD | Real money + latency; reserve for top apps |
| **Desktop** | Streamed VM / container with the app pre-installed | Heaviest; only for a flagship or two |

**This is where holaOS itself is the natural home.** holaOS is "the agent
environment for long-horizon work" with a runtime, harnesses, and durable
state — a per-repo prebuilt sandbox that boots an emulator is squarely the kind
of *workspace* holaOS is built to host. Rather than bolting a separate emulator
service onto every repo, the survivors could become holaOS workspaces whose
artifact is "launch the thing." That folds your scattered-repo problem into the
platform you already maintain, instead of building a parallel system.

**Decision gate before any Phase 3 work:** only repos marked KEEP in Phase 1
*and* with green artifacts in Phase 2 *and* that you genuinely want to demo
interactively qualify. Expect this to be a handful, not the whole set.

---

## Sequencing & what to build first

| Order | Deliverable | Effort | Unblocks |
|-------|-------------|--------|----------|
| 1 | Inventory collector (GitHub + local) → `inventory.json` / `INVENTORY.md` | S | All decisions |
| 2 | `STATUS.md` template + triage rubric | XS | The keepers list |
| 3 | Reusable build+artifact workflow templates (web/desktop/android) | M | Artifact visibility |
| 4 | Preview-capture step (screenshot/recording) added to templates | M | Eyeballing results |
| 5 | Web preview deploys for web keepers | S | Cheapest live sandbox |
| 6 | holaOS-hosted sandbox for 1–2 flagship survivors | L | The full vision |

**Recommended first concrete step:** build deliverables 1 and 2 (the inventory
collector + STATUS template). They are small, run safely over hundreds of repos
without building anything, and they produce the keepers list that makes every
later phase tractable. Everything downstream is wasted effort until that list
exists.

## Open questions to resolve before building

- **Inventory home:** does the triage toolkit live here in `holaOS` (it's your
  cleanest repo and already a "platform"), or in a dedicated `repo-ops` repo?
- **GitHub scope:** which account(s)/org(s) hold the repos? (This session is
  scoped to `dirtybananas/holaos` only, so the collector will need broader
  credentials when run for real.)
- **Local roots:** which directories on disk should the local scanner walk?
- **Android sandbox budget:** is interactive Android emulation worth real
  recurring cost, or is "APK + boot screenshot" (Phase 2) enough?
