---
name: zotero-notes-import
description: Import structured HTML reading notes into Zotero as item notes, matched by author + year reference. Use when the user wants to add their paper notes to Zotero, sync reading notes with their library, or import outliner notes into Zotero items. Trigger on phrases like "import my notes", "add notes to Zotero", "sync my reading notes".
---

# Zotero Notes Import

Imports structured HTML reading notes into Zotero as item notes, matching each entry to its Zotero item via author + year. Preserves personal annotation sections separately from PDF summary sections.

Notes are written via the **Zotero Web API** (not SQLite) so that keys are server-generated, sync state is correct, and Zotero can remain open during import.

Input is an **HTML outliner export** (e.g. File → Export → HTML from Bike or similar outliners). The nested `<ul>/<li>` structure is parsed directly.

---

## ⚠️ MANDATORY FIRST STEP — API credentials

**Before doing anything else**, check whether the user has already provided their Zotero credentials in this session. If yes, skip this step. If not, use the AskUserQuestion tool to ask:

```
Question: "Do you have your Zotero API key and numeric user ID ready?"

Options:
  A. "Yes — I'll paste them now"
  B. "No — I need to get them first"
```

If B: direct the user to https://www.zotero.org/settings/keys. The API key needs **read/write** access to their personal library and any group libraries. The **numeric userID** appears on that same page as "Your userID for use in API calls is XXXXXXX".

---

## Input

- **Notes file**: an HTML outliner export (e.g. from Bike: File → Export → HTML). Ask the user to export and copy the file to their Zotero folder (`~/Zotero/`), then provide the filename.
- **Zotero database**: opened read-only for item matching and group ID lookup.

---

## Process

### Step 1 — Parse the HTML outliner export

Read the HTML file. Outliners like Bike export a nested `<ul>/<li>` structure where each `<li>` contains a `<p>` and optionally a child `<ul>`.

Detect paper `<li>` elements by scanning **all `<li>` elements** recursively.

The recommended reference style is **Methods in Ecology & Evolution (MEE)**, which always includes a DOI. A typical entry looks like:

```
Smith, J., Brown, A., & Lee, K. (2021). Title of the paper. Journal Name,
590, 261–264. https://doi.org/10.xxxx/xxxxx
```

Detection relies on two signals that are always present in a well-formed MEE reference: a four-digit year in parentheses, and a DOI hyperlink (or an italicised journal name for references without a DOI). Author names need not be bold — the parser extracts the last name as the text before the first comma in the `<p>`.

```python
import re
from bs4 import BeautifulSoup

def is_paper_li(li):
    first_p = li.find('p')
    if not first_p:
        return False
    text = first_p.get_text()
    if not re.search(r'\(\d{4}\)', text):               # must have year in parens
        return False
    if not (first_p.find('a') or first_p.find('em')):  # must have DOI link or journal name
        return False
    return True

with open(notes_file, encoding='utf-8') as f:
    soup = BeautifulSoup(f, 'html.parser')

paper_lis = [li for li in soup.find_all('li') if is_paper_li(li)]
```

From the first `<p>` of each paper `<li>`, extract:
- **Last name**: text before the first comma in the `<p>`, stripped of leading whitespace. For multi-author entries this is the first author's last name.
- **Year**: four-digit year in parentheses.

The **child `<ul>`** of each paper `<li>` contains the note body. Identify the **personal annotation section** as any direct child `<li>` whose first `<p>` text matches a user-defined label (e.g. `^\s*NOTES` or any consistent heading the user uses). All other child `<li>` items are PDF-summary sections.

Ask the user what label they use for their personal annotation sections (e.g. "NOTES", "MY NOTES", initials, etc.) if not already known.

### Step 2 — Build the library ID map via SQLite (read-only)

Open the Zotero SQLite database read-only. Use it for two things: item matching, and mapping SQLite `libraryID` → real Zotero group ID (these are not the same number).

```python
import sqlite3, glob, os

candidates = glob.glob('/sessions/*/mnt/Zotero/zotero.sqlite')
LIVE_DB = candidates[0] if candidates else os.path.expanduser('~/Zotero/zotero.sqlite')
conn = sqlite3.connect(f"file:{LIVE_DB}?mode=ro", uri=True)

# Build libraryID → API prefix map
# libraryID=1 is always the personal library
# All others are groups — their real group ID is in the `groups` table
lib_prefix = {1: f"/users/{ZOTERO_USER_ID}"}
for group_id, lib_id in conn.execute("SELECT groupID, libraryID FROM groups"):
    lib_prefix[lib_id] = f"/groups/{group_id}"
```

### Step 3 — Match each reference to Zotero items

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

# Deduplicate by itemID
seen = set()
deduped = []
for r in results:
    if r[0] not in seen:
        seen.add(r[0])
        deduped.append(r)
results = deduped
```

- **No match**: record as unmatched, continue.
- **One or more matches**: post note for every matched item (papers often exist as duplicates across collections/libraries).

Close connection when done:

```python
conn.close()
```

### Step 4 — Compose the note HTML

```html
<div class="zotero-note znv1">
  <p><strong>Reading notes</strong> — [first 200 chars of reference]</p>
  <ul>
    <!-- non-personal child <li> items, preserving nested structure -->
  </ul>

  <hr/>
  <p><strong>Personal notes</strong></p>
  <ul>
    <!-- personal annotation child <li> items -->
  </ul>
</div>
```

Omit `<hr/>` and personal block if no personal annotation items. Omit summary `<ul>` if no non-personal items. Strip `id`, `data-created`, `data-modified` attributes. Preserve `<strong>`, `<em>`, `<mark>`.

### Step 5 — Check and post via Web API

```python
import requests, time

BASE    = "https://api.zotero.org"
HEADERS = {"Zotero-API-Key": ZOTERO_API_KEY, "Zotero-API-Version": "3"}

def already_has_note(prefix, item_key):
    r = requests.get(f"{BASE}{prefix}/items/{item_key}/children",
                     headers=HEADERS, params={"itemType": "note"})
    r.raise_for_status()
    return any("Reading notes" in c.get("data", {}).get("note", "") for c in r.json())

def post_note(prefix, item_key, note_html):
    payload = [{"itemType": "note", "parentItem": item_key,
                "note": note_html, "tags": [], "relations": {}}]
    r = requests.post(f"{BASE}{prefix}/items",
                      headers={**HEADERS, "Content-Type": "application/json"},
                      json=payload)
    if r.status_code not in (200, 201):
        raise RuntimeError(f"API error for {item_key}: {r.status_code} {r.text}")
    return r.json()

for item_id, item_key, lib_id, title, year in matched_items:
    prefix = lib_prefix.get(lib_id)
    if prefix is None:
        # unknown library — skip
        continue
    try:
        if already_has_note(prefix, item_key):
            record_skipped(item_key)
        else:
            post_note(prefix, item_key, note_html)
            record_inserted(item_key)
    except requests.HTTPError as e:
        if e.response.status_code == 403:
            record_permission_denied(item_key, prefix)
        else:
            raise
    time.sleep(0.15)
```

### Step 6 — Generate a uv-compatible script and save to ~/Zotero/

Rather than running the import directly (the sandbox cannot reach api.zotero.org), generate a self-contained Python script with a PEP 723 inline metadata header and save it to `~/Zotero/zotero_note_import.py`. The user runs it locally with `uv run ~/Zotero/zotero_note_import.py`.

The script header must be:

```python
# /// script
# requires-python = ">=3.11"
# dependencies = ["requests", "beautifulsoup4"]
# ///
```

The script should be fully self-contained: credentials, note HTML, and targets all embedded. Include a `already_has_note()` check so re-running is safe.

Tell the user:

```
Run with:
  uv run ~/Zotero/zotero_note_import.py

Then sync Zotero (Cmd+Shift+S).
```

### Step 7 — Report

After generating the script, report what will be imported:

| Status | Count | Papers |
|--------|-------|--------|
| ✓ Will insert note | N | [list of author–year + item key] |
| — Note already exists, will skip | N | [list] |
| ✗ No Zotero match found | N | [full reference] |
| ✗ Library not in group map | N | [list] |

For unmatched papers, print the full reference so the user can investigate manually.

---

## After import

> Run `uv run ~/Zotero/zotero_note_import.py`, then sync Zotero (Cmd+Shift+S). Notes will appear on the relevant items in whichever library they belong to. If a note seems missing, check that your API key has write access to the relevant group library at https://www.zotero.org/settings/keys.

---

## How manuscript-audit uses these notes

The manuscript-audit skill queries notes as a secondary source during Stage 1 (citation faithfulness):

```python
notes = conn.execute(
    "SELECT note FROM itemNotes WHERE parentItemID = ?",
    (matched_item_id,)
).fetchall()
```

Parse the note HTML to separate:
- **PDF summary sections** (before `<hr/>`) — secondary source, confirm against PDF.
- **Personal annotation section** (after `<hr/>`) — personal opinions, not faithful to source; exclude from faithfulness verdicts but surface as context.

```
Notes (from Zotero — secondary source, confirm against PDF):
  [Relevant excerpt from PDF summary section]

Personal context (use-case notes — not faithful to source):
  [Relevant excerpt from personal annotation section]
```
