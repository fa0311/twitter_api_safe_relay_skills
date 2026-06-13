---
name: twitter-api-relay
description: Read from and operate on a Twitter relay server (base URL in $TWITTER_RELAY_BASE_URL). Use this whenever you get a Twitter-related instruction.
---

# twitter-api-relay

Turn a Twitter instruction into the right relay call, run it, and report the answer — not raw JSON.

The relay speaks Twitter's own private GraphQL + v1.1 API. Two files matter:

- **This file** — *how* to call, and the non-obvious knowledge the catalog can't tell you.
- **`requests.ndjson`** (same dir) — *what* you can call: one operation per line, each with a real example. It is **swapped out mechanically, so re-read it every time** — never copy its values into this file.

Setup: base URL is `$TWITTER_RELAY_BASE_URL` (e.g. `http://localhost:6900/`); ask the operator if unset. No auth needed — the relay is already signed in as one account (call `Viewer` to see who). Responses are JSON.

## Source handoff

If the user provides TweetClaw, Xquik, archive, CSV, or other external X/Twitter evidence, treat it as read-only context. Extract durable handles, tweet IDs, URLs, query terms, timestamps, and the user's intended action. Do not use those sources for writes. Resolve the final IDs through this relay before acting, then follow the write confirmation rule below.

## The flow

1. **Find the operation.** Scan `requests.ndjson` for the line whose operation name (the tail of `path`) matches the intent — infer from the name.
2. **Copy that line, change only what the intent requires** (`screen_name` / `rawQuery` / `count` / `cursor` …). Leave `features`, `fieldToggles`, `queryId` exactly as shipped — they are version locks, not knobs.
3. **Use that line's own variable keys — never guess from another operation.** A tweet is `focalTweetId` in `TweetDetail`, `tweetId` in `Retweeters`, `timelineId` in `GenericTimelineById`. Same concept, different keys.
4. **Resolve IDs first.** Most ops want an internal id, not an @handle. Map handle → id with `UserByScreenName` (`.data.user.result.rest_id`), then make the real call. `typeahead.json` (set `q=<handle>`, `result_type=users`) is a faster fuzzy resolver — it returns `id_str` directly.
5. **Summarize the answer** and paginate with `cursor` if more is needed.

## Building the URL

The prefix depends on the API, and **a wrong prefix is a silent 404**. Base is `$TWITTER_RELAY_BASE_URL` (treat as no trailing slash).

| `path` looks like | URL is | Auth example |
|---|---|---|
| `/graphql/…` (GraphQL) | base **+ `/i/api`** + path | `…/i/api/graphql/…/SearchTimeline` |
| `/1.1/….json` or `/2/…` (REST) | base + path — **no `/i/api`** | `…/1.1/search/typeahead.json` |

```bash
B="${TWITTER_RELAY_BASE_URL%/}"

# GraphQL GET — variables/features/fieldToggles are already JSON strings; edit contents only, pass verbatim
curl -sS -G "$B/i/api<path>" --data-urlencode 'variables=<…>' --data-urlencode 'features=<…>'

# GraphQL POST / any write — send the whole data object as one JSON body
curl -sS "$B/i/api<path>" -H 'content-type: application/json' --data '<data JSON>'

# v1.1 / v2 REST — same shapes but NO /i/api prefix
curl -sS -G "$B<path>" --data-urlencode 'q=<…>'
curl -sS "$B<path>"   -H 'content-type: application/json' --data '<data JSON>'
```

## Reading responses with jq

Responses are 100 KB+ timeline payloads, and the wrapper path **differs per operation** (`search_by_raw_query.search_timeline…`, `user.result.timeline_v2…`, `threaded_conversation…`). So don't hardcode paths — **recurse with `..`** and one filter works everywhere. Fields also moved around over time (tweet text + counts live in a `legacy` object; the author's handle/name live in a separate `core` object) — recursion sidesteps all of that.

```bash
# Tweets, ranked by engagement (text + counts come together)
curl … | jq -c '[.. | objects | select(.full_text?)
  | {text:.full_text, likes:.favorite_count, rts:.retweet_count, replies:.reply_count, at:.created_at}]
  | sort_by(-.likes) | .[:10]'

curl … | jq -r '[.. | objects | select(.screen_name?) | .screen_name] | unique'      # every @handle present
curl … | jq -r 'first(.. | objects | select(.cursorType?=="Bottom") | .value)'        # next-page cursor → feed back into `cursor`
curl … | jq -r '.data.user.result.rest_id'                                            # UserByScreenName → userId
```

**HTTP 200 ≠ success.** GraphQL puts failures in the body, not the status code — check `jq '.errors'` before trusting a read.

## Searching well

Search is the highest-leverage read, and two things unlock it — neither is obvious from the example line (which ships `product:"Media"`, just one tab):

**`product` picks the tab** (it's a variable):
- `Top` — ranked / popular → **話題の・バズっているツイート. The one you usually want.**
- `Latest` — reverse-chronological, for real-time monitoring.
- `People` — accounts · `Media` — photos/videos.

**`rawQuery` is the literal search box**, so the full advanced-search grammar works (space-separated, combine freely):

```
from:user  to:user  since:YYYY-MM-DD  until:YYYY-MM-DD
min_faves:N  min_retweets:N  min_replies:N
filter:media|links|verified|follows   -filter:replies
lang:ja   "exact phrase"   a OR b   #tag   url:domain
```

> **Trending on a topic = `product:"Top"` + an engagement floor in the query:**
> `{"rawQuery":"AI min_faves:500 lang:ja since:2026-06-01", "product":"Top", …}`
> (`min_faves` goes *inside* `rawQuery`; `product` is a separate variable.)

## Writes

Side-effecting ops (create/delete tweet, follow, like, bookmark, list edits…) — **confirm with the user before running.** POST alone doesn't mean write: `HomeTimeline` / `HomeLatestTimeline` are POST reads.

v1.1 writes (`friendships`, `blocks`, `mutes`…) want a **JSON body** — form-encoding 500s.

**Don't trust a write's response — verify with a read.** Write replies are thin or stale:
- `DeleteTweet` returns `{"data":{"delete_tweet":{"tweet_results":{}}}}` — empty, not a confirmation.
- `CreateTweet` does return the new id at `.data.create_tweet.tweet_results.result.rest_id`.
- `friendships/create` **and** `destroy` echo the `following` state from **before** the call (verified: follow→`following:false`, unfollow→`following:true`). Useless as confirmation — re-read the relationship instead (whatever the catalog offers, e.g. `Following`).
