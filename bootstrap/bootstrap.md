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