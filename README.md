# UKIPO Pre-Upload Document Check

A single-file, offline, browser-based tool that checks Word documents (.docx) against the [One IPO Directions of 1 April 2026](https://www.gov.uk/government/publications/directions-for-the-use-of-the-one-ipo-digital-services/directions-for-the-use-of-the-one-ipo-digital-services) before they are filed with the UKIPO.

It runs entirely in the browser. No data leaves the device, no network calls, no installation. The whole tool is one HTML file.

## What it is and what it isn't

**It is** a pre-flight checker for patent admin and fee earners. It reads a DOCX, parses the underlying XML, and reports findings against a set of rules drawn from the One IPO Directions. The fee earner reviews the findings, decides what to do, and is responsible for the changes that follow.

**It isn't** a paid product, isn't supported by Anthropic or the UKIPO, doesn't update itself when the Directions change, and doesn't claim to catch every possible issue. It is a useful safety net, not a substitute for review.

If the Directions change, this repository's rules need updating. There is no auto-update channel.

## How to use it

1. Open the HTML file in a modern browser (Edge, Chrome, Firefox or Safari).
2. Drop a `.docx` file onto the upload zone, or click to pick one.
3. Click **Run checks**.
4. Read the findings. Each finding shows the rule it relates to, what was detected, where it was detected (page or part of the document), and a suggested remedy.
5. Fix what needs fixing in Word, save, and re-run if you want to confirm.

Maximum input size is 100 MB. The tool also refuses inputs that decompress to more than 500 MB.

## What it checks

The tool covers two broad areas: the One IPO Directions and a Document Inspector parity check.

**Directions coverage** (selected): Direction 10 (file format), Direction 11 (alt-text on images), Direction 13 (sequence listings), Direction 15 (combined Application Documents), Direction 18 (OLE objects), Direction 19 (comments), Direction 20 (tracked changes), Direction 21 (version history), Direction 22 (URLs), Direction 24 (font size and font choice), Direction 25 (Description title), and Schedule 2 paragraphs 3, 4, 6, 7 and 9 (frames, page numbering, page number position, margins, line spacing).

**Document Inspector parity**: hidden text, near-invisible text colour, decorative or review-style formatting, macros, embedded code, document properties metadata (author, last modified by, company, manager, template, custom properties from SharePoint or iManage), page watermarks, footnotes and endnotes, numbering anomalies, embedded image metadata (EXIF, XMP, PNG text chunks).

For the complete list, run the tool against any document. The "About this tool" panel inside the HTML lists every check the build performs.

## What it deliberately does not do

- It does not save reports, generate PDFs, or copy results to the clipboard. Findings live in the page only.
- It does not connect to the network, send telemetry, or store anything between runs.
- It does not check ODT files (Direction 10 permits ODT; this tool reads DOCX only).
- It does not check drawings or PDFs (Directions 12 and 26 are out of scope).
- It does not auto-fix anything. The fee earner remains responsible for changes.

## Architecture

Single HTML file. The bundled libraries and the application code are inlined. The only browser features used are the File API, `crypto.subtle` (for SHA-256 audit hashing and SHA-512 self-test integrity), and `DOMParser` (for XML).

**JSZip 3.10.1** is inlined and bracketed between `BEGIN JSZIP` and `END JSZIP` comments. Provenance, licence, and a SHA-512 reference are recorded in the comment block at the top of the file.

**Content Security Policy** is restrictive: `default-src 'none'`, `connect-src 'none'`, `form-action 'none'`. Network exfiltration is structurally blocked at the browser level. `script-src 'unsafe-inline'` is necessary because the JavaScript is bundled into the HTML; escaping discipline in the code is the actual XSS defence.

**Security design**: prototype-pollution defences using `Object.create(null)` for any object keyed from document data (relationship IDs, style IDs, header/footer part types). Recursion guards on style-chain walks. HTML escaping consistently applied at the rendering boundary. Hyperlinks in findings are rendered as text, never as clickable anchors. See `SECURITY.md` if present, or the comment blocks at the top of each check function in the HTML.

## Self-test harness

The tool includes a self-test for IT and audit. It is invisible to fee earners during normal use.

To run it, open the HTML file and append `#selftest` to the URL:

```
file:///.../ukipo_checker_v0_5_1.html#selftest
```

The self-test runs three reference documents (built in the browser from XML strings inside the HTML) through the full check pipeline and reports any deviation from expected results. It also verifies that the inlined JSZip library matches the SHA-512 hash captured when the build was produced.

A passing run is evidence that the tool is producing the results it was designed to produce. A failing run identifies which check disagreed with its expectation and is the starting point for diagnosis.

A "Return to main view" link returns the page to normal behaviour.

## Maintaining the tool

The HTML file is the source of truth. There is no build step, no bundler, and no external dependencies. To edit a check, edit the relevant function in the file directly, save, and refresh in the browser.

**Adding or changing a check**: locate the check function (named `checkDirectionNN_*` or similar), edit, save. Run the self-test to confirm nothing regressed. If the change affects what the self-test expects, edit the relevant `FIXTURE_*_PARTS` XML or the `TEST_FIXTURES` expectations table in the same file. Both live near the bottom of the application script and are documented inline.

**Adding a new check**: add a function alongside the others, add a call to it inside `collectFindings`, and add an entry to the `TEST_FIXTURES` expectations table so the self-test covers it.

**Updating JSZip**: download the new version, replace the bytes between `BEGIN JSZIP` and `END JSZIP` comments, update the SHA-512 reference in the top comment, and run the self-test (which will fail, intentionally, because the hash baked in for the integrity check is the old one). Open the failed self-test, copy the actual hash it computed, and paste it as the new value of `EXPECTED_JSZIP_SHA512`. Re-run the self-test; it should pass.

**Updating the rules to match a new Directions revision**: edit each affected check, update the wording of `description` and `suggestion` strings, update the version banner at the bottom of the file and the release notes inside the "About this tool" panel. Run the self-test.

## Release notes

Maintained inside the "About this tool" panel of the HTML file. Latest visible version of the tool at the time of this README is **v0.5.1**.

| Version | Notes |
|---------|-------|
| v0.5.1  | Self-test harness for IT and audit (in-browser fixture construction, JSZip integrity check). |
| v0.5.0  | Hard size limits (100 MB input, 500 MB unpacked). Adds document properties metadata, page watermark, footnotes and endnotes, numbering anomaly and embedded image metadata checks. |
| v0.4.3  | Heuristic checks demoted from fail to warn (Direction 13, 15, 21, 25). |
| Earlier | See the "About this tool" panel inside the HTML. |

## Browser support

Tested against current versions of Edge, Chrome and Firefox. Safari should work; the only browser feature with known compatibility differences is `crypto.subtle`, which all supported browsers implement. The tool does not work in Internet Explorer (no `crypto.subtle`, no `async`/`await`).

For offline use, save the HTML file locally and open it from the file system. The CSP is configured to allow this.

## Limitations to be aware of

- Pagination uses Word's own `lastRenderedPageBreak` markers. A document that has never been opened in Word will lack these markers; the tool falls back to "Body" for the location of findings.
- EXIF parsing is focused, not exhaustive. IFD0 ASCII tags are read; the EXIF SubIFD is not walked. GPS data presence is flagged; coordinates are not decoded. TIFF, HEIC and HEIF are flagged by extension without parsing.
- Watermark detection covers the classic Word watermark (VML `PowerPlusWaterMarkObject`) plus floating shapes in headers carrying common watermark words. Custom watermark designs may slip past.
- Tracked-change text is included in heuristic checks as if accepted. Run the checker again after accepting changes if you want to be sure.
- The `frame-ancestors` directive is set via `<meta>` CSP, which is documented to require a response header to take effect. If clickjacking is part of the threat model for how this tool is served, an `X-Frame-Options: DENY` header from the hosting environment is needed.

## Reporting an issue

Open an issue in this repository describing what document triggered it (without attaching the document itself if it contains client information), what the tool reported, and what you expected. Screenshots of the findings panel and the audit footer are helpful.

If the issue is a security concern, please raise it directly with the maintainer rather than opening a public issue.

## Licence

[Set this before publishing.] The bundled JSZip library is dual-licensed under the MIT and GPL-3.0 licences; see the comment block at the top of the HTML file for the upstream attribution.

## Acknowledgements

The One IPO Directions, dated 1 April 2026, are the source of the rules this tool checks. The Directions are Crown Copyright; references are not reproductions.
