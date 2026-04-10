# Dream Journal Template Generator — CHANGELOG

Format per entry:
- **Version / Timestamp** — short description
- **Issue** — what was wrong / what the bug was
- **Root Cause** — why it was wrong at the code level
- **Fix** — exactly what was changed and where
- **Scope** — which functions / lines were touched
- **Verification** — how to confirm the fix works
- **Notes** — anything to watch for going forward

---

## UI cleanup pass — 2026-04-05

### CTA system cleanup and breakpoint truth verification (Confirmed)

**What Changed:** The Dream Journal Generator UI was cleaned up to re-establish a single active CTA system per breakpoint and remove overlapping legacy action systems from the current architecture model.

Confirmed current-state changes:
- removed the legacy desktop action bar system from the intended UI architecture
- removed the old inline input action row system from the intended UI architecture
- established one CTA system per breakpoint
- desktop CTA truth: `.generate-strip`
- mobile CTA truth: `.mobile-floating-bar`
- confirmed desktop accordions remain collapsible
- kept print-only accordion force-open behavior as valid print behavior
- resolved merge conflict direction by keeping the cleaned local file structure as the source of truth

**Why It Changed:** Multiple overlapping UI systems were creating confusion about which controls were actually active, causing stale verification results and increasing regression risk during cleanup, merge resolution, and follow-up UI work.

**What Was Not Touched:**
- parser logic remained untouched
- pagination logic remained untouched
- print logic was not rewritten; valid existing print behavior remained in place

**Verification:** Cleanup was verified against the current `index-v2.html` file state using direct file inspection and exact selector / reference searches rather than stale snapshots or prior assumptions.

**Risk / Follow-up:**
- future UI work must preserve one CTA system per breakpoint
- future verification should be performed against the current file state, not stale snapshots or outdated cleanup assumptions

---

## UI stabilization layer — 2026-04-05

### Stage-based UI / navigation / action-flow stabilization (Confirmed)

**Issue:** After the parser and pagination fixes were already in place, the generator UI still had several stage-management problems that made the app feel unstable even when parsing/rendering were correct:
- Form / Review / Print did not consistently behave like isolated stages
- upload- and paste-populated states could surface different CTA behavior
- stale mobile and desktop action systems were still coexisting
- Review / Print navigation could fall back to the wrong stage
- screen preview behavior could diverge visually from print clipping in writing regions

**Root Cause:** These were UI-layer coordination problems, not parser or pagination failures.

The instability came from several overlapping causes:
1. stage progression had drifted during iteration, so some flows auto-advanced while others expected manual advancement
2. mobile action visibility was split across multiple partially overlapping CTA systems
3. tab transitions were not all being treated as explicit stage isolates, which allowed inactive panels or malformed wrapper boundaries to create mixed layout behavior
4. stale action-row and floating-action remnants were still present in markup / CSS / JS references
5. screen-preview writing-region containment did not fully mirror print-time clipping expectations

**Fix:** The UI flow was stabilized around a single intended stage model:
- Form -> Review -> Print is now the intended user flow
- upload and paste both remain on Form after content is populated
- the user manually advances from Form to Review
- the user manually advances from Review to Print

Confirmed UI changes completed in `index-v2.html`:
- restored the mobile floating action bar as the active mobile CTA system
- ensured `Continue to Review` appears for both upload-populated and paste-populated states
- treated pasted notes as meaningful content for CTA visibility
- removed the stale duplicate mobile action-row path and dead JS references tied to it
- made `switchTab()` explicit and deterministic for `generator`, `review`, `print`, and `saved`
- stabilized Review routing so Review no longer falls back to Form during normal stage transitions
- ensured Print -> Back to Review returns to Review
- stabilized layout isolation so inactive tab panels no longer remain in the page flow when another main stage is active
- normalized Form / Review stage container behavior so the desktop app no longer feels like two unrelated widths
- removed stale `.input-action-row` UI remnants and kept `.mobile-floating-bar` as the single active mobile action system
- fixed screen-preview writing-region containment so lined regions no longer visibly bleed into the footer area on screen and better match print clipping behavior

**Scope:** UI / navigation / stage-isolation layer in `index-v2.html` only.

This entry does **not** claim parser changes, extraction changes, pagination changes, journal-page render changes, or print-layout rewrites. It documents:
- stage workflow stabilization
- mobile / desktop action-flow cleanup
- tab visibility isolation
- review / print return-path fixes
- screen-preview containment alignment

**Verification:**
- paste -> `Continue to Review` appears without requiring a second tap
- upload -> `Continue to Review` appears after content is populated
- Review -> Print -> Back to Review returns to Review, not Form
- mobile sticky CTA visibility is driven by meaningful content state
- inactive Form / Review / Print / Saved panels do not remain in layout flow when another tab is active
- no screen-preview line bleed into the footer area

Short checklist:
- [ ] paste -> Continue to Review
- [ ] upload -> Continue to Review
- [ ] Review -> Print -> Back to Review
- [ ] mobile sticky CTA visibility
- [ ] no screen-preview line bleed into footer

**Notes:** This was a stabilization / cleanup pass after the parser and pagination work, not a parser-history rewrite. Parser, pagination, print rendering, and routing internals were not otherwise rewritten as part of this UI entry.

---

## DOCX ingestion layer — 2026-04-04

### Structured DOCX table parsing (Confirmed) — journal-style Word tables now load through structured extraction instead of flattened fallback

**Root Issue:** Real journal DOCX files were authored as actual Word tables/columns, not as flattened text:
- metadata table
- `Dream` / `Brainstorm` two-column table
- later paragraph-based `Possible Interpretation` / `Interpretation` content
- `Application` table

The old upload behavior could still fall back to flattened raw text parsing for this shape. That forced the parser to reconstruct ownership from flattened text, which is why `Dream` vs `Brainstorm` ownership was being misclassified in documents like `Deliverance for Jo 3.29.22.docx`. After structured matching was enabled, one more DOCX-ingestion bug remained: duplicated text was already being produced inside the structured extraction layer before anything reached parser/rendering.

**Why It Was Happening:** This was a DOCX ingestion / extraction failure, not a pagination or rendering failure, and not a general parser architecture failure.

The failure progressed in layers:
1. real Word table structure was not being preserved in document order, so the upload path could still behave like a flattened text document
2. structured table detection needed to recognize:
   - metadata tables
   - `Dream` / `Brainstorm` two-column tables
   - `Application` tables
3. DOCX table cells needed paragraph-aware header matching because heading text could live in the first non-empty paragraph of a cell, with body content after it
4. scalar header normalization needed to collapse DOCX duplication artifacts such as:
   - `DreamDream`
   - `BrainstormBrainstorm`
   - `ApplicationApplication`
5. the upload/data-flow path needed to treat structured DOCX as primary and bypass `extractDreamData(...)` entirely when structured extraction succeeded
6. the final remaining duplication bug was at the lowest structured-text layer: `collectWordNodeText(...)` appended text twice when walking Word XML

The first bad function for the final duplication bug was `collectWordNodeText(...)`. When the walker hit `<w:t>`, it pushed `node.textContent`, but then still recursed into that node's child text nodes. That caused the same text to be emitted twice, which is why structured extraction produced symptoms like:
- `DreamDream`
- `BrainstormBrainstorm`
- `Scene 1Scene 1`
- `ApplicationApplication`
- `March 29, 2022Date: March 29, 2022`
- duplicated narrative and brainstorm phrases in Dream Data

**Structural Fix:** `index-v2.html` now treats structured DOCX extraction as the primary ingestion path for journal-style Word files and uses flattened raw text as fallback only.

The progression of the completed fix was:
1. ordered DOCX body block extraction
2. structured table detection for:
   - metadata table
   - `Dream` / `Brainstorm` table
   - `Application` table
3. paragraph-aware header matching
4. scalar header normalization
5. structured DOCX upload-path bypass of `extractDreamData(...)`
6. low-level structured text dedup fix for paragraph/cell extraction

This means the table-based DOCX parser now supports:
- metadata tables
- two-column `Dream` / `Brainstorm` tables
- single-section `Application` tables
- later paragraph-based sections like `Possible Interpretation` / `Interpretation`
- structured DOCX as primary, flattened raw text as fallback only

**Implementation:** Completed in [`index-v2.html`](/Users/Sydharmony/Desktop/Dove%20Expressions/GitHub/dreamjournaltemplategenerator/index-v2.html).

The DOCX ingestion layer was updated to:
- extract ordered body blocks from `word/document.xml` so tables and paragraphs remain in document order
- analyze table cells by first meaningful paragraph, not by whole-cell flattened text
- classify tables from scalar normalized header strings
- strip matched header paragraphs from table cells while preserving the remaining body text
- materialize structured DOCX entries directly into dream fields
- short-circuit upload parsing when `__djDocxStructured: true`

The low-level successful fix was in `collectWordNodeText(...)`:
- after handling `tab`
- after handling `br`
- after handling `cr`
- after handling `t`
- after handling `instrText`

the function now returns immediately instead of recursing further into already-consumed Word XML text nodes. That change isolates the fix to the DOCX structured extraction layer and prevents paragraph/cell text from being emitted twice.

**Scope:** DOCX ingestion / extraction path in `index-v2.html` only.

This completed fix does **not** claim a parser architecture rewrite and does **not** claim a rendering/pagination fix. The intent was narrowly:
- preserve real Word table structure during extraction
- route structured DOCX fields directly when available
- eliminate duplicated text at XML collection time

**Verification / Before vs After:**

Before:
- structured DOCX could fall back to flattened raw text parsing
- parser could incorrectly re-enter `labeled_sections` flow when structured extraction should have won
- structured header scalars could look like `dreamdream`
- paragraph starts could look like `Scene 1Scene 1`
- metadata could look like `March 29, 2022Date: March 29, 2022`
- dream narrative and brainstorm bodies could contain repeated phrases

After:
- `__djDocxStructured: true`
- `structuredExtractionSucceeded: true`
- table `0` logs `matchedAsMetadata: true`
- table `1` logs `matchedAsDreamBrainstorm: true`
- table `2` logs `matchedAsApplication: true`
- upload logs show `bypassing extractDreamData`
- `extractDreamData entered` should **not** appear for the successful structured DOCX route
- no duplicated header scalars like `dreamdream`
- no duplicated paragraph starts like `Scene 1Scene 1`
- no duplicated metadata like `March 29, 2022Date: March 29, 2022`
- no duplicated narrative / brainstorm text in Dream Data

**What Was Intentionally Left Untouched:**
- parser logic
- `extractDreamData(...)`
- pagination
- rendering
- print layout
- CSS

**Notes:** This is a completed DOCX ingestion-layer fix for journal-style Word uploads. It should be understood as a successful structured extraction/data-flow correction in `index-v2.html`, not as a downstream parser, pagination, or rendering change.

## Documentation note — 2026-04-04

### Status correction (Not a fix) — recent April 4 investigation passes are not confirmed resolutions

**Issue:** Recent work around `Notary Authority`, `Deliverance for Jo`, and related parser/render regressions was described in a way that could read like those regressions were fixed.

**Root Cause:** The running investigation included narrow code passes and tracing work, but those passes were not accepted as confirmed end-to-end resolutions. Updating the docs without an explicit disclaimer would overstate the current state of the project.

**Fix:** Documentation only:
- added this changelog note to state clearly that the recent April 4 passes should **not** be treated as confirmed fixes
- recorded that no parser/render regression should be considered resolved from those notes alone
- left the earlier entries in place as historical implementation notes, not proof of successful outcome

**Scope:** Documentation only. This entry does **not** claim that parser, rendering, pagination, symbol extraction, repeated-mode handling, or upload-path regressions are fixed.

**Verification:** Current project status should be treated as: the April 4 investigation did **not** establish that anything was fully fixed. Re-test the actual regression files and screenshots before promoting any parser or rendering change as resolved.

**Notes:** This is an explicit record that the recent investigation entries were not sufficient evidence of a working fix.

---

## Parser layer — 2026-04-03

### Promotion (Confirmed) — Live entrypoint promotion to `index.html`

**Issue:** Production was still serving `index.html` while the active parser and rendering fixes were living in `index-v2.html`.

**Root Cause:** Deployment was updating the repository, but the live app entrypoint still pointed to the older `index.html` file instead of the tested `index-v2.html` build.

**Fix:**
- created `index-old.html` as a backup of the previous live entrypoint
- copied/promoted the full contents of `index-v2.html` into `index.html`
- retained `index-v2.html` in place for comparison and rollback

**Scope:** Entrypoint promotion only. No parser, rendering, pagination, CSS, auth, or print logic was changed during this step.

**Verification:** Confirmed that `index.html` now matches `index-v2.html` exactly, so the production root entrypoint reflects the tested parser and rendering fixes.

**Notes:** `index-v2.html` was intentionally kept temporarily so the promoted live file can still be compared against the staged candidate or rolled back quickly if needed.

### Parser (Confirmed) — Pasted dream parser stabilization (reflection-note fix)

**Issue:** In pasted dreams with an explicit `Brainstorming` heading, pre-Brainstorming dream-scene prose was still being diverted into `dreamAdditionalNotes` instead of staying in `dreamNarrative`.

**Root Cause:** The first bad stage was `extractDreamData(...)` in `unlabeled_or_template` mode. Inside the unlabeled parse loop, the `reflection-note` branch was still allowed to fire after `dreamNarrative` had already started:
- `isDreamReflectionLine(line)` matched lines like `I think they did it for me first...`
- `curField` switched from `dreamNarrative` to `dreamAdditionalNotes`
- later pre-Brainstorming scene paragraphs followed into Additional Notes

**Fix:**
- disabled the `reflection-note` diversion for pasted entries when a literal `Brainstorming` boundary is present
- kept the rest of the unlabeled parse loop unchanged so the parser still switches at the explicit `Brainstorming` heading

**Scope:** Parser layer in `index-v2.html` only — `extractDreamData(...)` unlabeled parse loop. No changes to title extraction, first-paragraph preservation, entry splitting, pagination, rendering, lined pages, or CSS.

**Verification:** Confirm that for pasted input with a literal `Brainstorming` heading:
- all pre-Brainstorming dream-scene prose remains in `dreamNarrative`
- `dreamAdditionalNotes` stays empty in this explicit-boundary path
- `dreamBrainstorm` begins only at the literal `Brainstorming` heading

**Notes:** This is a pasted-only guard for the explicit-boundary case. Uploaded parsing and non-pasted unlabeled behavior were intentionally left alone.

### Parser (Confirmed) — Mobile generate crash: `sectionTransitions` undefined

**Issue:** Mobile upload and pasted generation could fail with `Can't find variable: sectionTransitions`, blocking review generation.

**Root Cause:** `sectionTransitions` was declared only inside the `unlabeled_or_template` branch of `extractDreamData(...)`, but later shared-tail debug/reporting code referenced it outside that block. `laterSectionFallback` had the same scope problem.

**Fix:**
- lifted `sectionTransitions` to top scope inside `extractDreamData(...)`
- lifted `laterSectionFallback` to the same shared scope
- removed the inner shadowing assignment so all branches use the same shared state

**Scope:** `extractDreamData(...)` scope/initialization only. No parser classification, pagination, rendering, lined-page, or CSS changes.

**Verification:** Confirm mobile paste and mobile upload both generate successfully without the `Can't find variable: sectionTransitions` runtime error.

**Notes:** The same scope repair was also mirrored into `index-v2.html` so the staged backup file stays aligned with the live entrypoint.

### Parser (Labeled Uploads) — Recover leading narrative from single-block Brainstorm

**Issue:** Some labeled uploads, especially DOCX-derived ones, could place later dream-scene prose at the top of `dreamBrainstorm` because the Brainstorm body arrived as one newline-joined block.

**Root Cause:** `extractLeadingNarrativeRunFromBrainstorm(...)` relied on blank-line paragraph splitting. When only one block was produced, mixed narrative + reflective content was scored as a single unit, preventing correct recovery of the leading narrative run.

**Fix:** Added a single-block line-level fallback in `extractLeadingNarrativeRunFromBrainstorm(...)` that scans from the top, accumulates scene/narrative lines, and stops when true brainstorm/reflective content begins.

**Scope:** Labeled Brainstorm cleanup only. Multi-block behavior unchanged. No pasted parsing, pagination, rendering, lined-page, or CSS changes.

**Verification:** Leading dream-scene content now moves back to `dreamNarrative` while the trailing reflective paragraph remains in `dreamBrainstorm`.

### Parser (Confirmed)

**Issue:** Parser behavior was inconsistent across entry-splitting paths, and `labeled_sections` documents could mis-route content:
- pasted/imported raw text could be split for generation via `splitTextEntries(...)` in one branch and via `parseDreamEntriesFromText()` → `parseEntries()` in another, so the same input shape could be divided differently depending on branch
- in `labeled_sections`, `dreamNarrative` stayed open until the next recognized section heading, so reflective/interpretive transition lines could remain in narrative instead of moving to interpretation
- Brainstorm text could mix internal sub-section material (e.g. Interpretation, Application, Symbols, Scriptures) without explicit inline boundaries being applied before heuristic cleanup

**Root Cause:** Generation’s paste path used a second splitter (`splitTextEntries`) alongside the canonical `parseDreamEntriesFromText` pipeline. The labeled-section accumulation loop keyed transitions primarily on `detectHeading` / `detectSectionFromHeading`, without a narrative-specific guard for strong reflection/interpretation lead lines. Hybrid Brainstorm blocks were not always split on inline section labels before `cleanupLabeledSectionsBuckets(...)` scoring and rebalancing.

**Fix:**
- `handleGenerate()`: raw pasted text entry splitting goes through `parseDreamEntriesFromText()` only; removed use of `splitTextEntries(...)` and deleted the dead `splitTextEntries` helper
- `buildLabeledSectionsResult()` labeled loop: when `currentField === 'dreamNarrative'`, flush narrative and switch to `dreamInterpretation` on explicit reflection lead (`after reading this, i think…`), `isDreamReflectionLine(...)`, or `isInterpretationLikeLine(...)`
- `cleanupLabeledSectionsBuckets(...)` (Brainstorm paragraph pass): `splitHybridBrainstormBlock(...)` splits paragraphs that contain internal heading lines for Interpretation, Application, Symbols, or Scriptures and routes segments to those fields before remaining brainstorm cleanup
- Documented ordering: explicit boundaries and inline splits first; `cleanupLabeledSectionsBuckets(...)` and shared tail remain the secondary safety net

**Scope:** Parser layer in `index-v2.html` only — `handleGenerate()`, `buildLabeledSectionsResult()` (including `cleanupLabeledSectionsBuckets` and local `splitHybridBrainstormBlock`), removal of `splitTextEntries`. No pagination, rendering, preview scaling, UI structure, or OCR changes.

**Verification:** Generate from paste with multiple entries separated by `---` and from paste with multiple dated entries; confirm batch paths call `parseDreamEntriesFromText()` without `splitTextEntries`. For `labeled_sections` samples, confirm narrative no longer absorbs lines that match the narrative boundary guard, and Brainstorm paragraphs with inline Interpretation/Application/Symbols/Scriptures headings contribute to the expected fields after parse.

**Notes:** `labeled_sections` input is treated as potentially hybrid internally (not assumed purely labeled end-to-end). Heuristic cleanup after primary accumulation is not the sole section-boundary mechanism.

---

## index-v2-parser-fix-11 — 2026-04-02

### Fix 17 — Brainstorm residue recovery, scripture remainder split, sparse lined-page containment, and mobile parser debug crash fix

**Issue:** Real output still showed four live problems:
- brainstorm-like commentary was being stranded in `dreamNarrative`, leaving the Brainstorm page thin or blank
- mixed scripture/commentary blocks were being routed wholesale into `dreamScriptures`
- sparse lined/dotted writing areas could visually run into the footer / next page in preview
- mobile uploads could throw `Can't find variable: debugPageDump`

**Root Cause:** The first failing stage was `cleanupLabeledSectionsBuckets(...)` in `labeled_sections` mode. Raw labeled bucketing was already producing fields, but cleanup still handled mixed narrative/commentary/scripture blocks too coarsely:
- brainstorm/commentary scorers were too narrow for symbolic reflective notes
- scripture routing moved whole blocks instead of splitting verse lines from commentary remainder
- screen preview let sparse writing-region overflow/background paint past the intended body box
- `debugPageDump` was defined only inside `unlabeled_or_template` but referenced later from parser-debug code shared by all parse modes

**Fix:**
- widened `isLikelyBrainstormLine(...)` just enough to catch reflective commentary such as `In the situation with ...`, continuation-commentary lines, and glossary-style explanatory notes without changing parser modes
- widened `looksLikeQuotedCommentaryLine(...)` inside `buildLabeledSectionsResult()` so quoted/reflective commentary is pushed toward `dreamBrainstorm`
- added `extractScriptureAndRemainder(...)` inside `buildLabeledSectionsResult()` and replaced whole-block scripture routing in labeled cleanup so only true scripture/reference lines move to `dreamScriptures`, while the remainder continues through brainstorm / interpretation / application scoring
- lifted `debugPageDump` into parser scope so parser debug output cannot crash on mobile for non-unlabeled parse modes
- added a minimal screen-only containment rule for sparse lined/dotted writing sections so backgrounds stay inside the journal body and stop before the footer area

**Scope:** `index-v2.html` only, plus build marker bump to `index-v2-parser-fix-11`. No UI, generation, pagination, preview-scaling, auth/save-load, or print-layout rewrite.

**Verification target:**
- `Change of Menu Dream`:
  - `dreamNarrative` = scene prose only
  - `dreamBrainstorm` = reflective / symbolic commentary
  - `dreamSymbols` = glossary-style entries like `Hebrew`, `Restaurant`, `Cashier`, `Greet`, `IRON`, `ARM`
  - `dreamScriptures` = scripture/reference lines only
  - `dreamInterpretation` / `dreamApplication` remain populated
- mobile DOCX/template upload no longer throws `debugPageDump` error
- sparse Brainstorm page no longer shows ruled lines running into footer / next page header in preview

**Notes:** This pass stays focused on `labeled_sections` cleanup. `repeated_simple_sections` and `unlabeled_or_template` were not directly rewritten; shared helper widening was kept minimal and regression-safe.

## index-v2-parser-fix-10 — 2026-04-02

### Fix 16 — Core vocabulary detection updated to match Sydni's prophetic writing style

**Issue:** The parser kept routing interpretation content into the wrong fields — most visibly, `dreamInterpretation` was empty or minimal in documents where Sydni's interpretation was clearly present. Every previous fix had been a routing or cleanup patch. None had ever touched the root vocabulary functions. The result was a recurring cycle: fix one symptom, another broke the next week.

**Root Cause (vocabulary mismatch — three choke points):**

All three vocabulary/scoring locations shared the same foundational flaw: they only recognized academic analytical phrasing ("may represent", "symbolizes", "this dream means") and never matched Sydni's prophetic declaration style. Her interpretation content looks like:

> "God is getting ready to move me from the back to the front..."
> "God wants me to know that I am special to Him..."
> "I believe the dream is showing how the church of God has become a world stage..."
> "He is guiding me and he will never leave me nor forsake me."

The three affected choke points were:

1. **`isInterpretationLikeLine` (global, ~line 2495)** — the core line-level classifier. Called by `scoreInterpretationBlock`, `cleanupLabeledSectionsBuckets`, and every mode that scores sections. If this misses a line, every downstream consumer misses it too.

2. **`scoreInterpretationBlock` (local inside `buildLabeledSectionsResult`, ~line 3994)** — uses `isInterpretationLikeLine` per line, then runs a second regex pass on the block as a whole. Both passes used only academic vocabulary. Prophetic interpretation blocks scored near zero and were outscored by narrative blocks during bucket competition.

3. **`interpretationLooksAnalytical` (inside `extractDreamData`, ~line 5544)** — the most destructive single failure point. This one-line regex is the only guard protecting dreamInterpretation from being wiped. The swap logic says: if dreamNarrative is short AND dreamInterpretation doesn't look analytical, assume dreamInterpretation was actually the dream narrative and move it there — emptying dreamInterpretation completely. Because the regex only checked `may represent|this dream means|could mean|symbolizes|represents?`, Sydni's interpretation consistently failed the test, triggering the swap and leaving `dreamInterpretation = ''`.

These three failures are not separate bugs. They are the same root problem — vocabulary mismatch — surfacing at three different points in the parsing pipeline. Each one compounded the others.

**Fix:** Added Sydni's prophetic declaration patterns to all three locations:

- `god (is|was|has|wants?|will)` — covers "God is getting ready", "God has orchestrated", "God will..."
- `the lord (is|was|has|wants?|will)` — covers "The Lord is saying", "The Lord has shown"
- `i believe (the dream|god|this)` — covers "I believe the dream is showing..."
- `the dream is show(ing|s)` — direct interpretation declaration
- `god is getting ready` — exact phrase from Change of Menu Dream DOCX
- `god wants? (me|us|you) to` — exact phrase from Change of Menu Dream DOCX
- `he (is|has|will) (orchestrat|guid|call|mov|speak|show|reveal)` — covers "He is guiding me", "He will orchestrate"
- `this (is|was) (a)? (call|season|time|message|word)` — prophetic framing language
- `(lord|holy spirit) (is|was|has) (show|reveal|call|speak|saying)` — Holy Spirit references

**Scope:**
- `isInterpretationLikeLine` — global function, affects all parser modes
- `scoreInterpretationBlock` — local to `buildLabeledSectionsResult`, affects `labeled_sections` only
- `interpretationLooksAnalytical` — inside `extractDreamData`, affects `labeled_sections` only
- Build markers bumped to `parser-fix-10` (both `window.__DJ_BUILD` and `window.__djBuild`)

**Verification target (Change of Menu Dream DOCX):**
- `dreamInterpretation` contains the full "God is getting ready to move me..." block — not swapped to narrative, not empty
- `scoreInterpretationBlock` scores the Long Interpretation cell significantly higher than before (prophetic phrases now add 3–4 points each)
- `interpretationLooksNarrative && !interpretationLooksAnalytical` evaluates false for Sydni's interpretation text — swap does not trigger
- `dreamNarrative` contains only the actual dream scenes (restaurant, conference, theater)

**Notes:** This fix is the first time core vocabulary has been updated in any version of this parser. All prior fixes were routing or cleanup patches applied downstream of these functions. Going forward, when new document samples fail, check `isInterpretationLikeLine` first before building new routing logic.

---

## index-v2-parser-fix-09 — 2026-04-02

### Fix 15 — `labeled_sections` narrative cleanup now removes glossary / scripture / commentary / interpretation residue from `dreamNarrative`

**Issue:** Even after paired-column reconstruction, `dreamNarrative` in `labeled_sections` docs could still contain non-scene material:
- glossary-style symbol lines
- quoted reference/commentary
- scripture blocks
- interpretation prose
- application fragments

That made downstream symbol extraction work on dirty narrative text and left too much non-narrative content in the Dream section.

**Root Cause:** `buildLabeledSectionsResult()` was reconstructing the initial section buckets correctly enough to separate the first-level column ownership, but it did not perform a second narrative-focused cleanup pass on the labeled buckets before the shared parser tail. As a result, mixed content that landed in `dreamNarrative` stayed there until later helpers tried to infer symbols or interpretation from an already-contaminated narrative field.

**Fix:** Added a second-pass cleanup inside `buildLabeledSectionsResult()` through `cleanupLabeledSectionsBuckets(...)` that:
- strips glossary-style symbol lines from `dreamNarrative`, `dreamBrainstorm`, and `dreamInterpretation`
- keeps quoted commentary such as “Blue can represent God...” out of `dreamSymbols`
- moves quoted/commentary-style non-scene material from `dreamNarrative` into `dreamBrainstorm`
- moves scripture-like blocks from `dreamNarrative` / brainstorm / interpretation into `dreamScriptures`
- moves interpretation-heavy prose out of `dreamNarrative` into `dreamInterpretation`
- moves application-like response text into `dreamApplication`
- leaves only dream-scene prose in `dreamNarrative`

New local helpers added inside `buildLabeledSectionsResult()`:
- `looksLikeQuotedCommentaryLine(...)`
- `extractGlossaryLikeLinesFromBlock(...)`
- `looksLikeScriptureBlock(...)`
- `scoreNarrativeResidue(...)`

**Scope:** `buildLabeledSectionsResult()` only, build marker bumped to `index-v2-parser-fix-09`.

**Verification target (Change of Menu Dream):**
- `dreamNarrative` = dream scenes only
- `dreamSymbols` = glossary-style entries like `IRON`, `Restaurant`, `Cashier`, `Greet`
- quoted “Blue can represent God ...” commentary stays in brainstorm / notes, not symbols
- `dreamInterpretation` = real interpretation block
- `dreamApplication` = application block if present
- no glossary/scripture/interpretation clutter remains in `dreamNarrative`

**Notes:** This pass is still limited to `labeled_sections`. It does not change `repeated_simple_sections`, `unlabeled_or_template`, pagination, rendering, preview, or print.

## index-v2-parser-fix-08 — 2026-04-02

### Fix 14 — `labeled_sections` bucket rebalancing now moves mixed brainstorm / interpretation / application content after DOCX column flattening

**Issue:** Browser debug output for flattened two-column DOCX uploads showed that `labeled_sections` buckets were still contaminated after parser-fix-07:
- `dreamNarrative` still held brainstorm glossary + interpretation prose
- `dreamBrainstorm` still duplicated dream prose and long interpretation
- `dreamInterpretation` was underfilled
- `dreamApplication` could remain empty even when an Application heading existed

**Root Cause:** Parser-fix-07 repaired who owned the first body after consecutive headings, but it did not fully rebalance the already-bucketed content after Mammoth’s row-flattened output mixed scene prose, glossary lines, interpretation, and application prose into adjacent labeled buckets.

**Fix:** Added `cleanupLabeledSectionsBuckets(...)` inside `buildLabeledSectionsResult()` to perform post-bucketing rebalance before the shared parser tail:
- remove duplicated narrative scene prose from `dreamBrainstorm`
- move interpretation-heavy brainstorm content into `dreamInterpretation`
- recover application-like prose into `dreamApplication`
- let glossary-style lines move toward `dreamSymbols`
- preserve the existing shared parser tail so `extractListedSymbolEntries(...)`, `prioritizeListedSymbols(...)`, and `extractSymbolEntriesFromText(...)` still finalize symbols once the labeled buckets are cleaner

Support helpers added in the same local scope:
- `normalizeCompareText(...)`
- `joinNonEmptyChunks(...)`
- `hasFieldHeading(...)`
- `isDuplicateNarrativeChunk(...)`

**Scope:** `buildLabeledSectionsResult()` only, build marker bumped to `index-v2-parser-fix-08`.

**Verification target (Change of Menu Dream):**
- `dreamNarrative` contains dream scene prose only
- `dreamBrainstorm` contains brainstorm analysis only
- `dreamInterpretation` contains the full long-interpretation block
- `dreamApplication` is recovered if application exists
- `dreamSymbols` is available for the shared parser tail to populate once glossary lines are no longer trapped in dirty brainstorm content

**Notes:** This was a rebalance pass, not a new parser mode. `repeated_simple_sections` and `unlabeled_or_template` remain unchanged.

## index-v2-parser-fix-07 — 2026-04-02

### Fix 13 — `buildLabeledSectionsResult` reconstructs paired column headers before shared parser cleanup

**Issue:** In `labeled_sections` mode, Mammoth can flatten a two-column table into a consecutive-heading stream such as:

```text
Dream
Brainstorm
[narrative body]
[brainstorm body]

Long Interpretation
Application
[interpretation body]
[application body]
```

The existing labeled-section parser assumed the most recent heading owned all following content until the next heading. In this flattened shape, the second heading in each pair absorbed both column bodies. Result:
- `dreamBrainstorm` contained dream narrative prose
- `dreamApplication` could absorb interpretation prose
- symbol glossary lines stayed trapped inside brainstorm instead of reaching `dreamSymbols`

**Root Cause:** `buildLabeledSectionsResult()` handled headings as a single active cursor (`currentField`). When two headings appeared back-to-back with no meaningful body content between them, the first heading was flushed empty and the second heading became the owner of the entire combined body region.

**Fix:** Added paired-column reconstruction directly inside `buildLabeledSectionsResult()`:

- detect consecutive heading pairs with no meaningful body content between them
- collect the combined body region that follows the pair
- split that region into field A / field B bodies
- append the first body to heading A and the second body to heading B
- then continue normal labeled-section parsing for all non-paired headings

New local helpers inside `buildLabeledSectionsResult()`:
- `getHeadingRecord(...)`
- `findNextMeaningfulRecordIndex(...)`
- `collectTextUntilNextHeading(...)`
- `splitPairedColumnBodies(...)`

Pair splitting behavior:
- `dreamNarrative` + `dreamBrainstorm` uses narrative-vs-brainstorm scoring and falls back to `splitNarrativeLikeContentFromInterpretation(...)` when helpful
- `dreamInterpretation` + `dreamApplication` uses interpretation-vs-application scoring
- generic paragraph-block fallback is used only if no better split is found

**Scope:** `buildLabeledSectionsResult()` only, plus build marker bump to `index-v2-parser-fix-07`.

**Verification target (Change of Menu Dream):**
- `dreamNarrative` contains dream scene prose only
- `dreamBrainstorm` contains brainstorm content only
- `dreamInterpretation` contains interpretation only
- `dreamApplication` contains application only
- shared parser tail then extracts glossary-style lines from cleaned brainstorm into `dreamSymbols`
- no narrative duplication remains inside brainstorm

**Notes:** This fix does not touch `repeated_simple_sections`, `unlabeled_or_template`, generation, pagination, rendering, print, or preview code. It only repairs Mammoth’s consecutive-heading table flattening inside labeled-sections mode.

## index-v2-parser-fix-06 — 2026-04-02

### Fix 12 — Shared parser post-processing now runs for all parse modes

**Issue:** `repeated_simple_sections` and `labeled_sections` documents could parse their primary sections correctly, but symbol/scripture/emotion/application enrichment only reliably ran for `unlabeled_or_template` documents. In practice this left fields like `dreamSymbols` empty for repeated/labeled docs even when the helper logic already knew how to recover them from brainstorm or interpretation text.

**Root Cause:** `extractDreamData` previously exited too early once `buildRepeatedBlocksResult()` or `buildLabeledSectionsResult()` produced a mode-specific object. That early exit bypassed the shared post-processing pipeline later in `extractDreamData`:
- `extractNarrativeSymbolGlossary(...)`
- `extractListedSymbolEntries(...)`
- `prioritizeListedSymbols(...)`
- `extractSymbolEntriesFromText(...)`
- scripture / emotion / application cleanup that already runs near the final return

The mode-specific builders were correct, but they were terminating the parser before the shared cleanup layer could run.

**Fix:** Mode-specific builders no longer short-circuit `extractDreamData`. Instead:

```javascript
switch (docMode) {
  case 'repeated_simple_sections':
    Object.assign(result, buildRepeatedBlocksResult());
    break;
  case 'labeled_sections':
    Object.assign(result, buildLabeledSectionsResult());
    break;
  case 'unlabeled_or_template':
  default:
    break;
}
```

Then the existing shared post-processing continues to run once for **all** parse modes before the final return.

The unlabeled fallback main parse loop remains gated:

```javascript
if (docMode === 'unlabeled_or_template') {
  // fallback loop only
}
```

So repeated/labeled docs do **not** accidentally re-enter the unlabeled parser path.

**Scope:** `extractDreamData` only.
- Mode switch after `buildRepeatedBlocksResult()` / `buildLabeledSectionsResult()`
- Shared post-processing flow after the mode switch
- Build marker bumped to `index-v2-parser-fix-06`

**Verification:**
- Repeated-section docs now continue into shared symbol extraction instead of returning early.
- Labeled-section docs now continue into shared glossary/listed-symbol prioritization and scripture/emotion cleanup.
- `unlabeled_or_template` still uses the fallback parse loop exactly once.
- No pagination, rendering, preview, or print code changed.

**Notes:** This is a parser-layer flow fix, not a helper rewrite. The symbol/application/scripture/emotion helpers were already correct; they simply were not being reached for all document modes.

## index-v2-parser-fix-04 — 2026-04-02

### Fix 11 — `buildRepeatedBlocksResult`: recover narrative from empty dreamNarrative caused by consecutive column-header headings

**Issue:** Uploading a table-based docx (e.g., "Change of Menu Dream") produced a journal where ALL content — narrative, brainstorm, and long interpretation — appeared in the narrative section. Brainstorm and Long Interpretation sections were empty.

**Root Cause (full chain):**

1. The document uses a two-column table: "Dream" (left header) / "Brainstorm" (right header) in row 0, then narrative prose (left cell) / symbol analysis (right cell) in row 1.
2. Mammoth `extractRawText` flattens table cells row-by-row, left-to-right. Output: `"Dream\nBrainstorm\n[all left-column narrative]\n[all right-column brainstorm]"`.
3. In `detectRepeatedSimpleSectionBlocks`, "Dream" is immediately followed by "Brainstorm" with NO content between them. The "Dream" block is flushed as `{heading: 'dreamNarrative', text: ''}` (empty). The "Brainstorm" block accumulates ALL subsequent content (both narrative and brainstorm text).
4. `meaningfulBlocks` = 3 (dreamBrainstorm, dreamInterpretation, dreamApplication). `hasMultipleSectionTypes = true` → trigger fires → `repeatedBlocksDetected = true` → `docMode = repeated_simple_sections`.
5. `buildRepeatedBlocksResult` returns `dreamNarrative = ''`, `dreamBrainstorm = '[entire narrative + brainstorm content]'`.
6. `populateFormFromDreamData(..., { fallbackNarrativeToRaw: true })` sees `dreamNarrative = ''` → falls back to `data._raw` = **entire raw document text** → dumps into `dreamNarrative` form field.
7. User clicks Generate → `collectCurrentDreamFormData()` reads full raw text from narrative field → all content renders in narrative section.

**Fix:** Added empty-narrative recovery at the top of `buildRepeatedBlocksResult`. When `dreamNarrative` is empty but `dreamBrainstorm` is non-empty, the function now calls `splitNarrativeLikeContentFromInterpretation(dreamBrainstorm)` to split the combined block using narrative vs. interpretive content signals. The narrative portion goes to `dreamNarrative`; the analytical/symbol portion stays in `dreamBrainstorm`.

```javascript
// Added before the splitNarrativeAndInterpretationLines call:
if (!repeatedResult.dreamNarrative && repeatedResult.dreamBrainstorm) {
  const brainstormSplit = splitNarrativeLikeContentFromInterpretation(repeatedResult.dreamBrainstorm);
  if (brainstormSplit.narrative) {
    repeatedResult.dreamNarrative = brainstormSplit.narrative;
    repeatedResult.dreamBrainstorm = brainstormSplit.interpretation || '';
  }
}
```

**Scope:** `buildRepeatedBlocksResult` inside `extractDreamData`. No changes to detection logic, normalization, or generation pipeline.

**Verification (Change of Menu Dream 2.16.21.docx):**
- Before: `dreamNarrative = [entire raw doc]`, `dreamBrainstorm = ''`, `dreamInterpretation = ''`
- After: `dreamNarrative = [restaurant/conference/vaccine scenes]`, `dreamBrainstorm = [Hebrew/symbol analysis]`, `dreamInterpretation = [Long Interpretation content]`, `dreamApplication = [Application content]`

**Notes:** The split fires only when `dreamNarrative` is empty after block accumulation — a condition that occurs exclusively when the document uses consecutive column-header headings with no prose between them. It is safe for all other document shapes: when dreamNarrative already has content, the branch is bypassed entirely.

---

## index-v2-parser-fix-03 — 2026-04-02

### Fix 10 — `countDreamDocumentSectionHeadings`: broadened to cover extended heading variants

**Issue:** Documents using compound or slash-form section headings ("Dream Narrative", "Long Interpretation", "Brainstorm/Initial Thoughts") produced a `detectedHeadingCount` below 2, so `labeledSectionsDetected` was never set to `true`. The parser fell through to `unlabeled_or_template` mode instead of `labeled_sections` mode, causing sections to be misrouted or lost.

**Root Cause:** `countDreamDocumentSectionHeadings` used a narrow regex:
```regex
/^(dream|dream narrative|vision|brainstorm(?:ing)?|interpretation|application|symbols|scriptures?)[:]?$/i
```
This matched only single bare-word labels or a short fixed list. It did not match:
- `"Dream Narrative"` (compound form)
- `"Long Interpretation"` (modifier prefix)
- `"Brainstorm/Initial Thoughts"` (slash-combo form)
- `"Reflect"` / `"Reflection and Apply"` (application aliases)

**Fix:** Replaced the narrow regex filter with a three-stage check loop:
1. **Primary:** `normalizeSimpleSectionHeading(line) !== null` — uses the already-fixed normalization function to match all known heading variants (including "Dream Narrative", "Long Interpretation", etc.)
2. **Slash-combo:** Strip `/…` suffix and retry `normalizeSimpleSectionHeading` — handles "Brainstorm/Initial Thoughts" → "Brainstorm"
3. **Application aliases:** Explicit regex for "Reflect", "Reflection and Apply", "How I Will Respond", "Response" — which map to `dreamApplication` but are not in `normalizeSimpleSectionHeading`

**Scope:** `countDreamDocumentSectionHeadings` only. No changes to parsing logic, normalization functions, or generation pipeline.

**Verification (T4 test case):**
Input with headings: "Dream Narrative", "Brainstorm/Initial Thoughts", "Long Interpretation", "Reflect"
- Before fix: `detectedHeadingCount: 0` → `docMode: unlabeled_or_template` → `interpLen: 0` ❌
- After fix: `detectedHeadingCount: 4` → `docMode: labeled_sections` → all sections correctly populated ✓

**Notes:** `countDreamDocumentSectionHeadings` is a gate function — it only determines *whether* labeled-sections mode should be activated. The actual section routing during parsing is done by `detectSectionFromHeading` / `HEADING_RULES` / `buildLabeledSectionsResult`, which already handled these variants. The bug was that the gate never opened for extended forms.

---

## index-v2-parser-fix-02 — 2026-04-02

### Fix 9 — Critical regex character range bug in `normalizeSectionHeading` and `normalizeSimpleSectionHeading`

**Issue:** All section headings normalized to an empty string `""`. `detectSectionFromHeading` returned `null` for every heading including "Interpretation", "Application", "Brainstorm", etc. Interpretation and Application content bled into the wrong section or was lost entirely.

**Root Cause:** Both `normalizeSectionHeading` (inner function of `extractDreamData`, ~line 4328) and `normalizeSimpleSectionHeading` (global function, ~line 2485) contained the regex:
```javascript
.replace(/[\s:;,.!?-–—]+$/g, '')
```
The unescaped hyphen `-` between `?` (U+003F) and `–` (U+2013) created a Unicode range U+002D–U+2013. This range includes all ASCII lowercase and uppercase letters (a-z, A-Z). The regex was stripping every letter from the end of every heading, leaving only an empty string.

Effect: `normalizeSectionHeading("interpretation")` → `""` → `detectSectionFromHeading` returned `null` for all inputs. `normalizeOcrTolerantHeadingKey` called `normalizeSectionHeading` → also always returned `""`. `isInterpretationHeading`, `isApplicationHeading`, `resolveInterpAppField` all returned false/null. The entire heading detection layer was silently broken.

**Fix:** Changed the character class in both locations to escape the hyphen:
```javascript
// Old (BUG):
.replace(/[\s:;,.!?-–—]+$/g, '')
// New (FIXED):
.replace(/[\s:;,.!?–—\-]+$/g, '')
```
Hyphen is now at the end of the character class where it is treated as a literal character, not as a range operator.

**Scope:** Two regex literals:
1. `normalizeSectionHeading` inner function inside `extractDreamData` (~line 4334)
2. `normalizeSimpleSectionHeading` global function (~line 2487)

**Verification:**
- `normalizeSimpleSectionHeading('Interpretation:')` → `'dreamInterpretation'` ✓
- `normalizeSimpleSectionHeading('Application')` → `'dreamApplication'` ✓
- `detectSectionFromHeading('Brainstorm')` → `'dreamBrainstorm'` ✓
- T1, T2, T3 parser tests all pass after fix ✓

**Notes:** This was the root cause of all section-detection failures. `isInterpretationHeading`, `isApplicationHeading`, and `resolveInterpAppField` all indirectly depended on `normalizeSectionHeading` returning a non-empty string. The bug silently disabled all of them simultaneously. Any future change to these character classes must verify the hyphen is escaped or placed at a boundary position.

---

## index-v2-parser-fix-01 — 2026-04-02

### Fix 8 — Parser structural fixes: block-based parsing, MAJOR_SECTION guard, noise filtering

**Issue (composite):** The parser had multiple structural failures for documents with more than one instance of a section heading, documents using compound or alternate heading labels, and documents with template boilerplate mixed into content sections:

1. **Repeated section headings not combined:** Documents with "Dream" → content → "Dream" (continued) had the second block overwrite the first rather than appending.
2. **MAJOR_SECTION_FIELDS guard too aggressive:** The guard `if (openingMajorField !== headingField) { continue; }` was skipping any heading that didn't match the first major section seen, even when real content followed. This caused Interpretation/Application to be silently skipped in many documents.
3. **Template noise in section content:** Lines like "[ Write your interpretation here ]" or sidebar decoration clusters were being copied into output fields.
4. **`shouldUseSectionBlocks` trigger too narrow:** Required `Dream` or `Brainstorm` to appear 2+ times. Documents with one of each plus Interpretation/Application were excluded.
5. **`detectRepeatedSimpleSectionBlocks` trigger too narrow:** Required `meaningfulBlocks.length >= 3 && repeatedCount >= 1`; excluded documents with only 2 meaningful blocks.
6. **`parseSectionBlocks` not filtering template noise:** The block accumulator didn't call `isTemplateNoiseLabel` or `isDecorativeSidebarNoiseCluster`, so noise lines were included in block content.

**Root Cause:** The parser was designed around a single-pass, single-occurrence model. Each heading was expected to appear exactly once, and later occurrences were treated as metadata rather than continuation. The MAJOR_SECTION guard was implemented as an unconditional skip rather than a content-aware check.

**Fix (5 changes):**

1. **`detectRepeatedSimpleSectionBlocks` trigger broadened:** Changed from requiring `repeatedCount >= 1 && meaningfulBlocks.length >= 3` to activating whenever `meaningfulBlocks.length >= 2` and either a heading appears more than once OR there are 3+ distinct heading types.

2. **`shouldUseSectionBlocks` broadened:** Now activates when `uniqueHeadings.size >= 2` (any 2 distinct section types with real content), replacing the stricter requirement for Dream or Brainstorm to repeat.

3. **`parseSectionBlocks` noise filtering added:** Block accumulation loop now calls `isTemplateNoiseLabel(line)`, `isStandaloneJunk(line)`, and `isDecorativeSidebarNoiseCluster(line)` to skip template placeholder lines before accumulating content.

4. **`parseSectionBlocks` uses `detectHeading`:** Replaced inline heading detection in block loop with `detectHeading(line, records, i, true, current.heading)` — the same detection path used by the main parsing loop, ensuring all heading variants and aliases are recognized.

5. **MAJOR_SECTION_FIELDS guard fixed:** Changed from unconditional `continue` to content-aware check:
   - Old: `if (openingMajorField !== headingField) { continue; }`
   - New: `if (openingMajorField !== headingField && !hasRealContentAhead(lineRecords, li)) { continue; }`
   Only skips the heading if there is genuinely no real content following it. If content follows, the heading is accepted and content is routed to the correct field.

**Scope:**
- `detectRepeatedSimpleSectionBlocks` — trigger condition
- `shouldUseSectionBlocks` — activation threshold
- `parseSectionBlocks` — noise filtering + heading detection path
- `extractDreamData` main loop — MAJOR_SECTION_FIELDS guard condition

**Verification (T1–T3 parser tests):**
- T1 (single entry, standard sections): `docMode: labeled_sections`, all fields populated ✓
- T2 (repeated "Dream" blocks): blocks merged, `narrativeLen > 0` ✓
- T3 (brainstorm/interpretation/application, alternate labels): all sections routed correctly ✓

**Notes:** These fixes treat the MAJOR_SECTION guard as a spam-filter for template noise, not a strict structural constraint. It only skips headings when no real content follows — which is the correct behavior for template placeholders like an empty "Interpretation:" heading in a blank journal page.

---

## index-v2-stabilization-pass-01 — 2026-04-02

### Fix 7 — Generation pipeline stabilization: two canonical paths, form as single source of truth

**Issue:** The generation pipeline had a blurred, circular data flow:
- `generateFromCurrentState()` collected from the form, then called `showPages([data])`, which re-populated the form from that same data and then re-collected again — a redundant round-trip.
- For batch entries (2+), `showPages()` populated the form from entry[0] and re-read it through the form, but entries[1+] bypassed the form entirely, going directly to `_showPagesCore`. This meant entry[0] was form-normalized while entries[1+] were not.
- Manual form edits followed by "Generate" worked, but the generated output could silently drift if the `showPages` re-population overrode user edits with stale parsed data.
- No clear separation existed between "generate from what the user typed" vs "generate from parsed file objects."

**Root Cause:** `showPages(dataArray)` did three things at once: (1) set `lastParsedEntryForGeneration`, (2) repopulated the form from entry[0], (3) recollected from the form and stitched `[firstGenerationEntry, ...sourceData.slice(1)]`. This made entry[0] form-normalized and entries[1+] raw — two different data shapes in the same generation call.

**Fix:** Introduced two canonical generation functions:

1. **`generateFromCurrentForm()`** — single-entry path. Calls `collectCurrentDreamFormData()` and passes directly to `_showPagesCore([data])`. Never re-populates the form. Form is the only source of truth.

2. **`generateFromEntryArray(entries)`** — batch path. Takes a normalized entry array, applies `finalizeDreamTitle` and type inference to each entry via shallow copy (never mutates source), and passes directly to `_showPagesCore(normalized)`. Never reads from or writes to the form.

3. **`generateFromCurrentState()`** reduced to a one-line alias: `generateFromCurrentForm()`. Kept for backward compatibility (called from `applyUploadFlowResultToUi`).

4. **`showPages(dataArray)`** rewritten as a clean backward-compat router:
   - Single entry → `populateFormFromDreamData(entries[0])` + `generateFromCurrentForm()`
   - Multiple entries → `generateFromEntryArray(entries)` (form not touched)

5. **`handleGenerate()`** updated: all batch paths (`pendingPages > 1`, dated entries, separator-split entries) now call `generateFromEntryArray(...)` directly. Single-entry paths call `generateFromCurrentForm()` directly. Removed all `showPages(...)` calls from this function.

6. **`generateFromForm()`** updated to call `generateFromCurrentForm()`.

7. **Debug logs added** at:
   - File load: `[DJ] Loaded build: index-v2-stabilization-pass-01`
   - `collectCurrentDreamFormData()`: logs field lengths on every collection
   - `generateFromCurrentForm()`: logs collected data shape + final title + page count after render
   - `generateFromEntryArray()`: logs each normalized entry + final count + page count after render
   - `handleGenerate()`: logs which path was chosen and entry counts

**Scope:**
- `handleGenerate` — replaced `showPages(...)` calls with `generateFromEntryArray` / `generateFromCurrentForm`
- `generateFromForm` — updated alias
- `generateFromCurrentState` — reduced to alias for `generateFromCurrentForm`
- `generateFromCurrentForm` — new canonical single-entry function
- `generateFromEntryArray` — new canonical batch function
- `showPages` — rewritten as clean router
- `collectCurrentDreamFormData` — debug log added
- Build marker — updated to `index-v2-stabilization-pass-01`

**Verification (all confirmed in live preview):**
1. Single entry: `generateFromCurrentForm()` → 6 pages generated ✓
2. Manual form edit → `generateFromCurrentForm()` → preview reflects edit (narrative text confirmed in DOM) ✓
3. Batch (2 entries): `generateFromEntryArray([e1, e2])` → 13 pages generated ✓
4. Batch does NOT mutate form: form title unchanged after `generateFromEntryArray` call ✓
5. `window.__djBuild === 'index-v2-stabilization-pass-01'` confirmed in browser ✓

**Notes:**
- `showPages` is kept as a backward-compat router but no longer called from `handleGenerate`. Any new call sites should use `generateFromCurrentForm()` or `generateFromEntryArray()` directly.
- The `populateFormFromDreamData(entries[0])` call in batch paths inside `handleGenerate` is intentional — it updates the UI so the user can inspect the first entry in the form. It does NOT affect batch generation, which reads from the normalized entries array only.
- `generateFromCurrentState()` alias must be kept — it is called from `applyUploadFlowResultToUi` via the `shouldGenerate` flag on upload results.

---

## v2-pagegenfix-2026-04-02

### Fix 5 — Measurement context `availableHeight` floor: 0 → 60

**Issue:** All four measurement context functions (`createNarrativeMeasurementContext`, `createBrainstormMeasurementContext`, `createInterpretationMeasurementContext`, `createApplicationMeasurementContext`) used `Math.max(..., 0)` as the minimum floor for `availableHeight`. When the hidden measurement sheet rendered off-screen or the DOM was not fully painted, `getBoundingClientRect()` could return zeroes, producing `availableHeight = 0`. Any content passed to `paginateTokenStream` with `maxH = 0` triggered the `throw new Error('Overflow - invalid pagination')` guard.

**Root Cause:** `availableHeight = 0` causes the binary-search loop in `paginateTokenStream` to never satisfy `testH <= maxH` (since even a single word exceeds 0px). After the loop, `finalHeight > maxH` → the throw fires. This exception propagated through `document.fonts.ready.then()` as an unhandled Promise rejection, silently aborting the generation pipeline after the header page had already been appended.

**Fix:**
- `createNarrativeMeasurementContext` (line ~5944): Changed `Math.max(0, Math.floor(...) - SAFETY_PX)` → `Math.max(60, ...)`
- `createBrainstormMeasurementContext` (lines ~6006–6013): Both `availableHeight` and `finalMaxHeight` now use `Math.max(..., 60)`
- `createInterpretationMeasurementContext` (lines ~6072–6076): Both `availableHeight` and `finalMaxHeight` now use `Math.max(..., 60)`
- `createApplicationMeasurementContext` (lines ~6135–6139): Both `availableHeight` and `finalMaxHeight` now use `Math.max(..., 60)`

**Scope:** Four measurement context factory functions. No pagination logic changed. No rendering logic changed.

**Verification:** Open browser dev tools → Console. Generate pages with content in all four sections. Verify page count > 1. Verify no `Overflow - invalid pagination` error in console.

**Notes:** 60px matches the minimum used in `getLayoutConfig()`. It is enough height for at least one line of text at any reasonable font size. If `getBoundingClientRect` returns zeroes because the output section is still `display:none`, `_showPagesCore` calls `sec.classList.add('show')` and `void container.offsetHeight` to force a reflow before measurement — this floor is a secondary safety net for edge cases where that reflow is incomplete.

---

### Fix 6 — `_showPagesCore`: defensive try/catch around `planPages`

**Issue:** Page generation showed only the header/title page. No error surfaced in the UI. Console showed an unhandled Promise rejection: `Overflow - invalid pagination`.

**Root Cause:** `_showPagesCore` calls `addPage(buildHeaderPage(data))` first, then calls `planPages(data, layout, measure)` synchronously inside a `document.fonts.ready.then()` chain. There was no try/catch around `planPages`. When `planPages` → `paginateTokenStream` threw, execution halted after the header page had already been appended. The exception propagated as an unhandled rejection — nothing showed in the UI, and only the header page remained visible.

**Fix:** Wrapped `planPages(data, layout, measure)` in try/catch inside `_showPagesCore` (lines ~5408–5418):
```javascript
let descriptors = [];
descriptors.interpretationDescriptors = [];
descriptors.applicationDescriptors = [];
try {
  descriptors = planPages(data, layout, measure);
} catch (planErr) {
  console.error('[Dream Journal] planPages threw — page generation fell back to header-only. Error:', planErr);
  console.error('[Dream Journal] data passed to planPages:', {
    narrativeLength: (data.dreamNarrative || '').length,
    ...
  });
}
```
The default empty `descriptors` object has `interpretationDescriptors` and `applicationDescriptors` pre-set to `[]` so the downstream `renderPageDescriptors(container, descriptors.interpretationDescriptors || [], ...)` calls remain safe.

**Scope:** `_showPagesCore` only. No changes to `planPages`, `paginateTokenStream`, or any rendering function.

**Verification:** If measurement fails for any reason, the console will now show a clear `[Dream Journal] planPages threw` error with content lengths. Pages won't silently disappear — the header page still renders, and the error is actionable.

**Notes:** This is a defensive catch-all. Fix 5 (floor at 60px) is the proactive fix that prevents the throw. This catch is belt-and-suspenders for any future throw path from `planPages`. Do NOT remove it — it converts silent failures into visible errors.

---

## v2-parsefix-2026-04-02

### Fix 1 — `extractTopDocumentTitle`: skip bare date lines

**Issue:** For "Change of Menu Dream 2.16.21.docx", the document body opened with "February 16, 2021" on the first substantive line. `extractTopDocumentTitle` returned `{title: 'February 16, 2021'}`, causing the dream title to be set to the date string.

**Root Cause:** `extractTopDocumentTitle` looped through the first 8 lines looking for a title candidate. `isDateHeading(line)` was called elsewhere in the codebase but was NOT called inside this loop. Bare date lines were not excluded.

**Fix:** Added `if (isDateHeading(line)) continue;` as the first check inside the loop in `extractTopDocumentTitle` (line ~3265).

**Scope:** `extractTopDocumentTitle` inner function only.

**Verification:** Upload "Change of Menu Dream 2.16.21.docx". Verify title field shows "Change of Menu Dream", not "February 16, 2021".

**Notes:** `isDateHeading` is an existing robust function that handles multiple date formats (Month DD, YYYY / MM/DD/YYYY / etc.).

---

### Fix 2 — `extractTopDocumentTitle`: stop scanning on bare "Dream"/"Vision"

**Issue:** When the document heading section contained a bare line "Dream" (a template section label), `extractTopDocumentTitle` was returning `{title: 'Dream'}`. While this was blocked downstream by `isTemplatePlaceholderValue`, it caused unnecessary processing and could fail to be overridden by `enrichFromFilename` in some edge cases.

**Root Cause:** The loop only returned early if `isRealDreamStart(line)` or `isSceneMarker(line)` matched. A lone "Dream" or "Vision" heading did not match either function — it returned as a valid title candidate.

**Fix:** Added explicit break at end of the `if (/^(?:dream|vision)\b/i.test(line) ...)` block: if line is exactly "Dream" or "Vision" (case-insensitive), `break` the scanning loop and return `null` (no title found in header).

**Scope:** `extractTopDocumentTitle` inner function only.

**Verification:** Upload doc with "Dream" as first non-noise line. Verify title falls back to filename-derived title (e.g. "Change of Menu Dream").

---

### Fix 3 — Curly apostrophe gap in metadata regex (3 locations)

**Issue:** "Dreamer's Involvement: Participant" (where the apostrophe is U+2019 — Word's curly right single quote) was not being filtered as a metadata line. It leaked into `dreamNarrative` via `prefaceLines`.

**Root Cause:** The regex `dreamer'?s?` only covered the straight apostrophe U+0027. Microsoft Word documents produce U+2019 (curly right single quote) in words like "Dreamer's". The character class `[''']` (U+0027 + U+2018 + U+2019) was needed.

**Fix:** Changed `dreamer'?s?` → `dreamer[''']?s?` in three locations:
1. `extractTopDocumentTitle` loop (~line 3270)
2. `cleanedPreface` filter (~line 3782)
3. Strategy 6 `looksLikeMetadata` check (~line 4007)

**Scope:** Three separate regex literals. No function logic changed.

**Verification:** Upload a Word doc containing "Dreamer's Involvement: Participant" (with curly apostrophe). Verify this line does NOT appear in the Dream Narrative textarea.

**Notes:** The `[''']` character class covers: U+0027 (straight), U+2018 (left curly), U+2019 (right curly). All three forms are now handled.

---

### Fix 4 — `enrichFromFilename`: allow filename title to override bare heading keywords and date strings

**Issue:** In edge cases, `currentTitle` could be set to a bare section heading word ("Dream", "Vision") or a bare date string ("February 16, 2021"). `enrichFromFilename` would not override these because `canUseFilenameTitle` only checked for empty, `narrativeFallbackTitle`, or `dateFallbackTitle` exact matches.

**Root Cause:** `canUseFilenameTitle` was:
```javascript
!currentTitle || currentTitle === narrativeFallbackTitle || currentTitle === dateFallbackTitle
```
It did not account for bare section-heading keywords or free-form date strings produced by document content.

**Fix:** Added two additional conditions to `canUseFilenameTitle` (~line 5194):
```javascript
/^(?:dream|vision|brainstorm|interpretation|application)$/i.test(currentTitle) ||
isDateHeading(currentTitle)
```

**Scope:** `enrichFromFilename` inside `normalizeDreamEntry`. No other function changed.

**Verification:** Upload "Change of Menu Dream 2.16.21.docx". Verify title shows "Change of Menu Dream" regardless of what the parser extracted from document content.

---
