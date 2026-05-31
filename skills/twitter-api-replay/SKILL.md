---
name: twitter-api-replay
description: Read from and operate on a Twitter relay server (base URL in $TWITTER_REPLAY_BASE_URL). Use this whenever you get a Twitter-related instruction.
---

# twitter-api-replay

Translate instructions for the Twitter relay server into GraphQL / v1.1 calls, run them, and return a clear summary of the result.

- Base URL: environment variable `$TWITTER_REPLAY_BASE_URL` (e.g. `http://localhost:6900/`). If unset, ask the operator. Never hardcode the value into this file.
- Operation catalog: `requests.ndjson` in this directory (it is **swapped out mechanically**, so re-read it every time).
- No auth required. Responses are JSON.

## How to

Each line of `requests.ndjson` is one callable operation plus a concrete example of its variables. The tail of `path` is the operation name; `variables` holds the parameter example.

1. Pick the operation from the ndjson that matches the user's intent (you may infer meaning from the operation name).
2. Use that line as a template and rewrite **only the variables that correspond to the intent** (`screen_name` / `userId` / `rawQuery` / `count` / `cursor`, etc.). Keep `features`, `fieldToggles`, etc. exactly as they appear in the line.
   - **Variable names differ per operation. Always use the keys from that line's `variables` as-is** (e.g. to specify a tweet, `TweetDetail` uses `focalTweetId`, `Retweeters` uses `tweetId`, `GenericTimelineById` uses `timelineId`). Do not guess key names from other operations.
3. If an operation requires an internal ID (e.g. `userId`) and the user only gave an @handle, first call a resolver operation (e.g. `UserByScreenName`) and then call the real operation.
4. Return a summary that answers the question rather than raw JSON (paginate further with `cursor`).

## curl

How you build the URL depends on the kind of `path`. **Get the prefix wrong and you get a 404.** Use the environment variable `$TWITTER_REPLAY_BASE_URL` for the base (build it assuming no trailing slash).

- **GraphQL** (`path` is `/graphql/...`): `$TWITTER_REPLAY_BASE_URL` + **`/i/api`** + `path`
- **v1.1 REST** (`path` is `/1.1/...json`): `$TWITTER_REPLAY_BASE_URL` + `path` (**no** `/i/api` prefix)

```bash
# GraphQL GET: variables/features/fieldToggles in params are already JSON strings. Pass them verbatim, editing only the contents.
curl -sS -G "${TWITTER_REPLAY_BASE_URL%/}/i/api<path>" \
  --data-urlencode 'variables=<...>' --data-urlencode 'features=<...>'

# v1.1 GET example (no /i/api prefix)
curl -sS -G "${TWITTER_REPLAY_BASE_URL%/}<path>" \
  --data-urlencode 'variables=<...>' --data-urlencode 'features=<...>'

# GraphQL POST / v1.1 POST: send the data as a JSON blob
curl -sS "${TWITTER_REPLAY_BASE_URL%/}/i/api<path>" \
  -H 'content-type: application/json' --data '<data JSON>'

# v1.1 POST example (no /i/api prefix)
curl -sS "${TWITTER_REPLAY_BASE_URL%/}<path>" \
  -H 'content-type: application/json' --data '<data JSON>'
```


## Notes

- The current contents of the ndjson are always authoritative. Do not copy concrete values into this file.
- Confirm before running write (side-effecting) operations (create/delete tweet, follow/unfollow, etc.). Note that `HomeTimeline` / `HomeLatestTimeline` are reads even though they are POST — do not equate POST with writes.
- Do not take a write's response at face value; confirm with a read operation (e.g. `friendships/create` may return the `following` state from **before** the action, so confirm definitively with `friendships/show`).
