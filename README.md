# block-skills

Agent skills for [Block](https://block-bmg.pages.dev) — a workout and schedule
app whose data an AI agent can read and write through its MCP server,
`block-mcp`.

A skill is a set of instructions an AI coding agent (Claude Code, Cowork, or
any agent that reads the `SKILL.md` convention) loads when it recognises a
matching task. These skills teach an agent how to do things *with* Block that
the app's own UI doesn't cover.

## Skills

| Skill | What it does |
|---|---|
| [`block-log-import`](skills/block-log-import/SKILL.md) | Imports workout history into Block from any raw format — Strong/Hevy CSV exports, chat messages, training journals, photographed notes. Parses set notation, resolves exercise names against the library (including aliases), and writes sessions with import provenance. |

## Prerequisites

1. A Block account.
2. `block-mcp` connected to your agent, authenticated via OAuth. The skills
   call its tools (`list_exercises`, `get_workout_session`, `put_session_log`)
   and do nothing without them.

## Install

```bash
npx skills add chrs-dev/block-skills
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
