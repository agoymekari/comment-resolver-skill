# Bitbucket Cloud REST API — PR comments (fetch / reply / resolve)

Base: `https://api.bitbucket.org/2.0`. Bitbucket **Cloud** only. For remote→`{workspace}/{repo}`
parsing and auth (token vs. app password, `.claude/.sdlc.env` loading, required scopes),
reuse `pull-request/references/bitbucket-api.md` — not repeated here. Reply + resolve need
`pullrequest:write`.

Auth in every example below is either:
```
-H "Authorization: Bearer $BITBUCKET_TOKEN"      # preferred
-u "$BITBUCKET_USERNAME:$BITBUCKET_APP_PASSWORD" # app password, pullrequest:write
```

## Fetch all comments

```sh
curl -sS -u "$BITBUCKET_USERNAME:$BITBUCKET_APP_PASSWORD" \
  "https://api.bitbucket.org/2.0/repositories/$WORKSPACE/$REPO/pullrequests/$PR_ID/comments?pagelen=100"
```

Response is `{ "values": [ … ], "next": "<url?>" }` — follow `next` until absent.

## Classifying a fetched comment

Comments come from **any** author — a bot or a human — and the skill treats them the same.
The fields that matter for classification:

### Author (bot vs. human)

```jsonc
// bot / automation — an OAuth app user
"user": { "type": "app_user", "display_name": "EP Metrics", "kind": "oauth_app", "uuid": "{…}" }

// human reviewer — a normal account
"user": { "type": "user", "display_name": "Yoga Prasetyo", "account_id": "…" }
```

Author is used for **display and for natural-language pinpointing** (match `display_name`
fuzzily + a `content.raw` substring), *not* to decide whether a comment is actionable — a
human note and a bot finding are both fair game. Never hardcode a bot's `uuid` (it can differ
per workspace); match on `display_name` + `kind` when you do need to name the bot.

### Inline thread (locates code) vs. top-level note

```jsonc
{
  "id": 858269858,
  "content": { "raw": "…the discriminator may be undefined here…" },
  "inline": { "path": "src/pages/attendance-setting/FormShift.vue", "to": 195, "from": null },
  "resolution": null,     // null = open; object = already resolved (skip unless pinpointed)
  "deleted": false
}
```

- An **inline** comment has `.inline.path` + `.inline.to` (the line on the "to"/new side of
  the diff) — that locates the code to assess.
- A **top-level** comment has no `inline` key. It may still be actionable (a general review
  note) — assess it against the relevant code — or it may be automation noise (below).
- Comment bodies may be in **English or Bahasa Indonesia**.

### Bot finding format (one recognized pattern, not required)

Some bots tag findings, e.g. EP Metrics uses `**[<type>, importance: n/9]**` (`possible_bugs`,
`error_handling`, …) and may post a sibling **suggestion block** at the same `path`+`to`:

```jsonc
{ "id": 858269901,
  "content": { "raw": "**Code suggestion […]:** …\n```suggestion\n<code>\n```" },
  "inline": { "path": "…", "to": 675 } }
```

Group a suggestion with the finding at the same `path`+`to` into one logical thread. Treat the
tag/importance as a hint, not a gate — an untagged human comment is assessed the same way.

### Automation noise to DROP (top-level, NOT resolvable)

No `inline` key, and no ask to act on — just lifecycle/summary status. Examples (EP Metrics):
- `🤖 **AI PR review started** (analysis ID: …)`
- `# Review for PR#… ` + a fenced raw-model-response dump
- `## ✅ AI Code Review Complete` / "Finished AI Code Review" roll-up

Leave these alone. They are not threads → the `/resolve` endpoint does not apply → and they
are not yours to delete. Also skip already-resolved (`resolution != null`, unless pinpointed)
and deleted (`deleted == true`) comments.

## Reply to a comment (threaded)

```sh
curl -sS -u "$BITBUCKET_USERNAME:$BITBUCKET_APP_PASSWORD" -X POST \
  "https://api.bitbucket.org/2.0/repositories/$WORKSPACE/$REPO/pullrequests/$PR_ID/comments" \
  -H "Content-Type: application/json" \
  -d '{"content":{"raw":"Fixed in <sha> — build payload without mutating body."},"parent":{"id":858269881}}'
```

- `parent.id` makes it a threaded reply (omit for a new top-level comment).
- The reply is authored by the **credential owner** — confirm that's the intended account.
- Response includes the new comment `id`, `user.display_name`, and `links.html.href`
  (deep link to the comment in the diff). Report the html link.
- `content.raw` is markdown; a fenced ```suggestion block renders as an applyable suggestion.

## Resolve / reopen a thread

```sh
# resolve
curl -sS -u "$BITBUCKET_USERNAME:$BITBUCKET_APP_PASSWORD" -X POST \
  "https://api.bitbucket.org/2.0/repositories/$WORKSPACE/$REPO/pullrequests/$PR_ID/comments/$COMMENT_ID/resolve"

# reopen (undo)
curl -sS -u "$BITBUCKET_USERNAME:$BITBUCKET_APP_PASSWORD" -X DELETE \
  "https://api.bitbucket.org/2.0/repositories/$WORKSPACE/$REPO/pullrequests/$PR_ID/comments/$COMMENT_ID/resolve"
```

- Resolve targets the **finding's own comment id** (the inline thread root), not a reply.
- After a successful resolve, a re-fetch shows `resolution` populated (who/when).
- A `404`/`400` here usually means the id isn't an inline thread (e.g. you aimed at a
  top-level summary comment) — don't retry blindly; re-check the id.

## Failure notes

- `403` on reply/resolve → the credential lacks `pullrequest:write`. Stop; tell the
  developer to widen the token/app-password scope. Do not print the credential.
- `401` → bad/expired credential. Same: stop and report, never echo the value.
- Always surface Bitbucket's actual JSON error `message` to the developer verbatim.
