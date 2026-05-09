# claude-zotero-skills

Three Claude skills that turn Claude (via the [Cowork](https://www.anthropic.com/claude) desktop app or [Claude Code](https://www.anthropic.com/claude-code)) into a research assistant that works directly with your Zotero library.

| Skill file | What it does |
|---|---|
| `zotero-skill.md` | Query your Zotero SQLite database — search by author/year/keyword, list collections, find relevant citations, verify citation faithfulness, surface annotations and recently-read items (Zotero 9+) |
| `zotero-notes-import.md` | Import structured HTML reading notes from an outliner (e.g. [Bike](https://www.hogbaysoftware.com/bike/)) into Zotero as item notes, matched to items by author + year |
| `manuscript-audit-skill.md` | Four-pass manuscript audit: citation faithfulness check, citation gap detection, logical consistency review, and copyediting |

---

## Why skills rather than an MCP?

Zotero MCPs expose a fixed set of tools (search, get item, generate citation) that Claude calls one at a time. A skill is different: it gives Claude working knowledge of the Zotero schema so it can write arbitrary SQL, chain queries across tables, open PDFs with `pdfplumber`, and combine all of that into multi-step workflows in a single session — for example, checking citation faithfulness across an entire manuscript, or finding supplementary files alongside the main PDF. Because the skill loads as part of Claude's context rather than running as a separate server, it needs no installation, no API key, and no network access beyond Zotero's own sync.

The main thing MCPs offer that skills don't is write access to the live Zotero API. The notes-import skill works around this for one specific write task by generating a self-contained Python script you run locally.

---

## Installation

### 1. Get the files

```bash
git clone https://github.com/dougwyu/claude-zotero-skills.git
```

Or download the three `.md` files directly.

### 2. Load into Cowork

In the Cowork desktop app:

1. Open **Settings → Skills**
2. Click **Add skill** and point it at each `.md` file
3. Give Cowork access to your Zotero directory (the folder containing `zotero.sqlite` and the `storage/` subfolder)

### 3. Load into Claude Code

Place the skill files somewhere on your path and reference them in your `CLAUDE.md`, or load them manually at the start of a session:

```
/skill zotero-skill.md
```

---

## Customisation

The skills are templates. Before first use, ask Claude to customise them for your setup — it will read your Zotero directory and fill in the details automatically. Give Claude access to your Zotero folder, then say something like:

> "Customise the zotero skill for my library."

Claude will enumerate your libraries from the `groups` table and update the skill accordingly.

Things you may want to adjust manually:

**`zotero-skill.md`**
- The `litmap` project path (`~/src/Cowork/litmap` by default — change to wherever you cloned it)
- The embeddings database path (`~/LitLake/embeddings.db` by default)

**`zotero-notes-import.md`**
- Your personal annotation label — the skill defaults to asking you at runtime, but you can hardcode it (e.g. `MY NOTES`, your initials, etc.)
- The HTML structure detection in `is_paper_li()` — adjust if your outliner produces a different structure than nested `<ul>/<li>` with `<strong>` author names

**`manuscript-audit-skill.md`**
- No changes required; it delegates to the zotero skill for all library access

---

## Usage

### Zotero search and citation work

```
Find papers by Smith from 2020–2023 in my library.
```
```
Does Jones et al. 2019 actually support the claim that X?
```
```
Find citations for this paragraph: [paste text]
```
```
What did I highlight in the Chen 2022 paper?
```
```
What have I read most recently?
```

### Importing reading notes

1. Write your notes in an outliner that can export HTML (e.g. [Bike](https://www.hogbaysoftware.com/bike/))
2. Structure each paper entry with the author name in `<strong>` and the year in parentheses — e.g. **Smith** (2020)
3. Export to HTML and copy the file to your `~/Zotero/` folder
4. Tell Claude:

```
Import my notes from my-notes.html into Zotero.
```

Claude will match each entry to its Zotero item by author + year, generate a self-contained Python script, and ask you to run it:

```bash
uv run ~/Zotero/zotero_note_import.py
```

Then sync Zotero (`Cmd+Shift+S`). Re-running the script is safe — it skips items that already have a note.

### Manuscript audit

```
Audit my manuscript: [attach file or paste text]
```

Claude runs four passes in sequence:

1. **Citation faithfulness** — opens each cited PDF and checks whether the claim attributed to it is actually supported
2. **Citation gaps** — finds unsupported claims and suggests papers from your library that could fill them
3. **Logical consistency** — checks for contradictions, reasoning gaps, and scope creep
4. **Copyediting** — grammar, clarity, flow, and style

Expect 10–20 minutes for an 8,000-word manuscript.

---

## Semantic search with litmap

The `zotero-skill.md` and `manuscript-audit-skill.md` skills use **litmap** for semantic (embedding-based) search — finding papers by concept rather than keyword, clustering a collection into themes, or detecting citation gaps. This is Tier 4 in the zotero skill's search hierarchy.

**Repository:** [github.com/dougwyu/litmap](https://github.com/dougwyu/litmap)

### Setup

```bash
git clone https://github.com/dougwyu/litmap.git ~/src/Cowork/litmap
cd ~/src/Cowork/litmap
uv pip install -e .
```

Then build the embeddings index for your library:

```bash
litmap sync
```

This embeds all items in your Zotero library into `~/LitLake/embeddings.db`. The first run downloads the embedding model (~570 MB) and may take several minutes depending on library size. Subsequent syncs are incremental.

### Usage examples

```bash
# Find papers similar to a query
litmap search --query "biodiversity loss tropical forests" --top-k 10

# Find papers similar to a given paper (by DOI)
litmap search --paper "10.1126/science.1256014" --top-k 10

# Cluster a collection into themes
litmap cluster --collection "Chapter 2 refs" --output /tmp/clusters --format md
```

The skills call litmap automatically when you ask conceptual questions. You do not need to run it directly unless you want to rebuild the index (`litmap sync`) or explore outside of Claude.

---

## Requirements

- [Zotero](https://www.zotero.org/) 6 or 9 (Zotero 9 adds annotation and "Added By" support)
- [Claude Code](https://www.anthropic.com/claude-code) or [Cowork](https://www.anthropic.com/claude) desktop app
- Python 3.11+ with [uv](https://github.com/astral-sh/uv) (for the notes-import script and litmap)
- `pdfplumber` and `beautifulsoup4` (installed automatically by the notes-import script via `uv`)
- An outliner that exports HTML with nested `<ul>/<li>` structure (e.g. [Bike](https://www.hogbaysoftware.com/bike/)) — required only for `zotero-notes-import.md`

---

## License

MIT
