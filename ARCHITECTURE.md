# Dream Journal Template Generator — Architecture

> Keep this document updated when any layer's responsibilities or boundaries change.

---

## Data Flow Overview

```
┌─────────────────────────────────────────────────────────────┐
│  1. INPUT                                                    │
│     User uploads .docx / .txt file  OR  types into form     │
└──────────────────────────────┬──────────────────────────────┘
                               │
                               ▼
┌─────────────────────────────────────────────────────────────┐
│  2. PARSER  (extractDreamData / normalizeDreamEntry)         │
│     Raw document text → structured dream data object        │
└──────────────────────────────┬──────────────────────────────┘
                               │
                               ▼
┌─────────────────────────────────────────────────────────────┐
│  3. STATE  (form fields / populateFormFromDreamData)         │
│     Structured data → populated HTML form fields            │
│     (dreamTitle, dreamNarrative, dreamBrainstorm, etc.)     │
└──────────────────────────────┬──────────────────────────────┘
                               │  (user may edit at this point)
                               ▼
┌─────────────────────────────────────────────────────────────┐
│  4. PAGINATION  (planPages / paginateTokenStream)            │
│     Form field values → page descriptors                    │
│     Binary-search word-count pagination per section          │
└──────────────────────────────┬──────────────────────────────┘
                               │
                               ▼
┌─────────────────────────────────────────────────────────────┐
│  5. RENDER  (renderPageDescriptors / buildPageSheetFromDescriptor) │
│     Page descriptors → DOM page sheets appended to          │
│     #pagesContainer                                         │
└──────────────────────────────┬──────────────────────────────┘
                               │
                               ▼
┌─────────────────────────────────────────────────────────────┐
│  6. PRINT  (window.print / CSS @media print)                │
│     DOM page sheets → browser print dialog / PDF export     │
└─────────────────────────────────────────────────────────────┘
```

---

## Generation Paths (canonical — as of stabilization-pass-01)

There are exactly **two** paths into `_showPagesCore`. All generation must flow through one of them.

```
SINGLE ENTRY
  Form fields (user-editable)
      │
      ▼
  collectCurrentDreamFormData()        ← reads ALL form fields
      │
      ▼
  generateFromCurrentForm()            ← normalizes, calls _showPagesCore([data])
      │
      ▼
  _showPagesCore([data])

BATCH (2+ entries)
  Parsed entry objects (from file/paste)
      │
      ▼
  generateFromEntryArray(entries)      ← normalizes each entry (shallow copy)
      │                                   never reads form, never writes form
      ▼
  _showPagesCore(normalized)
```

### Routing

- `handleGenerate()` — chooses the path based on `pendingPages.length` and parse results. Calls `generateFromCurrentForm()` or `generateFromEntryArray()` directly.
- `showPages(dataArray)` — backward-compat router. Single → populate form + `generateFromCurrentForm`. Batch → `generateFromEntryArray`. No new code should call this.
- `generateFromCurrentState()` — one-line alias for `generateFromCurrentForm()`. Kept for backward compat only.
- `generateFromForm()` — one-line alias for `generateFromCurrentForm()`.

### What must NEVER happen

| Forbidden | Why |
|---|---|
| Batch path reads from form | Form may contain stale single-entry data; entries 2+ would be wrong |
| Batch path writes to form | Would overwrite user edits in the form |
| Single path bypasses `collectCurrentDreamFormData()` | Would ignore manual user edits made in the form |
| Any path calls `showPages()` for new code | `showPages` is legacy routing; use the canonical functions directly |
| `_showPagesCore` called outside the two generation functions | Bypasses normalization, debug logging, and `lastGeneratedData` tracking |

---

## Layer Responsibilities

### Layer 1 — Input
- Accepts file uploads (DOCX via mammoth.js, TXT via FileReader) or direct form editing
- Calls into the Parser layer with raw text string
- **Does NOT** make layout or rendering decisions

### Layer 2 — Parser (`extractDreamData`, `normalizeDreamEntry`)
**Responsibility:** Transform raw unstructured document text into a normalized `dreamData` object with keys:
- `dreamTitle` — string
- `dreamDate` — string
- `dreamNarrative` — string (raw multi-paragraph text)
- `dreamBrainstorm` — string
- `dreamInterpretation` — string
- `dreamApplication` — string
- `dreamSymbols`, `dreamScriptures`, `dreamAdditionalNotes`, etc.

#### Three Parsing Modes

The parser detects one of three document modes and routes accordingly:

| Mode | Trigger | Strategy |
|---|---|---|
| `repeated_simple_sections` | `detectRepeatedSimpleSectionBlocks` returns result | Block-based accumulation: repeated headings merge into the same field |
| `labeled_sections` | `countDreamDocumentSectionHeadings >= 2` | Bucket accumulation via `buildLabeledSectionsResult` — each heading type collects all content until the next heading |
| `unlabeled_or_template` | Neither of the above | Fallback: narrative-dominant parsing, MAJOR_SECTION_FIELDS guard, preface/scene detection |

#### Shared Post-Processing Contract

Mode-specific builders must **not** return from `extractDreamData` early.

- `buildRepeatedBlocksResult()` and `buildLabeledSectionsResult()` only produce mode-specific field buckets.
- Their output is merged into the shared `result` object via `Object.assign(...)`.
- After that, **all** parse modes continue through the same shared post-processing pipeline before final return.

This shared parser tail owns:
- narrative glossary stripping via `extractNarrativeSymbolGlossary(...)`
- listed symbol extraction via `extractListedSymbolEntries(...)`
- symbol prioritization via `prioritizeListedSymbols(...)`
- inferred symbol fallback via `extractSymbolEntriesFromText(...)`
- scripture and emotion auto-extraction / cleanup

For `labeled_sections`, the parser flow now has one extra rebalance step before the shared tail:

```text
bucket headings
→ reconstruct paired column headers
→ cleanupLabeledSectionsBuckets(...)
→ shared parser tail
```

That ordering matters. `cleanupLabeledSectionsBuckets(...)` is where `dreamNarrative` is cleaned back down to dream-scene prose before symbol / scripture / interpretation / application helpers run. It removes glossary lines, quoted commentary, scripture blocks, and other non-scene residue from `dreamNarrative`, and rehomes them into the appropriate labeled fields.

Only the unlabeled fallback parse loop is mode-gated:

```javascript
if (docMode === 'unlabeled_or_template') {
  // fallback loop only
}
```

This prevents repeated/labeled documents from re-entering the fallback parser while still guaranteeing that all parse modes receive the same downstream cleanup and enrichment.

### Parser Layer Clarifications

- `labeled_sections` input is treated as potentially **hybrid** internally (not assumed to be purely labeled end-to-end).
- Dream Narrative parsing includes a **boundary guard** to stop capture when reflective/interpretive transitions are encountered.
- Brainstorm content can contain inline sub-sections (`Interpretation`, `Application`, `Symbols`, `Scriptures`).
- Inline labels inside Brainstorm are treated as **hard parser boundaries** and routed to their target section.
- Heuristic cleanup remains a **safety net** after primary parsing; it is not the primary section-boundary mechanism.
- Raw pasted text now uses a single canonical parser flow for entry splitting.
- `splitTextEntries` has been removed as dead code.

#### Key Subfunctions

- `extractTopDocumentTitle` — scans first 8 lines for title candidate (skips dates, metadata, bare section headings)
- `enrichFromFilename` — overrides title from filename when current title is a placeholder/junk value
- `cleanedPreface` — strips metadata label+value lines from lines before the first section heading
- `detectSectionFromHeading` — maps heading text → field name using `normalizeOcrTolerantHeadingKey` + `HEADING_RULES`. Returns null for empty normalized strings.
- `normalizeSectionHeading` (inner) — strips markdown artifacts, lowercases, removes trailing separators. **Hyphen must be escaped or at boundary of character class** — unescaped hyphen creates a Unicode range that strips all letters.
- `normalizeSimpleSectionHeading` (global) — used by `countDreamDocumentSectionHeadings` as the primary heading recognizer for mode detection.
- `countDreamDocumentSectionHeadings` — counts how many lines are recognized section headings. Uses `normalizeSimpleSectionHeading` (covers extended variants like "Dream Narrative", "Long Interpretation"), slash-stripping for slash-combo forms ("Brainstorm/Initial Thoughts"), and explicit regex for application aliases ("Reflect", "How I Will Respond").
- `detectRepeatedSimpleSectionBlocks` — detects repeated heading pattern. Activates when `meaningfulBlocks.length >= 2` AND (any heading repeats OR 3+ distinct heading types exist).
- `parseSectionBlocks` — block accumulator for repeated-section mode. Uses `detectHeading` for heading detection; filters template noise via `isTemplateNoiseLabel`, `isStandaloneJunk`, `isDecorativeSidebarNoiseCluster`.
- `shouldUseSectionBlocks` — gate for block mode. Activates when `uniqueHeadings.size >= 2` (any 2 distinct section types with real content).
- `buildLabeledSectionsResult` — bucket accumulation for labeled-sections mode. Each field collects all matching blocks and joins with `joinSectionChunks`.
- `buildLabeledSectionsResult` also reconstructs paired column-header shapes when Mammoth flattens a two-column row as consecutive headings (`Dream` + `Brainstorm`, `Long Interpretation` + `Application`) followed by two body chunks. In that case it assigns the first body chunk to heading A and the second to heading B before falling through to the shared parser tail.
- `cleanupLabeledSectionsBuckets` — labeled-sections-only rebalance pass inside `buildLabeledSectionsResult`. Cleans non-scene residue out of `dreamNarrative`, deduplicates scene prose out of `dreamBrainstorm`, and reassigns glossary/scripture/interpretation/application content before shared parser-tail enrichment runs.
- `buildRepeatedBlocksResult` / `buildLabeledSectionsResult` — must merge into `result` and then fall through to shared post-processing. They are not final-return boundaries.
- `HEADING_RULES` — ordered rule array mapping heading regexes → field names. Covers compound forms ("Dream Narrative"), slash-combos ("Brainstorm/Initial Thoughts"), modifier prefixes ("Long Interpretation"), and aliases ("Reflect" → `dreamApplication`).

#### MAJOR_SECTION_FIELDS Guard (unlabeled_or_template mode)
Guards against treating narrative-continuation lines as new sections. Only skips a heading if `!hasRealContentAhead(lineRecords, li)`. A heading followed by real content is always accepted — this prevents the guard from suppressing valid Interpretation/Application sections.

**Must NOT:**
- Access or read DOM layout/rendering elements
- Make decisions about pagination or page count
- Write to form fields (that is the State layer's job via `populateFormFromDreamData`)

---

### Layer 3 — State (Form Fields)
**Responsibility:** Bridge between parsed data and pagination. The form acts as the canonical single source of truth for what gets paginated.

**Key functions:**
- `populateFormFromDreamData(data)` — writes parsed data into form textareas/inputs
- `collectCurrentDreamFormData()` — reads form fields back into a data object for pagination
- `setField(id, val)` — safely sets a form field; returns early if value is empty/whitespace

**Must NOT:**
- Generate pages or build DOM page elements
- Run measurements (getBoundingClientRect, offsetHeight)
- Contain any print-specific CSS logic

**Invariant:** `setField` never clears a field that has user content. It only sets; it does not overwrite with empty.

---

### Layer 4 — Pagination
**Responsibility:** Convert content strings into page descriptor objects. Determine word-count boundaries that fit each section's content within available page height.

**Key functions:**
- `getLayoutConfig()` — computes print geometry (page width, body height, section heights). Minimum floor: `Math.max(..., 60)`.
- `planPages(data, layout, measure)` — orchestrates all section pagination; returns array of page descriptors. Array carries `.interpretationDescriptors` and `.applicationDescriptors` as named properties.
- `paginateTokenStream(content, cssClass, subtitle, pageStyle, measureFactory)` — binary-search word-count pagination for narrative, brainstorm, interpretation, application sections. Throws `'Overflow - invalid pagination'` if `finalHeight > maxH` after binary search.
- `paginateBrainstormTokenStream` — brainstorm-specific token stream pagination
- Measurement context factories: `createNarrativeMeasurementContext`, `createBrainstormMeasurementContext`, `createInterpretationMeasurementContext`, `createApplicationMeasurementContext`
  - Each creates a hidden off-screen page shell, measures the available body height via `getBoundingClientRect()`
  - All use `Math.max(60, ...)` as minimum floor (prevents throw when measurement sheet renders off-screen)
  - All call `.cleanup()` after pagination to remove the hidden shell from the DOM

**Must NOT:**
- Append any elements to `#pagesContainer` or any visible DOM node
- Read from form fields directly (it receives pre-read `data` object)
- Touch CSS classes on visible page elements
- Change print CSS

**Overflow guard:** If `finalHeight > maxH` after binary search, `paginateTokenStream` throws. This indicates a single word is taller than the entire available page height — an impossible layout. With the 60px floor in measurement contexts, this should only occur in extreme pathological cases (e.g. an image or table row taller than 60px for a single "word").

---

### Layer 5 — Render
**Responsibility:** Transform page descriptors into DOM elements and append them to `#pagesContainer`.

**Key functions:**
- `_showPagesCore(dataArray)` — entry point; shows output section, resets preview frame, iterates entries, calls `planPages` then `renderPageDescriptors`
- `renderPageDescriptors(container, descriptors, data, applyStyle)` — iterates descriptor array, calls `buildPageSheetFromDescriptor` for each, appends to container
- `buildPageSheetFromDescriptor(descriptor, data)` — routes to the appropriate page builder based on descriptor type
- `buildHeaderPage(data)` — builds the metadata/title page (always appended first)
- `buildDreamPage`, `buildBrainstormPageFromDescriptor`, `buildInterpretationPageFromDescriptor`, `buildApplicationPageFromDescriptor` — section-specific page builders

**Error handling:** `_showPagesCore` wraps `planPages` in try/catch. If pagination throws for any reason, the error is logged to console with content lengths for debugging. The catch block uses safe empty defaults so downstream `renderPageDescriptors` calls do not throw.

**Must NOT:**
- Modify the `data` object it receives
- Run its own word-count or height calculations (measurements happen in Layer 4)
- Change print media CSS
- Alter form field values

---

### Layer 6 — Print
**Responsibility:** CSS `@media print` rules control physical page breaks, margins, and sizing for the browser's print dialog / PDF export.

**Must NOT be modified to fix:**
- Parser issues (Layer 2)
- Pagination overflow (Layer 4)
- Rendering bugs (Layer 5)
- Preview scaling (handled by CSS transform on `#pagesContainer`, separate from print CSS)

---

## What Must NEVER Cross Layers

| Forbidden crossing | Why |
|---|---|
| Parser reads DOM geometry | Parser runs before render; DOM is not ready. Geometry is undefined at parse time. |
| Parser writes to form fields directly | Only `populateFormFromDreamData` (State layer) sets form fields. Parser returns a data object. |
| Pagination appends to visible DOM | Pagination creates hidden off-screen measurement sheets only. Visible DOM is Render's job. |
| Pagination reads form fields | Pagination receives a pre-collected data object. It does not call `document.getElementById`. |
| Render modifies the data object it receives | Render is read-only with respect to data. Mutations must happen in Parser or State. |
| Print CSS changes to fix pagination math | Pagination math uses `getLayoutConfig()` which reads print geometry constants. Fix geometry in `getLayoutConfig`, not in CSS. |
| State layer re-parses document text | State layer only moves data between parsed object and form fields. Re-parsing belongs in Parser. |

---

## Key Constants

| Constant | Value | Purpose |
|---|---|---|
| `DREAM_PAGE_MEASURE_SAFETY_PX` | (see code) | Safety buffer subtracted from narrative available height to prevent subpixel render clipping |
| Measurement context floor | `60` | Minimum `availableHeight` in all four measurement contexts — prevents throw when off-screen measurement returns zeroes |
| `getLayoutConfig()` minimum body height | `Math.max(..., 60)` | Same 60px floor at the layout config level |

---

## Error Surfaces

| Error | Layer | Cause | Handling |
|---|---|---|---|
| `'Overflow - invalid pagination'` | Layer 4 `paginateTokenStream` | Single word taller than `availableHeight` | Caught by `_showPagesCore` try/catch (Layer 5). Logged to console with content lengths. |
| Silent header-only output | Layer 5 | Uncaught exception in `planPages` propagated through Promise chain | Fixed: try/catch in `_showPagesCore`. Error now surfaces in console. |
| Wrong title from document (date, bare "Dream") | Layer 2 | `extractTopDocumentTitle` not excluding date lines or bare section headings | Fixed: `isDateHeading` check + break on bare "Dream"/"Vision". |
| Metadata contaminating narrative | Layer 2 | Curly apostrophe (U+2019) not matched by straight-quote regex | Fixed: `[''']` character class in three regex locations. |
| All section headings normalize to `""` — `detectSectionFromHeading` returns `null` for everything | Layer 2 | Unescaped hyphen in `[\s:;,.!?-–—]` created Unicode range U+002D–U+2013, stripping ALL letters from normalized heading text | Fixed: `[\s:;,.!?–—\-]` in `normalizeSectionHeading` (inner) and `normalizeSimpleSectionHeading` (global). |
| Extended heading variants ("Dream Narrative", "Long Interpretation", "Brainstorm/Initial Thoughts") not counted by mode gate | Layer 2 | `countDreamDocumentSectionHeadings` used narrow bare-keyword regex, `detectedHeadingCount` stayed below 2, `labeled_sections` mode never triggered | Fixed: `countDreamDocumentSectionHeadings` now uses `normalizeSimpleSectionHeading` + slash-strip + application aliases. |
| Table-based docx: all content (narrative + brainstorm + interpretation) rendered in narrative section | Layer 2 + UI | Mammoth flattens two-column table row-by-row, producing `"Dream\nBrainstorm\n[all content]"`. "Dream" block is empty; "Brainstorm" absorbs all content. `dreamNarrative = ''` causes `populateFormFromDreamData` fallbackNarrativeToRaw to dump full raw text into narrative field. | Fixed: `buildRepeatedBlocksResult` now calls `splitNarrativeLikeContentFromInterpretation` on `dreamBrainstorm` when `dreamNarrative` is empty, recovering the narrative portion. |

---

## Function Call Chain: File Upload → Pages Generated

### Single File Upload → Auto-Generate
```
FileReader / mammoth.js
  └─ extractDreamData(text, filename)
       ├─ extractTopDocumentTitle(lines)
       ├─ detectSectionFromHeading(heading)
       ├─ cleanedPreface(prefaceLines)
       └─ normalizeDreamEntry(data, filename)
            └─ enrichFromFilename(data, filename)

  └─ applyUploadFlowResultToUi(result)
       ├─ populateFormFromDreamData(data)      [sets form fields]
       └─ generateFromCurrentState()           [alias → generateFromCurrentForm()]

  └─ generateFromCurrentForm()
       └─ collectCurrentDreamFormData()        [reads all form fields]
       └─ finalizeDreamTitle(data)
       └─ _showPagesCore([data])
```

### Manual Form Edit → Regenerate
```
[User edits textarea / input fields]

Generate button click
  └─ handleGenerate()
       └─ generateFromCurrentForm()            [single-entry path]
            └─ collectCurrentDreamFormData()   [reads edited form fields]
            └─ finalizeDreamTitle(data)
            └─ _showPagesCore([data])
```

### Batch Upload (2+ files / entries)
```
FileReader × N files
  └─ processMultiUploadFlow(files)
       └─ parseSingleDreamEntry(text) × N     [each file → normalized entry]

  └─ applyUploadFlowResultToUi(result)
       ├─ populateFormFromPendingEntry(first)  [UI only — shows first entry in form]
       └─ [pendingPages queued]

Generate button click
  └─ handleGenerate()
       └─ generateFromEntryArray(allData)      [batch path — does NOT read form]
            ├─ finalizeDreamTitle(e) × N
            └─ _showPagesCore(normalized)
```

### `_showPagesCore` (shared by both paths)
```
_showPagesCore(dataArray)
  ├─ buildHeaderPage(data)                     [title/metadata page — always first]
  ├─ getLayoutConfig()                         [print geometry constants]
  ├─ createContentMeasurer(layout)
  ├─ try { planPages(data, layout, measure) }  [wrapped in try/catch]
  │    ├─ paginateTokenStream(...)             [narrative pages]
  │    ├─ paginateBrainstormTokenStream(...)   [brainstorm pages]
  │    ├─ paginateTokenStream(..., createInterpretationMeasurementContext)
  │    └─ paginateTokenStream(..., createApplicationMeasurementContext)
  └─ renderPageDescriptors(container, descriptors, data, applyStyle)
       └─ buildPageSheetFromDescriptor(descriptor, data) [per page]
```
