# AssistNavigator — instructions for Claude Code

## Read first

1. `PROJECT-PLAN.md` — scope, features, phase order, decisions already made.
2. `docs/ASSIST-PROTOCOL.md` — **verified** wire protocol for quicksearch.dla.mil. Do not
   re-derive it and do not guess at endpoints; it was validated against the live site.
3. `docs/ARCHITECTURE.md` — stack, schema, algorithms, API contract, UI spec.

Build phases in order. Each phase in `PROJECT-PLAN.md` §5 has an acceptance test — do not move on
until it passes.

## Layout

    backend/
      assistnav/
        assist/        HTTP client, parsers, rate limiter   (Phase 1)
        store/         SQLAlchemy models, migrations, library  (Phase 2)
        docid/         ID grammar, normalisation, classification  (Phase 3)
        extract/       PDF text/geometry, Section 2, refs, OCR  (Phase 4)
        crawl/         BFS orchestrator, task queue  (Phase 5)
        api/           FastAPI routers, SSE  (Phase 5)
        cli.py
      tests/
        cassettes/     respx recordings — never hit the network in tests
        fixtures/      extracted-text golden files (text, not PDFs)
    frontend/
      src/{routes,components,graph,viewer,lib,store}
    docs/

## Hard rules

- **Never hit the live site in tests.** Only `pytest -m live` may, and it is excluded by default.
- **All network calls go through the single rate-limited client.** No ad-hoc `httpx.get`.
- **A WAF block returns HTTP 200.** Always check the body via `detect_waf_block`.
- **Never persist `/Transient/*.pdf` URLs** — they are one-shot.
- **Never hash raw PDF bytes for identity** — ASSIST watermarks each download with a timestamp.
  Identity is `(ident_number, dmxid)`; use `text_sha256` for content comparison.
- **Never discard a reference silently.** Unresolved references go to the coverage report.
- No authentication, no CAC, no restricted-distribution documents, no WAF evasion.

## Agent skills

### Issue tracker

Issues live in GitHub Issues for `roycohen2013/AssistNavigator`, via the `gh` CLI. See
`docs/agents/issue-tracker.md`.

### Triage labels

The five canonical roles, unchanged: `needs-triage`, `needs-info`, `ready-for-agent`,
`ready-for-human`, `wontfix`. See `docs/agents/triage-labels.md`.

### Domain docs

Single-context: `CONTEXT.md` and `docs/adr/` at the repo root, created lazily rather than
up front. See `docs/agents/domain.md`.

## Style

- Python 3.12, full type annotations, `ruff` + `mypy --strict` clean.
- Parsers raise `ParseError` with the offending HTML saved to the log directory — never return empty
  results on a markup change.
- Extraction is idempotent: re-running it on a revision must produce identical rows.
- Frontend: no `any`; generate the API client from FastAPI's OpenAPI schema rather than hand-writing
  fetch calls.
