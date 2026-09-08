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

## Identifying EP Metrics comments

The bot is an OAuth app user:

```jsonc
"user": {
  "type": "app_user",
  "display_name": "EP Metrics",
  "kind": "oauth_app",
  "uuid": "{…}"          // can differ per workspace — do NOT hardcode; match display_name + kind
}
```

### Actionable finding (inline)

```jsonc
{
  "id": 858269858,
  "content": { "raw": "**[possible_bugs, importance: 6/9]** The discriminator … " },
  "inline": { "path": "src/pages/attendance-setting/FormShift.vue", "to": 195, "from": null },
  "resolution": null,     // null = open; object = already resolved (skip)
  "deleted": false
}
```

- Body starts with `**[<type>, importance: n/9]**`. Types seen: `possible_bugs`,
  `error_handling`, and others the bot emits.
- `inline.path` + `inline.to` (line) locate the code. `inline.to` is the line on the "to"
  (new) side of the diff.

### Suggestion sibling (inline, same path+line)

```jsonc
{
  "id": 858269901,
  "content": { "raw": "**Code suggestion [error_handling, importance: 5/9]:** …\n```suggestion\n<code>\n```" },
  "inline": { "path": "…", "to": 675 }
}
```

Group a suggestion with the finding at the same `path`+`to` into one logical thread.

### Non-actionable summaries (top-level, NOT resolvable)

No `inline` key. Recognizable bodies:
- `🤖 **AI PR review started** (analysis ID: …)`
- `# Review for PR#… ` + a fenced raw-model-response dump
- `## ✅ AI Code Review Complete` roll-up

Leave these alone. They are not threads → the `/resolve` endpoint does not apply → and they
are not yours to delete.

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
