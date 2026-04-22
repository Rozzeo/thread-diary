# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Running locally

```bash
python -m http.server 3000
# open http://localhost:3000
```

No build step. The app uses React 18 + Babel Standalone loaded from CDN — JSX is transpiled in the browser at runtime.

## Architecture

The entire app lives in a single `index.html`. There is no bundler, no npm, no node_modules. All logic is written as `<script type="text/babel">` blocks that run in order. Each block exposes its exports via `Object.assign(window, {...})` so later blocks can use them.

**Script block order (matters — later blocks depend on earlier ones):**

| Block | What it does |
|---|---|
| `tokens.jsx` | Design tokens: `PALETTES`, `ACCENTS`, `INK`, `MOODS`, `MOOD_COLORS`, `PAPER_GRAIN`, date helpers |
| `icons.jsx` | SVG icon components + `HandDivider`, `StitchedCorners`, `StitchUnderline` |
| `seed.jsx` | Hardcoded demo entries (`SEED_ENTRIES`) and planted words (`SEED_PLANTED`) + `mockFeedback()` |
| `coach.jsx` | `runCoach(entry)` → calls Groq API → returns parsed JSON feedback. Falls back to `mockFeedback` if no key or request fails |
| `nudges.jsx` | Idle-typing nudge system: `pickNudge()`, `contextualNudge()`, `stageFor()` |
| `writer.jsx` | `Writer` component — textarea, mood picker, coach invocation, feedback cards |
| `calendar.jsx` | `CalendarScreen` — monthly mood grid + recent entries list |
| `garden.jsx` | `GardenScreen` — planted vocabulary words as sprout cards |
| `tweaks.jsx` | `TweaksPanel` — settings drawer (theme, API key) |
| `app.jsx` | `App` root — screen routing, state orchestration, `Nav`, `EntryReader` |

## Data flow

**AI:** `Writer.handleSave()` → `window.runCoach(entry)` → Groq `llama-3.3-70b-versatile` → JSON with grammar/vocabulary/style/mood/level fields.

**localStorage keys:**
- `thread:groq_key` — Groq API key (set via TweaksPanel ⚙️ or `?key=` URL param)
- `thread:entries` — array of `{ date: timestamp, mood, level, excerpt }` saved after each coach run
- `thread:planted` — array of vocabulary word objects planted by the user

**State sync:** `App` polls `localStorage` every 1 second to pick up `thread:entries` and `thread:planted` changes made by `Writer` (they run in the same window but `App` holds the shared state for `CalendarScreen` and `GardenScreen`).

## Editing the codebase

Since everything is inlined into `index.html`, find the relevant section by searching for the comment header (e.g. `// Coach:`, `// Writer screen`, `// Garden screen`). Each section ends with `Object.assign(window, ...)` and `</script>`.

**Deploy:** push to `master` — GitHub Pages auto-deploys from `https://rozzeo.github.io/thread-diary/`.

## Key design decisions

- No auth, no backend — all data is local to the browser's localStorage
- The `?key=gsk_...` URL param pre-fills the Groq key (useful for demos)
- `SEED_ENTRIES` fill calendar days that have no real user entry; real entries take priority by `dateKey`
- The coach prompt enforces strict JSON-only output; `mockFeedback()` is the hardcoded fallback when Groq is unavailable or the key is missing
