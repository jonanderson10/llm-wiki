# LLM Wiki

My take on Andrej Karpathy's LLM wiki, extended with conventions refined from running a live wiki — a named UPDATE operation for conversational facts, an automatic LINT counter, lazy session-start loading, a subagent briefing pattern via `VISITING.md`, and a bootstrapping prompt that asks for domain context before scaffolding anything.

> **Credit:** Original concept by [Andrej Karpathy](https://gist.github.com/karpathy/442a6bf555914893e9891c11519de94f)

## The idea

Most LLM-document workflows look like RAG: retrieve relevant chunks at query time, generate an answer, repeat. Nothing accumulates. This pattern is different: the LLM builds a persistent wiki that sits between you and your raw sources. Every new source is read, synthesized, and integrated into the existing knowledge base — updating entity pages, flagging contradictions, strengthening cross-references. The wiki compounds over time. You read it; the LLM writes it.

See [`setup-prompt.md`](./setup-prompt.md) for the full pattern, conventions, and a closing instruction you can paste directly into your agent to get started.

## What's in this repo

- **`setup-prompt.md`** — The idea file and bootstrapping prompt. Copy-paste it to your LLM agent of choice (Claude Code, Codex, etc.) to instantiate a wiki for your domain.

## What's different from the original

Karpathy's gist describes the core pattern well — the three-layer architecture, INGEST/QUERY/LINT operations, the role of `index.md`, `log.md`, and `hot.md`. This version keeps all of that and adds conventions that emerged from running a live wiki over several months:

**New operations**
- **UPDATE** — a named operation for pushing a fact into the wiki without a source file. In practice, plenty of knowledge arrives in conversation (a decision made in a meeting, a number that changed) and disappears into chat history without this. Karpathy's original only covers INGEST, QUERY, and LINT.

**Smarter session management**
- **Lazy session start** — `hot.md` is always read eagerly; `index.md` and the log are only read if `hot.md` doesn't resolve the question. Keeps short sessions cheap.
- **Inbox check at session start** — the original suggests draining the inbox only when you trigger an ingest. Checking at every session start means files never sit unclassified for more than one session.

**Operational hygiene**
- **LINT counter** — an `Ingests since last lint: N` line in `hot.md` that the agent increments on each ingest and resets after a lint pass. The original suggests periodic linting but provides no mechanism to track when it's due.
- **Conflict resolution path** — unresolved contradictions between pages are logged in `hot.md` under `## Open Conflicts` until you resolve them. The original says to flag conflicts inline but doesn't say what to do with them after.
- **`runbooks/`** added to the vault structure, separate from `analyses/`. Runbooks are executed repeatedly; analyses are read once. Mixing them buries operational docs.

**Subagent support**
- **`VISITING.md`** — a lightweight briefing file for visiting subagents. Tells them what the vault is, how to navigate it, and that they may read freely but must not write. Not in the original.

**Better bootstrapping**
- The closing instruction now asks for domain context (what is this wiki about, what sources, what questions) *before* scaffolding. The original jumps straight to "create the CLAUDE.md schema" — the schema should emerge from the domain, not be imposed on it.
