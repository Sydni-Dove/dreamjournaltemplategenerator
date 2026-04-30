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

---

## Hearing God App Runtime Contracts

These notes describe app-side contracts that must stay aligned with the shared generator ecosystem.

### Music Feed + Sticky Player

The Music Feed and sticky music player use a shared music context. The current contract is:

```text
Music Feed click
  -> selectTrack(playlist, track)
  -> MusicContext stores current playlist ID + current track ID
  -> currentTrack resolves from the selected playlist
  -> PersistentMusicPlayer renders currentTrack.url
```

Important boundaries:

- `selectTrack` requires both the `MusicPlaylist` and `MusicTrack`.
- Music Feed song buttons must call `onSelectTrack(playlist, track)`, not `onSelectTrack(track)`.
- `MusicContext` must preserve the selected track's `url`; it should not replace the clicked track with title-only data.
- `PersistentMusicPlayer` must derive embeds and direct audio from `currentTrack.url`.
- The iframe/audio element must be keyed by the selected track, using `currentTrack.id || currentTrack.url`, so a changed song remounts the media element and does not keep playing a stale YouTube/audio source.
- Direct audio playback should reset when `currentTrack.id` or `currentTrack.url` changes.

The playlist data source is `src/data/musicPlaylists.ts`. Track URLs are part of the playback contract and must not be removed. If a YouTube video disables embedding, replace that track with an embeddable/available video URL rather than changing player logic.

Current playlist curation notes:

- Jesus Culture tracks live in `Soaking`.
- Embassy Worship tracks live in `Warfare`.
- `Prayer` includes `Names Of God` by Mercy Culture Worship.
- `Warfare` includes `The Breaking` by Sharde Martin, `Yahweh Flow` by Tye Tribbett, and `Way Maker` by Maranda Curtis.
- `Prophetic Warfare (Instrumental)` includes Braam Official's `Prophetic Worship Instrumental; YAHWEH`.
- `Jump in the River` by Sharde Martin was removed.
- The DappyTKeys `Alone With God` URL should only be used by that specific DappyTKeys track, not as a fallback for unrelated songs.

### Prophetic Word Save Requirements

The prophetic word form must only require the main prophetic word/content field.

Validation contract:

```text
if (!wordContent.trim()) {
  block save
}
```

Fields that must remain optional:

- author / speaker / "who it's by"
- time
- date UI may default to the current date, but validation should not require the user to fill author/speaker or time

Save behavior:

- Empty author/speaker should be sent as an empty value that maps to `source: null` in the existing app save flow.
- Empty time should remain optional (`undefined` or `null`, depending on the existing local form model).
- Do not change Supabase table shape or service calls to enforce these UI validation rules.

### Facebook Community Link

The Facebook Community page's join button should route users to the private Facebook group:

```text
https://www.facebook.com/groups/1810675959353145
```
