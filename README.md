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

### 4. Install litmap (recommended)

The `zotero-skill.md` and `manuscript-audit-skill.md` skills use **litmap** for semantic search, citation gap detection, and library visualisation. See the [Semantic search with litmap](#semantic-search-with-litmap) section below for setup instructions.

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
2. Structure each paper as a top-level entry whose heading is the full reference. The recommended format is **Methods in Ecology & Evolution (MEE)** style, which always includes a DOI — this is what the skill uses to detect paper entries reliably. A generic MEE reference looks like:

   ```
   Smith, J., Brown, A., & Lee, K. (2021). Title of the paper. Journal Name,
   590, 261–264. https://doi.org/10.xxxx/xxxxx
   ```

   A complete note entry with child items and a personal annotation block:

   ```
   ▸ Smith, J., Brown, A., & Lee, K. (2021). Biodiversity loss in tropical forests.
     Nature, 590, 261–264. https://doi.org/10.1038/s41586-021-00001-x
       ▸ Main finding: forest loss accelerated 2000–2020
       ▸ Methods: remote sensing + field surveys across 50 sites
       ▸ JRS
           ▸ Good comparison dataset for Chapter 3
           ▸ Check whether their deforestation metric matches ours
   ```

   The skill identifies paper entries by looking for a four-digit year in parentheses and a DOI hyperlink (or italicised journal name) in the same line — author names do not need to be bold. The last name is extracted as the text before the first comma. Child items become the note body. Any child section whose heading matches your personal initials or annotation label (e.g. `JRS`) is stored separately as a personal annotation block, kept out of faithfulness verdicts during manuscript audit.

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

Then build the embeddings index for your library. There are two indexing commands:

```bash
# Embed abstracts and metadata (fast — minutes to an hour)
litmap sync

# Also embed full PDF text (slow — potentially many hours to several days)
litmap sync-fulltext
```

`litmap sync` indexes abstracts, titles, and metadata only, and is fast enough to run routinely. `litmap sync-fulltext` additionally reads and embeds the full text of every PDF in your library. Depending on your computer and the number of PDFs, this can take anywhere from a few hours to several days for a large library. The payoff is substantially better semantic search: queries match against the actual content of papers rather than just their abstracts, which is particularly valuable for finding relevant passages, detecting citation gaps, and manuscript auditing. Both commands write to `~/LitLake/embeddings.db`; subsequent runs are incremental and much faster.

The first run of either command also downloads the embedding model (~570 MB).

### What you can do with litmap

**Find papers by concept, not just keyword.** Standard Zotero search matches exact words. litmap matches meaning, so a query like "how do species interactions shape community assembly" will surface papers that discuss coexistence, competition, and niche theory even if those exact words don't appear in the title.

**Find papers similar to one you already have.** Give litmap a DOI and it returns the most conceptually similar papers in your library — useful for discovering related work you may have forgotten, or for building a citation cluster around a key paper.

**Cluster a collection into themes.** litmap groups a collection (or your entire library) into thematic clusters and produces a labelled outline. This is useful when starting to write — it shows you which themes are well-represented and which are thin, and can serve as a first draft of a section outline.

**Detect citation gaps in a manuscript.** The manuscript-audit skill uses litmap to find claims in your draft that lack citations, then searches your library semantically for papers that could support them. This catches gaps that keyword search would miss because the claim and the paper title use different vocabulary.

**Find papers by paper (not by query).** If a colleague recommends a paper you don't have, you can ask "what in my library is most like X?" using the paper's DOI — without needing to read it first.

**Visualise your library as a map.** `litmap map` produces a UMAP scatterplot of your papers with k-nearest-neighbour edges, coloured by semantic position — papers on similar topics cluster together. Output is an interactive Plotly HTML file you can pan and zoom in a browser, plus a publication-quality PNG/PDF at 300 DPI.

### Usage examples

```bash
# Find papers by concept
litmap search --query "biodiversity loss tropical forests" --top-k 10

# Find papers similar to a given paper (by DOI)
litmap search --paper "10.1126/science.1256014" --top-k 10

# Cluster a collection into themes (markdown outline)
litmap cluster --collection "Chapter 2 refs" --output /tmp/clusters --format md

# Cluster with an interactive HTML dendrogram as well
litmap cluster --collection "Chapter 2 refs" --output /tmp/clusters --format all

# Cluster the entire library
litmap cluster --output /tmp/clusters --format md

# Visualise the library as a UMAP map (interactive HTML + 300 DPI PNG/PDF)
litmap map --output /tmp/litmap_map
```

You can also drive litmap entirely through Claude — just ask conceptual questions and Claude will call litmap internally, so you rarely need to run it directly. The main reason to run it from the command line is to rebuild the index (`litmap sync` or `litmap sync-fulltext`) or to generate a cluster dendrogram you want to open in a browser.

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
