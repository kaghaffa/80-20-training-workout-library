# Golf Practice Log

A single-page app for logging **lessons**, **range sessions**, and **rounds** — with the log
structured from the start to be usable as input and context for AI-powered practice
recommendations.

Open `index.html` in any browser. No build step, no dependencies, no account. Data is stored
in `localStorage` on the device and can be exported to JSON at any time.

## Why the data model looks like this

Most golf apps store a score and a date. That is enough to draw a chart and not much else —
a model handed a list of scores can only tell you that you are shooting 89. This log is built
around the three things a coach actually needs to give advice, and it keeps them linked:

| Entry type | What it captures | Why a recommender needs it |
|---|---|---|
| **Lesson** | Focus areas, drills prescribed, the coach's cues verbatim, how well it clicked | **Intent** — what you were told to work on |
| **Range** | Per-block club, intent, shots, shots-on-intent, dominant miss, blocked vs. random structure | **Effort** — what you actually practised, and how it went |
| **Round** | Score to par, FIR, GIR, putts, scrambling, penalties, dominant miss, conditions | **Outcome** — whether any of it is transferring |

The join between them is the shared **focus area** vocabulary. Because a lesson and a range
session tag themselves with values from the same list, the app can answer a question no single
entry type can: *did you actually practise what you were taught?* That turns out to be a better
explanation for most plateaus than any new technical theory, so it is computed explicitly
(`rollups.adherence`) rather than left for a model to infer.

## Features

- Three entry types sharing one schema-driven form renderer
- Controlled vocabularies for clubs, miss patterns, focus areas, practice intent and conditions —
  so terms stay consistent across months of logging instead of drifting into free text
- Shot-level range blocks with a *shots on intent* success rate per club
- Insights view: scoring rates, miss patterns split by source, club coverage, lesson follow-through
- Rolling-window trend (last *n* rounds vs. the *n* before), suppressed below 6 rounds
- **AI context bundle** — copy, download, or POST the whole thing to a model
- Full JSON export / import
- Works offline, installable as a PWA

## Data model

Everything lives under one `localStorage` key, `golfPracticeLog.v1`:

```jsonc
{
  "schemaVersion": 1,
  "profile": { "handicap": null, "homeCourse": "", "goals": "" },
  "entries": [
    {
      "id": "e_m2k3j4_a7f2q",       // stable — used as the citation handle in AI output
      "type": "lesson",             // "lesson" | "range" | "round"
      "date": "2026-08-10",         // YYYY-MM-DD, the only required field on every type
      "createdAt": "2026-08-10T18:04:22.113Z",
      "durationMin": 60,
      "focus": ["Impact", "Iron play"],
      "notes": "…"
      // …plus the fields specific to the type
    }
  ]
}
```

The per-type fields are defined once, in the `TYPES` constant in `index.html`. That single
definition drives the form rendering, the save-time validation, **and** the `schema` block
published in the AI bundle — so the documentation a model receives cannot drift away from the
data the app actually stores. Adding a field means adding one object to `TYPES`.

### Conventions that matter downstream

- **An absent field means *not recorded*, never zero.** `compact()` strips nulls and empty
  values out of the AI bundle, and the prompt states this explicitly, so a model does not read
  an unlogged putt count as a round of zero putts.
- **Units are declared, not implied.** Every numeric field carries a `unit` in the schema
  (`minutes`, `yards`) or is a plain count.
- **Rates ship with their denominator.** `fairwaysHit` is meaningless without
  `fairwayAttempts`; both are stored, and the rollup emits `{made, of, pct}`.
- **Ratings are 1–5 self-reports** and are labelled as such. `fatigue` is inverted on purpose
  (5 = most fatigued) so a low quality score next to a high fatigue score reads as context
  rather than a skill regression.
- **Vocabularies are versioned with the schema.** Changing the meaning of a value means bumping
  `SCHEMA_VERSION` and adding a step to `migrate()`.

## The AI context bundle

The Insights tab builds a self-describing payload for a language model. `buildAIContext()` is
the entire integration surface — everything else in the app is logging and display.

```js
window.golfLog.buildAIContext({ windowDays: 90, maxEntries: 40 })
```

```jsonc
{
  "schema":      { "version": 1, "vocabularies": {…}, "entryTypes": {…} },
  "generatedAt": "2026-09-13T…",
  "window":      { "days": 90, "from": "…", "to": "…", "entriesInWindow": 24, "entriesIncluded": 24 },
  "profile":     { … },
  "rollups":     { "counts":…, "volume":…, "rounds":…, "practice":…, "misses":…, "adherence":…, "recency":… },
  "entries":     [ … ]   // raw logs, newest first, nulls stripped
}
```

Three deliberate choices:

1. **The schema travels with the data.** The model is told the units and the allowed vocabulary
   values, so its recommendations come back in terms this app already stores — a suggested drill
   names a real club from `vocabularies.clubs`, not "your mid iron".
2. **Rollups are precomputed.** Language models are unreliable at arithmetic across dozens of
   rows. Every rate, average and trend is computed deterministically in JS, and the prompt tells
   the model to trust those numbers rather than recompute them from `entries`. The raw entries
   are still included so it can cite specific sessions.
3. **Miss patterns are never merged.** `misses.inPractice` is weighted by shots hit;
   `misses.inRounds` is one vote per round. Different units, and different evidential weight —
   a range slice is much weaker evidence than a slice that cost you strokes on a card. They are
   reported separately and the prompt says not to combine them.

### Output contract

`RECOMMENDATION_OUTPUT_SCHEMA` (in `index.html`) pins the shape of a recommendation, so results
can be rendered directly instead of parsed out of prose:

- `diagnosis[]` — what is limiting the player, each with `evidence` (entry ids / rollup paths)
  and a `confidence` level
- `plan[]` — concrete sessions with per-block club, intent, shot count and success criteria
- `watchItems[]` — metrics to re-check next time, so the next round of data tests this plan
- `dataGaps[]` — what to start logging to make the next recommendation better

That last field closes the loop: the model's answer feeds back into what the app asks you to log.

## Wiring up live recommendations

Today the bundle is copied or downloaded and pasted into any assistant. To generate plans
in-app, POST it to a small server endpoint.

**Do not call the Anthropic API directly from this page.** It is a static client-side app;
an API key shipped to the browser is a key you have published. The browser sends the context
bundle, and the key stays on the server.

Client side — the only change needed in `index.html`:

```js
const ctx = window.golfLog.buildAIContext({ windowDays: 90 });

const res = await fetch('/api/recommend', {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify(ctx)
});
const plan = await res.json();   // matches RECOMMENDATION_OUTPUT_SCHEMA
```

Server side (TypeScript, `npm install @anthropic-ai/sdk`):

```typescript
import Anthropic from "@anthropic-ai/sdk";

const client = new Anthropic();   // reads ANTHROPIC_API_KEY from the environment

// PROMPT_INSTRUCTIONS and RECOMMENDATION_OUTPUT_SCHEMA are exported on
// window.golfLog — lift them into a shared module rather than retyping them.
export async function recommend(ctx: unknown) {
  const response = await client.messages.create({
    model: "claude-opus-5",
    max_tokens: 16000,
    thinking: { type: "adaptive" },
    system: PROMPT_INSTRUCTIONS,
    messages: [
      { role: "user", content: "PLAYER DATA\n```json\n" + JSON.stringify(ctx) + "\n```" },
    ],
    output_config: {
      format: { type: "json_schema", schema: RECOMMENDATION_OUTPUT_SCHEMA },
    },
  });

  const block = response.content.find((b) => b.type === "text");
  return block ? JSON.parse(block.text) : null;
}
```

Notes on that call:

- `claude-opus-5` with **adaptive thinking** — diagnosing a plateau from partial, noisy data is
  exactly the kind of reasoning that benefits from it. Drop to `output_config.effort: "low"`
  if you want cheaper, faster answers for routine check-ins.
- **Structured outputs** (`output_config.format`) guarantee the response validates against the
  contract, so the UI can render it without defensive parsing.
- The bundle is a few KB for a season of logging, so no chunking is needed. If you later add
  a chat-style follow-up, put the stable `schema` block first and the volatile `entries` last
  and mark a cache breakpoint between them.

## Running it

```
open golf-practice-app/index.html
```

Or serve the directory over HTTP to exercise the service worker and the install prompt:

```
python3 -m http.server --directory golf-practice-app 8000
```

The **Load sample data** button on an empty Insights or History tab populates a season of
representative entries — a player whose iron strike is genuinely improving while their tee shot
quietly costs them four strokes a round — which is enough to see every rollup and callout work.

## Status

This directory is self-contained and has no dependency on the rest of the repository. It lives
here only because the repo was the workspace available at the time; moving it to its own
repository is a directory copy, or `git subtree split -P golf-practice-app`.
