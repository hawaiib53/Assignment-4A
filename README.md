# Negotiation Brief Generator

This repo generates a draft **negotiation brief** for a small business
preparing to renegotiate a distributor or retailer contract (e.g., an
independent body care brand renewing with Sephora, Ulta, or a regional
distributor). Given a description of the situation — a Word document, typed
notes, or both — it produces a structured brief: a situation summary,
distributor context, an objectives worksheet, talking points, and a table
of anticipated hard questions with suggested counters.

There are two implementations of the same generator, sharing the same
underlying knowledge base and matching logic:

| | `web/negotiation_brief_generator.html` | `tools/generate_negotiation_brief.js` |
|---|---|---|
| Runs where | In the browser, standalone | On the command line (Node) |
| Who it's for | The business owner, directly | Claude / a script, driving it on the owner's behalf |
| Distributor lookups | Static knowledge base + pasted research notes only | Static knowledge base, or a fresh web search fed in via `--live-research` |
| Output | Rendered on-page, exportable as `.docx` or PDF (print) | Writes a `.docx` file directly |

Both are **rule-based, not an LLM call** — they parse the input you give
them and select pre-written content from a knowledge base by keyword match.
That's a deliberate choice: it's auditable (you can read every line of logic
that decides what goes in the brief), it needs no API key or network access
to run, and a business owner's contract details never have to leave their
machine.

**Every brief produced is explicitly a draft.** It's watermarked "DRAFT —
PENDING OWNER REVIEW," and in the web tool the Download/Print buttons are
disabled by the code until a sign-off checkbox and name field are filled
in. Nothing here is legal or financial advice, and nothing gets sent to a
distributor automatically — the point is to prepare a human for a
conversation, not to conduct it.

## How it works, end to end

1. **Input.** You provide a distributor name, optionally a `.docx` file
   describing the scenario, and/or free-text notes. The web page also
   accepts pasted "distributor research notes" for a fallback path (see
   below).
2. **Text extraction.** If a Word document is provided, its raw text is
   pulled out of the `.docx` (see "Reading `.docx` files" below for how).
3. **Keyword matching.** All the input text is lowercased and scanned
   against two keyword tables in `tools/knowledge_base.json`:
   - **Distributor match** — does the text mention `sephora`, `ulta`, etc.?
     If not, it falls back to a generic "Independent / Regional Distributor"
     profile.
   - **Issue match** — does it mention margin/keystone language, MOQ,
     exclusivity, marketing/trade spend, chargebacks, payment terms, or
     slotting fees? Each match pulls in a block of context and a
     pre-written difficult question + counter for that issue.
4. **Model assembly.** The matched knowledge-base content, your own input
   text, and a fixed set of general negotiation tactics are combined into
   one in-memory "brief model" (a plain JS object: company, distributor
   label, situation paragraphs, context sections, objectives rows, talking
   points, Q&A pairs, sign-off fields).
5. **Rendering.** The same model is rendered two ways: as HTML (the on-page
   brief, with live `<input>` fields for the objectives worksheet) and, on
   request, as a `.docx` file (see "Writing `.docx` files" below).
6. **Sign-off gate.** The brief is inert until a human explicitly accepts
   it — this is enforced in code (disabled buttons), not just written as a
   warning.

## Repo layout

```
web/
  negotiation_brief_generator.html   the interactive tool (open directly in a browser)
  playwright_test.js                 automated browser test for the above
tools/
  generate_negotiation_brief.js      Node CLI version of the same generator
  knowledge_base.json                distributor/issue/tactics data both versions read
research/
  distributor_negotiation_knowledge_base.md   human-readable research writeup, with sources
samples/
  sample_scenario_input*.docx        example Word-doc inputs
  sample_negotiation_brief*_DRAFT.docx        example generated outputs
  live_research_qh_distribution.json          example live-research file for the CLI
```

### `web/negotiation_brief_generator.html` — the interactive tool

One HTML file with everything inlined (CSS, JS, knowledge base) — no
build step, no dependencies, no network calls at runtime. It does two
things that are normally handled by a library, written from scratch so the
page stays self-contained:

- **Reading `.docx` files.** A `.docx` is a ZIP archive; Word compresses
  the parts inside it with DEFLATE. The page includes:
  - A ~30-line ZIP central-directory reader that finds the
    `word/document.xml` entry inside the uploaded file.
  - [`tiny-inflate`](https://github.com/foliojs/tiny-inflate) (MIT
    license, embedded verbatim with attribution), a small public-domain
    DEFLATE decompressor, to decompress that entry.
  - The browser's built-in `DOMParser` to parse the resulting XML and walk
    `<w:p>` paragraphs, pulling text out of `<w:t>` runs (with `<w:tab>`
    and `<w:br>` mapped to tab/newline characters).
  - This whole pipeline runs in `extractDocxText()` and is triggered the
    moment a file is chosen in the "Scenario notes" upload field.
- **Writing `.docx` files.** Going the other direction is simpler because
  the writer controls both ends: `buildDocxBlob()` builds the OOXML parts
  by hand (`word/document.xml`, `styles.xml`, the content-types and
  relationship files a `.docx` needs to open cleanly) and packages them
  into an **uncompressed** ("stored") ZIP — no DEFLATE encoder needed, just
  a CRC-32 checksum and the ZIP local/central-directory byte layout, both
  implemented directly (`crc32()`, `buildZipStore()`).
- **The rest of the page** is straightforward DOM code: text inputs feed a
  live "detected: Sephora · MOQ · Payment Terms"-style chip row
  (`updateChips()`, debounced on `input` events), a Generate button builds
  the brief model and renders it (`buildModel()` / `renderBrief()`), and
  the objectives worksheet is real `<input>` elements whose values are read
  back at export time.

### `tools/generate_negotiation_brief.js` — the CLI version

The same matching/model logic, restructured for scripted use (e.g., by
Claude, driving it on the owner's behalf in a chat session):

- `--scenario-docx <file>` shells out to `pandoc -t plain` to extract text
  (simpler than reading the ZIP by hand, since this runs somewhere with
  `pandoc` installed rather than in a browser sandbox).
- `--extract` runs just the detection step and prints whether the named
  distributor matched a built-in profile or fell back to generic — a cue
  that a live web search is worth doing before generating.
- `--live-research <file.json>` accepts a small JSON file (distributor
  name, researched bullet points, sources, optionally extra difficult
  questions) and merges it into the Distributor Profile section with
  sources cited — see `samples/live_research_qh_distribution.json` for the
  shape. This is how the tool incorporates a distributor that isn't
  Sephora or Ulta: something with real web access (a human, or Claude via
  its search tool) researches it once, and hands the findings back in as
  data.
- Uses the [`docx`](https://www.npmjs.com/package/docx) npm package
  (installed under `tools/node_modules`, not committed) to build the
  output file, rather than the hand-rolled writer in the web version.

### `tools/knowledge_base.json` — the shared data

Three parts, referenced by both implementations:

- `distributors` — per-distributor `match` keywords, a `label`, `context`
  bullets, and one or more `difficultQuestions`. Currently populated for
  Sephora and Ulta from the research in `research/`, plus a generic
  fallback entry.
- `issues` — the same shape, keyed by contract issue (margin, MOQ,
  exclusivity, marketing, chargebacks, payment terms, slotting), so
  whichever issues are mentioned in the input each contribute their own
  context and difficult question.
- `tactics` and `genericDifficultQuestions` — general small-business
  negotiation advice (BATNA, anchoring with target/acceptable/walk-away
  ranges, data-driven framing) and pressure-tactic questions that apply
  regardless of distributor or issue.

`research/distributor_negotiation_knowledge_base.md` is the prose version
of the same research, with the source links each bullet above was drawn
from — that's the file to edit first if you want to update or extend what
the JSON knows.

## Running it

**Web tool** — no setup: open `web/negotiation_brief_generator.html` in a
browser, or use the published version.

**CLI**:
```bash
cd tools && npm install   # installs the docx package
node generate_negotiation_brief.js \
  --company "Your Company" --distributor "Sephora" \
  --scenario-docx notes.docx --notes "extra context" \
  --owner "Your Name" --out brief_DRAFT.docx
```
Run with `--extract` first (see above) if the distributor might need a
live web lookup before generating.

**Tests**: `web/playwright_test.js` drives the HTML page in a real
Chromium instance — fills the form, uploads `samples/sample_scenario_input.docx`,
generates a brief, completes sign-off, and downloads the resulting
`.docx` — asserting on the content and that the file is well-formed. Run
`cd web && npm install && npm test`.

## Limitations

- The knowledge base reflects open web research as of July 2026 and will
  drift out of date — treat every specific figure (margins, chargeback
  windows, financial results) as a starting point to verify, not a fact to
  quote directly.
- The web tool's `.docx` reader handles the common case (a single
  `word/document.xml` compressed with DEFLATE or stored uncompressed,
  which covers documents from Word, Google Docs, and LibreOffice) but
  isn't a complete OOXML implementation — if a file fails to parse, the
  page says so and asks you to paste the text in instead of failing
  silently.
- This is keyword matching, not language understanding — it will miss an
  issue that's described without any of the matched terms, and the
  situation summary is your input text rendered back, not summarized or
  interpreted.
