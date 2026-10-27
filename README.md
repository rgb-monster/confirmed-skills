# Confirmed skills

Agent skills for the [Confirmed](https://confirmed.show) API. With them installed, your coding agent knows the
endpoints, fields and conventions before it writes any code, so it doesn't have to read the docs first or guess.

## Install

```bash
npx skills add rgb-monster/confirmed-skills
```

This works with Claude Code, Cursor, Codex and the other agents the [skills](https://skills.sh) installer supports.

In Claude Code, you can also add it as a plugin:

```
/plugin marketplace add rgb-monster/confirmed-skills
/plugin install confirmed@confirmed-skills
```

## Skills

| Skill | Covers |
| --- | --- |
| [`confirmed-api`](skills/confirmed-api/SKILL.md) | Shows and their spots, show meta, ticket sales, show types, venues, acts, the team, file uploads and the public shows feed |
| [`confirmed-showtime`](skills/confirmed-showtime/SKILL.md) | Driving a live show: next, previous, break, and which show is current |

## API keys

You can generate a key on your production's [Data & API](https://confirmed.show/to/data) page. Keep it in an
environment variable such as `CONFIRMED_API_KEY`, and out of your code.

## Docs

The full documentation is at [confirmed.show/api-docs](https://confirmed.show/api-docs). For agents, the same pages
are available as markdown, indexed at [confirmed.show/llms.txt](https://confirmed.show/llms.txt).

If anything here doesn't match how the API behaves, write to [support@confirmed.show](mailto:support@confirmed.show).
