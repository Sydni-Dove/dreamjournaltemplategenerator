# Dove Expressions — Cross-App Generator Architecture

This document describes the cross-app export contract between the Hearing God App, Dream Generator, and Prophetic Generator.

---

## Cross-App Export System

The Hearing God App is the source of truth for selected-entry generator export.

Hearing God App responsibilities:
- owns My Journal selection state
- filters selected entries by generator type
- maps selected entries into full structured generator payload objects
- constructs the generator URL
- encodes the payload as a URL-hash fragment
- opens the correct generator

Generator responsibilities:
- read `#exportData` from `window.location.hash`
- decode the base64 JSON payload
- read `payload.entries`
- stage imported entries for Review and Print
- render Review and Print from structured data

Generators must not re-parse structured hash payloads. The hash payload is already clean structured data.

---

## Payload Contract

Cross-app generator handoff uses this payload shape:

```json
{
  "v": 1,
  "entries": []
}
```

The payload is base64-encoded and appended to the generator URL as:

```text
#exportData=<encoded-payload>
```

The app sends full entry objects, not IDs.

---

## Updated Data Flow

Hearing God App:

```text
Selection
  -> build full structured payload
  -> encode as { v: 1, entries: [...] }
  -> open generator URL with #exportData
```

Generator:

```text
Read window.location.hash
  -> decode #exportData
  -> read payload.entries
  -> store imported entries in pendingPages / review state
  -> generate into lastGeneratedData after page generation
  -> render Review
  -> render Print
```

---

## Layer Separation

Parser Layer:
- only handles raw text, upload, and paste flows
- must not run on structured hash payloads

Review Layer:
- renders the full batch data it receives
- must not collapse imported/generated batches to the first entry
- may read from generator staging state such as `pendingPages`, `pendingReviewData`, or `lastGeneratedData`

Pagination Layer:
- splits already-correct structured content into printable pages
- does not mutate entry content

Rendering Layer:
- displays Review and Print output
- does not modify the structured data

---

## Deprecated Paths

The following handoff paths are deprecated and should not be restored:
- `savedDreamIds`
- `savedPropheticIds`
- query-param-based generator loading for selected-entry export
- ID-only generator handoff for selected-entry export
