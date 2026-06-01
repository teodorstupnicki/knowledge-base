You are setting up a personal LLM-Wiki in this directory, following Andrej
Karpathy's LLM-Wiki pattern. The goal is a persistent, compounding knowledge
base of interlinked markdown notes that YOU maintain across sessions — not
query-time RAG. You will use Claude Code's Skills system (.claude/skills/) so
the workflows are reusable, auto-loaded when relevant, and keep the main
session context lean.

STEP 1 — Interview me first. Ask these questions in one message and WAIT for my
answers. Do not scaffold anything until I reply:
  1. What is this knowledge base ABOUT? (domain / topic / purpose)
  2. What kinds of sources will I feed it? (web articles, PDFs, papers,
     podcast notes, book chapters, meeting transcripts, my own journal, etc.)
  3. What output formats do I care about beyond markdown pages? (comparison
     tables, slide decks, charts — or "just markdown for now")
  4. Roughly how many sources do I expect? (tens / hundreds / more)

STEP 2 — Once I answer, scaffold the structure. Tailor page types and
categories to MY domain (don't blindly use generic ones). Create:

  raw/                 -> empty, with a short README.md saying it's my
                          immutable source-of-truth dropbox that you never edit
  raw/assets/          -> for downloaded images
  wiki/                -> the notes you own and maintain
  wiki/index.md        -> catalog: every page linked + one-line summary,
                          grouped by category; seed it with the structure
  wiki/overview.md     -> a living top-level synthesis of the whole domain
  log.md               -> append-only; entries prefixed
                          "## [YYYY-MM-DD] <op> | <title>"
  CLAUDE.md            -> the lean schema (see STEP 3)
  .claude/skills/ingest/SKILL.md     (+ templates.md supporting file)
  .claude/skills/query/SKILL.md
  .claude/skills/lint/SKILL.md

STEP 3 — Write CLAUDE.md as a LEAN operating contract (it loads every session,
so keep it tight and skimmable — imperative bullets, not prose). It defines
only the always-true facts; the detailed workflows live in the skills. Include:
  - The directory layout and what each folder/file is for.
  - Page conventions: kebab-case filenames; required YAML frontmatter on every
    wiki page (title, type, tags, created, updated, source_count); and the rule
    that every entity/concept mentioned gets a [[wikilink]], added in BOTH
    directions, so the Obsidian graph stays connected.
  - The hard rule: NEVER create, edit, move, or delete anything under raw/.
  - A short "How this wiki is operated" note pointing to the three skills:
    /ingest to add a source, /query to ask a question, /lint to health-check;
    and that you may auto-load the query skill when I ask a question in natural
    language.
  - Do NOT duplicate the full workflow checklists here — they belong in the
    skills so they only load when needed.

STEP 4 — Create the three skills. Each is a DIRECTORY containing SKILL.md with
YAML frontmatter (name, description, and invocation control). Keep each
SKILL.md focused; descriptions are what I and you use to trigger them, so make
them precise.

  .claude/skills/ingest/SKILL.md
    Frontmatter:
      name: ingest
      description: Read a source (a path under raw/, or pasted text) and
        integrate it into the wiki — summary page, updated entity/concept
        pages, cross-links, contradiction flags, index and log updates.
      disable-model-invocation: true   # side effects + I control timing
      allowed-tools: Read, Write, Edit, Glob, Grep, Bash
    Body — an explicit, ordered checklist (a single source may touch 10-15
    pages):
      1. Read the source named in $ARGUMENTS. If it references local images in
         raw/assets/, view them too for extra context.
      2. Tell me the key takeaways and ASK what to emphasize before writing.
      3. Write or update a source-summary page in wiki/ (use the source
         template in templates.md).
      4. SEARCH the existing wiki FIRST (Glob/Grep). For each entity/concept in
         the source, EDIT the existing page if one exists; only create a new
         page when nothing fits. Never duplicate.
      5. Add [[wikilinks]] in both directions between the new/updated pages.
      6. Where the source contradicts an existing page, flag it inline with a
         short note (e.g. "> ⚠️ Contradicts [[page]]: ...").
      7. Update wiki/index.md (add/adjust the catalog entry + one-line summary).
      8. Append a log.md entry: "## [YYYY-MM-DD] ingest | <title>".
      9. Report which pages you touched.
    Also create .claude/skills/ingest/templates.md holding the short per-type
    page templates (source summary, entity, concept, comparison, overview) with
    the required frontmatter filled in. Reference it from SKILL.md.

  .claude/skills/query/SKILL.md
    Frontmatter:
      name: query
      description: Answer a question against the wiki with citations. Use this
        whenever the user asks a question that the knowledge base should
        answer, whether they type /query or just ask in natural language.
      # leave model-invocable so natural-language questions trigger it
    Body:
      1. Read wiki/index.md first to find candidate pages.
      2. Drill into the relevant pages and read them.
      3. Answer the question in $ARGUMENTS, citing the source pages you used
         (link them with [[wikilinks]]).
      4. If a web search would meaningfully improve the answer and tools allow,
         say so / use it, and mark which claims came from outside the wiki.
      5. END by offering to file the answer back into the wiki as its own page
         (so explorations compound), and append a log.md entry if I accept.

  .claude/skills/lint/SKILL.md
    Frontmatter:
      name: lint
      description: Health-check / audit the whole wiki. Use ONLY when the user
        explicitly asks to lint, audit, or health-check the wiki.
      disable-model-invocation: true   # I trigger this deliberately
      allowed-tools: Read, Glob, Grep, Bash
    Body — scan the whole wiki and produce a PRIORITIZED report covering:
      - contradictions between pages
      - stale claims superseded by newer sources
      - orphan pages (no inbound links)
      - important concepts mentioned but lacking their own page
      - missing cross-references
      - gaps a web search could fill
      Then suggest new questions to investigate and new sources to look for.
      Append a log.md entry: "## [YYYY-MM-DD] lint | <summary>".

STEP 5 — After creating everything: show me the file tree, confirm the three
skills are registered (note I can also run /help to see them), summarize how to
use /ingest, /query, and /lint, and tell me to run /ingest with my first
source.

Design rules to follow throughout:
  - CLAUDE.md = always-on, minimal. Skills = on-demand, detailed. Don't put the
    workflow checklists in CLAUDE.md.
  - Always set `name` explicitly in each skill's frontmatter (don't rely on the
    folder-name fallback).
  - Keep each SKILL.md well under 500 lines; move bulky reference material into
    supporting files in the same skill directory.