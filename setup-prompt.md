# LLM Wiki

A pattern for building personal knowledge bases using LLMs.

This is an idea file, designed to be copy-pasted to your LLM agent (e.g. Claude Code, OpenAI Codex, OpenCode, or similar). Its goal is to communicate the high-level idea and the conventions that make it work — but your agent will build out the specifics in collaboration with you.

---

## The Core Idea

Most people's experience with LLMs and documents looks like RAG: you upload a collection of files, the LLM retrieves relevant chunks at query time, and generates an answer. This works, but the LLM is rediscovering knowledge from scratch on every question. There's no accumulation. Ask a subtle question that requires synthesizing five documents, and the LLM has to find and piece together the relevant fragments every time. Nothing is built up. NotebookLM, ChatGPT file uploads, and most RAG systems work this way.

The idea here is different. Instead of just retrieving from raw documents at query time, the LLM incrementally builds and maintains a persistent wiki — a structured, interlinked collection of markdown files that sits between you and the raw sources. When you add a new source, the LLM doesn't just index it for later retrieval. It reads it, extracts the key information, and integrates it into the existing wiki — updating entity pages, revising topic summaries, noting where new data contradicts old claims, strengthening or challenging the evolving synthesis. The knowledge is compiled once and then kept current, not re-derived on every query.

This is the key difference: **the wiki is a persistent, compounding artifact.** The cross-references are already there. The contradictions have already been flagged. The synthesis already reflects everything you've read. The wiki keeps getting richer with every source you add and every question you ask.

You never (or rarely) write the wiki yourself — the LLM writes and maintains all of it. You're in charge of sourcing, exploration, and asking the right questions. The LLM does all the grunt work — the summarizing, cross-referencing, filing, and bookkeeping that makes a knowledge base actually useful over time. In practice, I have the LLM agent open on one side and Obsidian open on the other. The LLM makes edits based on our conversation, and I browse the results in real time — following links, checking the graph view, reading the updated pages. Obsidian is the IDE; the LLM is the programmer; the wiki is the codebase.

This can apply to a lot of different contexts:

- **Personal:** tracking your own goals, health, psychology, self-improvement — filing journal entries, articles, podcast notes, and building up a structured picture of yourself over time
- **Research:** going deep on a topic over weeks or months — reading papers, articles, reports, and incrementally building a comprehensive wiki with an evolving thesis
- **Reading a book:** filing each chapter as you go, building out pages for characters, themes, plot threads, and how they connect. By the end you have a rich companion wiki — think fan wikis like Tolkien Gateway, built personally as you read, with the LLM doing all the cross-referencing
- **Business/team:** an internal wiki maintained by LLMs, fed by Slack threads, meeting transcripts, project documents, customer calls — the wiki stays current because the LLM does the maintenance that no one on the team wants to do
- Competitive analysis, due diligence, trip planning, course notes, hobby deep-dives — anything where you're accumulating knowledge over time and want it organized rather than scattered

---

## Architecture

There are three layers:

**Raw sources** — your curated collection of source documents. Articles, papers, images, data files. These are immutable — the LLM reads from them but never modifies them. This is your source of truth.

**The wiki** — a directory of LLM-generated markdown files. Summaries, entity pages, concept pages, comparisons, an overview, a synthesis. The LLM owns this layer entirely. It creates pages, updates them when new sources arrive, maintains cross-references, and keeps everything consistent. You read it; the LLM writes it.

**The schema** — a document (e.g. `CLAUDE.md` for Claude Code or `AGENTS.md` for Codex) that tells the LLM how the wiki is structured, what the conventions are, and what workflows to follow when ingesting sources, answering questions, or maintaining the wiki. This is the key configuration file — it's what makes the LLM a disciplined wiki maintainer rather than a generic chatbot. You and the LLM co-evolve this over time as you figure out what works for your domain.

---

## Vault Structure

A suggested starting layout. Adapt freely to your domain:

```
/                       ← vault root
├── CLAUDE.md           ← agent schema and operating rules (you co-author; agent follows)
├── VISITING.md         ← lightweight briefing for visiting subagents (see below)
├── index.md            ← master content catalog (agent maintains)
├── log.md              ← append-only session log (agent maintains)
├── hot.md              ← session cache: active threads, recent decisions, next actions
├── raw/                ← immutable source documents (you add; agent reads)
│   ├── inbox/          ← staging area: unclassified files awaiting processing
│   └── ...             ← subdirectories by source type (articles/, meetings/, etc.)
└── wiki/               ← LLM-generated pages (agent owns entirely)
    ├── overview.md     ← "state of the domain" synthesis
    ├── projects/       ← one page per ongoing project or initiative
    ├── systems/        ← infrastructure, services, tools
    ├── concepts/       ← technical or domain concepts and patterns
    ├── people/         ← stakeholders, authors, collaborators
    ├── decisions/      ← decisions and ADRs, dated
    ├── runbooks/       ← operational step-by-step procedures (re-executed, not just re-read)
    └── analyses/       ← comparisons, investigations, query outputs
```

---

## Page Format

Every wiki page should include frontmatter so the LLM (and tools like Obsidian Dataview) can reason about the wiki structurally:

```yaml
---
title: <Human-readable title>
type: project | system | concept | person | decision | analysis | runbook | overview
tags: [<relevant tags>]
sources: [<list of raw/ filenames that informed this page>]
updated: YYYY-MM-DD
---
```

Suggested body conventions:
- Lead with a 2–3 sentence summary (no heading needed)
- Use `[[WikiLink]]` syntax for all cross-references to other wiki pages
- Use `> **Source:** raw/path/to/file` blockquotes when citing specific facts
- Flag contradictions and open questions with `> **Conflict:**`, `> **Open question:**` blockquotes
- Log unresolved conflicts in `hot.md` under `## Open Conflicts` until you resolve them — don't let the agent silently pick a winner
- Update the `updated` field any time a page is materially edited

---

## Operations

### Session Start

Every session, the agent should:

1. Read `hot.md` — resolves most recent-context questions without touching anything else
2. Check `raw/inbox/` — if files are present, drain it before proceeding (see INGEST Step 0)
3. Only if `hot.md` doesn't resolve the question: read `index.md`, then the last few `log.md` entries

The key principle: `hot.md` is always read eagerly; `index.md` and `log.md` are read lazily. This keeps short sessions cheap.

### INGEST

Triggered when you say "ingest [filename]" or "process [filename]".

**Step 0 — Drain the inbox:**
1. Check `raw/inbox/`. If empty, skip to Step 1.
2. Read enough of each file to classify it (~50 lines); propose a destination subdirectory and final filename. If the file has no inherent date, prepend `YYYY-MM-DD-` to the filename.
3. Show you the classification plan (one line per file) and wait for confirmation before moving anything.
4. On confirmation, `mv raw/inbox/<file> raw/<subdir>/<finalname>`.
5. Add moved files to the ingest queue.

**Step 1 — Ingest each file:**
1. Read the source file completely
2. Discuss key takeaways with you (3–5 bullet points; ask if anything is missing or wrong)
3. Write a summary page in `wiki/`
4. Update `index.md` with the new page entry
5. Identify all entities, systems, concepts, and people mentioned
6. Update or create their wiki pages — add new info, note conflicts, strengthen cross-references
7. Append an entry to `log.md`

A single ingest typically touches 5–15 wiki pages. The cross-referencing is where most of the value compounds.

**Tip:** separate "I have a file" from "I'm ready to work on this now." Drop files in `raw/inbox/` whenever they arrive. When you're ready, ask the agent to drain the inbox — it proposes classifications, waits for your confirmation, then ingests in sequence.

### UPDATE

Triggered when you say "update [page] with [context]" or share a fact without a source file — a decision made in a meeting, a number that changed, something said in conversation.

The agent reads the target page, applies the update, records the source as `(conversation YYYY-MM-DD)`, updates the `updated` date, and appends a brief log entry.

This operation matters because not everything worth knowing arrives as a file. Without a named UPDATE operation, conversational knowledge silently disappears into chat history.

### QUERY

Triggered when you ask a question.

1. Read `index.md` to identify relevant pages
2. Read those pages in full
3. Synthesize an answer with `[[WikiLink]]` citations
4. File as a wiki page if the answer required synthesizing 3+ pages, or reveals a pattern or decision not captured elsewhere

Good answers compound just like ingested sources do. An analysis you asked for, a comparison you needed, a connection you discovered — these shouldn't disappear into chat history.

Answers can take different forms: a markdown page, a comparison table, a slide deck (Marp), a chart (matplotlib). Let the format fit the question.

### LINT

Triggered when you say "lint the wiki", or automatically when the `Ingests since last lint` counter in `hot.md` reaches 10 (see hot.md section).

1. Read all pages in `wiki/`
2. Check for:
   - Contradictions between pages
   - Orphan pages (no inbound links)
   - Concepts mentioned but lacking their own page
   - Missing cross-references between related pages
   - Data gaps worth filling
3. Report findings as a checklist
4. Fix what can be fixed inline; flag what needs your input
5. Reset `Ingests since last lint` in `hot.md` to 0

### Session Close-Out

At the end of any session that modified wiki pages:
1. Update `hot.md` — evict done items, add new threads and next actions; increment `Ingests since last lint`
2. Append a `log.md` entry for work done this session
3. Verify `index.md` has entries for all new pages

Pure query sessions (no wiki modifications): close-out is optional.

---

## Indexing and Logging

Three special files keep the agent oriented as the wiki grows:

**`index.md`** is content-oriented. A catalog of everything in the wiki — each page listed with a link and a one-line summary, organized by category. The agent updates it on every ingest. When answering a query, the agent reads the index first to find relevant pages, then drills into them. This works well at moderate scale (~100 sources, hundreds of pages) and avoids the need for embedding-based infrastructure.

**`log.md`** is chronological. An append-only record of ingests, queries, lint passes, and updates. If each entry starts with a consistent prefix — e.g. `## [YYYY-MM-DD] ingest | Article Title` — the log becomes trivially parseable: `grep "^## \[" log.md | tail -5` gives the last 5 entries. Types: `ingest`, `query`, `lint`, `update`, `setup`.

**`hot.md`** is a session cache — short (~500 tokens), lossy, disposable. The agent updates it at the end of each session with whatever is most relevant right now: active project states, recent decisions, key numbers, immediate next actions. At the start of the next session, it reads this file first. This resolves most recent-context questions without touching the index or log.

`hot.md` is not a source of truth — permanent knowledge belongs in wiki pages; this just points at what's warm. Regenerate it from `log.md` and a handful of relevant wiki pages if it goes stale.

**Suggested `hot.md` contents:**
- Active threads — what's in motion, what's blocked
- Recent decisions — anything decided in the last 1–2 sessions
- Key numbers — metrics, dates, targets likely to be referenced again soon
- Next actions — what you said you'd do next
- Open conflicts — unresolved contradictions between wiki pages
- `Ingests since last lint: N` — a counter the agent increments on each ingest and resets after LINT

**Regenerate `hot.md` when:** its last-updated date is >7 days old, or >3 of its listed active threads are fully done.

---

## Visiting Subagents

If you use subagents (e.g. Claude Code's Agent tool, or similar), they arrive with no context about the wiki. A lightweight `VISITING.md` file solves this — it tells them what the vault is, how to navigate it (read `hot.md` → `index.md` → specific pages), what they'll find, and critically: they may read freely but must not modify `index.md`, `log.md`, `hot.md`, or any wiki page. Modifications are the primary agent's responsibility.

---

## Optional: CLI Tools

At some point you may want small tools that help the agent operate more efficiently. A search engine over the wiki is the most obvious one — at small scale the index file is enough, but as the wiki grows you want proper search. [`qmd`](https://github.com/qmd-lab/qmd) is a good option: local markdown search with hybrid BM25/vector search and LLM re-ranking, all on-device, with both a CLI and MCP server. You could also vibe-code a simpler search script with the agent's help as the need arises.

---

## Tips and Tricks

- **Obsidian Web Clipper** is a browser extension that converts web articles to markdown. Useful for quickly getting sources into `raw/`.
- **Download images locally.** In Obsidian Settings → Files and links, set the attachment folder to `raw/assets/`. Bind "Download attachments for current file" to a hotkey. After clipping an article, hit the hotkey and all images save to disk. LLMs can't read markdown with inline images in one pass — the workaround is to have the agent read the text first, then view referenced images separately for additional context.
- **Obsidian's graph view** is the best way to see the shape of your wiki — what's connected to what, which pages are hubs, which are orphans.
- **Marp** is a markdown slide format with an Obsidian plugin. Useful for generating presentations directly from wiki content.
- **Dataview** is an Obsidian plugin that runs queries over page frontmatter. If your agent writes YAML frontmatter (tags, dates), Dataview can generate dynamic tables and lists — e.g. all pages updated in the last 30 days.

---

## Why This Works

The tedious part of maintaining a knowledge base is not the reading or the thinking — it's the bookkeeping. Updating cross-references, keeping summaries current, noting when new data contradicts old claims, maintaining consistency across dozens of pages. Humans abandon wikis because the maintenance burden grows faster than the value. LLMs don't get bored, don't forget to update a cross-reference, and can touch 15 files in one pass. The wiki stays maintained because the cost of maintenance is near zero.

The human's job is to curate sources, direct the analysis, ask good questions, and think about what it all means. The LLM's job is everything else.

The idea is related in spirit to Vannevar Bush's Memex (1945) — a personal, curated knowledge store with associative trails between documents. Bush's vision was closer to this than to what the web became: private, actively curated, with the connections between documents as valuable as the documents themselves. The part he couldn't solve was who does the maintenance. The LLM handles that.

---

## Note

This document is intentionally abstract. It describes the pattern, not a specific implementation. The exact directory structure, schema conventions, page formats, tooling — all of that depends on your domain, your preferences, and your LLM of choice. Everything above is optional and modular. Pick what's useful, ignore what isn't. Your wiki might be text-only, small enough that the index file is all you need, uninterested in slide decks. That's fine.

---

## Closing Instruction (copy-paste to your agent)

> You are now my LLM Wiki agent. I want to build a personal knowledge base using the pattern described above.
>
> Before scaffolding anything, ask me:
> 1. What is this wiki about? What domain, project, or subject are you tracking?
> 2. What kinds of sources will you be feeding it? (meeting notes, articles, Slack threads, code, etc.)
> 3. What questions do you most want it to answer?
>
> Use my answers to propose a vault structure, schema conventions, and page types that fit this specific domain. Then create `CLAUDE.md` with the full agent schema, scaffold `index.md`, `log.md`, and `hot.md`, create `VISITING.md`, set up the `raw/` and `wiki/` directory structure, and walk me through the first ingest together.
>
> From that point forward, every interaction follows the schema.
