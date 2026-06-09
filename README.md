# skills

A collection of [Agent Skills](https://www.anthropic.com/news/skills) for AI agents.

The real skills live in `skills/`, and `SKILL.md` is plain Markdown — so any agent can read it directly, and the [`skills`](https://github.com/vercel-labs/skills) CLI can install it into Claude Code, Cursor, Codex, and others.

## Skills

- **[twitter-api-relay](skills/twitter-api-relay/SKILL.md)** — translate instructions into Twitter relay-server GraphQL / v1.1 calls and return a summarized result.

## Install

With the [`skills`](https://github.com/vercel-labs/skills) CLI (no clone needed):

```bash
# into the current project (.claude/skills/ etc.)
npx skills add fa0311/twitter_api_safe_relay_skills

# or globally, for every project (~/.claude/skills/)
npx skills add fa0311/twitter_api_safe_relay_skills -g

# preview what's in the repo without installing
npx skills add fa0311/twitter_api_safe_relay_skills --list
```

Or wire it in by hand from a clone — one source of truth, no duplication:

```bash
ln -s "$PWD/skills/twitter-api-relay" <tool-skills-dir>/twitter-api-relay
```

## Setup

This skill talks to a **relay server**, which is required and must be running first:

1. Stand up [fa0311/twitter_api_safe_relay](https://github.com/fa0311/twitter_api_safe_relay).
2. Point the skill at it via the `TWITTER_RELAY_BASE_URL` environment variable (see [`.env.example`](.env.example)) — e.g. `http://localhost:6900/`. Config is kept out of `SKILL.md` so the skill stays portable.

The relay holds the auth, so **the AI never sees your credentials or solves a captcha** — it only knows the operation catalog. That lets you hand Twitter API access to an AI securely, and with low token usage: the AI just picks an operation and edits a few variables instead of reasoning over the raw API.

## Updating `requests.ndjson`

The operation catalog (`skills/twitter-api-relay/requests.ndjson`) is captured from real traffic, not written by hand:

1. Run the dashboard server from [fa0311/twitter_api_safe_relay](https://github.com/fa0311/twitter_api_safe_relay).
2. Operate Twitter however you like — the requests show up as logs in the dashboard.
3. Hit the dashboard's **dump** button and paste the output into `skills/twitter-api-relay/requests.ndjson`.
