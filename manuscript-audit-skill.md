---
name: manuscript-audit
description: "Audit and polish scientific manuscripts for journal submission. Use this skill whenever the user has a manuscript draft (docx, pdf, or pasted text) for peer review. The skill performs four passes: (1) extracts all author-year citations and verifies claims against original PDFs in the user's Zotero library to ensure faithfulness, consulting the manuscript's reference list to disambiguate author-year citations; (2) identifies unsupported or weakly supported claims and suggests additional citations via semantic search of the library; (3) checks for logical inconsistencies, contradictions, and reasoning gaps within the manuscript; (4) copyedits for grammar, style, punctuation, flow, and journal conventions. Trigger on phrases like 'audit my manuscript,' 'review my citations,' 'fact-check my claims,' 'check my paper,' or 'prepare for submission.'. (Claude Code only — uses litmap for semantic citation-gap detection.)"
compatibility: "Claude Code only. Requires zotero skill, pdf-reading skill, file-reading skill (if manuscript uploaded as file), and a working `uv run litmap` install at ~/src/Cowork/litmap."
---

# Manuscript Audit for Scientific Journal Submission

> ⚠️ **Runtime: Claude Code only.** This skill calls `uv run litmap …` against `~/LitLake/embeddings.db` on the local machine. It will not work in the Cowork web sandbox. If you reached this skill from the Cowork web frontend, stop and switch to Claude Code.

A four-stage workflow to verify citations against sources, identify evidence gaps, detect logical flaws, and polish a manuscript draft before submission.

---

## Stage 1: Citation Extraction & Faithfulness Audit

### Input
- Manuscript (docx, pdf, or pasted text)
- User's Zotero library (via zotero skill)

### Process

1. **Extract reference list:** Locate the References or Bibliography section in the manuscript. Build a lookup table: `author_year → [full citation text, DOI/URL if present]`.

2. **Extract in-text citations:** Use regex to find all author-year patterns:
   - `Smith 2020`
   - `(Jones et al. 2019)`
   - `Smith and Brown 2018`
   - `(Smith 2020; Brown 2021)`
   
   For each citation, record: the matched text, the sentence/paragraph context, the section heading, and the claim being supported.

3. **Disambiguate via reference list:** For each in-text citation, match it to the reference list entry to confirm author names, year, and full publication details. Flag any mismatches (e.g., cited as "Smith 2020" but reference list shows "Smith, J. 2019").

4. **Check for retracted papers (Zotero 9):** Before opening any PDFs, query the `retractedItems` table for every paper matched in the database:

   ```python
   retracted = {
       r[0]: r[1]
       for r in conn.execute("SELECT itemID, data FROM retractedItems").fetchall()
   }
   ```

   If any cited paper's itemID appears in `retracted`, flag it at the very top of the Stage 1 report with a ⛔ **RETRACTED** verdict and the retraction data (journal notice, date if available). The author must address this before submission — retracted papers should not be cited without explicit acknowledgement of retraction status.

4b. **Locate PDFs in Zotero:** Query the Zotero SQLite database directly rather than grepping filenames. Filename-based search is unreliable for two reasons: (a) auto-generated filenames may not include the author name at all (e.g., an organisation such as "Nature Positive Initiative" may be saved under the document title), and (b) overly broad filename patterns return hundreds of false positives, making truncation errors likely. Instead, run a Python query against `zotero.sqlite`:

   ```python
   import sqlite3
   conn = sqlite3.connect('/mnt/Zotero/zotero.sqlite')
   query = '''
   SELECT i.key, c.lastName, c.firstName, idv_year.value AS year,
          idv_title.value AS title, ia.path
   FROM items i
   JOIN itemCreators ic ON ic.itemID = i.itemID AND ic.orderIndex = 0
   JOIN creators c ON c.creatorID = ic.creatorID
   LEFT JOIN itemData id_year ON id_year.itemID = i.itemID
       AND id_year.fieldID = (SELECT fieldID FROM fields WHERE fieldName = 'date')
   LEFT JOIN itemDataValues idv_year ON idv_year.valueID = id_year.valueID
   LEFT JOIN itemData id_title ON id_title.itemID = i.itemID
       AND id_title.fieldID = (SELECT fieldID FROM fields WHERE fieldName = 'title')
   LEFT JOIN itemDataValues idv_title ON idv_title.valueID = id_title.valueID
   LEFT JOIN itemAttachments ia ON ia.parentItemID = i.itemID
   WHERE c.lastName LIKE ? AND idv_year.value LIKE ?
   '''
   results = conn.execute(query, ('%He%', '2015%')).fetchall()
   ```

   Substitute the author's last name and year from the reference list. For multi-word organisation names, search by the first distinctive word as `lastName LIKE '%Organisation Name%'` or search by title keywords instead. The `path` field returned is of the form `storage:filename.pdf`; prepend `/mnt/Zotero/storage/<key>/` using the item key to get the full path. If multiple results are returned for the same author+year, use the title from the reference list entry to select the correct one.

   **Important:** Zotero PDF filenames are generated automatically and may contain errors — misspelled author names, wrong years, truncated titles. Never use a filename mismatch as evidence of a citation error in the manuscript. Always resolve ambiguity by matching against the full reference list entry (title, journal, DOI), not the filename alone.

4c. **Check for reading notes:** Once a Zotero item is matched, query any attached reading notes as a secondary source before opening the PDF:

   ```python
   from bs4 import BeautifulSoup

   notes = conn.execute(
       "SELECT note FROM itemNotes WHERE parentItemID = ?",
       (matched_item_id,)
   ).fetchall()

   for (note_html,) in notes:
       soup = BeautifulSoup(note_html, 'html.parser')
       hr = soup.find('hr')
       if hr:
           summary_text = ' '.join(t.get_text() for t in hr.previous_siblings)
           personal_text = ' '.join(t.get_text() for t in hr.next_siblings)
       else:
           summary_text = soup.get_text()
           personal_text = ''
   ```

   Use **PDF summary sections** (content before `<hr/>`) to quickly locate passages relevant to the claim — they save time when skimming a long PDF. Treat as helpful but non-authoritative; always confirm any finding against the PDF itself.

   Use **personal annotation sections** (content after `<hr/>`) as context only for understanding how the paper was intended to be used. Never cite personal annotation content in a faithfulness verdict.

   In the Stage 1 output, add a Notes subsection where reading notes exist:

   ```
   Notes (Zotero reading notes — secondary source, verify against PDF):
     [Relevant excerpt from PDF summary section]

   Personal context (use-case notes — not a faithfulness source):
     [Relevant excerpt from personal annotation section, if any]
   ```

5. **Extract and verify:** For each PDF found:
   - Extract full text using pdf-reading skill
   - Search for keywords from the claim (nouns, key concepts)
   - Extract relevant passages (context: 2–3 sentences before/after the keyword)
   - Compare the claim in the manuscript to the passage in the PDF
   - Record verdict: ✓ **Faithful** (claim matches source), ⚠ **Overstated** (claim goes beyond source), ✗ **Unsupported** (no matching passage found), or **Partial** (only part of the claim is supported)

5b. **Check for unquoted verbatim phrases:** While the PDF text is in hand, also scan the manuscript sentence(s) surrounding this citation for word-for-word borrowing from the source that is not enclosed in quotation marks. A run of 5 or more consecutive words appearing identically in both texts is a strong signal; 4 words is worth flagging if the phrasing is distinctive.

   ```python
   import re

   def ngrams(text, n):
       words = re.findall(r'\b\w+\b', text.lower())
       return [tuple(words[i:i+n]) for i in range(len(words) - n + 1)]

   manuscript_context = # the 1-3 sentences around the citation in the manuscript
   pdf_text = # full PDF text

   for n in (6, 5, 4):
       ms_grams = set(ngrams(manuscript_context, n))
       pdf_grams = set(ngrams(pdf_text, n))
       matches = ms_grams & pdf_grams
       if matches:
           # reconstruct matched phrase and check it isn't already in quotes
           for gram in matches:
               phrase = ' '.join(gram)
               # check manuscript context for surrounding quote marks
               pattern = r'["“”‘’][^"]*' + re.escape(phrase)
               if not re.search(pattern, manuscript_context.lower()):
                   flag_unquoted(phrase, manuscript_context)
   ```

   Flag any match with the verdict **⚠ Unquoted verbatim phrase** and report:
   - The matched phrase
   - The original sentence in the manuscript
   - The source sentence from the PDF
   - A suggested fix: either wrap in quotation marks and add a page reference, or paraphrase

   Report format:

   ```
   ⚠ Unquoted verbatim phrase — Smith et al. 2020

   Manuscript: "...measuring biodiversity is notoriously difficult and expensive..."
   Source:     "...measuring biodiversity is notoriously difficult in all fields..."

   Matched phrase: "measuring biodiversity is notoriously difficult"

   Fix: Either quote directly — 'measuring biodiversity is "notoriously difficult"
   (Smith et al. 2020, p. X)' — or paraphrase to make the wording clearly your own.
   ```

6. **Missing PDFs:** Before flagging a paper as absent, always: (a) search Zotero using variant spellings, partial author names, and keywords from the title; (b) check the manuscript's full reference list (which may be in a separate document if the PDF says "reference list located elsewhere") for the complete citation details, then re-search. Only flag as "PDF not found in library" after both steps have been attempted. When flagging, include the full reference entry from the reference list so the user can verify.

7. **Web fallback for missing PDFs:** If a PDF is not found in Zotero after step 6, attempt to retrieve the source from the web using the DOI or URL recorded in the reference list entry. Use the WebFetch tool with the DOI URL (e.g., `https://doi.org/10.xxxx/xxxxx`) or the direct URL if one is given. If the fetch succeeds, treat the retrieved content as the source for claim verification and proceed with the usual faithfulness check. If the fetch is blocked by the network proxy (`EGRESS_BLOCKED` error), record this explicitly and note: "PDF not in Zotero; web access blocked — claim could not be independently verified. Reference entry: [full citation]." In either case (success or blocked), include the full reference list entry in the report. **Never silently skip verification for a missing PDF** — always report what was attempted and what was found.

### Output Format

For each citation, report:

```
[Citation ID] Smith et al. 2020

Claim: "Biodiversity loss is accelerating globally (Smith et al. 2020)."

Verdict: ✓ Faithful
Source passage: "Our analysis shows accelerating declines in species richness 
across terrestrial and marine ecosystems over the past two decades."

Confidence: High (exact match to claim)

---
```

Alternatively for overstatement:

```
[Citation ID] Jones 2019

Claim: "All temperate forests show declining productivity (Jones 2019)."

Verdict: ⚠ Overstated
Source passage: "Productivity declines were observed in 68% of sampled 
temperate forests in North America."

Issue: The manuscript claims universality; the source reports 68% prevalence. 
Suggest: "Most temperate forests show declining productivity (Jones 2019)."

---
```

---

## Stage 2: Citation Gap Detection & Suggestion

> Calls `litmap search` against the user's local embeddings database for both REFS-marked and implicit citation gaps. Requires Claude Code runtime — see banner above.

### Step 0 — Resolve `(REFS)` Placeholders First

Before scanning for implicit gaps, check whether the author has explicitly marked citation gaps with a `(REFS)` placeholder. These are the highest-priority gaps because the author already knows a citation is needed.

**Detect placeholders:**

```bash
grep -n "REFS" manuscript.txt
```

Or in Python on the extracted text:

```python
import re
for m in re.finditer(r'\bREFS\b', text):
    start = text.rfind('\n', 0, m.start()) + 1
    end = text.find('\n', m.end())
    print(f"Line {text[:m.start()].count(chr(10))+1}: {text[start:end].strip()}")
```

For each `(REFS)` occurrence, extract **5–10 lines of surrounding context** — enough to understand the full claim being supported, including any citations already present in the same sentence or parenthetical cluster (they reveal the topic and the kind of evidence expected).

**Understand the claim:** Identify:
- The core assertion (what is being claimed?)
- The type of evidence expected (empirical study? review? methodology paper? grey literature?)
- The implied audience (e.g., a claim about audit standards needs assurance/accounting literature; a claim about SDMs needs ecology literature; a claim about market mechanisms needs economics literature)
- Whether the REFS is additive (joining a cluster of existing citations) or the sole citation for the claim

**Search Zotero for candidates — semantic + proper-noun pass:**

*Primary — semantic search (litmap):*

```bash
uv run --project ~/src/Cowork/litmap litmap search \
  --query "<claim assertion + 1–2 sentences of context>" \
  --top-k 8 --format json
```

Parse the JSON `results` array. Each entry has `zotero_key`, `title`, `authors`, `year`, `doi`, `similarity`.

*Complement — proper-noun pass (catches species names / acronyms / methods that semantic embeddings sometimes underweight):*

```python
import re, sqlite3

def extract_proper_nouns(text: str) -> list[str]:
    pattern = re.compile(
        r'\b[A-Z]{2,}\b'
        r'|\b[A-Z][a-z]+(?:\s[A-Z][a-z]+){1,2}\b'
    )
    skip = {"The", "A", "An", "In", "We", "Our", "This", "These", "For", "To"}
    seen, result = set(), []
    for m in pattern.findall(text):
        if m not in skip and m not in seen:
            seen.add(m); result.append(m)
    return result
```

For each extracted term, run a focused `LIKE` query against `itemDataValues` for title and abstract (use the SQL pattern from Tier 1 of the zotero skill). Merge results with the litmap output, deduping by `zotero_key`; semantic hits sort first by similarity, proper-noun-only hits append after.

**Filter and rank candidates:** Read the abstract of each candidate and assess:
- Does the paper directly support the claim, or only tangentially?
- Is it the right type of evidence (primary study, review, grey literature, canonical reference)?
- Is it already cited elsewhere in the manuscript? (Avoid suggesting duplicates unless the same paper is appropriate in multiple locations.)
- Does it serve the implied audience of the claim?

Discard weak matches. For strong candidates, retrieve full reference details (authors, year, title, journal, DOI) using the item key.

**Suggest no more than 3–4 candidates per REFS**, ranked by fit. For each, write one sentence explaining *why* it supports the specific claim — not just what the paper is about.

Note: if the REFS sits in a cluster with existing citations, candidates should complement rather than duplicate the existing papers. Read those existing citations' titles/abstracts briefly to understand what is already covered.

**Output format for REFS resolution:**

```
REFS — Line [N]
Context: "[the sentence(s) containing REFS, with surrounding citations if any]"
Claim: [1-sentence description of what evidence is needed]
Implied audience: [e.g., assurance/finance/ecology/policy]

Suggested citations:
1. [Zotero key] Author et al. (year). Title. Journal. DOI.  (similarity 0.87)
   → [1-sentence rationale]

2. [Zotero key] Author et al. (year). Title. Journal. DOI.  (similarity 0.78)
   → [1-sentence rationale]

[If no strong candidates found:]
No strong candidates found in your library. Consider searching [suggested alternative sources].
```

Deliver all REFS resolutions as a dedicated section before the implicit-gap analysis below.

---

### Implicit gap detection

#### Input
- The manuscript
- The faithfulness audit + reference list from Stage 1
- User's Zotero library

#### Process

1. **(Optional, ≥30 unique citations only) Up-front cluster overview.**

    ```bash
    uv run --project ~/src/Cowork/litmap litmap cluster \
      --manuscript <manuscript_path> \
      --output /tmp/audit_clusters \
      --format md
    ```

    Read `/tmp/audit_clusters.md` and present the thematic outline before per-claim analysis. Use it to spot topic areas under- or over-cited.

2. **Identify unsupported claims:** Flag all sentences/paragraphs with:
   - No citation
   - Citation verdict ✗ **Unsupported** or ⚠ **Overstated** (from Stage 1)
   - A citation to a paper that exists in Zotero but lacks the supporting passage

3. **Extract claim context:** For each flagged claim, extract:
   - The claim text (1–2 sentences)
   - The section heading
   - Key concepts (nouns, verbs, relationships)

4. **Search for candidates — semantic + proper-noun pass:**

    *Semantic (litmap, primary, always runs):*

    ```bash
    uv run --project ~/src/Cowork/litmap litmap search \
      --query "<claim_text>. <sentence_context>" \
      [--collection "<scope>"] \
      --top-k 5 --format json
    ```

    `--collection` is included only if the user named one ("audit my manuscript against my Chapter 2 refs only").

    *Proper-noun pass (always runs alongside):* use the same `extract_proper_nouns` helper from Step 0 and the same `LIKE` query against title + abstract. Merge with litmap results; semantic hits sort first by similarity, proper-noun-only hits append after.

    If both passes return nothing usable (litmap top result similarity < 0.75 AND proper-noun pass returns no usable candidates), record "no semantically similar papers in your library; consider broader literature search outside the local Zotero collection" for that claim and move on.

5. **Rank and suggest:** For each candidate:
   - Check if the result's abstract/title matches the claim
   - Assign relevance: **Perfect match**, **Strong**, **Moderate**, **Weak**
   - Extract a brief snippet from the paper that supports the claim
   - Write a 1-sentence justification for why this paper fits

6. **Prioritise suggestions:**
   - Up to 2 papers for claims with no citation
   - 1 replacement paper if the current citation is unsupported
   - 1–2 supplementary papers if the citation is weak or too narrow

   **Filter overlap with existing citations:** drop any candidate already cited in the manuscript (compare DOI first, then `lastname year` against the Stage 1 reference list).

#### Output Format (implicit gaps)

```
[Claim Location] Results section, paragraph 3

Unsupported claim: "Deep learning models outperform traditional SDMs
in predicting species distributions."

**Suggested citations:**

1. **Max et al. 2022** — "Deep learning for species distribution modeling:
A benchmark study"
   Relevance: Perfect match  (similarity 0.91)
   Snippet: "Deep neural networks achieved 12% higher AUC than MaxEnt
   models on average across 500 species."
   Justification: Directly compares deep learning to traditional SDMs with
   quantitative results.

2. **Brown & Kim 2021** — "Machine learning in biodiversity prediction"
   Relevance: Strong  (similarity 0.78)
   Snippet: "Recent advances in neural networks have improved predictive
   accuracy for spatial distribution models."
   Justification: Broader review of ML in SDMs; supports the claim but
   is less specific than Max et al. 2022.

---
```

#### Failure modes

- **Cluster step fails** (e.g., < 2 cited papers found in Zotero): skip silently, proceed to per-claim search.
- **`litmap` unavailable** (command not found, model fails to load, embeddings DB missing): emit the install/sync message ("Run `uv pip install -e .` from `~/src/Cowork/litmap`" or "Run `litmap sync` first") and abort Stage 2. Do NOT silently fall back to keyword/full-text-only search — the user explicitly requested a semantic audit.
- **Manuscript `--collection` scope mismatch**: if `--collection X` was passed but X has fewer than 5 papers, warn the user and offer to broaden.

---

## Stage 3: Logical Consistency Check

### Input
- The manuscript (full text)
- Citation audit and gap analysis from Stages 1–2

### Process

1. **Extract key claims:** Scan the manuscript section-by-section and list major claims:
   - Research questions and hypotheses
   - Assertions about the state of knowledge (e.g., "X is understudied")
   - Methodological choices and their justifications
   - Results and interpretations
   - Implications and future directions

2. **Check for contradictions:** 
   - Do any two claims directly contradict each other?
   - Are terms used inconsistently (e.g., "biodiversity" defined one way in intro, used differently in discussion)?
   - Are definitions circular or vague?

3. **Check for reasoning gaps:**
   - Are logical leaps made without justification? (e.g., "Study X found Y; therefore Z" without connecting Y to Z)
   - Does the conclusion follow from the results?
   - Are assumptions stated explicitly?
   - Is the scope of claims matched to the scope of evidence?

4. **Check for scope creep:**
   - Are generalizations overstated? (e.g., "This study of 10 species shows that...")
   - Are causal claims made where only correlations are shown?
   - Are statements qualified appropriately? (e.g., "may," "likely," "in this region")

5. **Flag undefined or under-defined terms:**
   - Are key concepts (e.g., "resilience," "ecosystem services") defined before use?
   - Are acronyms introduced before first use?

### Output Format

```
**Contradiction detected** (Abstract vs. Introduction)

Abstract: "Remote sensing cannot reliably measure forest biomass."
Introduction: "High-resolution satellite data enable accurate biomass estimation."

Recommendation: Clarify the distinction (e.g., "passive optical remote sensing 
cannot reliably measure biomass; active LiDAR-based approaches are more 
promising").

---

**Reasoning gap** (Methods → Results)

Methods: "We used species occurrence data from iNaturalist."
Results: "Species richness patterns aligned with predictions."

Issue: The connection between iNaturalist (which is spatially biased toward 
populated areas) and richness predictions is not established. Does this 
bias affect the conclusions?

Recommendation: Add a limitations paragraph addressing data bias.

---

**Scope creep** (Results → Discussion)

Results: "In our 10-site study, SDM accuracy was 0.82 AUC."
Discussion: "Deep learning SDMs are highly accurate for predicting global 
species distributions."

Issue: Results from 10 sites do not support a global claim.

Recommendation: Qualify: "Our results suggest that deep learning SDMs may 
achieve high accuracy; further validation across diverse regions is needed."

---
```

---

## Stage 4: Copyediting Pass

### Input
- The full manuscript

### Process

1. **Grammar & mechanics:**
   - Subject-verb agreement
   - Tense consistency (past for completed work, present for established facts)
   - Pronoun reference clarity
   - Active vs. passive voice (prefer active where possible, but passive acceptable in methods)

2. **Clarity & concision:**
   - Wordy phrases (e.g., "in order to" → "to", "due to the fact that" → "because")
   - Unclear pronoun antecedents
   - Overly complex sentences (break into 2–3 shorter sentences if needed)
   - Jargon without definition

3. **Flow & transitions:**
   - Do paragraphs flow logically?
   - Are transitions between sections smooth?
   - Do topic sentences align with paragraph content?

4. **Style & consistency:**
   - Parallel structure in lists or comparisons
   - Consistent capitalization and formatting
   - Consistent citation style (author-year format; check for mixed styles)
   - Number formatting (spell out <10; numerals ≥10; exception: start of sentence)

5. **Common scientific writing issues:**
   - Hedging (excessive "may," "might," "could" — use sparingly and intentionally)
   - Present perfect tense misuse (e.g., "Studies have shown that X" — who? when?)
   - "Data" as singular (data is plural; use "datasets" or "data points" for singular)
   - Abbreviations introduced but not used; or used before introduction

6. **Sentence-level revisions:** Provide before/after examples.

### Output Format

```
**Grammar issue** (Page 3, Results)

Original: "The results shows that deep learning models performs better 
than traditional SDMs."

Corrected: "The results show that deep learning models perform better 
than traditional SDMs."

Issue: Subject-verb agreement (plural "results" requires "show" and "perform").

---

**Clarity issue** (Page 1, Abstract)

Original: "We used remote sensing and machine learning, which is a powerful 
combination for predicting species distributions."

Revised: "We combined remote sensing with machine learning to predict 
species distributions with high accuracy."

Issue: "which is a powerful combination" is vague and wordy. The revision 
is more direct and specific.

---

**Flow issue** (Page 5, Discussion, para. 2)

Original: [Three sentences about climate change] [Abrupt shift] [One sentence 
about policy implications]

Revised: Add a transition: "These findings have important implications for 
conservation policy..." before the policy sentence.

---

**Style issue** (Inconsistent number formatting)

Original: "We sampled 10 sites across 3 regions in 4 years."

Check: Are numbers <10 and ≥10 formatted consistently? Should be either: 
"We sampled ten sites across three regions in four years" (spell out all <10) 
OR "We sampled 10 sites across 3 regions in 4 years" (numerals for all). 
Pick one and apply throughout.

---
```

---

## Integration & Output

Run all four stages in sequence. Deliver:

1. **Annotated manuscript** with inline comments (via track changes or comment blocks)
2. **Summary report** listing:
   - **Citation audit:** # faithful, # overstated, # unsupported, # missing PDFs
   - **Citation gaps:** # claims without support, # suggested citations
   - **Logical issues:** # contradictions, # reasoning gaps, # scope creep, # undefined terms
   - **Copyediting:** # grammar issues, # clarity issues, # style issues

3. **Prioritized revision checklist** (must-fix vs. nice-to-fix) so the user can tackle the most important issues first.

---

## Dependencies

- **zotero skill** — to query the Zotero library and retrieve PDFs
- **pdf-reading skill** — to extract text from PDFs and verify claims
- **file-reading skill** (if docx/pdf uploaded) — to read the manuscript
- Standard regex and text processing (built-in)

---

## Workflow Notes

- **Timing:** Expect 10–20 minutes for an 8,000-word manuscript, depending on citation count and PDF availability.
- **Zotero integration:** Assumes the user's Zotero library is accessible at `/mnt/Zotero/` (Cowork environment) or via the zotero skill.
- **Citation format:** Works with author-year citations (Smith 2020, Jones et al. 2019). Numbered citations [1], [2] require different parsing; confirm format before beginning.
- **Tone:** Citations and logical consistency feedback should be constructive and specific; copyediting suggestions should include examples and rationales.
- **Scope:** Focuses on peer-review readiness, not format compliance. For journal-specific formatting (e.g., Nature, Ecology Letters), recommend consulting target journal's author guidelines after audit.
