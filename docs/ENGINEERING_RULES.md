# Dream Journal Generator — Engineering Rules

## Purpose

This generator is not a basic form app. It is a browser-based document transformation pipeline.

Its job is to ingest messy source documents, extract structure, normalize content, and compose deterministic print-ready pages without corrupting meaning.

---

## Core Architecture

The system has strict layers. Each layer has one responsibility.

### 1. Extraction Layer
Responsible for reading source files and converting them into normalized raw text or structured source blocks.

Examples:
- PDF text extraction
- DOCX extraction
- pasted raw text

Rules:
- Extraction does not assign semantic meaning
- Extraction may preserve source metadata when useful
- Extraction should expose source artifacts, not hide them

### 2. Segmentation Layer
Responsible for detecting boundaries before semantic parsing.

Examples:
- entry boundaries
- title boundaries
- section heading boundaries
- repeated block boundaries

Rules:
- Segmentation is a first-class stage
- Do not combine segmentation with semantic assignment
- A bad segmentation decision corrupts everything downstream
- Boundary detection must happen before content assignment

### 3. Parser Layer
Responsible only for assigning segmented content into canonical fields.

Canonical fields include:
- title
- date
- time
- dreamNarrative
- dreamBrainstorm
- dreamInterpretation
- dreamApplication
- dreamSymbols
- additionalNotes

Rules:
- Parser output is the source of truth
- Parser decides structure once
- Parser must not drift line-by-line through heuristic state when a locked structure map is available
- Repeated sections must merge in document order without overwriting earlier content
- Pre-heading prose defaults to dreamNarrative unless proven otherwise

### 4. Normalization Layer
Responsible only for cleanup.

Allowed:
- trim whitespace
- normalize line breaks
- remove duplicates
- clean formatting artifacts

Not allowed:
- re-read raw text to reclassify structure
- move content between major sections unless parser explicitly marked incompleteness
- override parser-owned structure

### 5. Pagination Layer
Responsible only for splitting already-correct structured content across pages.

Rules:
- Never alter meaning
- Never reassign sections
- Never rewrite content
- Use word-boundary measurement, not character splitting
- No blank pages
- No duplicate pages
- No clipped words or sentences

### 6. Rendering Layer
Responsible only for visual composition.

Rules:
- Must display already-correct paginated content
- Must not alter parser output
- Must not compensate for parser mistakes
- Layout containment and text wrapping are rendering problems, not parser problems

### 7. Print Layer
Responsible only for deterministic print output.

Rules:
- Print output must match preview
- No UI chrome inside print pages
- No layout shift between preview and print
- No clipped content
- No hidden overflow

---

## Canonical Pipeline

The pipeline should work in this order:

1. Extract source input
2. Normalize source text minimally
3. Segment boundaries
4. Detect structure map
5. Assign content into canonical fields
6. Normalize/clean fields
7. Paginate
8. Render
9. Print/export

Do not skip stages.
Do not merge stages casually.

---

## Parser Design Rules

### Locked Structure First
The parser must detect document structure before assigning content.

That means:
- identify headings first
- classify heading types first
- build section ranges first
- only then assign content

### No Drift-Based State Machines
Avoid parser logic that depends on unstable live-state flags such as:
- currentField
- awaitingFirstBodyAfterTitle
- openingMajorField

Those patterns create regression loops.

### Consecutive Heading Rule
If headings appear back-to-back with no body between them:
- do not let the later heading steal ownership by default
- repair ownership narrowly based on actual body content
- keep the structure map intact

### Repeated Section Rule
If the same section appears more than once:
- merge in document order
- never overwrite earlier valid content
- record a debug warning that repeated sections were merged

### Symbol Extraction Rule
Symbols are not free prose.

Only promote content into `dreamSymbols` when there is strict glossary/list-style evidence.
Do not harvest:
- interpretation prose
- narrative prose
- URLs
- arbitrary “X is Y” free prose

If there is no strict symbol evidence, leave `dreamSymbols` empty.

---

## Debug and Observability Rules

Every parsed entry should expose structured debug metadata.

Minimum debug fields:
- `_docMode`
- `_detectedHeadings`
- `_headingCount`
- `_sectionRanges`
- `_warnings`
- `_titleSource`

Warnings should include:
- no headings detected
- repeated sections merged
- mixed labeled/unlabeled structure
- unknown heading encountered
- heading-like line ignored
- symbol extraction skipped due to lack of strict evidence

Do not debug by guesswork when the parser can report its decisions.

---

## Layer Discipline Rules

Before changing code, always identify the broken layer.

### If parser output is wrong:
fix parser or segmentation only

### If parser output is right but layout is wrong:
fix rendering or print only

### If parser output is right but page breaks are wrong:
fix pagination only

### If parser output is right but content gets reassigned later:
fix normalization or post-processing only

Never patch a downstream layer to hide an upstream bug.

---

## Regression Control Rules

Every fix must be tied to:
1. one failing document shape
2. one broken layer
3. one proven first bad stage

Every fix should be validated against real fixtures such as:
- startup prose before Brainstorm
- Brainstorm before Dream
- unlabeled prose only
- repeated Dream/Brainstorm sections
- template-style Dream / Brainstorm / Interpretation / Application
- DOCX flattened consecutive headings
- symbol-heavy glossary entries
- interpretation-heavy uploads

Do not trust synthetic examples alone.

---

## Working Method for LLM Coding

Use deterministic, narrow passes.

### Required flow
1. Identify the first bad stage
2. Explain why it is happening
3. Fix only that stage
4. Verify against real fixtures
5. Leave other layers untouched

### Never do this
- broad rewrites without proof
- mixed parser/render/pagination fixes in one pass
- “improve parser generally”
- add fallback logic before tracing the actual failure

### Preferred style
- one issue
- one layer
- one pass
- one verification report

---

## Practical Principle

This system should behave more like a compiler than a chat assistant.

That means:
- deterministic stages
- canonical intermediate structure
- explicit boundaries
- no silent reinterpretation
- objective validation before moving on

---

## Final Rule

Parser structure is final unless explicitly proven incomplete.

Everything else must respect that.