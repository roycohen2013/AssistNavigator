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
3. **Follow** — parse the PDF, extract every referenced standard, resolve each against ASSIST, and
   download them too, breadth-first to a chosen depth.
4. **Navigate** — an interactive citation graph beside a PDF viewer. Clicking an edge opens the
   citing document at the exact page and highlights the citation.

Everything runs locally against public, unclassified, Distribution Statement A material.

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
- **F4 — Recursive crawl.** Breadth-first, depth- and count-limited, resumable, rate-limited,
  pausable, with live progress.
- **F5 — Citation graph.** Interactive, filterable, multiple layouts, click-to-navigate, with the
  graph and PDF viewer side by side.

### 4.2 Necessary supporting features (not requested, but the tool is not usable without them)

- **F6 — Revision and staleness awareness.** ASSIST search returns a revision label (`H(1)`) and
  documents cite specific revisions. Surfacing "this document cites MIL-STD-810F; current is H(1)"
  is arguably the highest-value output of the whole system, and it falls out of the data model for
  almost free.
- **F7 — Resolution & coverage reporting.** Extraction is heuristic. Every crawl produces an explicit
  account of what resolved, what was ambiguous, what is external, and what failed — with one-click
  manual resolution. Without this the graph is a black box no engineer will trust.
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

**Done when:** fetching the same document twice reuses the cached PDF; the DB survives restart; a
corrupted download is detected and re-fetched.

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
confidence, `ref_spans` capture, ASSIST resolution with fuzzy title disambiguation, OCR fallback,
and idempotent re-extraction.

**Done when:** golden-file tests over MIL-STD-129 and MIL-STD-962 pass; every extracted reference
carries a page and rect that lands on the right text; re-extraction is byte-identical.

---

### Phase 5 — Crawl orchestrator + API *(3 days)*

BFS crawler with the resumable `crawl_tasks` queue, cycle handling, pause/resume/cancel, and the full
REST surface from ARCHITECTURE §7 including the SSE progress stream and the `/graph` endpoint.

**Done when:** `POST /crawls {root: 'MIL-STD-129', depth_limit: 2}` completes, is interruptible and
resumable across a server restart, and `/graph` returns a connected node/edge set that matches the DB.

---

### Phase 6 — Web UI: search, library, viewer, graph *(5 days)*

Search and library screens; PDF viewer with text layer, jump-to-page-and-highlight, and the Section 2
outline sidebar; the Cytoscape graph with the full encoding, layouts, filters, and interactions from
ARCHITECTURE §8.2; the split-pane wiring so clicking an edge opens the citing PDF at the citation;
the live crawl monitor.

**Done when:** a user types `MIL-STD-129`, starts a depth-2 crawl, watches it run, and clicks an edge
to land on the highlighted citation in the source PDF. **This is the MVP.**

---

### Phase 7 — Trust features *(2 days)*

Coverage report with manual resolution, staleness report, corpus full-text search, path explanation.

**Done when:** every unresolved reference from a real crawl is either resolvable in two clicks or
correctly classified as not-in-ASSIST.

---

### Phase 8 — Export & collections *(2 days)*

CSV / JSON / GraphML export, ZIP bundle with manifest, printable standards-package report,
collections, annotations.

---

### Phase 9 — Update monitoring & polish *(2 days)*

Scheduled re-check, "what changed" report, keyboard navigation, accessibility pass, empty and error
states, first-run onboarding, packaging.

---

### Phase 10 — Revision diff *(stretch)*

Text-level diff between two revisions of the same document.

**Estimated MVP (Phases 0–6): ~16 working days. Full scope: ~24.**

## 6. Risks

| Risk | Impact | Mitigation |
|---|---|---|
| **ASSIST changes its markup** | High — breaks everything | Parsers isolated in one module; `ParseError` with saved HTML; weekly opt-in live smoke test as the canary; all business logic tested against cassettes |
| **WAF blocks the client** | High | Conservative rate limits, honest identification, backoff + session reset, clear user-facing failure. No evasion — if DLA does not want automated access, the tool stops. |
| **Extraction misses or invents references** | High — silently wrong graph | Confidence scores, binding-vs-informational split, the coverage report as a permanent visible check, golden-file tests |
| **Legacy scanned PDFs have no text layer** | Medium | OCR fallback; documents still graphed from metadata; `needs_ocr` state is visible, not silent |
| **Combinatorial explosion at depth ≥ 3** | Medium | Hard caps, live estimates, binding-only default, on-demand one-level expansion instead of full crawl |
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

## 8. Decisions made for you

Flag any of these you want changed before code generation starts.

1. **Local-first single-user desktop-style web app**, not a hosted service. Avoids auth, quotas, and
   storage abstraction that buy nothing today.
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
7. **Default crawl depth 2, cap 100 documents.** Raisable, but the defaults should be safe.

## 9. Possible later milestones

Hosted multi-user deployment with shared libraries; a DoD-standards-aware assistant over the local
corpus (answer questions with citations into specific pages); PLM/requirements-tool integration;
watch lists with email alerts on revision changes.
