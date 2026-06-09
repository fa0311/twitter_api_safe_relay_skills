# skills

[Agent Skills](https://www.anthropic.com/news/skills) for AI agents. Each `SKILL.md` is plain Markdown, so any agent can read it — and the [`skills`](https://github.com/vercel-labs/skills) CLI can install it into Claude Code, Cursor, Codex, and more.

| Skill | What it does |
| --- | --- |
| [twitter-api-relay](skills/twitter-api-relay/SKILL.md) | Turn instructions into Twitter relay-server GraphQL / v1.1 calls, run them, and summarize the result. |

## Install

```bash
npx skills add fa0311/twitter_api_safe_relay_skills        # this project
npx skills add fa0311/twitter_api_safe_relay_skills -g     # all projects
npx skills add fa0311/twitter_api_safe_relay_skills --list # preview only
```

## Setup

`twitter-api-relay` needs a running relay server:

1. Start [fa0311/twitter_api_safe_relay](https://github.com/fa0311/twitter_api_safe_relay).
2. Set `TWITTER_RELAY_BASE_URL` to its address (e.g. `http://localhost:6900/`). See [`.env.example`](.env.example).

The relay keeps the auth, so the AI never touches your credentials or a captcha — it only sees the operation catalog. Secure to hand off, and cheap on tokens: the AI just picks an operation and tweaks a few variables.

## Updating the catalog

`requests.ndjson` is captured from real traffic, not written by hand:

1. Run the dashboard from [fa0311/twitter_api_safe_relay](https://github.com/fa0311/twitter_api_safe_relay).
2. Use Twitter as usual — requests appear in the dashboard.
3. Click **dump** and paste the output into [`skills/twitter-api-relay/requests.ndjson`](skills/twitter-api-relay/requests.ndjson).
