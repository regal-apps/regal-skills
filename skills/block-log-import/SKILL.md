---
name: block-log-import
description: >
  Import or backfill workout logs into Block from any raw format — Strong or
  Hevy CSV exports, chat messages, training journals, markdown tables,
  photographed notes. Use whenever someone asks to import, migrate, backfill,
  or "get my old workouts into Block", or shares raw workout logs that should
  be persisted. Parses set notation, resolves exercise names against the
  Block library including aliases, teaches personal aliases so each name is
  resolved only once, and writes sessions with import provenance.
---

# Block log import

Turn raw workout logs into Block sessions:
**parse → resolve every exercise name → write → report**.

Requires the `block-mcp` tools (`list_exercises`, `get_workout_session`,
`put_session_log`). If they are not connected, stop and say so — this skill
cannot import anything without them.

Work one training day at a time, oldest first. Never batch several dates into
one session.

## 0. Never import the same day twice

Before writing any date, call `get_workout_session` for it. If a session with
the same exercises already exists, skip that date and report it. Imports must
be safely re-runnable.

## 1. Parse the raw log

Extract, per training day: the **date**, an optional **title**, and an ordered
list of **exercises**, each with its ordered **sets**.

Per set, Block stores `weight` (kg), `reps`, `rir`, and `warmup`.

### Reading effort

Block stores **RIR** (reps in reserve). Convert whatever the source uses:

- `RIR n` → `n`
- `RPE n` (Strong, Hevy) → `10 − n`, floored at 0 (RPE 8 → RIR 2)
- A range (`2-3 RIR`, `RPE 7-8`) → take the **harder-sounding bound's**
  conservative reading, i.e. the **higher RIR** (`2-3 RIR` → 2 is the lower
  RIR; use it only if the source means "at least 2 left"). When ambiguous,
  prefer the lower RIR — assuming more effort keeps later progression
  estimates conservative.
- Nothing recorded → `null`. Never invent effort.

### Common notations

- `9x140`, `9 x 140`, `140 x 9` — reps and weight. Disambiguate by
  plausibility (reps rarely exceed ~30; loads rarely sit below reps for
  barbell work) and by staying consistent within one log.
- `WU`, `warmup`, `w/u`, Hevy's `set_type: warmup` → `warmup: true`, and
  leave `rir` null. Warm-ups are recorded but never count as working sets.
- Drop sets (`drop to 5x12.5`, Hevy `set_type: dropset`) → log each segment
  as its own set, directly after its parent set.
- Per-side or unilateral blocks (`[Set 1] 12x20 / 12x20`, "per leg") → log
  each performed set; weight is per side.
- Bodyweight movements → `weight: null`, unless added load is noted
  (`+5 kg`, `BW+10`), which becomes the weight.
- Failure notations (`0 RIR`, `AMRAP`, `to failure`) → `rir: 0`.
- Parenthetical remarks ("left shoulder felt off", "better form") are
  commentary, not data. Keep them out of Block; surface them in the report.

### What not to import

- **Time-based or circuit blocks with no reps and loads** ("3 rounds of 2 min:
  plank, dead bug") are not set-based. Skip them and list them in the report.
- **Sessions with no numbers at all.** Do not estimate. Tell the user what is
  missing and ask whether to skip the session or reconstruct it together.

## 2. Resolve every exercise name — before writing anything

Call `list_exercises` once and match each parsed name against both **names
and aliases**. Matching folds case, umlauts, accents and separators
automatically (`bankdruecken` = `Bankdrücken`), so your job is everything
semantic beyond that.

For each name, in order:

1. **It fold-matches a name or alias** → use it as-is. Nothing to do.
2. **It's the same movement under a different name** — a translation, a
   different word order, a synonym, an abbreviation ("Lat Pulldown" for
   "Latzug", "DB Row" for "Kurzhantel-Rudern") → pass that exercise's
   `exercise_id` **and** `remember_alias: true`. The mapping is persisted, so
   this name resolves by itself in every future import.
3. **It's genuinely a different movement** → pass the name with no
   `exercise_id`; Block creates a custom entry. Afterwards read the
   `near_matches` in the response: if it reveals an existing exercise you
   should have matched, tell the user rather than leaving a duplicate.

Hard rules:

- **Positional and equipment variants are distinct exercises.** Seated vs.
  standing calf raise, lying vs. seated leg curl, hanging vs. lying leg
  raise, machine vs. barbell press, high-bar vs. low-bar squat. Never map one
  onto another, even if only one exists in the library — create the missing
  variant instead. Merging them silently fuses two progressions into one and
  cannot be undone from the log.
- **Never guess an `exercise_id`.** Only use ids returned by
  `list_exercises`.
- When you cannot decide between rule 2 and rule 3, **ask**. Batch every open
  question and put them to the user *before* the first write, not one at a
  time mid-import.

## 3. Write the session

One `put_session_log` call per training day:

- `date` — the training date (`YYYY-MM-DD`).
- `title` — the session name from the log ("Upper A", "Push"), if any.
- `finished: true` and `source: "import"` — backfilled history is completed
  history, and provenance must say it was imported rather than logged live.
- `exercises` — in the order trained, each with its sets in order, warm-ups
  included and flagged.

## 4. Report

One compact line per session:

```
✓ 2026-07-23 Lower A — 6 exercises, 16 working sets
  aliased: "Hyper Extension" → Hyperextension
  created: Hack Squat
  skipped: core circuit (time-based)
```

Then a closing summary: sessions written, aliases taught, custom exercises
created, everything skipped and why, plus any commentary from the logs worth
keeping.

A second run over the same material should teach **zero** new aliases and
create **zero** new exercises. If it doesn't, say what changed — that is the
signal that something resolved inconsistently.
