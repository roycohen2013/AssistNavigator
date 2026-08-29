# AssistNavigator — Project Plan

A local-first web application that answers one question about a DoD standard, completely and
auditably: **what is the full set of standards this document obliges me to meet, and which of them
am I citing at the wrong revision?**

**Companion documents**
- [`docs/ASSIST-PROTOCOL.md`](docs/ASSIST-PROTOCOL.md) — the wire protocol, **verified against the
  live site on 2026-08-24**. Read this first; it removes most of the unknowns from Phase 1.
- [`docs/ARCHITECTURE.md`](docs/ARCHITECTURE.md) — stack, data model, algorithms, API contract,
  UI specification.

---

## 1. Problem

An engineer working to a MIL-STD does not read one document. MIL-STD-129 pulls in MIL-STD-130,
MIL-STD-2073-1, MIL-PRF-61002, MIL-DTL-64159, plus ASTM, SAE, ANSI, ISO and NATO STANAGs. Each of
those has its own Section 2. Today the workflow is: search QuickSearch, click through a JS-driven
download shim, open the PDF, scroll to Section 2, transcribe each ID by hand, search again, repeat.
Doing this properly for one standard is an afternoon; keeping it current across revisions is nobody's
job, so it does not happen.

The failure this causes is specific and expensive: you build to a cited revision that has since been
superseded, or you miss a binding requirement two hops down the citation chain.

## 2. The job

**The job is completeness.** Everything else in this document is in service of it.

The deliverable is a defensible list: every standard a document binds you to, each with its status,
its current revision, and the revision your document actually cites. An engineer should be able to
hand that list to a reviewer and defend every line of it — including the lines that say "we could not
resolve this."

Two things that look like the product are byproducts:

- **Currency.** "You cite MIL-STD-810F; current is H(1)" falls out of the completeness data for
  almost nothing. It is the most immediately useful *output*, but it is not the job — a currency
  check over an incomplete list is worse than useless, because it looks thorough.
- **The graph.** The graph is how a human checks a completeness claim: it makes an implausible
  answer look implausible. It is the primary *interface*, not the primary *value*. A tool that
  produced a perfect list and no graph would still do the job; a tool that produced a beautiful
  graph over a list with silent holes would not.

This ordering decides arguments. When graph aesthetics conflict with showing an unresolved
reference, the unresolved reference wins.

### 2.1 Two co-equal ways in

Documents enter the library by two paths, and neither is a fallback for the other:

- **Crawl.** Name a standard; the tool fetches it from ASSIST, extracts its references, and follows
  them outward.
- **Import.** Drop PDFs you already have — a customer data package, a program archive, documents
  pulled from ASSIST by hand years ago — into a watched folder, or drag them onto the window.

Both paths converge immediately: same library, same extraction pipeline, same graph, same reports.
An imported document's references are crawled exactly like a fetched one's, and a crawl can land on
a document you already imported. The design consequence is that **nothing downstream of the library
may assume a document came from ASSIST** — that assumption is what turns import into a second-class
path, and it is the single easiest way to get this architecture wrong.

Import is co-equal because the real starting point is usually a PDF someone emailed you, not a
document ID you can type.

## 3. Users and the jobs they hire this for

| User | Job |
|---|---|
| Design / systems engineer | "Give me the complete, current set of standards my part actually has to meet." |
| Quality / compliance engineer | "Prove which revision of each referenced standard applies, and flag where we cite a superseded one." |
| Proposal / contracts engineer | "Turn this SOW's cited standards into a bounded, reviewable package." |
| New hire | "Show me how this corner of the standards corpus fits together." |

## 4. Feature set

### 4.1 Core

- **F1 — ASSIST search.** By Document ID, title, keywords, or scope; filter by status, FSC/Area,
  date. Results show revision label and status inline.
- **F2 — Single-document download.** Current revision by default; any historical revision on demand.
  Local library with full provenance (source URL, ident, dmxid, timestamp).
- **F3 — Manual import.** A watched folder plus drag-and-drop. Document identity is read from the
  **PDF's own first-page header**, never from the filename — `scan001.pdf` and
  `MIL-STD-129 (2).pdf` are both normal and neither is evidence of anything. Imports that cannot be
  identified go to a queue for manual assignment; they are never silently skipped.
- **F4 — Reference extraction.** Parse the PDF and identify every cited standard, distinguishing
  *binding* references (Section 2, Applicable Documents) from incidental body mentions, and
  ASSIST-hosted documents from external ones (ASTM, SAE, ISO, ANSI, STANAG, DoDI).
- **F5 — Recursive crawl.** Breadth-first, depth- and count-limited, resumable, rate-limited,
  pausable, with live progress.
- **F6 — Coverage report.** Every crawl and every import produces an explicit account of what
  resolved, what was ambiguous, what is external, and what failed — with one-click manual
  resolution. This is not a report *about* the job; for a job defined as completeness, **it is the
  primary output.** The graph is the picture; this is the receipt.
- **F7 — Staleness report.** Every edge whose cited revision is behind the target's current
  revision, as a standing list. The headline artefact users will actually paste into a review.
- **F8 — Citation graph.** Interactive, filterable, multiple layouts, click-to-navigate, with the
  graph and PDF viewer side by side. **Expand-one-level is the primary navigation gesture**, not a
  power-user extra (see §5).

### 4.2 Necessary supporting features

- **F9 — Revision awareness.** Download the current revision by default; record the revision each
  document actually cites; flag every divergence. See §8.5 for what this deliberately does *not*
  include.
- **F10 — External-document nodes.** Roughly a third of the references in a typical MIL-STD are
  non-government (ASTM, SAE, ISO). They cannot be downloaded, but they must appear in the graph and
  in the coverage report, or the completeness claim is a lie by omission.
- **F11 — Full-text search across the local corpus** (SQLite FTS5) — "which of my 60 downloaded
  standards mention *electrostatic discharge*?"
- **F12 — Provenance and audit trail.** Every PDF records where it came from and when — ASSIST URL
  and dmxid, or import path and original filename. Note that ASSIST watermarks each download with a
  timestamp, so file hashes are not stable across downloads; the data model handles this explicitly.
- **F13 — Politeness and rate limiting.** Non-negotiable. Hard-coded conservative defaults, one
  choke point, honest client identification. See ASSIST-PROTOCOL §8.
- **F14 — Offline mode.** Once downloaded, the entire library, graph, viewer, and search work with no
  network. This matters for users on restricted networks — and it is why import must be co-equal:
  on such a network it is the *only* path in.

### 4.3 High-value additions

- **F15 — Export.** Citation tree as CSV / JSON / GraphML; a ZIP bundle of every PDF in a subtree
  plus a manifest; a printable "standards package" report listing each document, revision, status,
  and its citation path from the root.
- **F16 — Path explanation.** "Why is this document in my tree?" — shortest citation path from root.
- **F17 — Update monitoring.** Re-check the library against ASSIST on demand or on a schedule;
  report new revisions, cancellations, and supersessions since last check.
- **F18 — Collections / projects.** Group documents per program or part number, with notes.
- **F19 — Annotations and bookmarks** on PDF pages, stored locally.
- **F20 — OCR fallback** for legacy scanned specs without a text layer. Deliberately late (§8.8).
- **F21 — Revision diff.** Extract text from two revisions and show what changed. Genuinely useful,
  genuinely fiddly; deliberately last.

### 4.4 Explicit non-goals

- **No multi-user support, and the design actively assumes its absence.** No accounts, no
  permissions, no tenant column, no storage abstraction, no "we might need this later" seams. See
  §8.2 — this is a decision to pay a real cost later in exchange for a much simpler tool now.
- No authenticated ASSIST access, no CAC handling, no credential storage, ever.
- No attempt to obtain restricted-distribution (Statement B–F) or export-controlled documents.
- No redistribution or re-hosting of downloaded PDFs.
- No CAPTCHA solving, WAF evasion, or IP rotation. If DLA blocks the client, the client stops and
  says so.
- Not a requirements-management or compliance-matrix tool. It maps the corpus; it does not track
  your verification evidence.

## 5. Depth, and why the default is 1

The old plan defaulted to depth 2, capped at 100 documents. That is wrong, and the reason is a cost
asymmetry:

- A default that is **too shallow** costs one click on a `+18` badge.
- A default that is **too deep** costs twenty minutes of downloading, several hundred megabytes, a
  graph too dense to read, and — worst — a coverage report long enough that nobody audits it. The
  completeness job fails silently at exactly the moment the tool looks most impressive.

So: **`depth_limit=1`, `max_docs=25`.** Depth 2 stays available as a deliberate choice behind a
warning that states the estimated document count and byte volume before starting.

This makes **expand-one-level the primary navigation gesture.** It is not a fallback for a
too-timid default; it is the intended way to use the tool. You crawl a root, look at what came back,
and push outward along the branches that matter — which is how an engineer actually reads a standard
tree. The corollary is that expansion has to be *fast and obvious*: a `+N` badge on every node with
un-crawled children, one click to expand, no dialog, no new job to configure.

**On the evidence for depth 1.** It rests on one observed document (MIL-STD-129, ~6 binding
references at depth 1). That is thin, and if a typical document in your corpus cites 40, depth 1
plus a `+40` badge is a worse experience than I am predicting. Rather than defend the guess,
Phase 5 measures it: every crawl reports its **frontier count** — how many cited documents were
found but not fetched — and the UI shows it. After a week of real use the right default is a fact,
not an argument. If frontier counts routinely land under ~10, raise the default to 2.

## 6. Delivery plan

Nine phases plus a stretch. Each is independently demonstrable and ends with a concrete acceptance
test. Phases 0–7 constitute the MVP.

---

### Phase 0 — Scaffold *(0.5 day)*

Repo layout (`backend/`, `frontend/`, `docs/`), `uv` project, FastAPI skeleton serving a Vite build,
SQLite + Alembic baseline, ruff/mypy/pytest, Vitest, CI.

**Done when:** `uv run assistnav serve` opens a working (empty) app at `localhost:8787`.

---

### Phase 1 — ASSIST client *(2 days)*

`assistnav.assist` implementing ASSIST-PROTOCOL exactly: session bootstrap, WebForms search POST,
results-grid parser, document-details parser (metadata + revision tokens), and the two-hop PDF
download. Plus the shared rate limiter, WAF detection, `WafBlockedError`/`ParseError`, and
`respx` cassettes for every flow.

Confirm the four `[UNVERIFIED]` items from ASSIST-PROTOCOL: the status-dropdown value mapping,
results pagination, revision-ordering assumption, and the supersession/related-documents blocks on a
document that actually has them.

**Done when:** `assistnav fetch MIL-STD-129` writes a valid PDF to the library and prints correct
metadata, and the full test suite passes with zero network access.

---

### Phase 2 — Storage layer *(1 day)*

Full schema from ARCHITECTURE §3, migrations, library directory management, FTS5 index, settings.

The `revisions` table must accommodate a source-agnostic revision from day one — an imported PDF has
no `dmxid`. Building this in Phase 2 rather than retrofitting it in Phase 6 is what keeps import
co-equal instead of bolted on.

**Done when:** fetching the same document twice reuses the cached PDF; a revision row can be created
with no `dmxid`; the DB survives restart; a corrupted download is detected and re-fetched.

---

### Phase 3 — Document ID grammar *(2 days)*

`assistnav.docid` — the parser, normaliser, and classifier from ARCHITECTURE §4, with an exhaustive
test table covering every family plus the hard cases: `MIL-STD-2073-1E` (part number vs revision),
`MIL-DTL-38999/20K` (slash sheet), `MIL-STD-129R w/CHANGE 3`, `MIL-DTL-4` (single-digit ID),
hyphenated line breaks, and every external body.

**Done when:** 100+ parametrised cases pass, and running the parser over the MIL-STD-129 text fixture
produces exactly the expected reference set with no self-references and no watermark artefacts.

---

### Phase 4 — Extraction engine *(2.5 days)*

PyMuPDF text + word-box extraction, watermark/header/footer/TOC stripping, Section 2 location and
category tracking, two-column ID↔title re-association via geometry, edge classification with
confidence, `ref_spans` capture, ASSIST resolution with fuzzy title disambiguation, and idempotent
re-extraction.

**No OCR in this phase.** A PDF with no text layer is detected, marked `needs_ocr`, and still graphed
from its metadata — visibly incomplete, never silently empty. OCR itself is Phase 9 (§8.8).

**Done when:** golden-file tests over MIL-STD-129 and MIL-STD-962 pass; every extracted reference
carries a page and rect that lands on the right text; re-extraction is byte-identical; a text-layer-
less PDF produces a `needs_ocr` document rather than an exception or an empty reference set.

---

### Phase 5 — Crawl orchestrator, coverage report, API *(3.5 days)*

BFS crawler with the resumable `crawl_tasks` queue, cycle handling, pause/resume/cancel, and the full
REST surface from ARCHITECTURE §7 including the SSE progress stream and the `/graph` endpoint.

The **coverage report ships here, not in a later trust phase.** It is the primary output of a
completeness tool, so the crawler is not finished until it can account for every reference it saw.
This phase also emits the **frontier count** per §5.

The queue stays resumable. Depth 1 alone would not justify the `crawl_tasks` machinery, but
expand-one-level does: each expansion is a new increment of work against a library that already
exists, which is the same problem as resuming an interrupted crawl. Dropping it would mean building
it again under a different name.

**Done when:** `POST /crawls {root: 'MIL-STD-129'}` completes at depth 1, is interruptible and
resumable across a server restart, `/graph` returns a connected node/edge set matching the DB,
`/reports/coverage` accounts for **every** reference the crawl encountered with none unaccounted for,
and the response carries a frontier count.

---

### Phase 6 — Manual import *(1 day)*

`assistnav.import` — watch folder, drag-and-drop upload endpoint, first-page header parsing to
recover the document ID and revision, dedupe against the existing library by `(canonical_id,
text_sha256)`, and the unidentified-import queue.

Imported revisions flow into the same extraction pipeline as downloaded ones and are crawlable roots
like any other document.

**Done when:** dropping a renamed copy of MIL-STD-129 into the watch folder produces the same
library row, references, and graph nodes as fetching it from ASSIST; a PDF whose header cannot be
parsed appears in the unidentified queue with its first page rendered for manual assignment; and
re-dropping the same file is a no-op rather than a duplicate.

---

### Phase 7 — Web UI *(5.5 days)*

Search, library, and import screens; PDF viewer with text layer, jump-to-page-and-highlight, and the
Section 2 outline sidebar; the Cytoscape graph with the encoding, layouts, filters, and interactions
from ARCHITECTURE §8.2; the split-pane wiring so clicking an edge opens the citing PDF at the
citation; the live crawl monitor; the coverage report with one-click manual resolution; and the
**staleness report as a headline screen**, not a buried tab.

Expand-one-level gets first-class treatment: `+N` badges, one-click expansion, no configuration.

**Done when:** a user types `MIL-STD-129`, runs a depth-1 crawl, sees its coverage and staleness
reports, expands two nodes one level each, and clicks an edge to land on the highlighted citation in
the source PDF. **This is the MVP.**

---

### Phase 8 — Export, corpus search, collections *(2 days)*

CSV / JSON / GraphML export, ZIP bundle with manifest, printable standards-package report, corpus
full-text search, path explanation, collections, annotations.

Revision pinning lands here if §8.5's re-evaluation says it should.

---

### Phase 9 — OCR, update monitoring & polish *(2.5 days)*

`ocrmypdf` + Tesseract fallback for the `needs_ocr` backlog accumulated since Phase 4; scheduled
re-check and "what changed" report; graph virtualisation above ~500 nodes; keyboard navigation;
accessibility pass; empty and error states; first-run onboarding; packaging.

---

### Phase 10 — Revision diff *(stretch)*

Text-level diff between two revisions of the same document.

**Estimated MVP (Phases 0–7): ~17 working days. Full scope: ~22.**

This is up from the previous plan's ~16-day MVP, not down. Moving OCR out of the critical path saves
half a day and adding manual import costs one; promoting the coverage and staleness reports into the
MVP costs another day between them. That is a real increase and it is the right trade: an MVP
without manual import cannot be used on a restricted network at all, and an MVP without the coverage
report ships a completeness tool whose completeness is unauditable.

## 7. Risks

| Risk | Impact | Mitigation |
|---|---|---|
| **ASSIST changes its markup** | High — breaks the crawl path | Parsers isolated in one module; `ParseError` with saved HTML; weekly opt-in live smoke test as the canary; all business logic tested against cassettes. Manual import keeps the tool useful while a parser is broken — a second reason it is co-equal. |
| **WAF blocks the client** | High | Conservative rate limits, honest identification, backoff + session reset, clear user-facing failure. No evasion — if DLA does not want automated access, the tool stops. |
| **Extraction misses or invents references** | High — silently wrong completeness claim | Confidence scores, binding-vs-informational split, the coverage report as a permanent visible check, golden-file tests |
| **Depth-1 default is too shallow for a dense corpus** | Medium — tedium, not wrongness | Frontier count measured and displayed from Phase 5; expand-one-level is one click; raise the default once there is evidence (§5) |
| **Import header parsing is unreliable across decades of PDF layouts** | Medium | Unidentified queue with the first page rendered for manual assignment; never guess from the filename; never silently skip |
| **Legacy scanned PDFs have no text layer** | Medium | `needs_ocr` is a visible state from Phase 4; document still graphed from metadata; OCR arrives in Phase 9 |
| **Ambiguous document IDs** | Medium | Fuzzy title match against the title recovered from Section 2; manual resolution queue |
| **Very large PDFs (MIL-STD-810 ≈ 100 MB)** | Low | Streaming downloads, size caps, disk pre-flight |

## 8. Decisions

Flag any of these you want changed before code generation starts. §8.5 and §8.7 are calls made on
your behalf when the design interview ran out of evidence; they are the two most worth reversing.

### 8.1 The job is completeness

Currency and the graph are byproducts (§2). This is the tiebreaker for every later argument, and it
is why the coverage report is MVP and OCR is not.

### 8.2 Single-user, local-only — designed against multi-user

Not merely "multi-user is out of scope for now," but "the code may assume there is exactly one user."
No tenant IDs, no permission checks, no pluggable storage layer, no user column carried around for a
future that may never arrive. Bare SQLite, bare filesystem, no auth anywhere.

The cost is honest: if this ever needs to be shared, it is a rewrite of the storage and API layers,
not a migration. That is the right bet, because a speculative multi-user seam would slow every day of
the next three weeks to buy an option nobody has asked for.

### 8.3 Two co-equal import paths

Crawl and manual import, converging immediately on a shared library (§2.1). Nothing downstream of
the library may assume a document came from ASSIST.

### 8.4 Shallow defaults, expansion as the primary gesture

`depth_limit=1`, `max_docs=25`; depth 2 behind an explicit warning; expand-one-level as the intended
navigation (§5).

### 8.5 Revision authority: record and flag, but **do not pin** — *my call*

Settled: download the current revision by default, record the revision each document actually cites,
and flag every divergence. This is the staleness report and it is the highest-value output per unit
of effort in the whole system.

Cut: **revision pinning** — the ability to say "always resolve MIL-STD-810 to revision F for this
project." It is a plausible-sounding feature with no evidence behind it. Nobody in the interview
described needing it; it was proposed because it sounds like something a compliance tool should have.
Building it now means a pinning UI, pin-aware resolution in the crawler, pin-vs-current conflict
display in the graph, and a new class of "why did it download the wrong thing" bug — all for a
workflow that may not exist.

What is kept is the **seam**: `revisions` already stores every known revision of a document, and the
download endpoint already accepts an explicit `dmxid`. Adding pinning later is one nullable column on
`documents` plus a branch in the resolver — under a day. The feature is cut; the ability to add it
cheaply is not.

**Reverse this if:** in real use you find yourself repeatedly needing to hold a document at an
old revision because a contract specifies it. That is the evidence that was missing. Phase 8 is where
it would land.

### 8.6 Manual import identifies documents by PDF header, not filename

Filenames are noise: `scan001.pdf`, `MIL-STD-129 (2).pdf`, `129R_CH3_FINAL_v2.pdf`. The first page of
a military standard, by contrast, carries the ID and revision in a fixed, parseable position. Use the
filename as a weak hint for the unidentified queue and nothing more.

### 8.7 Depth-1 default is measured, not defended — *my call*

Keep depth 1, and instrument it (§5). The frontier count turns the open question into a
one-week measurement instead of an argument from a single document.

**Reverse this if:** frontier counts routinely come back under ~10, meaning the corpus is sparser
than feared and depth 2 is comfortably within the cap.

### 8.8 OCR is Phase 9, not Phase 4

OCR was on the MVP critical path because scanned legacy specs exist. But a missing text layer is a
*visible, honest* failure — `needs_ocr`, still graphed from metadata, listed in the coverage report —
and the completeness claim survives it because the gap is disclosed. Meanwhile OCR drags in
Tesseract, a binary dependency, per-page timing, and a quality problem that produces plausible-
looking wrong references. Wrong references are worse than absent ones for a completeness tool.

### 8.9 The resumable queue stays

Justified by expand-one-level rather than by long crawls (Phase 5).

### 8.10 Technical choices

1. **Python backend, TypeScript frontend.** PyMuPDF's word-level geometry is what makes
   click-an-edge-and-highlight-the-citation possible; the JS PDF ecosystem is materially weaker here.
2. **Cytoscape.js over React Flow** — better at scale and at automatic layout, which matters more
   than custom node chrome for a citation graph.
3. **Pure HTTP, no headless browser.** Validated: search and download both work with plain requests.
   Playwright would be slower, heavier, and more fragile.
4. **Binding-only crawling by default** (Section 2 references), with informational references
   extracted, stored, and shown in the graph but not followed.
5. **External standards are graphed but never fetched.** ASTM/SAE/ISO are paywalled by their
   publishers; the tool shows the dependency and stops there.

## 9. Compliance posture

The documents are public and unclassified, and QuickSearch serves them without authentication —
which is what makes this project reasonable. The constraints that keep it that way are hard
requirements, not preferences:

- Public endpoints only; no authentication bypass of any kind.
- Rate limits and backoff enforced at one choke point, on by default, conservative.
- The client identifies itself honestly.
- Downloaded PDFs stay local; each retains its ASSIST provenance stamp.
- Distribution statements are parsed and displayed. Any document that is not Statement A is flagged
  in the UI and never redistributed by the export feature. This applies to **imported** documents
  too — the tool must not become a laundering path for a restricted PDF a user happened to have.
- The export report reminds the user that ASSIST is the authoritative source and that local copies
  can go stale — the same warning DLA stamps on every page.

## 10. Possible later milestones

Hosted multi-user deployment with shared libraries (a rewrite, per §8.2, not a migration); a
DoD-standards-aware assistant over the local corpus (answer questions with citations into specific
pages); PLM/requirements-tool integration; watch lists with email alerts on revision changes.
