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

## License

MIT — see [LICENSE](LICENSE).
