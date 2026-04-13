# Recall — Design Document v1.0
**Personal Memory Assistant · Single-file HTML app**

---

## 1. Overview

Recall is a self-contained, offline-first personal memory capture tool that runs as a single HTML file in Chrome. It requires no server, no external APIs, and no installation. All data is stored locally in IndexedDB. The app supports text, voice dictation, audio recording, and photo capture — all organized into a beautiful card-based timeline.

| Property | Value |
|---|---|
| Deliverable | Single `.html` file |
| Target Browser | Chrome (desktop + mobile) |
| Storage | IndexedDB (no size limit) |
| External Dependencies | None — fully offline |
| APIs Used | Web Speech, MediaRecorder, getUserMedia, IndexedDB |
| Export | JSON (text + metadata only) |

---

## 2. Visual Design

### Aesthetic
Neumorphic dark clay. Soft raised surfaces, cool shadows, no harsh borders. The overall feel is tactile, premium, and calm — like a physical leather notebook reimagined in dark digital form.

### Color Palette

| Token | Hex | Usage |
|---|---|---|
| Base | `#1A1D2E` | Page background |
| Surface | `#1E2235` | App surface |
| Card | `#22273D` | Card background |
| Raised | `#2D3250` | Raised elements |
| Mid | `#4A5080` | Borders, dividers |
| Muted | `#7B82B0` | Secondary text |
| Accent | `#9B8EC4` | Purple accent, tags, highlights |
| Highlight | `#C4BAE8` | Highlighted text |
| Text | `#E8E4F8` | Primary text |

### Neumorphic Shadow Rules

- **Raised elements** — double shadow: dark shadow bottom-right (`#0D1018`), light shadow top-left (`#252A45`)
- **Pressed/active state** — inverted shadows, inset style for pressed buttons
- **No hard borders** — shadows define edges. Subtle 1px border at 5% opacity for legibility only
- **Border radius** — 16px for cards, 12px for inputs, 50% for circular action buttons
- **Typography** — System UI stack: `-apple-system, BlinkMacSystemFont, 'Segoe UI'`. No external font loads.

---

## 3. Layout Architecture

```
┌──────────────────────────────────────────────────┐
│  Scrubber │  Top Bar: Title · Search · Export     │
│           ├──────────────────────────────────────┤
│  (fixed   │  Capture Bar                         │
│   left    │  Text input · Dictate · Voice Note   │
│   ruler)  │  Photo · Submit                      │
│           ├──────────────────────────────────────┤
│           │  Memory Timeline (scrollable)         │
│           │  Cards: newest first                 │
│           │  photo thumbnail · audio player      │
│           │  tags · timestamp                    │
└──────────────────────────────────────────────────┘
```

### Timeline Scrubber
A vertical ruler fixed to the left edge of the viewport. Adaptive labels: shows month names when the full history spans more than 60 days; switches to day numbers when viewing a short date range. Clicking a label smoothly scrolls the timeline to that period. A floating indicator shows the current scroll position on the ruler.

---

## 4. Memory Card Anatomy

Each memory card can contain any combination of the following:

- **Text content** — up to 2000 characters, displayed with clamping at 4 lines; "show more" expands inline
- **Photo thumbnail** — displayed at top of card when present; tap to expand to full-screen lightbox
- **Audio player** — shown when a voice note or audio file is attached; simple play/pause + duration bar
- **Tags** — auto-generated date/time tag + any inline `#hashtags` parsed from text at save time
- **Timestamp** — relative ("2 hours ago") that switches to absolute date on hover/tap
- **Actions menu** — Edit · Delete · Copy text. Revealed on hover (desktop) or long-press (mobile)

---

## 5. Capture Bar — Input Modes

### Text Input
Auto-growing textarea. Detects `#hashtags` inline and highlights them in accent purple as typed. Press Enter or the submit button to save.

### Dictate Mode (speech-to-text)
Uses the Web Speech API. Live transcription appears in the text area as the user speaks — interim results shown in muted color, confirmed words in full color. Saved as text only; no audio file is stored. A pulsing mic indicator shows active state.

> **Fallback:** If Web Speech is unavailable (e.g. `file://` protocol), the mic button is hidden and a tooltip explains why. May require launching via `python3 -m http.server` for full functionality.

### Voice Note Mode (audio recording)
Uses MediaRecorder API. Saves an `audio/webm` blob to IndexedDB. No transcription. Displays a record/stop button with a live elapsed timer. The resulting card shows an audio player, not text. Can be combined with typed text on the same card.

### Photo Capture
Two sub-modes triggered from a single camera icon:
- **Camera** — opens `getUserMedia` live preview in a modal, tap-to-capture; works on mobile and desktop webcam
- **File upload** — standard file picker, accepts `image/*`

Photo is stored as a blob in IndexedDB after canvas resize to max 1600px (soft compression). Preview thumbnail shown in capture bar before submission. One photo per card.

---

## 6. Search

A search bar in the top navigation filters the timeline in real-time as the user types. Search matches against:
- Full text content
- Hashtags
- Auto-generated date tag (e.g. typing "March" or "2024" surfaces matching entries)
- Card type (e.g. "photo", "audio")

No search index required — client-side filter over the in-memory card array on each keystroke. Results highlight matched terms in cards.

---

## 7. Data Model (IndexedDB)

- **Database name:** `RecallDB`
- **Object store:** `memories`

```json
{
  "id":        "string",     // UUID v4
  "createdAt": "number",     // Unix timestamp (ms)
  "text":      "string",     // Body text (may be empty)
  "tags":      "string[]",   // Auto date-tag + parsed #hashtags
  "photoBlob": "Blob|null",  // Image binary
  "audioBlob": "Blob|null",  // Audio binary
  "audioType": "string",     // "voice-note" | "dictation" | null
  "edited":    "boolean",
  "editedAt":  "number|null"
}
```

---

## 8. Export

A single "Export" button in the top bar downloads a `recall-export-[date].json` file. The JSON array contains all memory objects with all text fields and metadata. Photo and audio blobs are replaced with a placeholder string noting their existence:

```
"[photo attached — not included in export]"
"[audio attached — not included in export]"
```

No binary data is exported in v1.

---

## 9. Adaptive Timeline Scrubber Logic

The scrubber label density adapts based on total data volume:

| Data Range | Label Format |
|---|---|
| 0–7 days | Individual day names (Mon, Tue…) |
| 8–90 days | Week markers (Week 1, Week 2…) |
| 90+ days | Month names (Jan, Feb…) |
| 2+ years | Year + quarter (2023 Q1…) |

The ruler height matches the scrollable timeline height. Each label's vertical position is **proportional to that period's share of total entries**, not elapsed time — so dense months get more ruler space than quiet ones.

---

## 10. Risks & Mitigations

| Risk | Severity | Mitigation |
|---|---|---|
| Web Speech API unavailable on `file://` | Medium | Hide dictate button; show tooltip; suggest local server |
| `getUserMedia` permission denied | Low | Fall back to file upload only; show permission guidance |
| IndexedDB quota exceeded | Low | Catch quota error; prompt user to export + delete old entries |
| Live transcription inaccuracy | Medium | Editable text after transcription; interim results visually distinguished |
| Large photo blobs | Low | Canvas resize to max 1600px before storage |

---

## 11. Build Phases

### Phase 1 — Core (Data + UI Shell)
- IndexedDB setup and schema
- Card timeline render (newest first)
- Text input + save
- Hashtag parsing and inline highlighting
- Full neumorphic UI shell

### Phase 2 — Media (Capture Modes)
- Dictate mode (Web Speech API)
- Voice Note mode (MediaRecorder API)
- Photo capture (camera modal + file upload)
- Audio player UI on cards
- Photo lightbox / full-screen viewer

### Phase 3 — Polish (Search + Export)
- Real-time search with term highlighting
- Adaptive timeline scrubber
- Edit / delete card actions
- JSON export
- Mobile responsiveness pass

---

## 12. Out of Scope (v1)

- Cloud sync
- Multi-user support
- AI / LLM features
- Push notifications
- Binary export (photos/audio)
- Drag-to-reorder cards
- Card linking / relations
- PWA / installable app shell

---

*Recall Design Document v1.0 — approved for development*
