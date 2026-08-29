# AssistNavigator — Architecture & Technical Specification

Companion to `PROJECT-PLAN.md` (what and why) and `ASSIST-PROTOCOL.md` (verified wire protocol).
This file is the **what to build** contract: stack, data model, algorithms, API surface, UI behaviour.

---

## 1. Shape of the product

A **local-first, single-user web application.** The user runs one command; a server starts on
`http://localhost:8787` and opens in their browser. All state — SQLite DB and downloaded PDFs — lives
in a local library directory. No accounts, no cloud, no multi-tenancy.

**This code may assume there is exactly one user** (PROJECT-PLAN §8.2). Do not add tenant IDs,
permission checks, user columns, or a pluggable storage abstraction "for later". Bare SQLite, bare
filesystem, no auth. Sharing this tool later is an accepted rewrite, not a migration.

Rationale: the corpus is large binary PDFs, the workload is bursty crawling, and engineers want the
library on their own disk next to their project files.

Default library location: `%LOCALAPPDATA%\AssistNavigator` on Windows, `~/.local/share/assistnavigator`
elsewhere; overridable with `ASSISTNAV_HOME`.

    <library>/
      assistnav.db            SQLite (WAL mode)
      pdfs/<canonical_id>/<revision_key>.pdf
      text/<canonical_id>/<revision_key>.txt   cached extracted text
      inbox/                  watched folder for manual import (§6)
      inbox/.processed/       imported originals, moved here after ingest
      logs/assistnav.log

`revision_key` is the `dmxid` for an ASSIST download and `imp-<revision.id>` for a manual import.
The path layout must not assume an ASSIST origin — see §6.

## 2. Stack

**Backend — Python 3.12**

| Concern | Choice | Why |
|---|---|---|
| Web framework | FastAPI + Uvicorn | async, typed, automatic OpenAPI for the frontend client |
| HTTP client | `httpx` (sync client in a worker thread pool) | cookie jar, redirect control, timeouts |
| HTML parsing | `selectolax` (fallback `lxml`) | fast, tolerant of the site's malformed WebForms HTML |
| PDF text + geometry | **PyMuPDF (`fitz`)** | gives word-level bounding boxes, which the citation-highlight feature depends on; `pdfplumber`/`pypdf` do not do this as well |
| Folder watching | `watchdog` | the manual-import inbox (§6) |
| OCR fallback | `ocrmypdf` + Tesseract (optional dependency, **Phase 9**) | legacy scanned specs; deliberately off the MVP critical path (PROJECT-PLAN §8.8) |
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
  resolution_state  TEXT NOT NULL,  -- unresolved | resolved | ambiguous | not_in_assist
                                    --  | restricted | failed
  resolution_note   TEXT,
  metadata_fetched_at TEXT,
  created_at        TEXT NOT NULL
  -- Revision pinning (PROJECT-PLAN §8.5) is deliberately absent. If it is ever needed it is one
  -- nullable `pinned_revision_id` column here plus a branch in the resolver. Do not add it now.
)

-- One row per known revision of a document, from EITHER source.
-- Nothing below the library layer may assume an ASSIST origin (PROJECT-PLAN §8.3).
revisions (
  id            INTEGER PK,
  document_id   INTEGER NOT NULL REFERENCES documents(id),
  source        TEXT NOT NULL,             -- 'assist' | 'import'
  dmxid         INTEGER,                   -- ASSIST download token; NULL for an imported PDF
  revision_label TEXT,                     -- 'D Change 3'
  doc_date      TEXT,
  is_current    INTEGER NOT NULL DEFAULT 0,
  pdf_path      TEXT,                      -- NULL until downloaded
  size_bytes    INTEGER,
  sha256        TEXT,                      -- local integrity only; NOT stable across downloads
  text_sha256   TEXT,                      -- watermark-stripped text hash; stable, cross-source
  page_count    INTEGER,
  has_text_layer INTEGER,
  needs_ocr     INTEGER NOT NULL DEFAULT 0,  -- no text layer, OCR not yet applied (Phase 9)
  ocr_applied   INTEGER NOT NULL DEFAULT 0,
  import_id     INTEGER REFERENCES imports(id),  -- provenance for source='import'
  downloaded_at TEXT,                      -- fetch time, or import time
  extracted_at  TEXT
)
-- Identity differs by source, so it needs two partial indexes rather than one UNIQUE:
CREATE UNIQUE INDEX rev_assist_ident ON revisions(document_id, dmxid) WHERE dmxid IS NOT NULL;
CREATE UNIQUE INDEX rev_text_ident   ON revisions(document_id, text_sha256)
                                     WHERE text_sha256 IS NOT NULL;
-- The second is what makes an imported copy of an already-downloaded document collapse onto the
-- existing row instead of duplicating it, and vice versa. It is why text_sha256 must be computed
-- for every revision from both paths, not just for content comparison.

-- One row per file offered to the manual importer. Nothing is ever silently dropped.
imports (
  id             INTEGER PK,
  original_path  TEXT NOT NULL,
  original_name  TEXT NOT NULL,
  file_sha256    TEXT NOT NULL,     -- of the file as offered; cheap re-drop detection
  state          TEXT NOT NULL,     -- pending | identified | unidentified | duplicate
                                    --  | not_a_standard | failed
  detected_id    TEXT,              -- canonical_id read from the PDF header, NOT the filename
  detected_revision TEXT,
  header_snippet TEXT,              -- first-page text used for the decision, for the review UI
  document_id    INTEGER REFERENCES documents(id),
  revision_id    INTEGER REFERENCES revisions(id),
  error          TEXT,
  created_at     TEXT NOT NULL,
  resolved_at    TEXT
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
  depth_limit INTEGER NOT NULL DEFAULT 1,     -- see PROJECT-PLAN §5
  max_docs INTEGER NOT NULL DEFAULT 25,
  include_external INTEGER NOT NULL DEFAULT 0,
  binding_only INTEGER NOT NULL DEFAULT 1,
  expanded_from_job_id INTEGER REFERENCES crawl_jobs(id),  -- set for expand-one-level jobs
  created_at TEXT, finished_at TEXT,
  stats_json TEXT       -- {queued, downloaded, skipped, unresolved, external, failed, bytes,
                        --  frontier}  -- frontier = cited documents found but not fetched (§6)
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

Input is a revision from **either** source. This pipeline never inspects `revisions.source`.

**Step 1 — Text and geometry.** PyMuPDF `page.get_text("dict")` for structured blocks and
`page.get_text("words")` for word boxes. Detect a missing text layer (mean chars/page below ~200):
set `needs_ocr=1`, record the document from its metadata alone, and stop — the document still
appears in the graph and is listed in the coverage report as text-less. OCR arrives in Phase 9
(§6.5); until then this is a disclosed gap, never a silent one or an exception.

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

## 6. Ingestion: the two paths

Documents enter the library by crawl (§6.0–6.3) or by manual import (§6.4). The two are co-equal:
they converge on the same `revisions` rows and hand off to the same extraction pipeline (§5), which
never asks which path a document arrived by.

### 6.0 Crawl orchestrator

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

**Defaults:** `depth_limit=1`, `max_docs=25`, `binding_only=true`, `include_external=false`
(PROJECT-PLAN §5). Depth 2 is offered behind a confirmation showing the estimated document count and
byte volume.

**Guarantees:**
- Every task is a `crawl_tasks` row, so a crash or restart resumes exactly where it left off.
- Cycles are handled by `UNIQUE(job_id, canonical_id)` — a document is visited once per job at its
  shallowest depth.
- Pause / resume / cancel from the UI.
- All network calls go through the single rate-limited client (ASSIST-PROTOCOL §8).
- Per-task failures are recorded and skipped, never fatal to the job.
- A crawl may start from an **imported** document exactly as from a fetched one.

### 6.1 Frontier count

On completion the job records `stats.frontier`: the number of distinct documents cited by crawled
documents that were *not* themselves fetched because the depth limit stopped them. This is displayed
after every crawl and per node as a `+N` badge.

Its purpose is evidential, not decorative — the depth-1 default rests on a single observed document,
and the frontier count is how that guess gets replaced by a measurement over the user's real corpus
(PROJECT-PLAN §5, §8.7).

### 6.2 Expand one level

The primary navigation gesture, not an advanced feature. Expanding node *X* creates a new
`crawl_jobs` row with `root_document_id = X`, `depth_limit = 1`, and `expanded_from_job_id` set to
the originating job. Documents already in the library are reused, not re-fetched, so an expansion
costs only its genuinely new documents.

This — not long crawls — is what justifies the resumable `crawl_tasks` queue: every expansion is an
increment of work against an existing library, which is structurally the same problem as resuming an
interrupted crawl (PROJECT-PLAN §8.9).

### 6.3 Coverage report

Not a reporting extra; the primary output of a completeness tool, and therefore MVP (Phase 5).

Every reference encountered during a job lands in exactly one bucket — `resolved`, `ambiguous`,
`not_in_assist` (external), `restricted`, `needs_ocr`, `failed`, or `frontier` (beyond the depth
limit) — and the buckets must sum to the total references seen. **A reference that cannot be
classified is a bug, not a rounding error.** The report is the artefact a user shows a reviewer to
defend the completeness claim, so its invariant is checked in tests, not merely intended.

### 6.4 Manual import pipeline

`assistnav.import`. Watched folder (`<library>/inbox/`) plus a drag-and-drop upload endpoint; both
enter the same pipeline. Every file produces an `imports` row before anything else happens, so a
file can never be dropped without a trace.

1. **Hash** the file. A `file_sha256` already present in `imports` → `state='duplicate'`, stop.
2. **Identify from the PDF's own first page**, never the filename (PROJECT-PLAN §8.6). Render page 1
   text and run the §4 grammar over the header region, taking the ID and revision from the standard
   cover-page position. Store the raw `header_snippet` regardless of outcome — the review UI shows
   it, so a failed parse is diagnosable without reopening the file.
3. **Not a standard** (no ID recoverable and no plausible cover page) → `state='not_a_standard'`.
   Listed, not deleted.
4. **No confident ID** → `state='unidentified'`, queued for manual assignment with page 1 rendered
   alongside a filename-derived *suggestion* (a hint only, never an automatic answer).
5. **Identified** → find or create the `documents` row; compute `text_sha256`; if a revision with
   that text hash already exists for the document, link the import to it and mark `duplicate` —
   this is how an imported copy of an already-downloaded PDF collapses onto one node. Otherwise
   create a `revisions` row with `source='import'`, `dmxid=NULL`.
6. **Extract** via §5, unchanged. Move the original into `inbox/.processed/`.
7. If online, optionally enrich the `documents` row from ASSIST metadata (status, current revision).
   This is what lets an imported document appear in the staleness report — but it is strictly
   optional, and every import step above works with no network at all.

### 6.5 OCR fallback *(Phase 9)*

Deferred off the MVP critical path (PROJECT-PLAN §8.8). When it lands: for revisions with
`needs_ocr=1`, if `ocrmypdf` is installed, run `ocrmypdf --skip-text --optimize 0` into a sidecar
file, set `ocr_applied=1`, clear `needs_ocr`, and re-extract. If Tesseract is absent, the revision
stays `needs_ocr` and the UI shows a one-line banner explaining how to enable it. OCR must never be
a hard install requirement, and OCR-derived references carry reduced confidence — a plausible wrong
reference is worse than a disclosed missing one.

## 7. HTTP API

All under `/api`. FastAPI generates OpenAPI; the frontend client is generated from it.

**Search & documents**
- `GET  /search?q=&field=id|title|keywords|scope&status=&fsc=&page=` — live ASSIST search
- `GET  /documents?filter=&status=&downloaded=&sort=` — local library
- `GET  /documents/{canonical_id}` — metadata, revisions, in/out reference counts
- `POST /documents/{canonical_id}/download` — body `{revision?: dmxid}`, fetch one PDF
- `GET  /revisions/{id}/pdf` — stream the local PDF to the viewer. Addressed by revision id,
  **not** by `dmxid`: an imported revision has no download token, and a dmxid-keyed viewer
  route would silently make imported documents unopenable.
- `POST /documents/{canonical_id}/refresh` — re-check ASSIST for a newer revision
- `POST /documents/{id}/resolve` — body `{ident_number}`, manual fix for an ambiguous ID

**Import**
- `POST /imports` — multipart upload, one or more PDFs (drag-and-drop path)
- `GET  /imports?state=` — the import queue, unidentified first
- `POST /imports/{id}/assign` — body `{canonical_id, revision_label?}`, manual identification
- `POST /imports/{id}/reject` — mark `not_a_standard`, keep the row
- `GET  /imports/{id}/page1` — rendered first page PNG for the review UI
- `GET  /imports/watch` — watch-folder status (path, enabled, last scan)

**Crawl**
- `POST /crawls` — body `{root, depth_limit=1, max_docs=25, binding_only, include_external}`
- `POST /crawls/{id}/expand` — body `{document_id}`, the expand-one-level gesture (§6.2)
- `GET  /crawls/{id}` / `POST /crawls/{id}/pause|resume|cancel`
- `GET  /crawls/{id}/events` — **SSE** stream of `{task, state, progress, stats}`
- `GET  /crawls/{id}/estimate` — pre-flight document count and byte estimate for a depth change

**Graph**
- `GET /graph?root=&depth=&kinds=&status=&include_external=&limit=`
  returns `{nodes: [{id, label, title, status, downloaded, source, is_external, needs_ocr,
  in_degree, out_degree, depth, frontier_count}],
  edges: [{source, target, kind, cited_revision, occurrences, stale}]}`
  `frontier_count` drives the `+N` expand badge; `source` distinguishes fetched from imported.
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

Persistent left rail: Library, Graph, Crawls, **Import**, Search, **Coverage**, **Staleness**,
Collections. Coverage and Staleness are top-level destinations, not tabs inside a Reports page —
they are the tool's primary outputs (PROJECT-PLAN §4.1). Import sits beside Crawls at equal weight,
not under a "more" menu.

Main area is a resizable split: **graph on the left, PDF viewer on the right**. Either pane can be
maximised. The split is the core interaction — clicking in the graph drives the viewer.

A persistent badge in the rail shows the count of unidentified imports and unresolved references.
Outstanding gaps in a completeness claim are always visible; the user should never have to go
looking for them.

### 8.2 Graph view

**Encoding**
- Node fill by status: Active green, Inactive amber, Canceled/Withdrawn red, External grey.
- Node border: solid = PDF held locally, dashed = known but not held. A small corner glyph marks
  `source='import'`, and another marks `needs_ocr` — both are provenance facts a reviewer will ask
  about, and neither changes the node's standing in the graph.
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
- **`+N` expand badge: permanently visible on every node with un-crawled children**, not revealed on
  selection. One click expands one level — no dialog, no job configuration, no confirmation. This is
  the primary way the graph grows (§6.2, PROJECT-PLAN §5); at depth-1 defaults the user will do it
  constantly, so it must cost one click and show progress in place.
- Right-click node: context menu — re-root, crawl from here, open on quicksearch.dla.mil, add to
  collection, hide.

**Performance:** depth-1 defaults keep MVP graphs well inside Cytoscape's comfortable range, so
virtualisation is Phase 9 work, not MVP. When it lands: collapse leaf clusters above ~500 nodes and
warn before rendering more than 1,000.

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

### 8.4 Library, import, crawls, reports

- **Library:** sortable/filterable table of every known document — ID, title, status, revision,
  held locally?, source, size, in/out-degree, last checked. Bulk select -> download / re-check /
  export.
- **Import:** the watch-folder path with a reveal-in-file-manager button, a drop zone, and the
  import queue. Unidentified items show the rendered first page beside an ID field pre-filled with a
  *suggestion* the user must confirm — never a silent auto-assignment. Items already imported are
  shown as duplicates rather than hidden, so a user who drops a folder twice sees why nothing
  happened.
- **Crawl monitor:** live SSE progress — a task list with per-document state, running counts,
  bytes downloaded, current rate-limit wait, and Pause/Cancel. On completion it shows the frontier
  count prominently (§6.1).
- **Coverage report:** every reference the crawl saw, in its bucket, with the buckets summing to the
  total (§6.3), and a one-click manual-resolve action for each ambiguous ID. Exportable as-is — it
  is the artefact a user hands a reviewer.
- **Staleness report:** "MIL-STD-XXX cites MIL-STD-810F; current is H(1)" — a standing top-level
  screen over the whole library, filterable by collection, sortable by how far behind each citation
  is, and exportable. This is the output users will paste into a design review.

### 8.5 Accessibility and keyboard

Full keyboard operation of the graph (arrow keys traverse edges, Enter opens, `/` focuses search,
`Esc` clears selection). Every colour encoding is paired with a shape or border encoding so the graph
is readable without colour discrimination. Meets WCAG 2.1 AA contrast.

## 9. Testing strategy

- **Unit:** `docid` normalisation table (target 100+ cases including every family in §4 and the
  hyphenation/revision edge cases); WAF detection; watermark stripping.
- **Golden-file extraction:** commit the extracted-text fixtures (not the PDFs) for MIL-STD-129 and
  MIL-STD-962; assert the exact expected reference set. Any grammar change shows up as a diff.
- **Coverage invariant:** a property test asserting that for any crawl, the coverage buckets sum to
  the total references encountered. An unclassifiable reference must fail the suite (§6.3).
- **Source equivalence:** the same document fetched from ASSIST and imported from a renamed file
  must produce identical `documents`, `refs`, and graph rows, and must collapse to one revision.
  This test is what keeps the two import paths co-equal as the code grows.
- **Import identification:** a table of cover-page header fixtures — modern, legacy, amended,
  change-noticed, and a deliberately misleading filename — asserting that the ID comes from the
  header and that a failed parse lands in the unidentified queue rather than raising or guessing.
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
| Scanned PDF, no text layer | `needs_ocr`, still graphed via metadata, counted in the coverage report as a disclosed gap; OCR is Phase 9, not a blocker |
| Imported PDF cannot be identified | `imports.state='unidentified'` with page 1 rendered for manual assignment; never guessed from the filename, never silently skipped |
| Imported PDF duplicates a downloaded one | collapsed onto the existing revision via `text_sha256`; the import row records the link rather than vanishing |
| Import of a non-Statement-A document | flagged like any other restricted document and excluded from export (PROJECT-PLAN §9) |
| Reference to a canceled/superseded doc | still downloaded and graphed; flagged red — knowing a spec depends on a canceled standard is the point |
| Huge PDF (MIL-STD-810 is ~100MB) | streaming download with progress, configurable size cap, resumable |
| Disk filling | pre-flight free-space check; per-library size budget in settings |
