---
name: zotero-notes-import
description: Import structured HTML reading notes into Zotero as item notes, matched by author + year. Use when the user wants to add their paper notes to Zotero, sync reading notes with their library, or import notes into Zotero items. Trigger on phrases like "import my notes", "add notes to Zotero", "sync my reading notes".
---

# Zotero Notes Import

Imports structured HTML reading notes into Zotero as item notes, matching each entry
to its Zotero item via author + year.

Adapt the database path and note structure (below) to match your own setup.

---

## ⚠️ MANDATORY FIRST STEP — Zotero must be closed

**Before doing anything else**, use the AskUserQuestion tool to ask the user to confirm that Zotero is closed:

```
Question: "Have you closed Zotero? This skill writes directly to zotero.sqlite.
Running with Zotero open will cause data conflicts and may corrupt your library."

Options:
  A. "Yes, Zotero is closed — proceed"
  B. "No, Zotero is still open — I'll close it now"
```

If the user selects B, **stop immediately** and output:

> Please close Zotero fully (including any background sync processes), then run this
> skill again. On macOS you can confirm it is closed with `pgrep -x Zotero` returning
> nothing; on Windows, check Task Manager.

Do not proceed past this point until the user explicitly confirms Zotero is closed.

---

## Input

- **Notes file**: an HTML file containing reading notes, one `<li>` per paper, each
  headed by a reference with the first author's surname in `<strong>` tags.
- **Zotero database**: `/path/to/Zotero/zotero.sqlite`

If the user has not provided the notes file path, ask them to provide it.

---

## Process

### Step 1 — Parse the HTML notes file

Read the HTML file. The structure is a nested `<ul>/<li>` outline. Paper entries can
appear at **any nesting level**.

Detect paper `<li>` elements by scanning **all `<li>` elements** recursively and
applying this filter:

```python
import re
from bs4 import BeautifulSoup

def is_paper_li(li):
    first_p = li.find('p')
    if not first_p:
        return False
    text = first_p.get_text()
    if not re.search(r'\(\d{4}\)', text):              # must have year in parens
        return False
    if not first_p.find('strong'):                     # must have <strong> author
        return False
    if not (first_p.find('a') or first_p.find('em')): # must have DOI link or journal name
        return False
    return True

paper_lis = [li for li in soup.find_all('li') if is_paper_li(li)]
```

Adapt `is_paper_li` to match your notes format if it differs.

From the first `<p>` of each paper `<li>`, extract:
- First author's last name: text inside the first `<strong>` tag, stripping trailing
  commas and whitespace.
- Year: the four-digit year in parentheses, e.g. `(2013)`.

The **child `<ul>`** of each paper `<li>` contains the note body. If your notes have
a personal/opinions section distinct from the summary, identify it by a consistent
marker (e.g. a `<li>` whose text starts with a known prefix) and handle it separately
in Step 5.

### Step 2 — Work on a local copy of the database

The Zotero database may be on a FUSE-mounted filesystem. SQLite cannot commit
transactions directly to FUSE mounts — writes will fail with a "disk I/O error".
Always work on a local copy and sync back when done:

```python
import shutil, subprocess, sqlite3

LIVE_DB = '/path/to/Zotero/zotero.sqlite'
LOCAL_DB = '/tmp/zotero_import_work.sqlite'

shutil.copy2(LIVE_DB, LOCAL_DB)
conn = sqlite3.connect(f'file:{LOCAL_DB}?mode=rwc', uri=True)
conn.execute("PRAGMA journal_mode=WAL")

# ... do all inserts on conn ...

conn.commit()
conn.close()

# rsync handles large files over FUSE reliably; plain cp may time out
subprocess.run(['rsync', LOCAL_DB, LIVE_DB], check=True)
```

### Step 3 — Match each reference to a Zotero item

For each parsed reference, query the local copy:

```python
query = '''
SELECT i.itemID, i.key, i.libraryID,
       idv_title.value AS title,
       idv_year.value  AS year
FROM items i
JOIN itemCreators ic  ON ic.itemID   = i.itemID AND ic.orderIndex = 0
JOIN creators c       ON c.creatorID = ic.creatorID
LEFT JOIN itemData    id_year  ON id_year.itemID  = i.itemID
    AND id_year.fieldID  = (SELECT fieldID FROM fields WHERE fieldName = 'date')
LEFT JOIN itemDataValues idv_year  ON idv_year.valueID  = id_year.valueID
LEFT JOIN itemData    id_title ON id_title.itemID = i.itemID
    AND id_title.fieldID = (SELECT fieldID FROM fields WHERE fieldName = 'title')
LEFT JOIN itemDataValues idv_title ON idv_title.valueID = id_title.valueID
WHERE c.lastName LIKE ?
  AND idv_year.value LIKE ?
  AND i.itemTypeID NOT IN (14, 26)
'''

results = conn.execute(query, (f'%{last_name}%', f'{year}%')).fetchall()

# Deduplicate by itemID (same item can appear multiple times via multiple PDFs)
seen = set()
deduped = []
for r in results:
    if r[0] not in seen:
        seen.add(r[0])
        deduped.append(r)
results = deduped
```

- If **no match**: record as unmatched, continue, report at the end.
- If **one or more matches**: insert the note for **every matched item**. Papers
  frequently exist as duplicates in Zotero. Attaching the note to all copies ensures
  it is visible whichever copy the user is viewing.

### Step 4 — Check for existing notes and insert

For each matched item, skip if a note already exists:

```python
if conn.execute("SELECT 1 FROM itemNotes WHERE parentItemID = ?", (item_id,)).fetchone():
    continue  # note already present — skip
```

When inserting, the note item's `libraryID` **must match the parent item's `libraryID`**.
Zotero will silently hide a note whose library differs from its parent — do not
hardcode `libraryID=1`:

```python
import random, string, time

def generate_zotero_key(conn):
    chars = string.ascii_uppercase + string.digits
    while True:
        key = ''.join(random.choices(chars, k=8))
        if not conn.execute("SELECT 1 FROM items WHERE key = ?", (key,)).fetchone():
            return key

max_version = conn.execute("SELECT MAX(version) FROM items").fetchone()[0] or 0
now_iso = time.strftime('%Y-%m-%d %H:%M:%S')

# parent_library_id comes from i.libraryID in the query result above
conn.execute(
    "INSERT INTO items (itemTypeID, libraryID, key, dateAdded, dateModified, "
    "clientDateModified, version, synced) VALUES (26, ?, ?, ?, ?, ?, ?, 0)",
    (parent_library_id, generate_zotero_key(conn), now_iso, now_iso, now_iso, max_version + 1)
)
new_item_id = conn.execute("SELECT last_insert_rowid()").fetchone()[0]

conn.execute(
    "INSERT INTO itemNotes (itemID, parentItemID, note, title) VALUES (?, ?, ?, ?)",
    (new_item_id, item_id, note_html, f"Reading notes — {last_name} ({year})")
)
```

### Step 5 — Compose the note HTML

Build the note content as a single HTML string. Adapt the structure to suit your
note format — the key requirement is that Zotero expects a `<div class="zotero-note znv1">` wrapper:

```html
<div class="zotero-note znv1">
  <p><strong>Reading notes</strong> — [reference string]</p>
  <ul>
    <!-- main note content, preserving nested structure -->
  </ul>

  <hr/>
  <p><strong>Personal notes</strong></p>
  <ul>
    <!-- personal/opinions section, if any -->
  </ul>
</div>
```

Omit the `<hr/>` and personal block entirely if there is no personal section. Strip
`id`, `data-created`, and `data-modified` attributes from all elements. Preserve
`<strong>`, `<em>`, and `<mark>` formatting.

### Step 6 — Commit, sync back, and report

```python
conn.commit()
conn.close()
subprocess.run(['rsync', LOCAL_DB, LIVE_DB], check=True)
```

Report a summary table:

| Status | Count | Papers |
|--------|-------|--------|
| ✓ Note inserted | N | [list of author–year] |
| — Note already existed, skipped | N | [list] |
| ✗ No Zotero match found | N | [list of full references] |

For unmatched papers, print the full reference so the user can investigate manually.

---

## After import

Remind the user:

> Notes have been written to `zotero.sqlite`. Open Zotero and the notes will appear
> on the relevant items. Notes are attached to whichever library each item lives in —
> check the correct library if a note seems missing.
