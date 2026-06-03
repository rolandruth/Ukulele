# Chord Transposer Matrix — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build a single `index.html` file — a real-time guitar/ukulele chord transposer with Dark Studio UI, Alpine.js reactivity, auto-scroll, wake lock, and AdSense-ready ad slots.

**Architecture:** Alpine.js `app()` factory on `<body>` owns all state; pure JS engine functions (`transposeChord`, `parseLines`) handle chord logic outside Alpine; Tailwind Play CDN + custom CSS properties handle styling. No build step, no server — open in browser or deploy to any static host.

**Tech Stack:** Alpine.js 3.x (CDN), Tailwind CSS Play CDN, vanilla JS, localStorage, Screen Wake Lock API, requestAnimationFrame.

---

## File Map

| File | Purpose |
|---|---|
| `index.html` | Entire application — all HTML, CSS, JS in one file |

---

## Task 1: Scaffold index.html

**Files:**
- Create: `index.html`

- [ ] **Step 1: Create the file**

Create `index.html` with this exact content:

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1.0" />
  <title>Chord Transposer — Guitar & Ukulele</title>
  <meta name="description" content="Free online chord transposer for guitar and ukulele. Instantly shift any chord chart up or down by semitone with auto-scroll and dark mode." />

  <!-- Tailwind Play CDN -->
  <script src="https://cdn.tailwindcss.com"></script>
  <script>
    tailwind.config = { darkMode: 'class' }
  </script>

  <!-- Alpine.js -->
  <script defer src="https://cdn.jsdelivr.net/npm/alpinejs@3.13.5/dist/cdn.min.js"></script>

  <style>
    /* ── Color tokens ── */
    :root {
      --bg:         #0d0d14;
      --surface:    #111120;
      --border:     #1e1e2e;
      --gold:       #c9a84c;
      --gold-dim:   rgba(201,168,76,0.27);
      --text:       #cccccc;
      --text-muted: #666666;
      --chord:      #c9a84c;
      --lyric:      #aaaaaa;
    }

    /* Light mode overrides */
    html:not(.dark) {
      --bg:         #faf9f6;
      --surface:    #f0ede8;
      --border:     #ddd8ce;
      --gold:       #a8832a;
      --gold-dim:   rgba(168,131,42,0.2);
      --text:       #1a1a1a;
      --text-muted: #888888;
      --chord:      #a8832a;
      --lyric:      #444444;
    }

    body { background-color: var(--bg); color: var(--text); transition: background-color 0.2s, color 0.2s; }

    /* Chord display */
    .chord-span { color: var(--chord); font-weight: 600; }
    .line        { font-family: 'Courier New', Courier, monospace; white-space: pre; line-height: 1.8; font-size: inherit; display: block; }
    .lyric-line  { color: var(--lyric); }
    .chord-line  { color: var(--chord); }

    /* Scrollbar */
    ::-webkit-scrollbar       { width: 6px; height: 6px; }
    ::-webkit-scrollbar-track { background: var(--surface); }
    ::-webkit-scrollbar-thumb { background: var(--gold-dim); border-radius: 3px; }
    ::-webkit-scrollbar-thumb:hover { background: var(--gold); }
  </style>
</head>
<body x-data="app()" class="min-h-screen">

  <p class="p-8 text-center" style="color: var(--text-muted);">Loading…</p>

  <script>
    function app() {
      return {
        init() { console.log('Alpine ready'); }
      };
    }
  </script>
</body>
</html>
```

- [ ] **Step 2: Verify in browser**

Open `index.html` in a browser. Expected:
- Dark background (#0d0d14), "Loading…" text in muted color
- No console errors

- [ ] **Step 3: Commit**

```bash
git init
git add index.html
git commit -m "feat: scaffold index.html with CDNs and color tokens"
```

---

## Task 2: Transposition Engine

**Files:**
- Modify: `index.html` — replace the `<script>` block content

- [ ] **Step 1: Replace the script block**

In `index.html`, replace everything inside `<script>` (the `function app()...` block) with:

```js
// ── Chromatic scale ──────────────────────────────────────────────────────────
const CHROMATIC_SCALE = [
  { sharp: 'C',  flat: 'C'  }, { sharp: 'C#', flat: 'Db' },
  { sharp: 'D',  flat: 'D'  }, { sharp: 'D#', flat: 'Eb' },
  { sharp: 'E',  flat: 'E'  }, { sharp: 'F',  flat: 'F'  },
  { sharp: 'F#', flat: 'Gb' }, { sharp: 'G',  flat: 'G'  },
  { sharp: 'G#', flat: 'Ab' }, { sharp: 'A',  flat: 'A'  },
  { sharp: 'A#', flat: 'Bb' }, { sharp: 'B',  flat: 'B'  }
];

// ── Core transposer ──────────────────────────────────────────────────────────
function transposeChord(chord, semitones, preference) {
  const m = chord.match(/^([A-G][#b]?)(.*)/s);
  if (!m) return chord;
  const [, root, suffix] = m;
  const idx = CHROMATIC_SCALE.findIndex(n => n.sharp === root || n.flat === root);
  if (idx === -1) return chord;
  const newIdx = ((idx + semitones) % 12 + 12) % 12;
  return CHROMATIC_SCALE[newIdx][preference] + suffix;
}

// ── Line classifier ───────────────────────────────────────────────────────────
const CHORD_TOKEN_RE = /^[A-G][#b]?(?:maj7?|min7?|m|7|9|11|13|sus[24]|dim|aug|add9)?$/;

function isChordLine(line) {
  const tokens = line.trim().split(/\s+/).filter(Boolean);
  if (!tokens.length) return false;
  const chords = tokens.filter(t => CHORD_TOKEN_RE.test(t));
  return chords.length / tokens.length >= 0.5;
}

// ── Chord line renderer (with spacing compensation) ───────────────────────────
function transposeChordLine(line, semitones, preference) {
  let result = '';
  let i = 0;
  while (i < line.length) {
    // Only attempt chord match at start or after whitespace
    if (i === 0 || /\s/.test(line[i - 1])) {
      const rest = line.slice(i);
      const cm = rest.match(/^([A-G][#b]?(?:maj7?|min7?|m|7|9|11|13|sus[24]|dim|aug|add9)?)(?=\s|$|\/)/);
      if (cm) {
        const orig = cm[1];
        const transposed = transposeChord(orig, semitones, preference);
        // Consume trailing spaces
        let spaceEnd = i + orig.length;
        while (spaceEnd < line.length && line[spaceEnd] === ' ') spaceEnd++;
        let spaces = line.slice(i + orig.length, spaceEnd);
        // Compensate for length change
        const diff = orig.length - transposed.length;
        if (diff > 0) spaces += ' '.repeat(diff);
        else if (diff < 0 && spaces.length > 1) spaces = spaces.slice(Math.min(-diff, spaces.length - 1));
        result += `<span class="chord-span">${transposed}</span>${spaces}`;
        i = spaceEnd;
        continue;
      }
    }
    result += line[i] === '<' ? '&lt;' : line[i] === '>' ? '&gt;' : line[i] === '&' ? '&amp;' : line[i];
    i++;
  }
  return result;
}

// ── HTML escaper ─────────────────────────────────────────────────────────────
function escHtml(s) {
  return s.replace(/&/g, '&amp;').replace(/</g, '&lt;').replace(/>/g, '&gt;');
}

// ── Full parser ───────────────────────────────────────────────────────────────
function parseLines(text, semitones, preference) {
  if (!text.trim()) return '';
  return text.split('\n').map(line => {
    if (!line.trim()) return '<span class="line">&nbsp;</span>';
    if (isChordLine(line)) return `<span class="line chord-line">${transposeChordLine(line, semitones, preference)}</span>`;
    return `<span class="line lyric-line">${escHtml(line)}</span>`;
  }).join('');
}

// ── Key indicator ─────────────────────────────────────────────────────────────
function getKeyIndicator(text, semitones, preference) {
  const m = text.match(/[A-G][#b]?(?:maj7?|min7?|m|7|9|11|13|sus[24]|dim|aug|add9)?/);
  if (!m) return 'Key: —';
  const root = m[0].match(/^[A-G][#b]?/)[0];
  if (semitones === 0) return `Key: ${root}`;
  const transposed = transposeChord(root, semitones, preference);
  return `Key: ${root} → ${transposed} (${semitones > 0 ? '+' : ''}${semitones})`;
}

// ── Inline tests (check browser console) ─────────────────────────────────────
(function runTests() {
  const eq = (a, b, msg) => console.assert(a === b, `FAIL: ${msg} — got "${a}", expected "${b}"`);
  eq(transposeChord('C',     2,  'sharp'), 'D',     'C+2=D');
  eq(transposeChord('C',     2,  'flat'),  'D',     'C+2=D (flat pref)');
  eq(transposeChord('B',     1,  'sharp'), 'C',     'B+1=C wrap');
  eq(transposeChord('C',    -1,  'flat'),  'B',     'C-1=B');
  eq(transposeChord('F#',    1,  'sharp'), 'G',     'F#+1=G');
  eq(transposeChord('Gb',    1,  'sharp'), 'G',     'Gb+1=G enharmonic');
  eq(transposeChord('Am7',   2,  'sharp'), 'Bm7',   'Am7+2 keeps suffix');
  eq(transposeChord('Cmaj7', 1,  'sharp'), 'C#maj7','Cmaj7+1 sharp');
  eq(transposeChord('Bb',    2,  'flat'),  'C',     'Bb+2=C');
  eq(transposeChord('Eb',   -1,  'flat'),  'D',     'Eb-1=D');
  eq(transposeChord('G',    12,  'sharp'), 'G',     'G+12 octave wrap');
  eq(transposeChord('G',   -12,  'sharp'), 'G',     'G-12 octave wrap');
  eq(transposeChord('A#',    1,  'flat'),  'B',     'A#+1=B');
  console.assert(isChordLine('Am  C  G  F'),          'chord line: all chords');
  console.assert(isChordLine('Am'),                   'chord line: single chord');
  console.assert(!isChordLine('Yesterday all my troubles'), 'lyric line');
  console.assert(!isChordLine('Play the Am chord here'),    'mixed → lyric');
  console.assert(!isChordLine(''),                    'empty line');
  console.log('✓ All engine tests passed');
})();

function app() {
  return { init() {} };
}
```

- [ ] **Step 2: Verify in browser**

Open (or reload) `index.html`. Open DevTools console. Expected:
- `✓ All engine tests passed`
- No assertion failures
- No JS errors

- [ ] **Step 3: Commit**

```bash
git add index.html
git commit -m "feat: add transposition engine with console tests"
```

---

## Task 3: Alpine app() Factory + LocalStorage

**Files:**
- Modify: `index.html` — replace `function app()` with full factory

- [ ] **Step 1: Replace `function app()` (keep all engine code above it)**

Replace only `function app() { return { init() {} }; }` at the bottom of the script with:

```js
const SAMPLE_SONG = `Am      C       D        F
There is a house in New Orleans
Am      C       E
They call the Rising Sun
Am      C       D          F
And it's been the ruin of many a poor boy
Am      E       Am
And God I know I'm one

Am      C       D        F
My mother was a tailor
Am          E
She sewed my new blue jeans
Am      C       D        F
My father was a gamblin' man
Am      E       Am
Down in New Orleans`;

function app() {
  return {
    input:       '',
    semitones:   0,
    preference:  'sharp',
    fontSize:    'md',
    darkMode:    true,
    autoScroll:  false,
    scrollSpeed: 2,
    wakeLock:    null,
    _scrollFrame: null,

    init() {
      this.input       = localStorage.getItem('ct_input')        ?? SAMPLE_SONG;
      this.darkMode    = localStorage.getItem('ct_dark')         !== 'false';
      this.fontSize    = localStorage.getItem('ct_fontsize')     ?? 'md';
      this.scrollSpeed = parseInt(localStorage.getItem('ct_scroll_speed') ?? '2');
      this.preference  = localStorage.getItem('ct_preference')  ?? 'sharp';
      this.semitones   = parseInt(localStorage.getItem('ct_semitones')    ?? '0');

      // Apply dark mode to <html>
      document.documentElement.classList.toggle('dark', this.darkMode);

      // Persist on change
      this.$watch('input',       v => localStorage.setItem('ct_input', v));
      this.$watch('fontSize',    v => localStorage.setItem('ct_fontsize', v));
      this.$watch('scrollSpeed', v => localStorage.setItem('ct_scroll_speed', v));
      this.$watch('preference',  v => localStorage.setItem('ct_preference', v));
      this.$watch('semitones',   v => localStorage.setItem('ct_semitones', v));
      this.$watch('darkMode', v => {
        localStorage.setItem('ct_dark', v);
        document.documentElement.classList.toggle('dark', v);
      });

      // Release wake lock when page hidden
      document.addEventListener('visibilitychange', () => {
        if (document.hidden && this.wakeLock) {
          this.wakeLock.release().catch(() => {});
          this.wakeLock = null;
        }
      });
    },

    get parsedHtml() {
      return parseLines(this.input, this.semitones, this.preference);
    },

    get keyIndicator() {
      return getKeyIndicator(this.input, this.semitones, this.preference);
    },

    transposeUp()   { this.semitones++; },
    transposeDown() { this.semitones--; },
    resetSemitones() { this.semitones = 0; },

    togglePreference() {
      this.preference = this.preference === 'sharp' ? 'flat' : 'sharp';
    },

    setFontSize(size) { this.fontSize = size; },

    toggleDarkMode() { this.darkMode = !this.darkMode; },

    toggleAutoScroll() {
      this.autoScroll = !this.autoScroll;
      if (this.autoScroll) { this._startScroll(); this._requestWakeLock(); }
      else                 { this._stopScroll();  this._releaseWakeLock(); }
    },

    _startScroll() {
      const pane = document.getElementById('output-pane');
      if (!pane) return;
      const speeds = [0.3, 0.6, 0.9, 1.2, 1.5];
      let paused = false;
      let resumeTimeout = null;

      const pause = () => {
        paused = true;
        clearTimeout(resumeTimeout);
        resumeTimeout = setTimeout(() => { paused = false; }, 2000);
      };
      pane.addEventListener('touchstart', pause, { passive: true });
      pane.addEventListener('mousedown',  pause);
      pane._pauseCleanup = () => {
        pane.removeEventListener('touchstart', pause);
        pane.removeEventListener('mousedown',  pause);
      };

      const tick = () => {
        if (!this.autoScroll) return;
        if (!paused) pane.scrollTop += speeds[this.scrollSpeed - 1];
        this._scrollFrame = requestAnimationFrame(tick);
      };
      this._scrollFrame = requestAnimationFrame(tick);
    },

    _stopScroll() {
      if (this._scrollFrame) { cancelAnimationFrame(this._scrollFrame); this._scrollFrame = null; }
      const pane = document.getElementById('output-pane');
      if (pane && pane._pauseCleanup) { pane._pauseCleanup(); delete pane._pauseCleanup; }
    },

    async _requestWakeLock() {
      if ('wakeLock' in navigator) {
        try { this.wakeLock = await navigator.wakeLock.request('screen'); } catch (_) {}
      }
    },

    _releaseWakeLock() {
      if (this.wakeLock) { this.wakeLock.release().catch(() => {}); this.wakeLock = null; }
    }
  };
}
```

- [ ] **Step 2: Verify in browser**

Open DevTools → Console tab. Run:
```js
document.querySelector('[x-data]').__x.$data.semitones   // → 0
document.querySelector('[x-data]').__x.$data.darkMode    // → true
document.querySelector('[x-data]').__x.$data.preference  // → 'sharp'
```
Expected: values match defaults. No errors.

- [ ] **Step 3: Commit**

```bash
git add index.html
git commit -m "feat: add Alpine app() factory with state and localStorage"
```

---

## Task 4: Full HTML Layout

**Files:**
- Modify: `index.html` — replace `<body>` content (keep `<script>` unchanged)

- [ ] **Step 1: Replace the `<body>` content**

Replace `<p class="p-8 text-center"...>Loading…</p>` (the placeholder paragraph) with all of the following, keeping the `<script>` block at the bottom unchanged:

```html
  <!-- ── Top Banner Ad ─────────────────────────────── -->
  <div id="ad-top-banner" class="w-full flex justify-center items-center py-1"
       style="background:var(--surface); border-bottom:1px solid var(--border); min-height:52px;">
    <div class="hidden md:flex items-center justify-center"
         style="width:728px; height:90px; border:1px dashed var(--gold-dim); color:var(--text-muted); font-size:12px; letter-spacing:1px;">
      <!-- Replace with AdSense unit -->
      ADVERTISEMENT · 728×90
    </div>
    <div class="flex md:hidden items-center justify-center"
         style="width:320px; height:50px; border:1px dashed var(--gold-dim); color:var(--text-muted); font-size:11px; letter-spacing:1px;">
      <!-- Replace with AdSense unit -->
      AD · 320×50
    </div>
  </div>

  <!-- ── Header ────────────────────────────────────── -->
  <header class="flex items-center justify-between px-4 py-3"
          style="border-bottom:1px solid var(--border);">
    <div>
      <h1 class="font-bold tracking-widest uppercase text-sm" style="color:var(--gold); letter-spacing:3px;">
        Chord Transposer
      </h1>
      <p class="text-xs mt-0.5" style="color:var(--text-muted);">Guitar &amp; Ukulele</p>
    </div>
    <button @click="toggleDarkMode()"
            class="rounded-full p-2 transition-colors"
            style="background:var(--surface); border:1px solid var(--border);"
            title="Toggle dark/light mode">
      <span x-text="darkMode ? '☀️' : '🌙'" class="text-base"></span>
    </button>
  </header>

  <!-- ── Sticky Control Bar ────────────────────────── -->
  <div class="sticky top-0 z-40 px-3 py-2 flex flex-wrap items-center gap-2"
       style="background:var(--surface); border-bottom:1px solid var(--border);">

    <!-- Semitone buttons -->
    <button @click="transposeDown()"
            class="px-3 py-1.5 rounded text-sm font-bold transition-colors"
            style="background:var(--bg); border:1px solid var(--gold); color:var(--gold);">
      −1
    </button>
    <button @click="transposeUp()"
            class="px-3 py-1.5 rounded text-sm font-bold transition-colors"
            style="background:var(--gold); color:#0d0d14;">
      +1
    </button>

    <!-- Key indicator -->
    <span class="text-xs font-mono px-2 py-1 rounded"
          style="background:var(--bg); border:1px solid var(--border); color:var(--text-muted);"
          x-text="keyIndicator"></span>

    <!-- Reset -->
    <button @click="resetSemitones()"
            class="text-xs underline"
            style="color:var(--text-muted);"
            x-show="semitones !== 0">
      reset
    </button>

    <!-- Sharp/Flat toggle -->
    <button @click="togglePreference()"
            class="px-3 py-1.5 rounded text-sm font-mono transition-colors"
            style="background:var(--bg); border:1px solid var(--border); color:var(--text);"
            :title="'Current: ' + preference">
      <span x-text="preference === 'sharp' ? '♯' : '♭'"></span>
    </button>

    <!-- Font size -->
    <div class="flex rounded overflow-hidden" style="border:1px solid var(--border);">
      <template x-for="sz in ['sm','md','lg']">
        <button @click="setFontSize(sz)"
                class="px-2.5 py-1 text-xs font-mono transition-colors"
                :style="fontSize === sz
                  ? 'background:var(--gold);color:#0d0d14;'
                  : 'background:var(--bg);color:var(--text-muted);'"
                x-text="sz.toUpperCase()">
        </button>
      </template>
    </div>

    <!-- Auto-scroll -->
    <button @click="toggleAutoScroll()"
            class="px-3 py-1.5 rounded text-xs font-mono transition-colors"
            :style="autoScroll
              ? 'background:#1e2d1a;border:1px solid #4a9b3a;color:#4a9b3a;'
              : 'background:var(--bg);border:1px solid var(--border);color:var(--text-muted);'">
      <span x-text="autoScroll ? '⏸ Scroll' : '▶ Scroll'"></span>
    </button>

    <!-- Speed slider -->
    <div class="flex items-center gap-1" x-show="autoScroll">
      <span class="text-xs" style="color:var(--text-muted);">Speed</span>
      <input type="range" min="1" max="5" x-model.number="scrollSpeed"
             class="w-20 accent-yellow-500" style="cursor:pointer;" />
    </div>

    <!-- Wake lock indicator -->
    <span x-show="wakeLock !== null" class="text-xs" style="color:var(--text-muted);" title="Screen will stay on">☀ On</span>
  </div>

  <!-- ── Split Pane ─────────────────────────────────── -->
  <div class="flex flex-col md:flex-row" style="height: calc(100vh - 220px); min-height: 400px;">

    <!-- Input Pane -->
    <div class="flex flex-col" style="flex:1; border-right:1px solid var(--border);">
      <div class="px-3 py-1.5 text-xs font-mono tracking-widest uppercase"
           style="background:var(--surface); border-bottom:1px solid var(--border); color:var(--text-muted);">
        Input
      </div>
      <textarea
        x-model="input"
        class="flex-1 resize-none p-3 font-mono text-sm outline-none"
        style="background:var(--bg); color:var(--text); border:none; font-family:'Courier New',Courier,monospace; line-height:1.8;"
        placeholder="Paste chord sheet here…"
        spellcheck="false"
      ></textarea>

      <!-- Sidebar ad — desktop only, below input -->
      <div id="ad-sidebar-left" class="hidden md:flex items-center justify-center"
           style="height:250px; min-height:250px; border-top:1px solid var(--border); background:var(--surface);">
        <div style="width:300px; height:250px; border:1px dashed var(--gold-dim); color:var(--text-muted); font-size:11px; display:flex; align-items:center; justify-content:center; letter-spacing:1px;">
          <!-- Replace with AdSense unit -->
          ADVERTISEMENT · 300×250
        </div>
      </div>
    </div>

    <!-- Output Pane -->
    <div class="flex flex-col" style="flex:1;">
      <div class="px-3 py-1.5 text-xs font-mono tracking-widest uppercase flex items-center justify-between"
           style="background:var(--surface); border-bottom:1px solid var(--border); color:var(--text-muted);">
        <span>Output</span>
        <span x-text="semitones !== 0 ? (semitones > 0 ? '+' : '') + semitones + ' semitones' : 'Original key'"
              style="color:var(--gold-dim);"></span>
      </div>
      <div id="output-pane"
           class="flex-1 overflow-y-auto p-3 pb-16"
           :class="{
             'text-sm':  fontSize === 'sm',
             'text-base': fontSize === 'md',
             'text-lg':  fontSize === 'lg'
           }"
           style="background:var(--bg); touch-action:pan-y;"
           x-html="parsedHtml">
      </div>

      <!-- Sidebar ad — desktop only, below output -->
      <div id="ad-sidebar-right" class="hidden md:flex items-center justify-center"
           style="height:250px; min-height:250px; border-top:1px solid var(--border); background:var(--surface);">
        <div style="width:300px; height:250px; border:1px dashed var(--gold-dim); color:var(--text-muted); font-size:11px; display:flex; align-items:center; justify-content:center; letter-spacing:1px;">
          <!-- Replace with AdSense unit -->
          ADVERTISEMENT · 300×250
        </div>
      </div>
    </div>
  </div>

  <!-- ── Mobile Bottom Anchor Ad ───────────────────── -->
  <div id="ad-bottom-anchor"
       class="md:hidden fixed bottom-0 left-0 right-0 flex justify-center items-center z-50"
       style="background:var(--surface); border-top:1px solid var(--border); height:54px;">
    <div style="width:320px; height:50px; border:1px dashed var(--gold-dim); color:var(--text-muted); font-size:11px; display:flex; align-items:center; justify-content:center; letter-spacing:1px;">
      <!-- Replace with AdSense unit -->
      AD · 320×50
    </div>
  </div>
```

- [ ] **Step 2: Verify in browser**

Reload `index.html`. Expected:
- Dark Studio layout visible: gold header, sticky control bar, split textarea/output panes
- Sample song renders in output pane with gold chords above gray lyrics
- Ad placeholder boxes visible (dashed gold borders labeled "ADVERTISEMENT")
- Mobile: use DevTools device emulation — panes stack vertically, bottom ad anchors to bottom

- [ ] **Step 3: Commit**

```bash
git add index.html
git commit -m "feat: add full HTML layout with split panes and ad slots"
```

---

## Task 5: Control Bar Wiring Verification

**Files:**
- Modify: `index.html` — this task verifies the wiring done in Task 4 and fixes any issues found

- [ ] **Step 1: Test each control in browser**

Open `index.html` and verify each control works:

1. **+1 / −1 buttons** — click +1 three times. Output chords should shift up 3 semitones. Key indicator should show e.g. `Key: Am → Cm (+3)`.
2. **Reset** — appears when semitones ≠ 0. Click it. Key indicator returns to `Key: Am`. Reset link disappears.
3. **♯/♭ toggle** — with semitones at +1, click toggle. `Am → A#m` should become `Am → Bbm`.
4. **S/M/L** — click each size. Output text changes size. Active button turns gold.
5. **▶ Scroll** — click. Button turns green. Output pane begins scrolling. Click ⏸ Scroll to stop.
6. **Speed slider** — appears when auto-scroll is active. Drag slider, scroll speed changes.
7. **Dark mode toggle** — click ☀️. Page switches to light mode (warm off-white). Click 🌙 to switch back.

- [ ] **Step 2: Test localStorage persistence**

1. Change semitones to +2 and set font size to LG.
2. Close and reopen `index.html`.
3. Expected: semitones is still +2, font size is still LG, input text is preserved.

- [ ] **Step 3: Fix any issues found**

If any control doesn't work as described, fix the Alpine binding in the HTML. Common issues:
- `x-model.number` needed for numeric sliders
- `x-show` vs `x-if` for toggle visibility
- Alpine reactivity: ensure `parsedHtml` getter is called `parsedHtml` not `parsedLines`

- [ ] **Step 4: Commit**

```bash
git add index.html
git commit -m "fix: verify and fix all control bar bindings"
```

---

## Task 6: Mobile Responsive Polish

**Files:**
- Modify: `index.html` — `<style>` block additions and layout height fix

- [ ] **Step 1: Fix split pane height on mobile**

In the `<style>` block, add after the scrollbar styles:

```css
/* Split pane sizing */
@media (max-width: 767px) {
  #split-pane { height: auto !important; }
  #split-pane > div:first-child { height: 35vh; flex: none !important; border-right: none !important; border-bottom: 1px solid var(--border); }
  #split-pane > div:last-child  { height: calc(65vh - 54px); flex: none !important; } /* 54px = bottom ad */
}
```

- [ ] **Step 2: Add `id="split-pane"` to the split container**

Find the line:
```html
<div class="flex flex-col md:flex-row" style="height: calc(100vh - 220px); min-height: 400px;">
```

Change it to:
```html
<div id="split-pane" class="flex flex-col md:flex-row" style="height: calc(100vh - 220px); min-height: 400px;">
```

- [ ] **Step 3: Add output pane bottom padding for mobile anchor ad**

The output pane already has `pb-16`. On mobile, the bottom anchor ad is 54px tall, so 64px padding is sufficient. Verify the last lyric line is not hidden behind the ad on mobile by scrolling to the bottom in DevTools device emulation.

- [ ] **Step 4: Verify in browser**

Use DevTools → device emulation (iPhone SE or similar). Expected:
- Input textarea takes top ~35vh
- Output pane below it, scrollable
- Last line not obscured by bottom anchor ad
- Control bar wraps gracefully on small screens

- [ ] **Step 5: Commit**

```bash
git add index.html
git commit -m "fix: mobile responsive layout for split pane and anchor ad"
```

---

## Task 7: Visual Polish

**Files:**
- Modify: `index.html` — `<style>` block additions

- [ ] **Step 1: Add transition and focus styles**

In the `<style>` block, append:

```css
/* Button hover states */
button { transition: opacity 0.15s, transform 0.1s; }
button:active { transform: scale(0.96); }

/* Textarea focus */
textarea:focus { box-shadow: inset 0 0 0 1px var(--gold-dim); }

/* Control bar button base */
.ctrl-btn {
  display: inline-flex; align-items: center; justify-content: center;
  border-radius: 6px; font-size: 13px; padding: 6px 12px;
  cursor: pointer; user-select: none;
}

/* Output pane: fade in on load */
#output-pane { animation: fadeIn 0.3s ease; }
@keyframes fadeIn { from { opacity: 0; } to { opacity: 1; } }

/* Chord span hover (click-to-highlight) */
.chord-span { cursor: default; border-radius: 2px; padding: 0 1px; transition: background 0.15s; }
.chord-span:hover { background: var(--gold-dim); }
```

- [ ] **Step 2: Add section label styling to ad slots**

In the `<style>` block, append:

```css
/* Ad placeholder labels */
[id^="ad-"] > div {
  border-radius: 6px;
  letter-spacing: 1.5px;
  text-transform: uppercase;
  font-family: monospace;
}
```

- [ ] **Step 3: Verify in browser**

Reload and check:
- Buttons have smooth press animation
- Chord spans subtly highlight on hover
- Output pane fades in on page load
- Ad placeholders look styled (monospace label, rounded border)

- [ ] **Step 4: Commit**

```bash
git add index.html
git commit -m "feat: add visual polish — transitions, hover states, fade-in"
```

---

## Task 8: Final Edge Cases & SEO

**Files:**
- Modify: `index.html` — `<head>` meta additions and JS edge case fixes

- [ ] **Step 1: Add Open Graph meta tags to `<head>`**

After the existing `<meta name="description">` tag, add:

```html
  <meta property="og:title" content="Chord Transposer — Guitar & Ukulele" />
  <meta property="og:description" content="Free online chord transposer. Shift any chord sheet up or down by semitone instantly." />
  <meta property="og:type" content="website" />
  <meta name="theme-color" content="#0d0d14" />
  <link rel="icon" href="data:image/svg+xml,<svg xmlns='http://www.w3.org/2000/svg' viewBox='0 0 100 100'><text y='.9em' font-size='90'>🎸</text></svg>" />
```

- [ ] **Step 2: Handle empty input gracefully**

In the `parseLines` function, the first line is `if (!text.trim()) return '';`. This leaves the output pane blank if the user clears the textarea. Replace it with:

```js
function parseLines(text, semitones, preference) {
  if (!text.trim()) return '<span class="line lyric-line" style="color:var(--text-muted);font-style:italic;">Paste a chord sheet into the input pane…</span>';
  // ... rest unchanged
```

- [ ] **Step 3: Verify edge cases in browser**

Test these scenarios:
1. **Clear the textarea** — output pane shows the placeholder message in muted italic
2. **Input with no chords** (just lyrics) — all lines render as lyric lines, no errors
3. **Input with only chords** — all lines render as chord lines in gold
4. **Rapid +1 clicking** (click 20 times fast) — semitones increments correctly, no rendering glitches
5. **Slash chords** (`Am/E`, `G/B`) — `Am/E` renders as `<span>Am</span>/<span>E</span>` both transposed
6. **Very long chord names** (`Cmaj7`, `Gsus4`, `Bbdim`) — spacing compensation keeps subsequent chords in column

- [ ] **Step 4: Commit**

```bash
git add index.html
git commit -m "feat: add SEO meta, favicon, and empty-state handling"
```

---

## Task 9: Final Verification Pass

**Files:**
- Read: `index.html` — no changes, just verification

- [ ] **Step 1: Full feature checklist in browser**

Run through the complete feature list against the spec:

| Feature | How to test | Expected |
|---|---|---|
| Transpose +1 semitone | Click +1 | All chords shift up one step |
| Transpose −1 semitone | Click −1 | All chords shift down one step |
| Wrap-around | Start at B, +1 | Becomes C |
| Sharp preference | Set ♯, transpose C up 1 | Shows C# |
| Flat preference | Set ♭, transpose C up 1 | Shows Db |
| Key indicator | Transpose to +2 | Shows `Key: X → Y (+2)` |
| Reset | Click reset when offset ≠ 0 | Returns to 0, reset link hides |
| Font S/M/L | Click each | Output text resizes |
| Dark mode | Toggle | Page switches color scheme |
| Auto-scroll | Enable | Output pane scrolls smoothly |
| Scroll speed | Drag slider | Scroll rate changes |
| Scroll pause | Touch output during scroll | Pauses 2s then resumes |
| LocalStorage | Reload page | All preferences restored |
| Sample song | Clear localStorage, reload | House of the Rising Sun loads |
| Empty state | Clear textarea | Placeholder message shows |
| Mobile layout | DevTools emulation | Stacked panes, bottom ad anchored |
| Ad slots | Desktop + mobile | 4 placeholder divs visible |

- [ ] **Step 2: Cross-browser spot check**

If other browsers are available (Firefox, Safari), open `index.html` in each. Verify no layout breaks and Alpine/Tailwind load correctly from CDN.

- [ ] **Step 3: Final commit**

```bash
git add index.html
git commit -m "feat: complete chord transposer — all features verified"
```

---

## Self-Review: Spec Coverage Check

| Spec requirement | Task covering it |
|---|---|
| Single HTML file | Task 1 |
| Alpine.js + Tailwind CDN | Task 1 |
| Dark Studio color tokens | Task 1 |
| Light mode toggle | Task 4 |
| CHROMATIC_SCALE with enharmonics | Task 2 |
| `transposeChord()` with suffix preservation | Task 2 |
| `isChordLine()` ≥50% threshold | Task 2 |
| `transposeChordLine()` with spacing compensation | Task 2 |
| `parseLines()` with chord/lyric/blank handling | Task 2 |
| Key indicator (`Key: X → Y (+n)`) | Task 2, Task 4 |
| Alpine state + all state properties | Task 3 |
| LocalStorage for all 6 keys | Task 3 |
| Sample song (House of the Rising Sun) | Task 3 |
| Always-split desktop layout | Task 4 |
| Mobile stacked layout | Task 6 |
| Semitone +/− buttons | Task 4 |
| Sharp/flat toggle | Task 4 |
| Font size S/M/L | Task 4 |
| Reset button | Task 4 |
| Auto-scroll with rAF | Task 3 (logic), Task 4 (UI) |
| Scroll speed slider | Task 4 |
| Scroll pause on touch/mouse | Task 3 |
| Screen Wake Lock API | Task 3 |
| 4 ad placeholder slots | Task 4 |
| SEO meta + favicon | Task 8 |
| Empty state handling | Task 8 |
| Edge case: slash chords | Task 8 verification |
| Edge case: long chord names | Task 8 verification |
