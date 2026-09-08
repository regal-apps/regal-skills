# regal-skills

Agent skills for the **Regal** product line — a set of household apps whose
data an AI agent can read and write through one MCP server, `regal-mcp`
(`https://mcp.das-regal.workers.dev/mcp`).

A skill is a set of instructions an AI coding agent (Claude Code, Cowork, or
any agent that reads the `SKILL.md` convention) loads when it recognises a
matching task. These skills teach an agent how to do things *with* the apps
that their own UI doesn't cover.

Today `regal-mcp` exposes the tools of [Block](https://block-bmg.pages.dev) —
workouts and daily schedule — so that's where the skills are. Skills for the
other apps are additive.

## Skills

| Skill | App | What it does |
|---|---|---|
| [`block-log-import`](skills/block-log-import/SKILL.md) | Block | Imports workout history from any raw format — Strong/Hevy CSV exports, chat messages, training journals, photographed notes. Parses set notation, resolves exercise names against the library (including aliases), and writes sessions with import provenance. |

## Prerequisites

1. A Block account.
2. `regal-mcp` connected to your agent and authenticated via OAuth. The skills
   call its tools (`list_exercises`, `get_workout_session`, `put_session_log`)
   and do nothing without them.

## Install

```bash
npx skills add regal-apps/regal-skills
```

Or copy the skill folder you want into your project's `.claude/skills/`.

## How exercise names are handled

Block ships a seed exercise library with curated aliases, so standard English
gym vocabulary resolves onto it without creating duplicates — "Bench Press"
finds "Bankdrücken", "Lat Pulldown" finds "Latzug". When a skill resolves a
name the library doesn't know yet, it records that mapping as a personal
alias, so the same name is never ambiguous twice.

Matching is deterministic: exact matches after folding case, umlauts, accents
and separators are accepted silently; anything less certain is surfaced as a
suggestion rather than guessed. Similar-but-distinct movements — seated vs.
standing calf raise — are kept apart on purpose, because merging two
progressions cannot be undone.

## Vocabulary: muscles and equipment

Every exercise carries a `muscle_group` and an `equipment` value. **The API
speaks English enum values only**; the German words in the Block app are
display labels and never appear in a tool call or response.

`muscle_group` — the RP-aligned set of 17:

```
chest, lats, upper_back, lower_back, traps,
front_delts, side_delts, rear_delts,
biceps, triceps, forearms,
quads, hamstrings, glutes, calves, abs,
other
```

`equipment` — 6 values:

```
barbell, dumbbell, machine, cable, bodyweight, other
```

`other` is the untagged fallback in both lists, not a category: an exercise
tagged `other` is invisible to every volume view (`get_volume`, the app's
Tagebuch chart). Custom exercises created through `put_routine` or
`put_session_log` start out as `other`; the user tags them in the app's
exercise picker. When a skill creates one, it says so in its report so that
tagging happens.

## Tool reference

All tools take and return JSON; dates are `YYYY-MM-DD`, times `HH:MM` (24h),
weights kg, effort as RIR (reps in reserve). Read tools are annotated
read-only.

| Tool | Kind | Purpose |
|---|---|---|
| `list_routines` | read | All routines with ordered exercises and target sets |
| `list_exercises` | read | The library (seed + the user's custom entries) with aliases, `muscle_group`, `equipment`, `unilateral`; optional `query` for folded substring + fuzzy search |
| `put_routine` | write | Create/update a routine as one document; exercises by name or `exercise_id`, optional `remember_alias`; flags `created` entries and `near_matches` |
| `schedule_workout` | write | Snapshot a routine onto a date as a planned session with a workout slot |
| `put_day` / `get_day` / `get_week` | write / read | A day's timeline of slots (generic, meal, workout) with statuses; `get_week` is seven `get_day`s |
| `get_adherence` | read | Slot status roll-ups for a range, split at `today` into elapsed and upcoming |
| `get_volume` | read | Working sets per muscle for a range, logged apart from planned — see below |
| `get_workout_session` | read | Sessions of a date with prescription and log as separate lists, plus lifecycle `state` |
| `put_session_log` | write | Write a session's performed sets as one document; `date` creates an ad-hoc session (`finished`, `source: import` for backfill), `session_id` rewrites an existing log |
| `set_session_state` | write | Settle `planned` / `started` / `finished`; finishing marks the slot done |
| `merge_session_into_plan` | write | Fold a duplicate logged session into the planned one of the same day |
| `put_metric` / `get_metrics` | write / read | Daily values (bodyweight, waist, steps, kcal, …), one per date and kind, with note and `source` |
| `get_exercise_history` | read | One exercise's logged sets across sessions in a range, oldest first |

### `get_volume`

Working sets per muscle over a date range — the weekly volume check in one
call instead of counting through every session.

Parameters:

| Name | Required | Meaning |
|---|---|---|
| `from` | yes | Range start, inclusive |
| `to` | yes | Range end, inclusive; must not precede `from` |
| `today` | no | Cutoff between upcoming and missed plans; defaults to today in Europe/Berlin |

Response:

```json
{
  "from": "2026-09-07",
  "to": "2026-09-13",
  "today": "2026-09-09",
  "muscles": [
    { "muscle_group": "chest", "logged": 6, "planned": 3 },
    { "muscle_group": "lats",  "logged": 4, "planned": 4 },
    { "muscle_group": "other", "logged": 2, "planned": 0 }
  ],
  "total": { "logged": 12, "planned": 7 }
}
```

- `logged` counts sets actually performed, from every session in range
  whatever its state, by Block's one set-counting rule: a unilateral pair or a
  drop set is **one** set, a warm-up is none.
- `planned` counts the prescribed sets of sessions still ahead — never
  started **and** dated `today` or later. A plan that was never trained stops
  counting once its day is over; a started or finished session contributes
  its log, never its plan.
- Muscles without any sets are omitted. The list is ordered by total volume,
  ties by name, `other` always last.
- The Block app's Tagebuch tab runs the same computation, so these numbers
  match what the user sees for the same week.

## License

MIT — see [LICENSE](LICENSE).
