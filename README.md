# Building an LLM Knowledge Base with Claude Code + Obsidian

A practical, low-friction setup based on Karpathy's [LLM Wiki pattern](https://gist.github.com/karpathy/442a6bf555914893e9891c11519de94f).

---

## The idea in one paragraph

Most "chat with your docs" tools work like RAG: you upload files, and the model re-discovers the relevant fragments from scratch on every question. Nothing accumulates. The LLM-Wiki pattern flips this: instead of retrieving raw chunks at query time, the LLM **incrementally builds and maintains a persistent wiki** — a folder of interlinked markdown notes that sits between you and your raw sources. Every time you add a source, the model reads it, extracts what matters, and folds it into the existing notes: updating entity pages, revising summaries, flagging contradictions, adding cross-links. The knowledge is compiled once and *kept current*, so the wiki compounds with every source and every question. You curate sources and ask questions; the LLM does all the bookkeeping (summarizing, cross-referencing, filing) that makes humans abandon wikis.

**The mental model:** Obsidian is the IDE, Claude Code is the programmer, and the wiki is the codebase. You keep both open side by side — Claude edits the notes from your conversation, you browse the results live (graph view, links, updated pages).

---

## The three layers

| Layer | What it is | Who owns it |
|---|---|---|
| **`raw/`** | Your curated source documents (articles, PDFs, notes, transcripts). Immutable — the model reads but never edits them. This is your source of truth. | You |
| **`wiki/`** | LLM-generated markdown: summaries, entity pages, concept pages, comparisons, an overview. The model creates, updates, and cross-links these. | Claude |
| **`CLAUDE.md`** (the schema) | The config file that tells Claude *how* the wiki is structured, what the conventions are, and exactly what to do when ingesting, querying, or maintaining. This is what turns Claude from a generic chatbot into a disciplined wiki maintainer. | You + Claude, co-evolved |

Two navigation files live at the root of the wiki:

- **`index.md`** — content-oriented catalog. Every page listed with a link and a one-line summary, grouped by category. Updated on every ingest. Claude reads this first when answering a question, then drills into the relevant pages. (This avoids needing embedding-based RAG infrastructure at small-to-moderate scale — roughly hundreds of pages.)
- **`log.md`** — chronological, append-only record of what happened and when. Each entry starts with a consistent prefix like `## [2026-06-01] ingest | Article Title` so it stays greppable: `grep "^## \[" log.md | tail -5`.

---

## One-time setup (≈10 minutes)

1. **Install Obsidian** — <https://obsidian.md>. Create a new vault in an empty folder, e.g. `~/knowledge-base`.
2. **Install Claude Code** — `npm install -g @anthropic-ai/claude-code` (needs Node.js 18+). Docs: <https://docs.claude.com/en/docs/claude-code/overview>.
3. **Open the vault in your terminal and start Claude Code there:**
   ```
   cd ~/knowledge-base
   claude
   ```
   Claude Code now operates inside the same folder Obsidian is showing you.
4. **Run the bootstrap prompt below** (next section). It scaffolds the whole structure for you.
5. **Optional but recommended — Obsidian Web Clipper:** a browser extension that converts web articles to clean markdown. Use it to drop sources into `raw/` with one click. <https://obsidian.md/clipper>
6. **Optional — local images:** In Obsidian → *Settings → Files and links*, set "Attachment folder path" to `raw/assets/`. Then in *Settings → Hotkeys*, bind "Download attachments for current file" to a hotkey (e.g. Ctrl+Shift+D). After clipping an article, hit the hotkey to pull its images to local disk so Claude can view them.

---

## The bootstrap prompt

Paste this into Claude Code once, in your vault folder. It interviews you briefly, then builds the entire structure — folders, the `CLAUDE.md` schema, the `index.md`/`log.md` files, and three reusable slash commands (`/ingest`, `/query`, `/lint`).

> Copy everything between the lines.

```text
You are setting up a personal LLM-Wiki in this directory, following Andrej
Karpathy's LLM-Wiki pattern. The goal is a persistent, compounding knowledge
base of interlinked markdown notes that YOU maintain — not query-time RAG.

STEP 1 — Interview me first. Ask me these questions (one message, wait for my
answers), and do NOT scaffold anything until I reply:
  1. What is this knowledge base ABOUT? (domain / topic / purpose)
  2. What kinds of sources will I feed it? (e.g. web articles, PDFs, papers,
     podcast notes, book chapters, meeting transcripts, my own journal)
  3. What output formats do I care about beyond markdown pages? (e.g.
     comparison tables, slide decks, charts — or "just markdown for now")
  4. Roughly how many sources do I expect? (tens / hundreds / more)

STEP 2 — Once I answer, scaffold the wiki. Tailor the page types and
categories to MY domain (don't blindly use generic ones). Create:

  raw/                  -> empty, with a short README explaining it's my
                           immutable source-of-truth dropbox
  raw/assets/           -> for downloaded images
  wiki/                 -> the notes you will own and maintain
  wiki/index.md         -> catalog: every page linked + one-line summary,
                           grouped by category. Seed it with the structure.
  wiki/overview.md      -> a living top-level synthesis of the whole domain
  log.md                -> append-only, entries prefixed
                           "## [YYYY-MM-DD] <op> | <title>"
  CLAUDE.md             -> the schema (details below)
  .claude/commands/ingest.md, query.md, lint.md -> slash commands (below)

STEP 3 — Write CLAUDE.md as the operating contract. It must define, concretely
(not vaguely — be specific enough that a fresh session behaves identically):
  - The directory layout and what each folder/file is for.
  - Page conventions: filename style (kebab-case), required YAML frontmatter
    (title, type, tags, created, updated, source_count), and the use of
    [[wikilinks]] for every entity/concept mentioned so the Obsidian graph
    stays connected.
  - The page TYPES for my domain (e.g. source summary, entity, concept,
    comparison, overview) and a short template for each.
  - The INGEST workflow, as an explicit checklist: read the source -> tell me
    the key takeaways and ASK what to emphasize -> write/update a source
    summary page -> create or UPDATE relevant entity & concept pages (search
    the existing wiki FIRST and edit rather than duplicate) -> add wikilinks
    both directions -> flag any contradictions with existing pages inline ->
    update index.md -> append a log.md entry. Note that a single source may
    touch 10-15 pages.
  - The QUERY workflow: read index.md first, drill into relevant pages,
    answer WITH citations to the source pages, and then OFFER to file good
    answers back into the wiki as their own page so explorations compound.
  - The LINT workflow: scan for contradictions, stale claims superseded by
    newer sources, orphan pages (no inbound links), important concepts
    mentioned but lacking a page, missing cross-references, and gaps a web
    search could fill. Report findings + suggest new questions/sources.
  - A rule that you NEVER modify anything under raw/.

STEP 4 — Write the three slash commands so the workflows are one keystroke and
never drift. Use this shape (filename = command name):

  .claude/commands/ingest.md  -> frontmatter "description: Ingest a source
    into the wiki" and body: "Ingest the source: $ARGUMENTS  (a path under
    raw/, or paste text). Follow the INGEST workflow in CLAUDE.md exactly,
    step by step. Do not skip the index.md and log.md updates."

  .claude/commands/query.md   -> "description: Ask the wiki a question" and
    body: "Answer this against the wiki: $ARGUMENTS  Follow the QUERY workflow
    in CLAUDE.md. Cite source pages. End by offering to file the answer back
    in as a page."

  .claude/commands/lint.md    -> "description: Health-check the wiki" and
    body: "Run the LINT workflow in CLAUDE.md across the whole wiki and give
    me a prioritized report plus suggested next sources and questions."

STEP 5 — After creating everything, show me the tree, summarize how to use
the three commands, and tell me to type /ingest with my first source.

Keep CLAUDE.md tight and skimmable — it's read every session, so favor clear
imperative checklists over prose.
```

> **Why slash commands?** A known failure of this pattern is the wiki going stale because the model "forgets" to do the bookkeeping — putting *"remember to update the index"* in `CLAUDE.md` isn't reliable on its own. Encoding the ingest/query/lint workflows as explicit, repeatable commands makes the disciplined behavior the default path rather than something the model has to remember.
>
> *(Note: `.claude/commands/*.md` is still fully supported; Claude Code's newer equivalent is `.claude/skills/<name>/SKILL.md`, which also adds autonomous invocation. Commands are simpler to start with — you can graduate to skills later.)*

---

## The daily workflow (the low-friction loop)

Once bootstrapped, the entire system is three commands:

1. **Add a source.** Clip a web article with Web Clipper, drop a PDF, or paste notes into `raw/`.
2. **`/ingest raw/that-article.md`** — Claude reads it, tells you the takeaways, asks what to emphasize, then writes the summary, updates every relevant page, adds cross-links, flags contradictions, and updates `index.md` + `log.md`. One source typically touches 10–15 pages.
3. **`/query what's the strongest evidence for X?`** — Claude reads the index, pulls the relevant pages, answers with citations, and offers to file the answer back as a new page so your explorations compound too.
4. **`/lint`** (occasionally) — Claude health-checks the whole wiki: contradictions, stale claims, orphan pages, missing concept pages, and gaps worth researching. Great for finding your next questions.

Ingest sources **one at a time** while you're learning the pattern — read the summaries, check the updates, steer what gets emphasized. Once you trust it, you can batch-ingest with less supervision.

---

## Useful extras

- **Graph view** (Obsidian): the best way to see the *shape* of your wiki — hubs, clusters, and orphan pages. Open it while Claude works.
- **Git:** the wiki is just markdown files, so `git init` gives you version history, diffs of every ingest, branching, and easy team sharing — for free.
- **Dataview** (Obsidian plugin): if Claude adds YAML frontmatter (it will, per the schema), Dataview generates dynamic tables from it (e.g. "all sources from this month sorted by date").
- **Marp** (Obsidian plugin): turn any wiki page into a markdown slide deck — handy when a query answer should become a presentation.
- **A search engine, eventually:** at hundreds of pages, `index.md` is enough. Beyond that, a local markdown search tool (e.g. [qmd](https://github.com/tobi/qmd) — on-device BM25 + vector + rerank, usable as a CLI or MCP server) lets Claude search instead of scanning the index. Add it only when you feel the need.

---

## Why this works

The hard part of a knowledge base was never the reading or the thinking — it's the *bookkeeping*: keeping summaries current, updating cross-references, noticing when new data contradicts old claims, staying consistent across dozens of pages. Humans abandon wikis because that maintenance burden grows faster than the value. An LLM doesn't get bored, doesn't forget a cross-reference, and can touch 15 files in one pass — so the maintenance cost drops to near zero and the wiki actually stays alive. Your job is to curate sources, direct the analysis, and ask good questions. Everything else is the model's job.

It's the same spirit as Vannevar Bush's 1945 Memex — a private, actively curated store where the *connections* between documents are as valuable as the documents. Bush couldn't solve who does the maintenance. Now the LLM does.