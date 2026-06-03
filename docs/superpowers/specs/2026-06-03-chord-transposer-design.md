# Interactive Guitar/Ukulele Chord Transposer Matrix — Design Spec

**Date:** 2026-06-03  
**Status:** Approved  

---

## Overview

A single self-contained `index.html` utility for musicians to paste chord sheets and transpose them in real time. Designed for dwell-time ad revenue in the music niche — musicians keep it open on a stand or desk while practicing. The tool must work offline, load instantly, and never obstruct the chord view.

---

## Delivery Format

- **Single file:** `index.html` — no build step, no server, no Node
- **CDNs:** Tailwind CSS + Alpine.js (both via CDN in `<head>`)
- **Deployable to:** Any static host (Netlify, GitHub Pages, Cloudflare Pages) or opened directly in a browser

---

## Visual Design

**Theme:** Dark Studio  
Dark navy/black background with gold accent — premium recording-studio aesthetic. Easy on eyes in low-light rooms and on gigs.

**Color tokens (CSS custom properties):**
```
--bg:           #0d0d14
--surface:      #111120
--border:       #1e1e2e
--gold:         #c9a84c
--gold-dim:     #c9a84c44
--text:         #cccccc
--text-muted:   #666666
--chord:        #c9a84c
--lyric:        #aaaaaa
```

A dark mode toggle is included. Dark Studio is the default. Light mode uses a warm off-white background (`#faf9f6`) with near-black text (`#1a1a1a`) and the same gold accent (`#c9a84c`) slightly darkened to `#a8832a` for contrast.

---

## Layout

**Desktop (≥768px) — Always-Split:**
```
┌─────────────────────────────────────────┐
│  TOP BANNER AD (728×90, sticky)         │
├─────────────────────────────────────────┤
│  HEADER: Title + Dark Mode toggle       │
├─────────────────────────────────────────┤
│  CONTROL BAR (sticky, full width)       │
│  [−1] [+1]  Key: C  [♯/♭]  [S|M|L]     │
│  [▶ Auto-scroll] [Speed ●───]  [☀Wake]  │
├──────────────────┬──────────────────────┤
│  INPUT PANE      │  OUTPUT PANE         │
│  (textarea)      │  (rendered output)   │
│                  │                      │
│  [300×250 ad]    │  [300×250 ad]        │
├─────────────────────────────────────────┤
│  (no bottom anchor on desktop)          │
└─────────────────────────────────────────┘
```

**Mobile (<768px):**  
Split collapses to vertical stack. Input pane: fixed ~35vh, scrollable. Output pane: fills remaining viewport. Bottom anchor ad (320×50) fixed to viewport bottom with padding to never cover the last chord line.

---

## Architecture

### State (Alpine.js `x-data` on `<body>`)

```js
{
  input: '',           // raw textarea content
  semitones: 0,        // current shift offset (−12 to +12)
  preference: 'sharp', // 'sharp' | 'flat'
  fontSize: 'md',      // 'sm' | 'md' | 'lg'
  darkMode: true,      // boolean
  autoScroll: false,   // boolean
  scrollSpeed: 2,      // 1–5
  wakeLock: null,      // WakeLockSentinel handle
}
```

All persistent keys saved to `localStorage`:
- `ct_input` — textarea content
- `ct_dark` — dark mode
- `ct_fontsize` — font size
- `ct_scroll_speed` — scroll speed
- `ct_preference` — sharp/flat
- `ct_semitones` — transposition offset

Restored via `x-init` on page load.

### File structure inside `index.html`
1. `<head>` — meta, title, Tailwind CDN, Alpine CDN
2. `<style>` — CSS custom properties, monospace chord alignment, ad slot styling, scrollbar, transitions
3. `<body x-data="app()">` — full UI markup
4. `<script>` — `app()` factory with all engine logic

---

## Transposition Engine

### Chromatic Scale
```js
const CHROMATIC_SCALE = [
  { sharp: "C",  flat: "C"  },
  { sharp: "C#", flat: "Db" },
  { sharp: "D",  flat: "D"  },
  { sharp: "D#", flat: "Eb" },
  { sharp: "E",  flat: "E"  },
  { sharp: "F",  flat: "F"  },
  { sharp: "F#", flat: "Gb" },
  { sharp: "G",  flat: "G"  },
  { sharp: "G#", flat: "Ab" },
  { sharp: "A",  flat: "A"  },
  { sharp: "A#", flat: "Bb" },
  { sharp: "B",  flat: "B"  }
];
```

### `transposeChord(chord, semitones, preference)`
1. Extract root with `/^([A-G][#b]?)(.*)/`
2. Find root index — check both `sharp` and `flat` keys across all 12 entries
3. `newIndex = ((index + semitones) % 12 + 12) % 12`
4. Return `CHROMATIC_SCALE[newIndex][preference] + suffix`

Supported suffixes: `m`, `7`, `maj7`, `min7`, `9`, `11`, `13`, `sus2`, `sus4`, `dim`, `aug`, `add9`, and combinations.

### `parseLines(text)` — Line Classifier
Processes input line by line. A line is a **chord line** if ≥50% of its non-whitespace tokens match the chord regex:
```
/\b[A-G][#b]?(?:maj7?|min7?|m|7|9|11|13|sus[24]|dim|aug|add9)?\b/
```

- **Chord lines:** Each chord token wrapped in `<span>` with transposed value. Rendered `font-family: monospace; white-space: pre`.
- **Lyric lines:** Rendered as-is in monospace at matching line-height.
- **Blank lines:** Preserved as verse spacers.
- **Mixed lines** (e.g., "Play the Am chord here"): Treated as lyric lines — embedded chords are NOT transposed.
- **Bar marker lines** (`|`, `/`): Preserved as-is.

Chord and lyric lines share identical monospace line-height so chords stay pixel-aligned above their syllables even when chord name length changes during transposition.

### Current Key Indicator
The control bar displays the detected tonic (first chord token in the input) with the current shift delta: `Key: Am → Bm (+2)`. Updates live on every semitone change.

---

## Features

### Auto-Scroll
- Toggle in control bar starts/stops smooth scroll on the output pane
- Speed slider 1–5 maps to 0.3–1.5px per `requestAnimationFrame` tick
- Scroll pauses on user touch/mouse interaction, resumes after 2s of inactivity
- Speed preference persisted to localStorage

### Screen Wake Lock
- `navigator.wakeLock.request('screen')` called when auto-scroll activates
- Released on auto-scroll off or `visibilitychange` (page hidden)
- Silently skipped on unsupported browsers — no error shown

### Dark Mode Toggle
- Toggles a `dark` class on `<html>`; Tailwind dark-mode utilities handle the rest
- Preference persisted to localStorage and restored on load

### Font Size Toggle
- Three-way toggle: S / M / L
- Applied as Tailwind text-size class on the output pane wrapper
- Preference persisted to localStorage

### Reset Button
- Small inline link in control bar
- Sets `semitones = 0`, clears the key indicator delta

### Pre-loaded Sample Song
"House of the Rising Sun" chord progression (Am–C–D–F / Am–E–Am–E / Am–C–D–F / Am–E–Am) with lyrics. Public domain. Uses 6 distinct chords — good showcase for the transposer across both guitar and ukulele voicings.

---

## Ad Placements

All slots are `<div>` elements with a gold-bordered placeholder labeled "Advertisement" in muted text. Each has a `<!-- Replace with AdSense unit -->` comment for easy swap-in without touching app logic.

| Slot ID | Format | Location | Notes |
|---|---|---|---|
| `ad-top-banner` | 728×90 / 320×50 | Below header, sticky | Highest visibility during setup |
| `ad-sidebar-left` | 300×250 | Below input pane | Desktop only (`hidden md:block`) |
| `ad-sidebar-right` | 300×250 | Below output pane | Desktop only — peripheral to chord view |
| `ad-bottom-anchor` | 320×50 | Fixed viewport bottom | Mobile only — dwell ad during auto-scroll |

---

## Edge Cases & Constraints

- Input with no recognizable chords: output pane renders the raw text unchanged
- Transposing beyond +12 or −12: semitone counter allows free movement; modulo wraps correctly
- Very long chord names (e.g., `Cmaj7#11`): monospace pre-formatting ensures lyrics below don't shift
- Offline use: all CDN assets cached by the browser after first load; no server calls at runtime
- AdSense placeholder divs have fixed dimensions so layout doesn't reflow when real ads load
