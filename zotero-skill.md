---
name: zotero
description: "Use this skill whenever the user wants to query, search, or explore their Zotero reference library — or when they need help with citations in their writing. This includes: finding papers in specific collections or group libraries, searching by author/year/topic, verifying whether cited papers actually support the claims they are cited for, finding additional citations relevant to a passage of text, reading PDF content for summaries or quotes, and any other task that involves the Zotero database or its attached PDFs. Trigger on phrases like: \"find papers on X in my Zotero\", \"check my citations\", \"are these references accurate?\", \"find more citations for this paragraph\", \"what does [Author Year] actually say?\", \"search the [collection] library\", \"summarise papers about X\", \"what have I read recently?\", \"show me my annotations on X\", \"what did I highlight\"."
---

# Zotero Skill

> ⚠️ **Runtime: Claude Code only.** This skill runs `uv run litmap …` against `~/LitLake/embeddings.db` on the local machine. It will not work in the Cowork web sandbox. If you reached this skill from the Cowork web frontend, stop and switch to Claude Code.

## Setup

**Database:** `~/Zotero/zotero.sqlite` (mounted dynamically — use the path resolution snippet below)
**PDFs:** `<zotero_dir>/storage/<itemKey>/<filename>.pdf`
**Reference file:** `<zotero_dir>/cowork-zotero-reference.md`

Read the reference file at the start of every session — it holds
environment-specific facts (this machine's libraries, connection quirks,
the installed Zotero version's field-ID values). **Always run live queries
for counts, collection listings, and field IDs — do not trust hardcoded
numbers anywhere, including in this skill.**

```python
import sqlite3, glob, os

# Resolve Zotero mount path dynamically (session ID changes each run)
candidates = glob.glob('/sessions/*/mnt/Zotero/zotero.sqlite')
if candidates:
    db_path = candidates[0]
else:
    db_path = os.path.expanduser('~/Zotero/zotero.sqlite')
zotero_dir = os.path.dirname(db_path)

# Zotero holds a write lock while running. `mode=ro` fails with
# "database is locked"; `immutable=1` reads a consistent read-only
# snapshot even with the app open. Never attempt writes.
conn = sqlite3.connect(f"file:{db_path}?immutable=1", uri=True)
cur = conn.cursor()

# Field IDs are NOT stable across Zotero major versions (Zotero 9
# remapped every one). Always resolve them by name at runtime:
fids = dict(cur.execute("SELECT fieldName, fieldID FROM fields").fetchall())
# e.g. fids['title'], fids['abstractNote'], fids['date'],
#      fids['publicationTitle'], fids['url'], fids['DOI']
```

Build every metadata query with `fids[...]`, never a literal `fieldID = N`.
The SQL examples below use placeholders like `{title}` to mean
`fids['title']` — substitute the resolved integer (e.g. via f-string)
before executing. `date` values are stored as `"YYYY-MM-DD …"`; use
`substr(value,1,4)` for the year.

**Always query across all libraries unless the user specifies otherwise.**

**Always exclude trashed items.** Items the user deleted sit in Zotero's
Trash (table `deletedItems`) and remain in `items` until the trash is
emptied — so a deduped/removed paper still appears in raw queries. Append
this to every metadata/full-text search unless the user explicitly asks to
include trash:

```sql
AND i.itemID NOT IN (SELECT itemID FROM deletedItems)
```

## Libraries

Query the database directly to enumerate the user's libraries:

```sql
SELECT libraryID, name FROM groups
```

libraryID 1 is always the personal library. All others correspond to group libraries. At the start of a session, list the available libraries so the user can refer to them by name.

## Four tiers of search

Choose the right tier for the task:

```
Exact author/year/title or specific Zotero collection?     → Tier 1
A specific keyword/phrase across PDF text?                 → Tier 2
Deep claim verification needing PDF read?                  → Tier 3
Conceptual / paraphrased / "find similar" / "organise"?    → Tier 4
```

When in doubt, Tier 1 first to scope, then Tier 4 within scope.

**Tier 1 — Metadata** (instant, whole library)
Query `itemData`/`itemDataValues`/`creators` for title, author, abstract, journal,
date, DOI, tags. Use for: finding papers by author/year, listing a collection,
keyword searches in titles/abstracts.

**Tier 2 — Full-text index** (fast, indexes all items with PDFs)
Query `fulltextWords` + `fulltextItemWords` for keyword presence across PDFs without
opening them. Good for "find papers that discuss [term]" across a large collection.
Words are lower-cased and individual (not phrases). For phrase search, intersect
multiple word queries or fall back to Tier 3.

```sql
SELECT i.itemID, i.libraryID, tv.value AS title
FROM fulltextWords fw
JOIN fulltextItemWords fiw ON fw.wordID = fiw.wordID
JOIN items i ON fiw.itemID = i.itemID
JOIN itemData td ON i.itemID = td.itemID AND td.fieldID = {title}
JOIN itemDataValues tv ON td.valueID = tv.valueID
WHERE fw.word = 'keyword'
  AND i.itemID NOT IN (SELECT itemID FROM deletedItems)
```

**Tier 3 — PDF reading** (slower, for deep analysis)
Open PDFs when you need to verify a specific claim, extract a quote, or check
citation faithfulness.

Pick the tool by *what you need to do with the content*:

**1. Text only** (claims, quotes, statistics) → **`pdftotext`** — fast, no
page limit, no dependency:

```python
import subprocess
text = subprocess.run(["pdftotext", "-q", pdf_path, "-"],
                       capture_output=True, text=True).stdout
```

**2. Understand/interpret a figure, chart, or plot** → **the Read tool with
the `pages` parameter**, not pdfplumber. Only the Read tool can actually see
and interpret a figure (axes, trends, legend, values); pdfplumber merely
returns pixels/coordinates. It is also more efficient — one step, no
render→save→re-read detour. Respect the ~20-pages-per-request cap; for a
large PDF pass a tight page range. (Consistent with the global CLAUDE.md
rule: visual content → Read tool.)

**3. pdfplumber — supporting role only.** Use it to (a) extract the exact
data **table** behind a figure when the numbers exist as vector text, or
(b) **crop a single panel's bounding box** on a busy multi-panel page so the
Read tool gets a clean, tightly-scoped image. Also the fallback when
`pdftotext` yields empty output (scanned PDF). Installed globally:

```python
import pdfplumber
with pdfplumber.open(pdf_path) as pdf:
    page   = pdf.pages[n]
    tables = page.extract_tables()                  # data behind a figure
    images = page.images                            # locate figure bboxes
    page.crop(bbox).to_image(resolution=200).save(out_png)  # isolate a panel
```

Use `re.finditer` to locate specific passages rather than printing the entire
text. Always read SI attachments too when they exist (see PDF-path section).

## Tier 4 — Semantic (model-backed)

Use when the query is conceptual or paraphrased — i.e. when keyword search would miss synonyms, acronyms, or rephrasings. Tier 4 calls `litmap search` or `litmap cluster` against the local embeddings database. The first call after a fresh model download or a long idle takes 10–30 seconds while the embedding model warms up; subsequent calls within the same session are sub-second.

### Pattern 4a — Natural-language query → ranked papers

```bash
uv run --project ~/src/Cowork/litmap litmap search \
  --query "biodiversity loss accelerating in tropics" \
  --top-k 10 --format json
```

Returns JSON of the form:

```json
{
  "query": "biodiversity loss accelerating in tropics",
  "results": [
    {"zotero_key": "AAAA0001", "title": "...", "authors": ["..."],
     "year": "2022", "doi": "10.1/...", "similarity": 0.91, "abstract": "..."}
  ]
}
```

Summarise the top results with similarity scores. The `zotero_key` is portable — pass it to Tier 1/2/3 to open the PDF or fetch full metadata.

### Pattern 4b — Paper-to-paper similarity

```bash
uv run --project ~/src/Cowork/litmap litmap search \
  --paper "10.1126/science.1256014" --top-k 10 --format json
```

Same JSON shape as 4a; the focal paper is automatically excluded. Use for *"what else have I read that's like X?"*

### Pattern 4c — Cluster a collection or library into themed groups

```bash
uv run --project ~/src/Cowork/litmap litmap cluster \
  --collection "Chapter 2 refs" \
  --output /tmp/litmap_clusters \
  --format md
```

Read `/tmp/litmap_clusters.md` and present the outline inline. For visual exploration, mention that the `.html` dendrogram is also available via `--format all`.

To cluster the entire library, omit `--collection`.

## Zotero 9 — New tables and fields

Zotero 9 added several tables and columns. Use them where relevant.

**`itemAnnotations`** — PDF/EPUB annotations (highlights, notes, underlines, images):
```sql
SELECT ia.itemID, ia.type, ia.authorName, ia.text, ia.comment,
       ia.color, ia.pageLabel, ia.sortIndex, ia.position, ia.isExternal
FROM itemAnnotations ia
WHERE ia.parentItemID = ?   -- parentItemID is the attachment's itemID
ORDER BY ia.sortIndex
```
- `type`: `highlight`, `note`, `image`, `underline`, `ink`
- `text`: the highlighted/underlined text verbatim
- `comment`: the annotation's note/comment body
- `pageLabel`: the page label shown in the reader (may differ from physical page number)
- Use this table when the user asks "what did I highlight in X", "show me my annotations",
  "what notes did I make on Y", or wants to review their reading notes in structured form.
  Prefer this over reading the PDF directly for annotation retrieval — it's faster and
  returns exactly what was marked.

**`itemAttachments.lastRead`** — Unix timestamp of when the attachment was last opened
in the Zotero reader (new in Zotero 9; NULL if never opened in Zotero 9+):
```sql
SELECT ia.lastRead, i.key, idv.value AS title, ia.path
FROM itemAttachments ia
JOIN items i ON ia.itemID = i.itemID
JOIN items parent ON ia.parentItemID = parent.itemID
JOIN itemData id_t ON parent.itemID = id_t.itemID AND id_t.fieldID = {title}
JOIN itemDataValues idv ON id_t.valueID = idv.valueID
WHERE ia.lastRead IS NOT NULL
ORDER BY ia.lastRead DESC
LIMIT 20
```
Use for "what have I read recently?" queries. `lastRead` is seconds since Unix epoch;
convert with `datetime(ia.lastRead, 'unixepoch')`.

**`groupItems`** — tracks who added/last modified items in group libraries ("Added By" /
"Modified By" feature in Zotero 9):
```sql
SELECT gi.createdByUserID, gi.lastModifiedByUserID,
       u1.username AS addedBy, u2.username AS modifiedBy
FROM groupItems gi
JOIN users u1 ON gi.createdByUserID = u1.userID
JOIN users u2 ON gi.lastModifiedByUserID = u2.userID
WHERE gi.itemID = ?
```

**`retractedItems`** — flags papers that have been retracted (new in Zotero 9):
```sql
SELECT ri.itemID, ri.data, ri.flag
FROM retractedItems ri
```
When performing citation faithfulness checks, always query this table first for all
cited papers. Flag any retracted source prominently at the top of the report — a
retracted paper should never be cited without explicit acknowledgement of its status.

---

## Finding a PDF path — including supplementary files

When retrieving attachments for a paper, always fetch **all** file attachments, not
just the main PDF. Supplementary information files (SI, appendices, supporting data)
are stored as sibling attachments under the same `parentItemID`.

```sql
SELECT ia.path, ia.contentType, i.key AS attachmentKey
FROM itemAttachments ia
JOIN items i ON ia.itemID = i.itemID
WHERE ia.parentItemID = ?
  AND ia.path IS NOT NULL
  AND ia.path != ''
ORDER BY ia.itemID
```

> ⚠️ Zotero 9 **removed `itemAttachments.orderIndex`** (no replacement
> ordering column exists). Do not `ORDER BY ia.orderIndex` — it throws
> `no such column`. `ORDER BY ia.itemID` approximates insertion order
> (main PDF is normally added first); never rely on attachment order for
> correctness — classify main-vs-SI by the filename heuristic below.

Classify each result by filename:
- **Main article** — the primary PDF (usually matches the paper title or author-year)
- **Supplementary** — filename contains any of: `supplement`, `supporting`, `appendix`,
  `SI`, `S1`, `S2`, `ESM`, `Online Resource`, `Data S`, `Table S`, `Figure S`

PDF path: `<zotero_dir>/storage/<attachmentKey>/<filename>`
where `<filename>` is the part of `ia.path` after `storage:`.

**Always read supplementary files when they exist.** Information critical to a claim
is often in the SI — extended methods, species lists, robustness checks, data tables.
Read the main PDF first; then check whether any SI attachments exist and read them
too, prioritising sections most relevant to the claim being checked.

Non-PDF SI (e.g. `.xlsx`, `.csv`) can be read with pandas or standard file tools
if the content is relevant to the task.

## Core query patterns

### Find item by author + year (all libraries)
```sql
SELECT DISTINCT i.itemID, i.key, i.libraryID,
       tv.value AS title, dv.value AS date, jv.value AS journal,
       GROUP_CONCAT(c.lastName || ', ' || c.firstName, '; ') AS authors
FROM items i
JOIN itemCreators ic ON i.itemID = ic.itemID AND ic.orderIndex = 0
JOIN creators c ON ic.creatorID = c.creatorID
JOIN itemData td ON i.itemID = td.itemID AND td.fieldID = {title}
JOIN itemDataValues tv ON td.valueID = tv.valueID
LEFT JOIN itemData dd ON i.itemID = dd.itemID AND dd.fieldID = {date}
LEFT JOIN itemDataValues dv ON dd.valueID = dv.valueID
LEFT JOIN itemData jd ON i.itemID = jd.itemID AND jd.fieldID = {publicationTitle}
LEFT JOIN itemDataValues jv ON jd.valueID = jv.valueID
WHERE c.lastName LIKE '%LastName%' AND dv.value LIKE '%YEAR%'
  AND i.itemID NOT IN (SELECT itemID FROM deletedItems)
GROUP BY i.itemID
```

### List items in a collection (with metadata)
```sql
SELECT DISTINCT i.itemID, i.key,
       tv.value AS title, dv.value AS date, jv.value AS journal,
       av.value AS abstract,
       GROUP_CONCAT(c.lastName || ', ' || c.firstName, '; ') AS authors
FROM collectionItems ci
JOIN items i ON ci.itemID = i.itemID
JOIN itemData td ON i.itemID = td.itemID AND td.fieldID = {title}
JOIN itemDataValues tv ON td.valueID = tv.valueID
LEFT JOIN itemData dd ON i.itemID = dd.itemID AND dd.fieldID = {date}
LEFT JOIN itemDataValues dv ON dd.valueID = dv.valueID
LEFT JOIN itemData jd ON i.itemID = jd.itemID AND jd.fieldID = {publicationTitle}
LEFT JOIN itemDataValues jv ON jd.valueID = jv.valueID
LEFT JOIN itemData ad ON i.itemID = ad.itemID AND ad.fieldID = {abstractNote}
LEFT JOIN itemDataValues av ON ad.valueID = av.valueID
LEFT JOIN itemCreators ic ON i.itemID = ic.itemID AND ic.orderIndex = 0
LEFT JOIN creators c ON ic.creatorID = c.creatorID
WHERE ci.collectionID = ?
  AND i.itemID NOT IN (SELECT itemID FROM deletedItems)
GROUP BY i.itemID ORDER BY dv.value DESC
```

### Find collection by name
```sql
SELECT collectionID, collectionName, libraryID, parentCollectionID
FROM collections
WHERE collectionName LIKE '%search_term%'
ORDER BY libraryID, collectionName
```

**Disambiguation:** The same collection name can appear in multiple libraries. Always
check `libraryID` and, if ambiguous, confirm with the user which one they mean —
or query both and note the distinction in your output.

### Get abstract for an item
```sql
SELECT v.value FROM itemData d
JOIN itemDataValues v ON d.valueID = v.valueID
WHERE d.itemID = ? AND d.fieldID = {abstractNote}
```

## Field IDs — resolve at runtime, never hardcode

Zotero's numeric `fieldID`s are an internal detail and **change across major
versions** (Zotero 9 remapped all of them). Do not memorise or hardcode them.
Build the lookup once per session and reference it by name:

```python
fids = dict(cur.execute("SELECT fieldName, fieldID FROM fields").fetchall())
```

The `{title}`, `{date}`, `{abstractNote}`, `{publicationTitle}`, `{url}`,
`{DOI}` placeholders in the SQL examples above all mean `fids['<name>']`.
The reference file records the *current* installed version's values for
quick human reference only — the query path must still resolve live.

---

## Task: Find relevant citations for a passage

When the user provides a paragraph and asks for relevant citations:

1. Identify the core claims and key concepts — what is the sentence actually saying?
2. Identify which library/collection to search (ask if not specified)
3. Search Tier 1 (title/abstract keywords) — cast a moderately wide net
4. If Tier 1 returns too few results, extend to Tier 2 (full-text index)
5. Read abstracts to filter for genuine relevance
6. Present candidates grouped by match strength:
   - **Strong match** — paper directly illustrates the claim
   - **Relevant** — paper supports a related part of the sentence
   - **Lower relevance** — brief note on why it doesn't fit
7. For each strong match, give a one-sentence note explaining *why* it's relevant
8. For borderline candidates, offer to open the PDF to confirm

---

## Task: Verify citation faithfulness

When the user provides a paragraph with citations and asks whether the citations
faithfully represent the source:

1. **Check for retractions first** — query `retractedItems` for all cited papers.
   Flag any retracted paper prominently before proceeding.
2. Find each cited paper in the database (Tier 1, by author + year)
3. Read the abstract first — flag obvious mismatches immediately
4. Find and open the PDF for each paper (Tier 3)
5. For each specific claim attributed to a paper, search the PDF text with
   `re.finditer` for relevant passages
6. Assess each claim:
   - **Faithful** — claim is directly supported by the paper
   - **Overstated** — claim goes beyond what the paper actually says
   - **Misattributed** — the figure/finding is from a paper the cited paper *itself*
     cites, not from the cited paper directly (a common failure mode)
   - **Unsupported** — the claim is simply not in the paper

7. Present a verdict table: one row per (citation × claim) combination, with the
   supporting quote from the paper where possible

**Critical check for statistics:** When a claim involves a specific number (%, count,
ratio), always verify that exact figure appears in the cited paper. A common failure
is citing a *review* paper for a statistic that originated in a paper the review
cites — the review paper's text will say something like "X et al. found that < 5%..."
rather than presenting it as its own finding. In that case, the original source
should be cited instead, or in addition.

---

## Task: Check whether a paper is in the library

Search by last name across all libraries, then filter by year:

```sql
SELECT i.itemID, i.libraryID, tv.value AS title, dv.value AS date
FROM items i
JOIN itemCreators ic ON i.itemID = ic.itemID AND ic.orderIndex = 0
JOIN creators c ON ic.creatorID = c.creatorID
JOIN itemData td ON i.itemID = td.itemID AND td.fieldID = {title}
JOIN itemDataValues tv ON td.valueID = tv.valueID
LEFT JOIN itemData dd ON i.itemID = dd.itemID AND dd.fieldID = {date}
LEFT JOIN itemDataValues dv ON dd.valueID = dv.valueID
WHERE c.lastName LIKE '%LastName%'
  AND i.itemID NOT IN (SELECT itemID FROM deletedItems)
GROUP BY i.itemID
```

---

## Scoping to a specific library or collection

- **By library name:** find the libraryID from the `groups` table, add `WHERE i.libraryID = N`
- **By collection name:** find collectionID first, then join via `collectionItems`
- **Excluding bulk collections:** the personal library may have large unsorted collections
  (e.g. `import`, `need_metadata`) — exclude these when doing relevance searches unless
  the user specifically wants them included

## Notes

- The SQLite database is read-only — never attempt writes
- PDF reading: `pdftotext` for text-only; **Read tool (`pages`) to interpret
  figures/charts**; `pdfplumber` only to extract figure data-tables or crop a
  panel, and as scanned-PDF fallback — pdftotext/pdfplumber installed globally
- For large PDFs, use `re.finditer` for keyword search rather than printing all text
- Some collections share names across libraries — always check `libraryID`
- Reference file counts are approximate; always query live for actual counts
- `itemAnnotations` is the canonical source for annotation data in Zotero 9; prefer it
  over parsing annotation text from PDFs

## Cross-tier integration rules

- Tier 4 results carry `zotero_key`. Pass it straight to Tier 1 SQL (`WHERE i.key = ?`) for full metadata, or to Tier 3 for PDF reading.
- When the user has specified a collection scope ("in collection X, find papers about Y"), pass `--collection "<name>"` to litmap.
- When the user has specified a *library*, litmap cannot scope by library directly. Run Tier 4 unscoped, then filter results by checking each `zotero_key` against `libraryID` via a single Tier 1 SQL query.
- Auto-sync runs before every litmap call. Mention the sync in the user response only if it added papers (>0 new embeddings).

## Tier 4 errors and edge cases

| Condition | Skill response |
|---|---|
| `litmap` command not found | "`litmap` is not installed. Run `uv pip install -e .` from `~/src/Cowork/litmap`." |
| First-run model download | "First-run model download (~570 MB), this takes ~1 minute." |
| `~/LitLake/embeddings.db` missing | "Embeddings database not found. Run `litmap sync` once to embed the library." |
| Auto-sync took > 30s on incremental | Note: "Sync took longer than usual — if you've recently added many papers, this is expected." |
| Top result similarity < 0.5 | "No strong semantic matches in your library. Consider rephrasing or broadening the query." |
