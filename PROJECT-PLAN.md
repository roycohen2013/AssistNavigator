# AssistNavigator — Project Plan

A local-first web application for navigating the DoD standards corpus published at
`assist.dla.mil` / `quicksearch.dla.mil`: name a standard, get the PDF, follow its citations
outward automatically, and explore the resulting dependency graph.

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

## 2. What the tool does

1. **Find** — search ASSIST by document ID, title, keywords, or scope.
2. **Fetch** — download the PDF of the current (or any historical) revision to a local library.
   Documents can also be **imported by hand**, from a watched folder or drag-and-drop, and get
   identical treatment.
3. **Follow** — parse the PDF, extract every referenced standard, resolve each against ASSIST, and
   download them too, breadth-first, one level by default.
4. **Navigate** — an interactive citation graph beside a PDF viewer. Clicking an edge opens the
   citing document at the exact page and highlights the citation.
5. **Account for itself** — every crawl ends with a coverage report saying what resolved, what did
   not, and what is not in ASSIST at all. A completeness claim you cannot check is worse than none.

Everything runs locally against public, unclassified, Distribution Statement A material.

**The success criterion is completeness**: the bounded, checkable set of standards a document
actually depends on. Currency ("you cite Rev F, current is H(1)") is the highest-value byproduct.
The graph is the navigation surface over both, not an end in itself.

### On revisions — read this before trusting the tool

A document's Section 2 typically reads: *"the issues of these documents are those cited in the
solicitation or contract."* **Neither the citing document nor ASSIST is the authority on which
revision applies — the contract is, and this tool cannot see your contract.**

So the tool never silently picks a revision for you. It downloads the **current** revision by
default, records the **cited** revision on every citation edge, flags divergence visibly, and lets
you fetch the cited revision in one click or **pin** a known revision on the root. Completeness here
is a claim about *the set of documents*, never about which revision binds you.

## 3. Users and the jobs they hire this for

| User | Job |
|---|---|
| Design / systems engineer | "Give me the complete, current set of standards my part actually has to meet." |
| Quality / compliance engineer | "Prove which revision of each referenced standard applies, and flag where we cite a superseded one." |
| Proposal / contracts engineer | "Turn this SOW's cited standards into a bounded, reviewable package." |
| New hire | "Show me how this corner of the standards corpus fits together." |

## 4. Feature set

### 4.1 Core (the request as stated)

- **F1 — ASSIST search.** By Document ID, title, keywords, scope; filter by status, FSC/Area, date.
  Results show the revision label and status inline.
- **F2 — Single-document download.** Current revision by default; any historical revision on demand.
  Content-addressed local library with full provenance (source URL, ident, dmxid, timestamp).
- **F3 — Reference extraction.** Parse the PDF and identify every cited standard, distinguishing
  *binding* references (Section 2, Applicable Documents) from incidental body mentions, and
  ASSIST-hosted documents from external ones (ASTM, SAE, ISO, ANSI, STANAG, DoDI).
- **F4 — Recursive crawl.** Breadth-first, resumable, rate-limited, pausable, with live progress.
  **Defaults to one level, cap 25.** Depth 2 is a deliberate choice behind a warning and a live
  document count; going deeper is normally done by expanding one node at a time from the graph.
- **F5 — Citation graph.** Interactive, filterable, multiple layouts, click-to-navigate, with the
  graph and PDF viewer side by side. Expanding a node one level is a first-class action, not a
  fallback for a crawl that stopped early.
- **F5a — Manual PDF import.** A watched folder and drag-and-drop, with identical extraction,
  graphing, and search. Documents are identified by parsing the ID from the PDF's own page-1 header
  rather than the filename, falling back to asking when ambiguous. Provenance is recorded as
  `manual` rather than `assist`.

  This is a co-equal path, not a fallback. It independently covers restricted-distribution documents
  the tool must never fetch, legacy specs sourced elsewhere, and the day ASSIST changes its markup.

### 4.2 Necessary supporting features (not requested, but the tool is not usable without them)

- **F6 — Revision awareness and staleness.** Every citation edge records the revision it names; every
  document records its current revision. Divergence ("cites MIL-STD-810F, current is H(1)") is drawn
  in the graph, not buried in a report, and is the highest-value output of the system. Includes
  one-click fetch of a cited historical revision, and **revision pinning** on the root for when the
  governing contract revision is known. See the revisions note in §2 — the tool reports, it does not
  adjudicate.
- **F7 — Resolution & coverage reporting. Part of the MVP, not a later nicety.** Extraction is
  heuristic. Every crawl produces an explicit account of what resolved, what was ambiguous, what is
  external, and what failed — with one-click manual resolution. Since the product's whole claim is
  *completeness*, an unfalsifiable claim is worse than no claim: this report is what makes it
  checkable.
- **F8 — External-document nodes.** Roughly a third of the references in a typical MIL-STD are
  non-government (ASTM, SAE, ISO). They cannot be downloaded, but they must appear in the graph or
  the dependency picture is misleadingly incomplete.
- **F9 — Full-text search across the local corpus** (SQLite FTS5) — "which of my 60 downloaded
  standards mention *electrostatic discharge*?"
- **F10 — Provenance and audit trail.** Every PDF records where it came from and when. Note that
  ASSIST watermarks each download with a timestamp, so file hashes are not stable across downloads —
  the data model handles this explicitly.
- **F11 — Politeness and rate limiting.** Non-negotiable. Hard-coded conservative defaults, one
  choke point, honest client identification. See ASSIST-PROTOCOL §8.
- **F12 — Offline mode.** Once downloaded, the entire library, graph, viewer, and search work with no
  network. This matters for users on restricted or air-gapped networks.

### 4.3 High-value additions

- **F13 — Export.** Citation tree as CSV / JSON / GraphML; a ZIP bundle of every PDF in a subtree
  plus a manifest; a printable "standards package" report listing each document, revision, status,
  and why it is in the tree (its citation path from the root).
- **F14 — Update monitoring.** Re-check the library against ASSIST on demand or on a schedule;
  report new revisions, cancellations, and supersessions since last check.
- **F15 — Collections / projects.** Group documents per program or part number, with notes.
- **F16 — Annotations and bookmarks** on PDF pages, stored locally.
- **F17 — Path explanation.** "Why is this document in my tree?" — shortest citation path from root.
- **F18 — Revision diff.** Extract text from two revisions and show what changed. Genuinely useful,
  genuinely fiddly; deliberately last.

### 4.4 Explicit non-goals

- No authenticated ASSIST access, no CAC handling, no credential storage, ever.
- No attempt to obtain restricted-distribution (Statement B–F) or export-controlled documents.
- No redistribution or re-hosting of downloaded PDFs.
- No CAPTCHA solving, WAF evasion, or IP rotation. If DLA blocks the client, the client stops and
  says so.
- Not a requirements-management or compliance-matrix tool. It maps the corpus; it does not track
  your verification evidence.

## 5. Delivery plan

Ten phases. Each is independently demonstrable and ends with a concrete acceptance test. Phases 1–6
constitute the MVP that satisfies the original request.

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

Also the **manual import path**: watched folder, ID detection from the PDF's page-1 header, provenance
tagging. It lands here rather than late because everything downstream — extraction, graphing, search —
must treat manual and fetched documents identically, and building it now prevents an ASSIST-shaped
assumption leaking into those layers.

**Done when:** fetching the same document twice reuses the cached PDF; the DB survives restart; a
corrupted download is detected and re-fetched; a PDF dropped in the watch folder is identified and
registered with `provenance='manual'` without any network access.

---

### Phase 3 — Document ID grammar *(2 days)*

`assistnav.docid` — the parser, normaliser, and classifier from ARCHITECTURE §4, with an exhaustive
test table covering every family plus the hard cases: `MIL-STD-2073-1E` (part number vs revision),
`MIL-DTL-38999/20K` (slash sheet), `MIL-STD-129R w/CHANGE 3`, `MIL-DTL-4` (single-digit ID),
hyphenated line breaks, and every external body.

**Done when:** 100+ parametrised cases pass, and running the parser over the MIL-STD-129 text fixture
produces exactly the expected reference set with no self-references and no watermark artefacts.

---

### Phase 4 — Extraction engine *(3 days)*

PyMuPDF text + word-box extraction, watermark/header/footer/TOC stripping, Section 2 location and
category tracking, two-column ID↔title re-association via geometry, edge classification with
confidence, `ref_spans` capture, cited-revision capture on every edge, ASSIST resolution with fuzzy
title disambiguation, and idempotent re-extraction.

**OCR is not in this phase** — it moved to Phase 9. Modern documents carry a text layer (verified);
a document without one is marked `needs_ocr`, still graphed from its metadata, and skipped. This
keeps Tesseract off the critical path.

**Done when:** golden-file tests over MIL-STD-129 and MIL-STD-962 pass; every extracted reference
carries a page and rect that lands on the right text, plus the revision it cited where one is named;
re-extraction is byte-identical.

---

### Phase 5 — Crawl orchestrator + API *(3 days)*

BFS crawler with the resumable `crawl_tasks` queue, cycle handling, pause/resume/cancel, single-node
one-level expansion, and the full REST surface from ARCHITECTURE §7 including the SSE progress stream
and the `/graph` endpoint.

Also the **coverage report** (`/reports/coverage`), promoted here from Phase 7. The product claims
completeness; the claim ships with the means to check it or it does not ship.

**Done when:** `POST /crawls {root: 'MIL-STD-129'}` completes at the depth-1 default, is interruptible
and resumable across a server restart, `/graph` returns a node/edge set matching the DB, expanding one
node adds exactly its children, and the coverage report accounts for every extracted reference as
resolved, ambiguous, external, or failed — with none silently dropped.

---

### Phase 6 — Web UI: search, library, viewer, graph *(5 days)*

Search and library screens; PDF viewer with text layer, jump-to-page-and-highlight, and the Section 2
outline sidebar; the Cytoscape graph with the full encoding, layouts, filters, and interactions from
ARCHITECTURE §8.2; the split-pane wiring so clicking an edge opens the citing PDF at the citation;
the live crawl monitor; the coverage report screen; and **staleness rendered on the graph itself** —
edges whose cited revision trails the target's current revision are drawn distinctly, with the
one-click fetch of the cited revision.

**Done when:** a user types `MIL-STD-129`, runs the default depth-1 crawl, watches it run, clicks an
edge to land on the highlighted citation in the source PDF, expands one node a level deeper from the
graph, sees any stale citation marked without opening a report, and can open the coverage report to
see exactly what the crawl could not resolve. **This is the MVP.**

---

### Phase 7 — Depth, search, and provenance *(2 days)*

Corpus full-text search (FTS5), path explanation ("why is this document in my tree?"), revision
pinning on the root, and the standalone staleness report across the whole library rather than one
crawl. Manual resolution of ambiguous IDs from the coverage report.

**Done when:** every unresolved reference from a real crawl is either resolvable in two clicks or
correctly classified as not-in-ASSIST; a pinned root revision changes which revision the tool treats
as governing throughout the tree.

---

### Phase 8 — Export & collections *(2 days)*

CSV / JSON / GraphML export, ZIP bundle with manifest, printable standards-package report,
collections, annotations.

---

### Phase 9 — OCR, update monitoring & polish *(2 days)*

**OCR fallback** (`ocrmypdf` + Tesseract, optional dependency) for scanned legacy specs marked
`needs_ocr` in Phase 4. Scheduled re-check against ASSIST, "what changed" report — no more than daily,
since ASSIST itself updates nightly on business days. Keyboard navigation, accessibility pass, empty
and error states, first-run onboarding, packaging.

---

### Phase 10 — Revision diff *(stretch)*

Text-level diff between two revisions of the same document.

**Estimated MVP (Phases 0–6): ~16 working days. Full scope: ~24.**

The MVP total is unchanged after the 2026-08-29 design review: OCR leaving the critical path (Phase 4
→ 9) roughly pays for manual import arriving (Phase 2) and coverage reporting moving forward (Phase 7
→ 5). What changed is *what you get at the end of it* — a shallower default crawl, an auditable
completeness claim, and revision divergence visible on the graph.

## 6. Risks

| Risk | Impact | Mitigation |
|---|---|---|
| **ASSIST changes its markup** | High — breaks everything | Parsers isolated in one module; `ParseError` with saved HTML; weekly opt-in live smoke test as the canary; all business logic tested against cassettes |
| **WAF blocks the client** | High | Conservative rate limits, honest identification, backoff + session reset, clear user-facing failure. No evasion — if DLA does not want automated access, the tool stops. |
| **Extraction misses or invents references** | High — silently wrong graph | Confidence scores, binding-vs-informational split, the coverage report as a permanent visible check, golden-file tests |
| **Legacy scanned PDFs have no text layer** | Medium | OCR fallback; documents still graphed from metadata; `needs_ocr` state is visible, not silent |
| **Combinatorial explosion at depth ≥ 2** | Medium | Depth-1 default, cap 25, live estimates, binding-only default, on-demand one-level expansion as the normal path |
| **User trusts a revision the tool never claimed was governing** | High — silently wrong compliance work | The tool reports cited vs. current and never adjudicates; divergence is drawn on the graph; §2 states the limit in the plan itself |
| **Ambiguous document IDs** | Medium | Fuzzy title match against the title recovered from Section 2; manual resolution queue |
| **Very large PDFs (MIL-STD-810 ≈ 100 MB)** | Low | Streaming downloads, size caps, disk pre-flight |

## 7. Compliance posture

The documents are public and unclassified, and QuickSearch serves them without authentication —
which is what makes this project reasonable. The constraints that keep it that way are hard
requirements, not preferences:

- Public endpoints only; no authentication bypass of any kind.
- Rate limits and backoff enforced at one choke point, on by default, conservative.
- The client identifies itself honestly.
- Downloaded PDFs stay local; each retains its ASSIST provenance stamp.
- Distribution statements are parsed and displayed. Any document that is not Statement A is flagged
  in the UI and never redistributed by the export feature.
- The export report reminds the user that ASSIST is the authoritative source and that local copies
  can go stale — the same warning DLA stamps on every page.

## 8. Decisions settled

Reviewed and confirmed in a design interview on 2026-08-29. Items 8–12 came out of that session and
changed the plan; the rest were confirmed as originally proposed.

1. **Local-first single-user desktop-style web app**, not a hosted service. Explicitly designed
   *against* multi-user until someone asks. Avoids auth, quotas, and storage abstraction.
2. **Python backend, TypeScript frontend.** PyMuPDF's word-level geometry is what makes
   click-an-edge-and-highlight-the-citation possible; the JS PDF ecosystem is materially weaker here.
3. **Cytoscape.js over React Flow** for the graph — better at scale and at automatic layout, which
   matters more than custom node chrome for a citation graph.
4. **Pure HTTP, no headless browser.** Validated: search and download both work with plain requests.
   Playwright would be slower, heavier, and more fragile.
5. **Binding-only crawling by default** (Section 2 references), with informational references
   extracted, stored, and shown in the graph but not followed. Following every incidental mention
   explodes the tree.
6. **External standards are graphed but never fetched.** ASTM/SAE/ISO are paywalled by their
   publishers; the tool shows the dependency and stops there.
7. **Completeness is the success criterion**, currency the highest-value byproduct, the graph the
   navigation surface over both. Not comprehension-for-its-own-sake, not a staleness monitor.
8. **Default crawl depth 1, cap 25** — changed from depth 2 / cap 100. Depth 1 answers "what does
   this document directly require" in about two minutes, which is the common question. Depth 2 is
   behind a warning and a live count; expanding one node at a time is the normal way deeper.
   *Caveat: based on one measurement (MIL-STD-129 cited ~6 resolvable documents), not a survey. If
   real documents prove denser, revisit.*
9. **The tool never picks a revision for you.** Current by default, cited revision recorded on every
   edge, divergence drawn on the graph, one-click fetch of the cited revision, pinning on the root.
   Driven by a domain fact: Section 2 defers to "the issues cited in the solicitation or contract",
   so the contract governs and the tool cannot see it. See §2.
10. **Manual PDF import is a co-equal path**, in Phase 2, not a fallback. Also covers restricted
    documents, externally sourced legacy specs, and ASSIST markup changes.
11. **Coverage reporting is MVP scope** (Phase 5), not a later trust feature. A completeness claim
    ships with the means to falsify it or it does not ship.
12. **OCR is post-MVP** (Phase 9, was Phase 4). Modern documents have text layers; scanned ones are
    marked `needs_ocr` and still graphed from metadata. Keeps Tesseract off the critical path.
13. **Automated access proceeds** under the conservative limits in ASSIST-PROTOCOL §8. There is no
    published API, no bulk product, no terms-of-use page, and no stated rate limit — only a generic
    CFAA banner. Public unauthenticated documents fetched slowly and identified honestly are within
    normal use; the manual-import path (item 10) exists so the tool degrades rather than escalates.

## 9. Possible later milestones

Hosted multi-user deployment with shared libraries; a DoD-standards-aware assistant over the local
corpus (answer questions with citations into specific pages); PLM/requirements-tool integration;
watch lists with email alerts on revision changes.
