# Steward: Vision

*A personal memory system for prospectors. Meets your data where it lives, surfaces it when you need it, and never asks you to migrate.*

---

## The Problem Is Fragmentation, Not Volume

The dominant story about digital overload is that we have too much stuff. That story is wrong, or at least shallow. The real pain is not volume — it is **fragmentation**, and fragmentation has a precise geometry.

Information lives along three axes:

- **Channel**: notes apps, AI chat transcripts, email, iMessage, Dropbox, browser history, local files, terminals, voice memos, screenshots. Call it ten, in practice.
- **Context**: work and personal, at minimum. Often more — side projects, family, hobbies.
- **Time**: what is live right now, what is from last month, and the decade of sediment underneath.

Ten channels times two contexts times three time strata is roughly **sixty possible locations** for any single item. The cost of adding a new channel is not additive, it is multiplicative: every new place you can put something multiplies every future lookup.

This is a **cache coherence problem** in systems-design terms. And the most expensive consequence is not storage — it is that your working memory gets consumed by **routing metadata**. You stop thinking about the thing and start thinking about where the thing is. "Was that in a Claude chat or a ChatGPT chat? Was it in the Obsidian vault or a loose markdown file? Was that from the work laptop or personal?" The content is gone; only the index query remains, and the index is broken.

Steward exists to collapse that cost.

## Why Existing Tools Fail

### Storage without retrieval

Every second-brain app, every LLM chat history, every cloud drive is optimized for the wrong half of the problem. They preserve everything and surface almost nothing. They are **landfills with a search bar**. A chat transcript is not a memory; it is a pile. The test of memory is whether the right thing shows up at the right moment without you having to phrase a perfect query for it. On that test, nothing currently ships.

This is the core indictment: **the industry delivers storage without retrieval**, then calls it memory.

### Organization and retrieval are opposites

The second failure is a category error baked into almost every productivity system ever sold. **Organization is front-loaded work** — taxonomy, tagging, folders, decisions at save time about how future-you will want to search. It always fails, because you cannot predict future-you's query. **Retrieval is lazy** — no work at save time, cost paid only when a real need surfaces. These are not complements. They are opposites. Effort spent on organization is effort not spent on retrieval, and the trade is bad.

Steward's bet: **if retrieval is good enough, organization becomes unnecessary.** Optimize for the lazy side of the dichotomy and the front-loaded side withers away on its own.

### The second-brain trap

Every Notion, every Obsidian, every Roam, every Mem, wants you to **migrate into it**. That is the business model — become the walled garden, own the surface. The consequence is that they violate the first rule of a personal memory system: they refuse to meet you where your data already is. They do not reduce fragmentation; they add one more channel to it. The tool that was supposed to unify your information became the sixty-first location.

Steward refuses this trade.

## The User: Built for Prospectors, Not Minimalists

There are two psychologies around accumulation and they need completely different tools.

**Hoarders** accumulate because letting go feels bad. Stuff is a burden they cannot put down. The correct advice for a hoarder is the minimalist playbook: throw away more, declare bankruptcy, trust the fade, stop saving things you will never look at again. Marie Kondo is right for them.

**Prospectors** accumulate because past-self is a useful collaborator. When an old Dropbox account surfaces unexpectedly, the dominant feeling is not "oh no, more debt" — it is **"oh wow, valuable."** Their archive pays dividends. An old note, a four-year-old chat log, a screenshot from 2021 — any of these can turn out to have real operational value in a current problem. The minimalist playbook is actively harmful to a prospector: it tells them to throw away the compost their future work is built on.

Steward is built for prospectors. The explicit test: when old information resurfaces, does the user feel relief or dread? If relief, Steward is for them. If dread, they should use a different tool, probably a trash can.

The desire for total recall is not greedy — it is rational. Fallible memory is a hardware limitation, not a virtue. Every external memory technology humans have ever invented — writing, libraries, search engines, Evernote, LLMs — is a chip off the same desire for photographic recall. The reason it has not been delivered is not that it was wrong to want; it is that the tools keep delivering storage and calling it memory.

## Core Principles

Steward is defined as much by what it refuses as by what it builds.

- **Local-first.** Data stays on the user's machine. Cloud is opt-in, per query, never by default.
- **Never migrate.** Meet data where it already lives. Do not ask the user to move anything, ever.
- **Guest in your data, not a landlord.** Read in place. Leave originals untouched. Own nothing.
- **Plain text and open formats.** The artifacts Steward produces must outlive Steward.
- **Append-only.** Never destructive. Never reorganizes source data. History is sacred.
- **Citations always.** Every retrieved answer cites its source with a click-through to the original. No black-box recall.
- **Inspectable.** Every layer — the index, the embeddings, the traces, the logs — is just files a human can open.
- **Frictionless capture.** If logging an item takes more than ten seconds, capture is broken.
- **Earn each step.** Do not build Layer N until Layer N-1 has stuck as a habit. No speculative scaffolding.

These are not preferences. They are the load-bearing walls.

## What Steward Is, and Isn't

Steward **is** an integration layer: a thin, local, inspectable system that sits on top of data the user already has, indexes it where it lives, and surfaces it on demand with citations. It is a retrieval engine, a capture habit, and a set of adapters — in that order of importance.

Steward **is not** a note-taking app. Not a second brain. Not a knowledge graph product. Not a place to put things. It has no walled garden and no migration path, because there is nothing to migrate into.

## The Unique Opportunity: Integration Is the Gap

Three categories describe the problem space, and only one is open.

- **Capture** is solved. ScreenPipe, Rewind, browser history, shell history, voice memo apps, and the ambient logging of modern OSes already capture more than enough.
- **Retrieval primitives** are solved at the component level. SQLite FTS5, sqlite-vec, Whisper, local embedding models, LlamaIndex, ripgrep — the building blocks ship and work.
- **Integration is unsolved.** Nothing assembles these pieces into a system that meets a real user in their real editor, real browser, real chat, real terminal, sitting on top of their real files. Every attempt so far has either built a walled garden or shipped a component library.

That is Steward's seat at the table: **the integration layer that respects existing tools.**

A general-purpose version of this is a research problem — it has to compromise for users it does not know. A personal version, shaped exactly to one builder's channels, habits, and editor of choice, is a weekend project that compounds. Steward is the personal version, built first for one prospector, designed so its principles generalize later if they want to.

## Layer 0: The Prospector Log

The foundational artifact of Steward is almost embarrassingly simple. It is a single plain-text markdown file, `prospector.md`, with one line per entry and exactly three fields:

```
date | source/location | sentence about what's there and why future-self cares
```

No tags. No categories. No folders. No schema. Three fields.

The prospector log is not a note-taking surface. It is a **human-judgment layer**. The human — the only entity in the system with taste and stakes — marks "this matters." The automation's only job is to index the marks and make them reachable. Everything else Steward does, at every higher layer, is downstream of this file.

> **The prospector log is the signal. The automation is the plumbing.**

This is the philosophical anchor. A vision built on a three-field text file cannot drift into becoming a walled garden, because there is no garden — there is a text file, and then a pile of adapters that respect it. If Layer 0 does not stick as a daily habit, no higher layer is worth building. If it does stick, every higher layer is an obvious extension of it.

Everything else in the spec — requirements, architecture, implementation, risks — is downstream of this.
