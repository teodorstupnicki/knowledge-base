# Building an LLM Knowledge Base with Claude Code + Obsidian

A practical guide to turning an Obsidian vault into a queryable, self-maintaining knowledge base — with Claude Code as the agent that ingests, distills, links, and queries your notes.

---

## The idea in one paragraph

Obsidian gives you a folder of plain Markdown files connected by `[[wikilinks]]`. Claude Code is a CLI agent that reads and writes files in a directory and follows instructions you put in a `CLAUDE.md`. Combine them and you get a knowledge base where (a) you own the data as plain text, (b) an LLM can navigate the link graph the same way you do, and (c) the same agent that *queries* the vault is the one that *maintains* it. The methodology underneath this — what Karpathy calls a "knowledge compilation pipeline" — is: raw input → distilled atomic notes → linked graph → synthesized answers. Each layer is a transformation Claude Code can do for you on demand.

---

## Architecture

```
┌──────────────────────────────────────────────────────────────┐
│  Storage layer — Obsidian vault (plain .md on disk)          │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐      │
│  │ 0-inbox/ │  │ 1-notes/ │  │ 2-mocs/  │  │ 3-sources│      │
│  │ raw      │  │ atomic   │  │ Maps of  │  │ articles │      │
│  │ capture  │  │ ideas    │  │ Content  │  │ books    │      │
│  └──────────┘  └──────────┘  └──────────┘  └──────────┘      │
└──────────────────────────────────────────────────────────────┘
            ▲                                  ▲
            │ reads/writes                     │ reads
            │                                  │
┌───────────┴──────────────────────────────────┴───────────────┐
│  Agent layer — Claude Code                                   │
│  • CLAUDE.md     → vault conventions                         │
│  • .claude/commands/  → slash commands (/distill, /query …)  │
│  • .claude/agents/    → sub-agents (optional)                │
└──────────────────────────────────────────────────────────────┘
            ▲
            │
┌───────────┴──────────────────────────────────────────────────┐
│  Interface layer                                             │
│  • Obsidian (read, edit, graph view)                         │
│  • Claude Code CLI (ingest, distill, query)                  │
└──────────────────────────────────────────────────────────────┘
```

The vault on disk is the single source of truth. Obsidian is your reading/editing UI. Claude Code is the agent. They never fight because they both just touch Markdown files.

---

## Prerequisites

- **Obsidian** — install from obsidian.md (free)
- **Node.js 18+** — required by Claude Code
- **Claude Code** — `npm install -g @anthropic-ai/claude-code`
- **An Anthropic API key or Claude subscription** — for Claude Code auth
- **Git** — strongly recommended for vault history

You don't need any plugins to start. Once it's working you'll probably want Obsidian's *Dataview* and *Templater* plugins, but skip them for now.

---

## Step 1 — Set up the vault

Create the vault directory and open it in Obsidian once so it gets initialized:

```bash
mkdir -p ~/vaults/kb
cd ~/vaults/kb
git init
```

Then in Obsidian: *Open folder as vault* → pick `~/vaults/kb`.

### Folder structure

There are religious wars about Obsidian folder structure. The structure below is opinionated but works well for an LLM-driven KB because each folder maps to a *stage in the compilation pipeline*:

```
kb/
├── 0-inbox/        # raw, unprocessed capture (paste here, distill later)
├── 1-notes/        # atomic notes — one idea per file
├── 2-mocs/         # Maps of Content — index notes that link clusters
├── 3-sources/      # source material: articles, papers, books, transcripts
├── 4-daily/        # daily notes (optional)
├── _attachments/   # images, PDFs
├── _templates/     # note templates
└── CLAUDE.md       # instructions for Claude Code
```

The numeric prefixes keep them sorted in the file tree by pipeline stage.

### Naming convention

Pick **one** and never change it. I recommend:

- Atomic notes: `Title Case With Spaces.md` (e.g. `Eventual Consistency.md`)
- MOCs: `MOC - Topic.md` (e.g. `MOC - Distributed Systems.md`)
- Sources: `SRC - Author - Title.md`
- Daily: `YYYY-MM-DD.md`

This matters because Claude Code will be creating files autonomously — a strict convention prevents drift.

### Atomic note template

Save this as `_templates/atomic.md`:

```markdown
---
created: {{date}}
tags: []
status: seedling
---

# {{title}}

> One-line claim or definition.

## Body

…

## Related
- [[ ]]

## Sources
- [[SRC - …]]
```

`status` follows the digital-garden convention: `seedling` (rough), `budding` (developed), `evergreen` (stable). Claude Code will respect and update this field.

---

## Step 2 — Bootstrap Claude Code in the vault

```bash
cd ~/vaults/kb
claude
```

First run will prompt for auth. Once you're in the REPL, run:

```
/init
```

This generates a starter `CLAUDE.md`. We're going to replace it with something tailored.

---

## Step 3 — Write CLAUDE.md

This is the most important file in the system. It's loaded automatically into Claude Code's context every session and tells Claude *how this specific vault works*. Treat it like the README you'd give a new teammate.

Save this as `~/vaults/kb/CLAUDE.md`:

```markdown
# Vault operating manual

This is a personal knowledge base built on Obsidian conventions.
You are the maintenance agent. Read this file fully before any task.

## Folder semantics

- `0-inbox/`       → raw, unprocessed. Anything here is fair game to refactor.
- `1-notes/`       → atomic notes, one idea each. Stable filenames; rename only with caution.
- `2-mocs/`        → Maps of Content. Index notes that link clusters of atomic notes.
- `3-sources/`     → source material (articles, papers, transcripts). Treat as read-mostly.
- `4-daily/`       → daily journals. Don't edit past days unless asked.
- `_attachments/`  → binary files. Don't touch.
- `_templates/`    → note templates. Use them; don't modify without asking.

## Atomic note rules

1. One idea per note. If a note grows two distinct theses, split it.
2. Title is the claim or concept itself ("Eventual Consistency", not "Notes on EC").
3. First line under the H1 is a one-sentence summary or definition. Always keep this.
4. Use `[[wikilinks]]` liberally. A note with zero outgoing links is a smell.
5. Frontmatter `status` field: seedling → budding → evergreen. Bump when you substantially expand a note.
6. Tags in frontmatter, lowercase, kebab-case (e.g. `distributed-systems`).

## Linking rules

- When you create a new atomic note, search the vault for 2–5 existing notes it should link to and add them under `## Related`.
- When you reference a concept that doesn't yet have a note, create a stub link `[[New Concept]]` — Obsidian shows these as unresolved and they become a backlog.
- Never invent a link target you didn't verify exists or didn't create yourself.

## MOC rules

- An MOC is itself a note. It contains a short framing paragraph plus structured links.
- When a topic has 5+ atomic notes, propose creating an MOC.
- MOCs link *to* atomic notes; atomic notes link *back to* their MOC under `## Related`.

## Sources

- Every claim that came from a specific source must cite the source note under `## Sources`.
- If the source isn't yet in `3-sources/`, create it with a `SRC - Author - Title.md` stub.

## What you should never do

- Never delete a note without explicit confirmation.
- Never rename a file without explicit confirmation (it breaks links).
- Never edit `_templates/` or `_attachments/`.
- Never collapse multiple atomic notes into one without confirmation.
- Don't make up content for fields you don't know — leave them blank.

## Default workflow when given content to ingest

1. Drop verbatim into `0-inbox/` with a timestamped filename.
2. Read it. Identify atomic ideas (typically 1–5 per inbox item).
3. For each idea: check if an atomic note already exists. If yes, propose updates. If no, draft a new atomic note.
4. Link new notes to existing related notes (search the vault first).
5. If a source is involved, create or update the source note in `3-sources/`.
6. Report what you did as a short summary with file paths.

## Default workflow when asked a question

1. Search the vault for relevant atomic notes (use grep/glob, not memory).
2. Cite the notes you used by filename in your answer.
3. If the answer is partial because notes are missing, say so explicitly and suggest what to add.
```

Tweak this to taste, but keep the structure. The "what you should never do" section is what stops Claude from being too eager and shredding your vault.

---

## Step 4 — Core workflows

You now have everything to start using the system. The four workflows below cover ~90% of daily use.

### 4.1 Ingest

You read an article, paste a transcript, or dump a chat. Drop the raw content into `0-inbox/` (manually or via Claude Code) and ask:

```
> Distill the latest item in 0-inbox into atomic notes. Link to existing notes where possible.
```

Claude will read the inbox file, search `1-notes/` for related concepts, and propose new notes plus updates to existing ones. Review before accepting — your judgment is the quality filter.

### 4.2 Distill

Sometimes a note in `1-notes/` has grown bloated. Ask:

```
> Read 1-notes/Distributed Systems.md and propose a split into atomic notes.
```

Claude will identify the distinct ideas and suggest filenames + content. Accept the ones you like.

### 4.3 Link

Run a periodic linking pass:

```
> Audit 1-notes/ for orphan notes (zero incoming or outgoing links). For each orphan, suggest 2–3 plausible links based on content.
```

This is the maintenance Claude is best at — boring, mechanical, and high-leverage.

### 4.4 Query

This is where the KB pays off:

```
> Using only notes in this vault, explain the tradeoffs between active-active and active-passive failover. Cite the specific notes you used.
```

Because of the CLAUDE.md rules, Claude will grep the vault, pull the relevant atomic notes, synthesize, and cite filenames. If the KB is missing context, it will tell you — and you can ingest a source to fill the gap.

---

## Step 5 — Custom slash commands

Repetitive operations deserve a slash command. Claude Code reads markdown files in `.claude/commands/` and exposes them as `/command-name`.

Create `.claude/commands/distill.md`:

```markdown
---
description: Distill the most recent inbox item into atomic notes
---

Find the most recently modified file in `0-inbox/`. Read it.

For each distinct idea in that file:
1. Search `1-notes/` to check if it already exists.
2. If it exists, propose specific edits to expand or refine it.
3. If it doesn't, draft a new atomic note using `_templates/atomic.md`.
4. For every new note, find 2–5 related existing notes and link to them.

If the content has a clear source, ensure a note exists in `3-sources/`.

Output a plan first. Wait for approval before writing files.
```

Now in any session: `/distill` and Claude executes that workflow.

A few more worth having:

`.claude/commands/orphans.md`:

```markdown
---
description: Find and triage orphan notes
---

List all notes in `1-notes/` with zero outgoing wikilinks OR zero incoming wikilinks.
For each, output:
- filename
- one-line summary (read the note)
- 2–3 suggested links with reasoning

Don't edit anything. Just produce the report.
```

`.claude/commands/moc.md`:

```markdown
---
description: Generate or refresh a Map of Content for a topic
---

Topic: $ARGUMENTS

1. Search `1-notes/` for all notes related to the topic.
2. Group them into 3–6 sub-themes.
3. Either create `2-mocs/MOC - {topic}.md` or refresh it if it exists.
4. The MOC should have:
   - A 2–3 sentence framing paragraph.
   - Sub-theme headings, each with bulleted wikilinks to atomic notes.
   - A "Loose threads" section listing unresolved questions or gaps.

Show me the MOC content before writing.
```

Use as `/moc Distributed Systems`.

`.claude/commands/query.md`:

```markdown
---
description: Answer a question using only the vault
---

Question: $ARGUMENTS

1. Grep/glob the vault for notes relevant to the question.
2. Read the matched notes fully.
3. Synthesize an answer.
4. Cite each claim with the source note filename in the form `[[Note Title]]`.
5. If the vault is missing context to answer fully, end with a "Gaps" section listing what's missing.

Do not use information not present in the vault.
```

Use as `/query What are the failure modes of Raft under network partition?`.

---

## Step 6 — Optional: sub-agents for separation of concerns

Claude Code supports **sub-agents** (`.claude/agents/*.md`) — specialized agents with their own system prompts and tool restrictions. Useful when you want different behavior for different jobs.

Example: a *librarian* sub-agent with read-only access for queries, and a *gardener* sub-agent that does write-heavy maintenance. Keeps the query agent from accidentally mutating files mid-answer.

Skip this until the basics feel solid — it adds complexity for marginal gain early on.

---

## Step 7 — Optional: MCP for external sources

If you want to ingest from external systems automatically (Slack, Gmail, GitHub, web), wire them in via MCP servers configured in Claude Code. The flow becomes:

```
External source → MCP → Claude Code → 0-inbox/ → distill → 1-notes/
```

For your context — given the cloud/Azure work — an MCP server that pulls from a specific Azure DevOps Wiki or a GitHub repo could autopopulate `3-sources/` with internal docs.

Again: skip until the manual flow is habitual.

---

## Daily loop, in practice

A realistic week:

- **Capture** throughout the day: paste interesting things into `0-inbox/`, write half-formed thoughts into `4-daily/`.
- **Once a day** (or every few days): run `/distill` until inbox is empty.
- **Once a week**: run `/orphans` and clean up dangling notes; refresh the MOC for whatever topic you're actively learning.
- **On demand**: `/query` when you need to retrieve or synthesize.
- **`git commit`** at the end of each session. Vault history is gold.

The vault gets denser and more useful as the link graph thickens. After ~3 months of consistent use, queries become noticeably more powerful because Claude has a real graph to traverse.

---

## Gotchas

- **Filename instability is fatal.** Once a note is created and linked, renaming breaks every backlink. Lock down the filename when status hits `budding`.
- **Don't let the inbox rot.** If `0-inbox/` has 50 items, distillation feels like a chore and you'll stop. Keep it under 10.
- **Beware the "atomic" trap.** Notes can be too small. If a note is one sentence with no body, it's probably a property of another note, not a note of its own.
- **Claude will hallucinate links if you let it.** The CLAUDE.md rule "never invent a link target" matters — without it, you'll get notes referencing files that don't exist.
- **Source quality is the ceiling.** Your KB is only as good as what you feed it. Garbage transcripts → garbage atomic notes.
- **Plain text is a feature, not a limitation.** Resist the urge to store everything in databases or vector stores. Markdown + grep + an LLM beats most "real" KB systems for personal use.

---

## What this gets you that ChatGPT-with-files doesn't

1. **Persistence and ownership** — the vault is yours, on disk, in version control.
2. **Incremental quality** — every interaction makes the KB better, not just that one answer.
3. **Citation and traceability** — every synthesized answer points back to source notes.
4. **Composable** — the same vault is queryable by Obsidian, grep, ripgrep, any LLM, any RAG pipeline you bolt on later.
5. **Auditable** — git diff shows exactly what changed every session.

---

## Where to take it next

- Add **Dataview** plugin in Obsidian to query frontmatter (e.g. all `status: seedling` notes).
- Add a `_scripts/` directory with Python tools Claude can call (e.g. a script that emits the link graph as JSON).
- Wire an **MCP server** so a second LLM (a smaller, faster one) can query the vault for cheap lookups while Claude Code handles writes.
- Periodic **eval**: pick 10 questions you should be able to answer from the KB, run them as `/query`, score the results. Iterate the CLAUDE.md based on failure modes.

That last one is the most underrated. The CLAUDE.md is a prompt you tune over time — same as any other system prompt — and the only honest way to tune it is against real queries.
