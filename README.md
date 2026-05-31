# skills

A collection of [Agent Skills](https://www.anthropic.com/news/skills) for AI agents.

The real skills live in `skills/`, and each tool references them via symlinks (e.g. `.claude/skills/` for Claude Code). One source of truth, no duplication. `SKILL.md` is plain Markdown, so any agent can also just read it directly.

```bash
# wire a skill into another tool
ln -s "$PWD/skills/twitter-api-replay" <tool-skills-dir>/twitter-api-replay
```

Config values are kept out of `SKILL.md` via environment variables — see `.env.example`.

## Skills

- **[twitter-api-replay](skills/twitter-api-replay/SKILL.md)** — translate instructions into Twitter relay-server GraphQL / v1.1 calls and return a summarized result.

**Requires a running relay server** — [fa0311/twitter_api_safe_relay](https://github.com/fa0311/twitter_api_safe_relay). Stand it up first, then point `TWITTER_REPLAY_BASE_URL` at it.

The relay holds the auth, so **the AI never sees your credentials or solves a captcha** — it only knows the operation catalog. That lets you hand Twitter API access to an AI securely, and with low token usage (the AI just picks an operation and edits a few variables instead of reasoning over the raw API).

### Updating `requests.ndjson`

The operation catalog is captured from real traffic, not written by hand:

1. Run the dashboard server from [fa0311/twitter_api_safe_relay](https://github.com/fa0311/twitter_api_safe_relay).
2. Operate Twitter however you like — the requests show up as logs in the dashboard.
3. Hit the dashboard's **dump** button and paste the output into `skills/twitter-api-replay/requests.ndjson`.
