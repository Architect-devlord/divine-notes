# CLAUDE.md — Divine_Notes

This is an LLM-maintained wiki covering the entire Divine World project — design concepts and actual implementation, together — following the pattern in `LLM_Wiki_Pattern.md` (same folder). Read that first if you haven't, then `index.md`. Short version: you own `wiki/` entirely. Devlord curates sources, directs emphasis, and corrects you when you get something wrong (this has already happened once — see `log.md`'s `oracle_stuff` correction — and the fix was to record the correction, not just quietly apply it).

This structure lives inside/alongside the `divine-world-core` repo — that's why `CLAUDE.md` sits at this level rather than nested deeper, so Claude Code picks it up automatically.

## Structure

```
Divine_Notes/
├── CLAUDE.md              ← you are here
├── index.md                ← content catalog, covers design + codebase + a pointer to raw/. Read first.
├── log.md                   ← append-only chronological record, one shared timeline for both halves
├── LLM_Wiki_Pattern.md        ← the source pattern doc
├── Wiki.base                   ← dynamic table view (Obsidian Bases)
├── raw/                          ← Devlord's own original notes. SELF-CONTAINED. See boundary rule below.
│   ├── index.md                   ← the ONLY thing outside raw/ is allowed to link to
│   └── *.md                        ← individual raw notes; can link to each other freely
└── wiki/
    ├── design/                      ← concept pages: robot bodies, bases, cross-reality architecture
    │   ├── Robots.md                  ← the hub. Stays at this level, NOT inside Robots/.
    │   ├── Robots/                      ← the 7 body pages (Normal_Agents + the 6 God Agents)
    │   │   └── *.md
    │   └── Bases/                        ← the 7 base pages
    │       └── *.md
    └── codebase/                      ← implementation pages: what divine-world-core actually does
        └── *.md
```

Devlord organized `wiki/design/` this way himself (moved the flat files into `Robots/`/`Bases/` after the last delivery) — mirror this layout for any new design page rather than dropping it flat into `wiki/design/`. A new robot body page goes in `Robots/`, a new base page goes in `Bases/`, anything hub-level (like `Robots.md` itself) stays at the `wiki/design/` root.

## The raw/ boundary rule — read this before touching anything in or near raw/

Devlord's own words: _"the files inside the raw folder can be interconnected but not with the files outside the raw folder."_ Concretely:

- Files inside `raw/` may link to each other freely (`raw/index.md` ↔ any `raw/*.md`, or `raw/*.md` ↔ `raw/*.md`).
- **Nothing outside `raw/` links to a specific file inside it.** If a `wiki/design/` or `wiki/codebase/` page needs to point at the raw notes, it points at `raw/index.md` — never at `raw/Creaking.md` or any other individual file.
- **Nothing inside `raw/` links outside `raw/`.** Don't add a link from a raw note back to a `wiki/` page, even if it's the note that inspired that page.
- Why: this lets Devlord rename, split, merge, reorganize, or delete anything inside `raw/` at any time without breaking a single link anywhere else in the vault. `raw/` is explicitly temp/working space he manages himself — the wiki's job is to know it exists and point at its index, not to depend on its internal shape.

**The filename collision this creates, and how it's handled**: `raw/` and `wiki/design/` currently share 8 filenames — `Robots`, `Normal_Agents`, `Creaking`, `Elder_Guardian`, `Warden`, `Ender_Dragon`, `Wither`, `Oracle` (each raw note became a same-named, differently-structured wiki page). A bare `[[Creaking]]` is genuinely ambiguous once both exist in one vault — Obsidian can't tell which one you mean. **Every link to one of these 8 names, anywhere in the vault, must be path-qualified**: `[[wiki/design/Robots/Creaking|Creaking]]` or `[[raw/Creaking|Creaking]]`, never bare `[[Creaking]]`. The `|Creaking` alias keeps the visible text identical to what a bare link would've shown — this is purely a resolution fix, not a visible change. If you add a new page whose name collides with something in `raw/`, apply the same treatment immediately, don't wait for it to cause a problem.

Every other page name in this vault (the 7 base pages, all 11 codebase pages) is unique vault-wide and can use plain bare `[[Name]]` links without qualification.

## Conventions

**Page structure**: YAML frontmatter (design pages: `type`/`category`/`serves`/`status`; codebase pages: `type`/`status`), a `💡` opening line stating what the page is, `##`/`###` section headers, a closing insight where it fits (design pages use `💡 **Key Insight**`).

**Terminology** (design side): **Civilian Agent** (mind, formerly "NPC") pilots a **Normal Agent** (body). **God Agent** (mind, formerly "god") pilots one of six specialized bodies. See [[wiki/design/Robots|Robots]] → Terminology before reintroducing either retired term anywhere new.

**Accuracy over completeness** (codebase side): if you haven't read a file, don't write a page asserting what it does beyond what's inferable from how already-read files call it. `index.md`'s "Identified, not yet ingested" list exists so partial knowledge is recorded honestly rather than either invented or omitted.

**Cross-linking between design and codebase**: encouraged, not required for every page. Add a link where a reader of one side would genuinely benefit from the other (see [[wiki/design/Robots/Oracle|Oracle]] ↔ [[wiki/codebase/oracle-two-systems|oracle-two-systems]] for the model example) — don't force a connection that isn't there.

**When code and a design page disagree**: don't silently "fix" either one. Note the discrepancy explicitly (see [[wiki/codebase/known-issues|known-issues]]'s save-cadence example) — sometimes the code is right and the design doc is stale, sometimes the design doc is describing a future target the code hasn't caught up to yet, and either way the discrepancy itself is the useful fact to preserve.

**Never link to `docs/` or `py_backend/oracle_stuff/`** as if current — both confirmed outdated by Devlord. See [[wiki/codebase/known-issues|known-issues]] and [[wiki/codebase/oracle-two-systems|oracle-two-systems]].

## Workflows

### Ingest (new source arrives — either a new raw note or unread code)

1. If it's a new raw brainstorm note: add the file to `raw/`, add an entry to `raw/index.md`. Don't touch `wiki/` yet unless asked to synthesize it.
2. If it's meant to become a design page: read it, discuss key takeaways with Devlord before committing to a synthesis, write/update the `wiki/design/` page(s), respecting the boundary rule above if it originated from `raw/`.
3. If it's unread source code: read enough to write an accurate page — full-depth reads of every file aren't a realistic bar for a codebase this size. Write/update `wiki/codebase/` page(s).
4. Either way: update `index.md`, append a `log.md` entry.

### Code changes (you're the one modifying divine-world-core, not just reading it)

Update the relevant `wiki/codebase/` page(s) in the same session, not later. A wiki that drifts from the code is worse than no wiki. If a `# FIX (...)`-style comment in the code explains _why_ something changed, that reasoning belongs in the wiki page too.

### Query

Read `index.md` first, drill into the pages it points to, read underlying source/raw notes directly if a wiki page doesn't have enough depth. If the answer required real synthesis and is durable, offer to file it back as a wiki update.

### Lint (periodic or on request)

Check: pages describing code that's since changed; the "Identified, not yet ingested" list still being accurate; orphan pages; any bare link to one of the 8 colliding names that should be path-qualified; anything in `raw/` that's been linked to individually instead of through its index; anything "retired"/"deprecated" quietly reintroduced as current.

## Known open items (as of 2026-07-03)

- The Electron frontend (`dw_agent/`) is the top codebase-ingest priority now — the last major untouched area, and genuinely unknown how much of what it visualizes duplicates vs. extends what's already documented.
- No physical robot hardware sourced/built yet — every design-side component recommendation is a plan, not a confirmed BOM.
- [[wiki/design/Robots/Ender_Dragon|Ender_Dragon]] and [[wiki/design/Robots/Wither|Wither]] both have an open local-vs-central-compute question their own Sanctuary/Citadel bases are meant to resolve with real thrust data.
- `AUDIT_REPORT.md`'s critical duplicate-Config-class finding — not confirmed fixed as of the last codebase ingest.
- This vault hasn't had a lint pass yet — young enough that issues are unlikely to have accumulated, but worth doing once a few more real ingests land on either side.