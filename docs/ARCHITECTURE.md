# AssistNavigator — Architecture & Technical Specification

Companion to `PROJECT-PLAN.md` (what and why) and `ASSIST-PROTOCOL.md` (verified wire protocol).
This file is the **what to build** contract: stack, data model, algorithms, API surface, UI behaviour.

---

## 1. Shape of the product

A **local-first, single-user web application.** The user runs one command; a server starts on
`http://localhost:8787` and opens in their browser. All state — SQLite DB and downloaded PDFs — lives
in a local library directory. No accounts, no cloud, no multi-tenancy.

Rationale: the corpus is large binary PDFs, the workload is bursty crawling, and engineers want the
library on their own disk next to their project files. A hosted multi-user version is a possible
later milestone (see PROJECT-PLAN §9) but designing for it now would add auth, quotas, and storage
abstraction for no near-term benefit.

Default library location: `%LOCALAPPDATA%\AssistNavigator` on Windows, `~/.local/share/assistnavigator`
elsewhere; overridable with `ASSISTNAV_HOME`.

    <library>/
      assistnav.db            SQLite (WAL mode)
      pdfs/<ident>/<dmxid>.pdf
      text/<ident>/<dmxid>.txt        cached extracted text
      logs/assistnav.log

## 2. Stack

**Backend — Python 3.12**

| Concern | Choice | Why |
|---|---|---|
| Web framework | FastAPI + Uvicorn | async, typed, automatic OpenAPI for the frontend client |
| HTTP client | `httpx` (sync client in a worker thread pool) | cookie jar, redirect control, timeouts |
| HTML parsing | `selectolax` (fallback `lxml`) | fast, tolerant of the site's malformed WebForms HTML |
| PDF text + geometry | **PyMuPDF (`fitz`)** | gives word-level bounding boxes, which the citation-highlight feature depends on; `pdfplumber`/`pypdf` do not do this as well |
| OCR fallback | `ocrmypdf` + Tesseract (optional dependency) | legacy scanned specs |
| DB | SQLite + SQLAlchemy 2.0 Core, Alembic migrations | single-file, zero-config, FTS5 built in |
| Fuzzy matching | `rapidfuzz` | title-based disambiguation of search hits |
| Jobs | in-process asyncio worker + `crawl_tasks` table | resumable across restarts without Redis/Celery |
| Progress | Server-Sent Events | one-way, trivially simple, no WebSocket plumbing |
| Packaging | `uv` + a `assistnav` console script | `uv run assistnav serve` |

**Frontend — TypeScript**

| Concern | Choice | Why |
|---|---|---|
| Framework | React 19 + Vite | |
| Data | TanStack Query | caching, background refetch, SSE-driven invalidation |
| Styling | Tailwind + shadcn/ui | dense, professional, keyboard-friendly defaults |
| **Graph** | **Cytoscape.js** with `fcose` + `elk` layouts | handles 1k+ nodes, has compound nodes, mature layout options, exports SVG/PNG. React Flow is nicer for custom node UI but degrades past a few hundred nodes and has weaker automatic layout — wrong trade for a citation graph |
| PDF viewer | `pdfjs-dist` driven directly (not `react-pdf`) | we need the text layer and custom highlight rects |
| Client state | Zustand | graph filters, selection, split-pane sizing |
| Tests | Vitest + Playwright | |

The built frontend is served as static files by FastAPI, so there is one process and one port.

## 3. Data model

SQLite. All timestamps UTC ISO-8601.

```sql
-- One row per logical document (a Document ID), whether or not we have the PDF.
documents (
  id                INTEGER PK,
  canonical_id      TEXT UNIQUE NOT NULL,  -- normalised, e.g. 'MIL-STD-2073-1'
  display_id        TEXT NOT NULL,         -- as ASSIST prints it
  ident_number      INTEGER UNIQUE,        -- ASSIST identity; NULL if unresolved/external
  title             TEXT,
  status            TEXT,                  -- Active | Inactive | Canceled | Withdrawn | Unknown
  current_revision  TEXT,                  -- e.g. 'R(3)'
  fsc_area          TEXT,
  doc_date          TEXT,
  doc_category      TEXT,
  preparing_activity TEXT,
  lead_std_activity TEXT,
  custodians_json   TEXT,
  is_external       INTEGER NOT NULL DEFAULT 0,  -- ASTM/SAE/ISO/ANSI/STANAG/DoDI etc.
  external_body     TEXT,                        -- 'ASTM', 'SAE', 'ISO', 'NATO', 'DoD', ...
  resolution_state  TEXT NOT NULL,  -- unresolved | resolved | ambiguous | not_in_assist | failed
  resolution_note   TEXT,
  metadata_fetched_at TEXT,
  created_at        TEXT NOT NULL
)

-- One row per downloadable revision of a document.
revisions (
  id            INTEGER PK,
  document_id   INTEGER NOT NULL REFERENCES documents(id),
  dmxid         INTEGER NOT NULL,          -- ASSIST download token
  revision_label TEXT,                     -- 'D Change 3'
  doc_date      TEXT,
  is_current    INTEGER NOT NULL DEFAULT 0,
  pdf_path      TEXT,                      -- NULL until downloaded
  size_bytes    INTEGER,
  sha256        TEXT,                      -- local integrity only; NOT stable across downloads
  text_sha256   TEXT,                      -- watermark-stripped text hash; stable
  page_count    INTEGER,
  has_text_layer INTEGER,
  ocr_applied   INTEGER NOT NULL DEFAULT 0,
  downloaded_at TEXT,
  extracted_at  TEXT,
  UNIQUE(document_id, dmxid)
)

-- A citation edge. Source is a specific revision; target is a logical document.
refs (
  id              INTEGER PK,
  src_revision_id INTEGER NOT NULL REFERENCES revisions(id),
  dst_document_id INTEGER NOT NULL REFERENCES documents(id),
  raw_text        TEXT NOT NULL,        -- 'MIL-STD-2073-1E'
  cited_revision  TEXT,                 -- 'E' — may differ from current!
  kind            TEXT NOT NULL,        -- binding | informational | supersession | related
  section         TEXT,                 -- '2.2.1', '2.3', 'body', 'metadata'
  category_hint   TEXT,                 -- 'DEPARTMENT OF DEFENSE STANDARDS' etc.
  occurrences     INTEGER NOT NULL DEFAULT 1,
  confidence      REAL NOT NULL,        -- 0..1
  UNIQUE(src_revision_id, dst_document_id, kind)
)

-- Where each citation physically appears, for click-to-highlight navigation.
ref_spans (
  id        INTEGER PK,
  ref_id    INTEGER NOT NULL REFERENCES refs(id) ON DELETE CASCADE,
  page      INTEGER NOT NULL,   -- 0-indexed
  x0 REAL, y0 REAL, x1 REAL, y1 REAL,   -- PDF user-space, PyMuPDF origin
  snippet   TEXT
)

crawl_jobs (
  id INTEGER PK, root_document_id INTEGER, root_query TEXT,
  status TEXT,          -- queued | running | paused | done | failed | cancelled
  depth_limit INTEGER NOT NULL DEFAULT 2,
  max_docs INTEGER NOT NULL DEFAULT 100,
  include_external INTEGER NOT NULL DEFAULT 0,
  binding_only INTEGER NOT NULL DEFAULT 1,
  created_at TEXT, finished_at TEXT,
  stats_json TEXT       -- {queued, downloaded, skipped, unresolved, failed, bytes}
)

crawl_tasks (
  id INTEGER PK, job_id INTEGER NOT NULL REFERENCES crawl_jobs(id),
  canonical_id TEXT NOT NULL, depth INTEGER NOT NULL,
  parent_document_id INTEGER,
  state TEXT NOT NULL,  -- pending | resolving | downloading | extracting | done | skipped | failed
  attempts INTEGER NOT NULL DEFAULT 0, error TEXT,
  updated_at TEXT,
  UNIQUE(job_id, canonical_id)
)

collections (id, name, description, created_at)
collection_items (collection_id, document_id, note, added_at)
annotations (id, revision_id, page, x0,y0,x1,y1, color, note, created_at)
settings (key TEXT PK, value TEXT)

-- Full-text search across the local corpus.
CREATE VIRTUAL TABLE doc_fts USING fts5(
  body, canonical_id UNINDEXED, revision_id UNINDEXED, page UNINDEXED,
  tokenize='porter unicode61'
);
```

## 4. Document ID grammar and normalisation

This is the heart of the system; get it wrong and the graph is garbage. Implement as
`assistnav.docid` with an exhaustive unit-test table.

### 4.1 Recognised ASSIST families

| Family | Pattern | Example |
|---|---|---|
| Modern defense | `MIL-(STD|PRF|DTL|HDBK|SPEC)-<digits>[-<digits>][/<digits>][<rev>]` | `MIL-STD-2073-1E`, `MIL-DTL-38999/20K` |
| Legacy defense | `MIL-<1-3 letters>-<digits>[/<digits>][<rev>]` | `MIL-C-5541`, `MIL-A-8625F` |
| Military sheet | `MS<digits>`, `AN<digits>`, `AND<digits>` | `MS27488` |
| Federal spec | `<2-4 letters>-<letter>-<digits>[<rev>]` | `QQ-P-416F`, `TT-P-1757`, `MMM-A-132`, `PPP-B-601` |
| Commercial Item Description | `A-A-<digits>[<rev>]` | `A-A-59588` |
| Federal standard | `FED-STD-<digits>[<rev>]` | `FED-STD-595C` |

### 4.2 Recognised non-ASSIST (external nodes — never downloadable)

`STANAG <n>`, `ASTM <letter><digits>[-<yy>]`, `SAE (AS|AMS|ARP|J)<n>`, `ISO <n>[-<part>]`,
`ANSI[/x] <alnum.>`, `IEEE <n>`, `IPC-<alnum>`, `NAS<n>`, `UL <n>`, `EIA/JEDEC <alnum>`,
`NIST SP <n>`, `FIPS PUB <n>`, `DoD(I|M|D) <n.n>`, `CFR` cites, `NASA-STD-<n>`.

These become `documents` rows with `is_external=1`, `resolution_state='not_in_assist'`. They appear
in the graph as distinct grey nodes so the user can see the full dependency surface, but the crawler
never tries to fetch them.

### 4.3 Normalisation rules

1. Uppercase; collapse whitespace; normalise Unicode dashes to ASCII hyphen.
2. Strip trailing revision/change: `MIL-STD-129R w/CHANGE 3` -> base `MIL-STD-129`,
   `cited_revision = 'R Change 3'`. Handle `w/CHANGE n`, `NOTICE n`, `(n)`, `AMENDMENT n`,
   `SUPPLEMENT n`.
3. **Preserve** dash-suffixed part numbers that are part of the identity (`MIL-STD-2073-1`) and slash
   sheets (`MIL-DTL-38999/20`). Distinguishing `-1` (part of ID) from `A` (revision) is the single
   trickiest rule: a revision suffix is 1–2 letters attached with no separator; a part number is
   digits after a hyphen.
4. Repair line-break hyphenation from PDF extraction (`MIL-\nSTD-810` -> `MIL-STD-810`).
5. Drop the ASSIST download watermark line before any matching (see ASSIST-PROTOCOL §6).
6. Never emit a self-reference: a document citing its own ID (MIL-STD-129 mentions `MIL-STD-129R`
   173 times in its own headers) must be discarded.

### 4.4 Validated observations from the live corpus

Extraction run on MIL-STD-129 Rev R Change 3 produced, among others: `MIL-STD-130`,
`MIL-PRF-61002`, `MIL-DTL-64159`, `MIL-STD-290`, `MIL-STD-2073-1`, `MIL-DTL-4`, plus external
`STANAG 4281`, `STANAG 4329`, `ISO 15394`, `ASTM D5486`, `ANSI MH10.8.2`, `SAE AS5502`,
`DoDM 4140.27`. Note `MIL-DTL-4` — a legitimate 1-digit document ID ("Tires and Inner Tubes;
Packaging of"). Do not add a minimum-digit-count rule; it would drop real documents.

## 5. Reference extraction pipeline

Input: a downloaded PDF. Output: `refs` + `ref_spans` rows.

**Step 1 — Text and geometry.** PyMuPDF `page.get_text("dict")` for structured blocks and
`page.get_text("words")` for word boxes. Detect a missing text layer (mean chars/page below ~200)
and route to OCR (§6.4).

**Step 2 — Strip furniture.** Remove the ASSIST download watermark, page headers/footers (repeated
lines across many pages), and the table-of-contents region (lines ending in dot-leaders plus a page
number: `\.{4,}\s*\d+\s*$`). Skipping the TOC matters — it is full of section titles that look like
citations.

**Step 3 — Locate Section 2.** Find the heading `^\s*2\.?\s+APPLICABLE DOCUMENTS` (excluding TOC
hits), bounded by `^\s*3\.?\s+(DEFINITIONS|REQUIREMENTS)`. Within it, track the ALL-CAPS category
headers observed live: `DEPARTMENT OF DEFENSE SPECIFICATIONS`, `DEPARTMENT OF DEFENSE STANDARDS`,
`DEPARTMENT OF DEFENSE HANDBOOKS`, `FEDERAL SPECIFICATIONS`, `FEDERAL STANDARDS`,
`INTERNATIONAL STANDARDIZATION AGREEMENTS`, `NON-GOVERNMENT PUBLICATIONS`. Track subsection numbers
`2.2.1`, `2.2.2`, `2.3`.

Section 2 is laid out as a two-column `ID - Title` list which linearised text scrambles (verified: it
does). Use the word bounding boxes to re-associate IDs with their titles by vertical alignment — the
recovered title is a strong cross-check when resolving the ID against ASSIST search.

**Step 4 — Tokenise.** Run the §4 grammar over the whole document, recording page and bbox for each
hit.

**Step 5 — Classify each edge.**

| Location | `kind` | `confidence` |
|---|---|---|
| Inside Section 2.2.x / 2.3 | `binding` | 0.95 |
| Body text outside Section 2 | `informational` | 0.7 |
| From `qsDocDetails` supersession block | `supersession` | 1.0 |
| From `qsDocDetails` related block | `related` | 1.0 |

Merge duplicates by `(src_revision, dst_document, kind)`, summing `occurrences` and collecting all
spans.

**Step 6 — Resolve.** For each unseen `canonical_id`: check `documents`, else ASSIST search by
Document ID.
- Exact match on the Document ID column -> `resolved`, store `ident_number`.
- Multiple hits -> disambiguate with `rapidfuzz` against the title recovered in Step 3; if still
  under threshold, mark `ambiguous` and surface it in the Unresolved queue for manual pick.
- Zero hits and the ID matches an external family -> `not_in_assist`.
- Zero hits otherwise -> `unresolved` (likely an extraction artefact; shown in the coverage report).

**Step 7 — Index.** Write page text into `doc_fts`.

Extraction must be **idempotent and re-runnable**: `POST /api/revisions/{id}/reextract` deletes and
rebuilds that revision's refs, so the grammar can be improved without re-downloading anything.

## 6. Crawl orchestrator

**Algorithm.** Breadth-first from the root document.

    queue = [(root, depth=0)]
    while queue and downloaded < max_docs:
        doc, d = queue.pop_front()
        if doc in library and not force_refresh: reuse
        else: resolve -> fetch details -> download current revision -> extract
        if d < depth_limit:
            for ref in refs(doc):
                if ref.kind == 'binding' or not binding_only:
                    if ref.is_external and not include_external: skip
                    enqueue(ref.dst, d+1)

**Defaults:** `depth_limit=2`, `max_docs=100`, `binding_only=true`, `include_external=false`.
Depth 2 from a typical MIL-STD is already tens of documents and hundreds of megabytes; the UI must
show a live estimate and let the user stop.

**Guarantees:**
- Every task is a `crawl_tasks` row, so a crash or restart resumes exactly where it left off.
- Cycles are handled by `UNIQUE(job_id, canonical_id)` — a document is visited once per job at its
  shallowest depth.
- Pause / resume / cancel from the UI.
- All network calls go through the single rate-limited client (ASSIST-PROTOCOL §8).
- Per-task failures are recorded and skipped, never fatal to the job.

### 6.4 OCR fallback

If `has_text_layer` is false and `ocrmypdf` is installed: run `ocrmypdf --skip-text --optimize 0`
into a sidecar file, set `ocr_applied=1`, then extract normally. If Tesseract is absent, mark the
revision `needs_ocr` and show a one-line banner in the UI explaining how to enable it. OCR must never
be a hard install requirement.

## 7. HTTP API

All under `/api`. FastAPI generates OpenAPI; the frontend client is generated from it.

**Search & documents**
- `GET  /search?q=&field=id|title|keywords|scope&status=&fsc=&page=` — live ASSIST search
- `GET  /documents?filter=&status=&downloaded=&sort=` — local library
- `GET  /documents/{canonical_id}` — metadata, revisions, in/out reference counts
- `POST /documents/{canonical_id}/download` — body `{revision?: dmxid}`, fetch one PDF
- `GET  /documents/{canonical_id}/pdf?dmxid=` — stream the local PDF to the viewer
- `POST /documents/{canonical_id}/refresh` — re-check ASSIST for a newer revision
- `POST /documents/{id}/resolve` — body `{ident_number}`, manual fix for an ambiguous ID

**Crawl**
- `POST /crawls` — body `{root, depth_limit, max_docs, binding_only, include_external}`
- `GET  /crawls/{id}` / `POST /crawls/{id}/pause|resume|cancel`
- `GET  /crawls/{id}/events` — **SSE** stream of `{task, state, progress, stats}`

**Graph**
- `GET /graph?root=&depth=&kinds=&status=&include_external=&limit=`
  returns `{nodes: [{id, label, title, status, downloaded, is_external, in_degree, out_degree,
  depth}], edges: [{source, target, kind, cited_revision, occurrences, stale}]}`
- `GET /graph/path?from=&to=` — shortest citation path, for "why is this document in my tree?"

**References**
- `GET /documents/{id}/refs?direction=out|in`
- `GET /refs/{id}/spans` — pages + rects for highlight
- `POST /revisions/{id}/reextract`

**Corpus**
- `GET  /fts?q=&limit=` — full-text search across downloaded PDFs, returns doc + page + snippet
- `GET  /reports/coverage?crawl_id=` — resolved / unresolved / external / failed breakdown
- `GET  /reports/staleness` — every edge whose `cited_revision` != target's `current_revision`
- `POST /export` — body `{format: 'csv'|'json'|'graphml'|'zip'|'pdf-report', scope}`

**Collections / annotations**: standard CRUD.

## 8. Frontend specification

### 8.1 Layout

Persistent left rail (Library, Graph, Crawls, Search, Reports, Collections). Main area is a resizable
split: **graph on the left, PDF viewer on the right**. Either pane can be maximised. The split is the
core interaction — clicking in the graph drives the viewer.

### 8.2 Graph view

**Encoding**
- Node fill by status: Active green, Inactive amber, Canceled/Withdrawn red, External grey.
- Node border: solid = PDF downloaded locally, dashed = known but not downloaded.
- Node size scaled by in-degree (how many documents depend on it).
- Edge style: solid = `binding`, dotted = `informational`, double-line = `supersession`.
- Edge colour: red when `cited_revision` is behind the target's `current_revision` (staleness is a
  first-class visual, not a buried report).
- The root node is visually pinned and larger.

**Layouts:** hierarchical by crawl depth (default), `fcose` force-directed, radial-from-root.
Layout choice persists per crawl.

**Controls:** depth slider (re-filters client-side, no refetch), toggles for
binding-only / show-external / show-not-downloaded, status filter chips, FSC/Area filter, text filter
that dims non-matching nodes.

**Interactions**
- Hover node: highlight it and its immediate neighbours, dim everything else; tooltip with title +
  status + revision.
- Click node: metadata panel; PDF opens in the right pane if downloaded, otherwise a Download button.
- Double-click node: re-root the graph on it, pushing a breadcrumb entry.
- **Click edge: open the *source* PDF in the right pane, jump to the exact page, and highlight the
  citation rect.** This is the feature that makes the tool worth using — it answers "where and in
  what context does A cite B?" in one click.
- Right-click node: context menu — re-root, crawl from here, open on quicksearch.dla.mil, add to
  collection, hide, expand one level.
- Selecting a node with un-crawled children shows a `+N` badge; clicking it expands one level
  on demand.

**Performance:** virtualise above ~500 nodes — collapse leaf clusters into a single expandable node
and warn before rendering more than 1,000.

**Navigation aids:** breadcrumb trail of visited roots, browser back/forward via URL state
(`/graph?root=MIL-STD-129&depth=2&layout=hier`), minimap, `Fit`, `Reset`, export PNG/SVG.

### 8.3 PDF viewer

pdf.js with a real text layer (so the user can select and copy). Additional behaviour:
- Jump-to-page-and-highlight driven by `ref_spans`.
- A **Section 2 outline** in the sidebar listing every extracted reference, each a clickable link
  that both scrolls the PDF and selects the corresponding node in the graph — navigation works in
  both directions.
- Inline chips next to recognised document IDs in the text layer: click to open that document.
- Search within document; highlight-and-annotate saved to `annotations`.

### 8.4 Library, crawls, reports

- **Library:** sortable/filterable table of every known document — ID, title, status, revision,
  downloaded?, size, in/out-degree, last checked. Bulk select -> download / re-check / export.
- **Crawl monitor:** live SSE progress — a task list with per-document state, running counts,
  bytes downloaded, current rate-limit wait, and Pause/Cancel.
- **Coverage report:** what the crawl resolved vs. what it could not, with a one-click manual-resolve
  action for each ambiguous ID. This is how the user learns to trust the graph.
- **Staleness report:** "MIL-STD-XXX cites MIL-STD-810F; current is H(1)" — directly useful when
  reviewing an old spec or a contract data package.

### 8.5 Accessibility and keyboard

Full keyboard operation of the graph (arrow keys traverse edges, Enter opens, `/` focuses search,
`Esc` clears selection). Every colour encoding is paired with a shape or border encoding so the graph
is readable without colour discrimination. Meets WCAG 2.1 AA contrast.

## 9. Testing strategy

- **Unit:** `docid` normalisation table (target 100+ cases including every family in §4 and the
  hyphenation/revision edge cases); WAF detection; watermark stripping.
- **Golden-file extraction:** commit the extracted-text fixtures (not the PDFs) for MIL-STD-129 and
  MIL-STD-962; assert the exact expected reference set. Any grammar change shows up as a diff.
- **Network:** all ASSIST interactions recorded as `respx` cassettes from the verified flows in
  ASSIST-PROTOCOL §9. **The test suite must never hit the live site.**
- **One opt-in live smoke test** (`pytest -m live`) that downloads MIL-STD-962 end-to-end. This is
  the canary for the site changing its markup — run it in CI weekly, not per-commit.
- **E2E:** Playwright over a seeded library — search, crawl, graph render, edge-click-to-highlight.

## 10. Failure modes to handle explicitly

| Failure | Handling |
|---|---|
| WAF block | backoff, session reset, surface plainly after 5 consecutive |
| Site markup changes | parsers raise `ParseError` with the offending HTML saved to `logs/`; the UI says "ASSIST changed its page layout" rather than showing an empty result |
| Document exists but PDF is restricted | mark `resolution_state='restricted'`, show a link to ASSIST, never retry |
| Scanned PDF, no OCR available | `needs_ocr`, still graphed via metadata, banner explains the fix |
| Reference to a canceled/superseded doc | still downloaded and graphed; flagged red — knowing a spec depends on a canceled standard is the point |
| Huge PDF (MIL-STD-810 is ~100MB) | streaming download with progress, configurable size cap, resumable |
| Disk filling | pre-flight free-space check; per-library size budget in settings |
