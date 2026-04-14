# Using Claude as a Zotero research assistant — two skills for direct SQLite access

I've been using Claude (via the Cowork desktop app) as an AI assistant that works
directly with my Zotero library, and I wanted to share the approach in case it's
useful to others.

The setup uses "skills" — plain markdown files that load into Claude's context and
tell it how to work with your Zotero data. Two skills cover the main use cases:

**zotero** (read) — queries the SQLite database directly to search by author/year/
keyword, list collections, find relevant citations for a passage, verify whether a
cited paper actually supports a claim, retrieve PDF text, and surface annotations and
recently-read items (Zotero 9+). No Zotero app needs to be running.

**zotero-notes-import** (write) — imports structured HTML reading notes into Zotero's
`itemNotes` table, matched to items by author + year. Requires Zotero to be closed;
works on a local copy of the database and syncs back on completion.

---

## Why skills rather than an MCP?

The Zotero MCPs I've seen expose a fixed set of tools (search, get item, generate
citation) that Claude calls one at a time. A skill is different: it gives Claude a
working knowledge of the Zotero schema so it can write arbitrary SQL, chain queries
across tables, open PDFs with pdfplumber, and combine all of that into multi-step
workflows in a single session — for example, checking citation faithfulness across an
entire manuscript, or finding supplementary files alongside the main PDF. Because the
skill loads as part of Claude's context rather than running as a separate server, it
needs no installation, no API key, and no network access.

The main thing MCPs offer that skills don't is write access to the live Zotero API
(creating items, modifying metadata). The notes-import skill works around this for
one specific write task, but if you need general write access, an MCP would be the
better fit.

---

## The skills

Both files are templates — you'll need to update the database path and (for the query
skill) populate the libraries table with your own libraryIDs. Everything else should
work as-is.

**zotero-skill.md** — paste the contents into a file called `SKILL.md` inside a
folder called `zotero/` in your Claude skills directory.

**zotero-notes-import-skill.md** — same pattern, folder called `zotero-notes-import/`.

The notes-import skill expects a specific HTML input format (nested `<ul>/<li>` with
`<strong>` author names and years in parentheses). The `is_paper_li` function in
Step 1 is the part most likely to need adaptation for your own note style.

Happy to answer questions about the setup.
