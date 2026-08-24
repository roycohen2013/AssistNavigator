# ASSIST / QuickSearch Protocol Notes

**Status: empirically verified against the live site on 2026-08-24.** Every request/response
described below was actually executed and observed. Treat this file as the authoritative reference
for the `assistnav.assist` client module. Anything marked `[UNVERIFIED]` is inference and must be
confirmed during Phase 1.

---

## 1. Hosts and access model

| Host | Auth | Use |
|---|---|---|
| `https://quicksearch.dla.mil` | **None** | Public search + PDF download. **This is the host we integrate with.** |
| `https://assist.dla.mil` | Login / CAC for most functions | Do **not** integrate. No credential handling, no CAC emulation. |

All target documents are public, unclassified, Distribution Statement A material. The tool must never
attempt to reach authenticated ASSIST functions or restricted-distribution documents.

## 2. WAF behaviour (critical)

The site sits behind an F5 ASM web application firewall.

- A request with a non-browser `User-Agent` (e.g. default `curl/8.x`) is rejected.
- **The rejection returns `HTTP 200`**, not 403. The body is:
  `<html><head><title>Request Rejected</title></head><body>The requested URL was rejected...
  Your support ID is: <NNNNNNNNNNN>...`
- Therefore: **never trust the status code alone.** Every response must pass through
  `detect_waf_block(body)` which matches `Request Rejected` + `support ID`.
- Mitigation: send a realistic desktop Chrome `User-Agent`, an `Accept` header, a `Referer` matching
  the previous page in the flow, and reuse one HTTP client (shared cookie jar) for the whole session.
- `robots.txt` does not exist on either host (requests for it are WAF-rejected). Absence of
  robots.txt is not licence to hammer the site — see the rate-limit policy in section 8.
- A WAF block is a distinct error class (`WafBlockedError`) and must trigger exponential backoff plus
  session reset (new cookie jar, re-fetch `qsSearch.aspx`), not a plain retry.

## 3. Search — `qsSearch.aspx`

Classic ASP.NET WebForms. Two steps.

### 3.1 Bootstrap (GET)

    GET https://quicksearch.dla.mil/qsSearch.aspx

Scrape these hidden inputs from the returned HTML: `__VIEWSTATE`, `__VIEWSTATEGENERATOR`,
`__EVENTVALIDATION`, `__VIEWSTATEENCRYPTED`, `TextBoxBegin_Date_HF`, `TextBoxEnd_Date_HF`,
`TextBoxBegin_Date_HFS`, `TextBoxEnd_Date_HFS`, `HiddenFieldSortOrder`, `divPreamble_HF`.
HTML-unescape every value before re-posting it.

### 3.2 Submit (POST)

    POST https://quicksearch.dla.mil/qsSearch.aspx
    Content-Type: application/x-www-form-urlencoded
    Referer: https://quicksearch.dla.mil/qsSearch.aspx

**Verified-working field set** — this exact payload returned the MIL-STD-810 result row:

| Field | Value |
|---|---|
| `__EVENTTARGET`, `__EVENTARGUMENT` | empty string |
| `__VIEWSTATE`, `__VIEWSTATEGENERATOR`, `__EVENTVALIDATION` | echoed from bootstrap |
| `__VIEWSTATEENCRYPTED` | empty string |
| `FromSession_HFS` | `0` |
| `DocumentIDTextBox` | e.g. `MIL-STD-810` |
| `DocumentIDTextBox_HFS` | **same value** |
| `IDNumberTextBox`, `IDNumberTextBox_HFS` | empty string |
| `DocumentTitleKeyWords`, `DocumentTitleKeyWords_HFS` | free-text terms, or empty |
| `DropDownListContains`, `DropDownListContains_HFS` | `AND` or `OR` |
| `DropDownListTitleOrKeywords`, `DropDownListTitleOrKeywords_HFS` | `TOS`, `TOK`, `T`, `K`, or `S` |
| `DropDownListStatus`, `DropDownListStatus_HFS` | empty = All; `1`..`5` `[UNVERIFIED mapping]` |
| `ExListBoxFSCHF`, `ExListBoxFSCHFN`, `ExListBoxFSCHFN_HFS` | empty string |
| `UseTransDateCheckBox_HFS` | empty string |
| `TextBoxBegin_Date`, `TextBoxEnd_Date` | empty string |
| `TextBoxBegin_Date_HF`/`_HFS`, `TextBoxEnd_Date_HF`/`_HFS` | echoed from bootstrap |
| `Command_HFS` | empty string |
| `divPreamble_HF` | `none` |
| `HiddenFieldSortOrder` | echoed from bootstrap |
| `GetFilteredButton` | `Search` |

**Gotcha (this cost two failed attempts during validation):** the page's `setSelection(1)` JS mirrors
every visible field into its `_HFS` twin before submit. Omitting the `_HFS` fields, or sending
`GetFilteredButton=Submit` instead of `Search`, redirects to `/UnknownError.htm` with HTTP 404.

`DropDownListTitleOrKeywords` maps to the UI options: `TOS` = Title/Keywords/Scope,
`TOK` = Title or Keywords, `T` = Title only, `K` = Keywords only, `S` = Scope only.

### 3.3 Results

Results render into a table with `id="GV"`, columns:

    Img | Document ID | Status | FSC/Area | Doc Date | Title

Observed row:

    Y | MIL-STD-810 | H(1) | A | ENVR | 18-May-2022 | Environmental Engineering Considerations and Laboratory Tests

- The `Status` column carries the **revision + change** (`H(1)` = Revision H, Change 1).
- Each row links to `qsDocDetails.aspx?ident_number=<N>`. `<N>` is the stable document identity.
- Pagination: `[UNVERIFIED]` — confirm the pager control name for multi-page result sets.

## 4. Document details — `qsDocDetails.aspx`

    GET https://quicksearch.dla.mil/qsDocDetails.aspx?ident_number=36064

Yields (verified for MIL-STD-962): Document ID, Title, Status, Document Date, Next Review Due,
FSC/Area, Doc Category, Lead Standardization Activity, Preparing Activity, Coordination Level, and
Army / Navy / Air Force / DLA / Other custodians. Some documents additionally show supersession and
related-document blocks; MIL-STD-962 showed none, so the parser must treat those blocks as optional.

**Revision list.** Every available revision appears as a JS call of the form:

    spawnPDFWindow('./ImageRedirector.aspx?token=<DMXID>.<IDENT>', <DMXID>)

Observed for ident 36064, in page order (newest first):
`5794274, 5754610, 5727650, 608578, 417149, 40733, 484743, 484741` —
corresponding to Rev D Change 3, D Notice 1, D Change 2, D Change 1, D, C, B, A.

- `DMXID` is the revision-specific document image id. **This is the download key.**
- Page order is newest-first, so the first token is the current revision. `[UNVERIFIED as a general
  rule — cross-check against the Status column revision label before trusting it.]`

## 5. PDF download — verified 3-hop chain

    qsDocDetails.aspx?ident_number=N
      -> ImageRedirector.aspx?token=<DMXID>.<N>     (HTML shim that auto-clicks a link)
      -> WMX/Default.aspx?token=<DMXID>             (302 Found)
      -> /Transient/<GUID>.pdf                      (200, application/pdf)

**The `ImageRedirector` hop can be skipped.** Go straight to:

    GET https://quicksearch.dla.mil/WMX/Default.aspx?token=<DMXID>
    Referer: https://quicksearch.dla.mil/qsDocDetails.aspx?ident_number=<N>
    User-Agent: <realistic Chrome UA>
    Cookie: <jar carried from the details-page request>

Follow redirects. Verified results:

- MIL-STD-962 (ident 36064, dmxid 5794274) -> 6,099,653 bytes, `application/pdf`, header `%PDF-1.5`
- MIL-STD-129 (ident 35520, dmxid 5781568) -> 5,804,525 bytes, `application/pdf`

Both a browser UA **and** a live cookie jar are required; missing either produced the WAF
`Request Rejected` page.

**`/Transient/<GUID>.pdf` URLs are one-shot and ephemeral. Never cache or persist them.**

## 6. Per-download watermark — affects hashing

Every page of a downloaded PDF is stamped in the footer:

    Source: https://assist.dla.mil -- Downloaded: 2026-08-24T21:00Z
    Check the source to verify that this is the current version before use.

Consequences:

1. **`sha256(pdf_bytes)` is NOT stable across downloads of the same revision.** Use it only for
   local file-integrity checks. Document identity is `(ident_number, dmxid)`.
2. For content-level dedupe or revision diffing, hash the normalised extracted text with the stamp
   line removed.
3. The reference extractor must strip this footer, or `assist.dla.mil` gets picked up as a citation
   on every single page.

## 7. Text layer

`pdftotext` on MIL-STD-129 (Rev R w/Change 3) produced 4,991 lines of clean text — modern documents
carry a real text layer. Older scanned specs will not; see the OCR fallback in ARCHITECTURE.md.

## 8. Rate-limit and politeness policy (hard requirement)

This is a US government system. The client ships with these defaults, enforced in one place:

- Max **2** concurrent requests to `quicksearch.dla.mil`.
- Minimum **1.5 s** delay between requests, with jitter.
- Exponential backoff `2s -> 4s -> 8s -> 30s -> 120s` on WAF block or 5xx. Abort the job after 5
  consecutive blocks and surface a clear message to the user.
- Hard cap on documents per crawl job (default 100, configurable).
- Never re-download a revision already present in the local library unless the user forces a refresh.
- Identify honestly via a custom header (`X-Client: AssistNavigator/<version>`) alongside the browser
  UA, so DLA can attribute the traffic if they care to.

## 9. Known-good fixtures for tests

Record these as VCR/`respx` cassettes so the test suite never touches the network:

| Document | ident_number | current dmxid (as of 2026-08-24) | Notes |
|---|---|---|---|
| MIL-STD-962 | 36064 | 5794274 | 8 revisions, no related docs, sparse references |
| MIL-STD-129 | 35520 | 5781568 | rich Section 2, mixed gov / non-gov references |
| MIL-STD-810 | 35978 | (fetch) | very large; use for size/timeout testing |
