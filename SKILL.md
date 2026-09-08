---
name: ep-metrics-triage
description: >-
  Use this skill when a developer wants to work through the automated AI code-review comments
  that the "EP Metrics" bot posts on a Bitbucket Cloud pull request — triage each finding,
  fix the valid ones, reply with the outcome, and resolve the thread. Trigger phrases include
  "resolve the EP Metrics comments", "triage the AI review comments on this PR", "handle the
  bot review findings", "action the EP Metrics findings", "address the PR review comments",
  "go through the automated review on PR #N". Targets Bitbucket Cloud PRs. For each actionable
  finding it decides valid-fix / valid-wontfix / invalid, applies the fix through the normal
  build+commit path when valid, posts a threaded reply stating the outcome (referencing the
  commit for fixes), and resolves the inline thread — every outward-facing write gated behind
  developer confirmation. Composes pull-request (Bitbucket API + auth), code-review (validity
  lens), coding-standards-* (implementing fixes), and commit-workflow (landing them).
---

# EP Metrics Review-Comment Triage

> **Classification: INTERNAL.** Operates on private repos and PRs; treat all fetched
> content as internal-confidential. Never echo Bitbucket credentials (see Guardrails).

The "EP Metrics" bot runs an automated AI code review on Bitbucket PRs and leaves inline
findings (each tagged `[type, importance: n/9]`) plus a few non-actionable summary comments.
This skill takes those findings from **fetched → triaged → fixed/answered → replied →
resolved**, one thread at a time, with a human gate in front of every write.

It does **not** reinvent PR/commit machinery — it composes existing skills:

- **`pull-request`** → `references/bitbucket-api.md` for remote parsing + auth conventions
  (`.claude/.sdlc.env`, token/app-password precedence). This skill's
  `references/bitbucket-comments-api.md` only adds the *comment* endpoints.
- **`code-review`** → the rubric/lens for judging whether a finding is actually valid.
- **`coding-standards-frontend` / `coding-standards-backend`** → how a fix is written.
- **`commit-workflow`** → landing a fix (conventional message, co-author trailer, push,
  first-push Jira self-test). This skill never hand-rolls git writes.

## Pattern: triage loop with a write gate

Reading comments and reading code is safe. Editing code, committing, replying on the PR,
and resolving threads are outward-facing and hard to reverse — so they happen only after
the developer approves the triage plan. The skill presents a plan, waits for a "yes", then
executes and reports.

## Step 0 — Pre-flight

1. **Identify the PR.** From a URL the developer pastes
   (`…/{workspace}/{repo}/pull-requests/{id}`) or, if none given, resolve from the current
   branch: `GET …/pullrequests?q=source.branch.name="<branch>"&state=OPEN`. Confirm the
   matched PR title with the developer before proceeding.
2. **Load credentials** exactly as `pull-request/references/bitbucket-api.md` describes
   (`set -a && [ -f .claude/.sdlc.env ] && source .claude/.sdlc.env; set +a`). Required
   scopes here: `repository:read` (fetch), **`pullrequest:write`** (reply + resolve). If
   the vars are unset after sourcing, STOP and tell the developer what to fill in — never
   invent or print a secret.
3. **Local checkout must match the PR** to make fixes: on the PR's **source branch**, clean
   working tree, up to date with `origin`. If you're not going to fix anything (reply/resolve
   only), a checkout isn't required — but say so.

## Step 1 — Fetch and classify EP Metrics comments

```sh
curl -sS -u "$BITBUCKET_USERNAME:$BITBUCKET_APP_PASSWORD" \
  "https://api.bitbucket.org/2.0/repositories/$WORKSPACE/$REPO/pullrequests/$PR_ID/comments?pagelen=100"
```

(Or `Authorization: Bearer $BITBUCKET_TOKEN`.) Page through `next` links if present.

Filter to the bot: `user.display_name == "EP Metrics"` **and** `user.kind == "oauth_app"`.
Match on display name + kind, not a hardcoded UUID (the app UUID can differ per workspace).

Then split the bot's comments into two buckets — see
`references/bitbucket-comments-api.md` for the exact shapes:

- **Actionable findings** — inline comments (have `.inline.path` + `.inline.to`) whose body
  starts with `**[<type>, importance: n/9]**`. A finding may have a sibling **suggestion
  block** (`**Code suggestion [...]**` on the same path+line) — group them into one thread.
- **Non-actionable summaries** — the top-level `AI PR review started`, the raw-response
  dump (`Review for PR#…`), and the `✅ AI Code Review Complete` roll-up. These are **not**
  inline threads and are **not resolvable**. Leave them untouched; never delete them.

**Skip** any finding thread that is already `resolved` (`.resolution != null`) or that
already has a human reply after the bot — report these as "already handled".

## Step 2 — Triage each finding (validity)

For every actionable finding, open the referenced file+lines in the local checkout and
judge it with the `code-review` lens. Assign exactly one verdict:

| Verdict | Meaning | Action in Step 3–5 |
| --- | --- | --- |
| **VALID-FIX** | Finding is correct and worth fixing now. | Implement → commit → reply w/ commit → resolve. |
| **VALID-WONTFIX** | Real but intentional / out of scope / deferred (e.g. tracked as a separate ticket). | Reply explaining the decision → resolve (with developer OK). |
| **NEEDS-INFO** | Correct only under an assumption you can't verify locally (e.g. "confirm backend normalizes these names" — the MIN-1 style finding). | Reply stating what must be confirmed and by whom → **leave open** unless developer says resolve. |
| **INVALID** | False positive: wrong line, already handled, based on a wrong assumption, or not applicable to this code. | Reply with the concrete reason → resolve. |

Rules:
- Ground every verdict in the actual code you read — quote the line. Never accept or reject
  a finding on the bot's say-so alone.
- Do not fabricate CWE/CVE/severity or invent APIs in a fix — if a fix needs an unverified
  fact, it's NEEDS-INFO, not VALID-FIX.
- Low-importance style nits (importance ≤ 3) may be batched; call them out but don't
  over-engineer.

**→ Present the full triage table to the developer and wait for a "yes" before any write.**
Let them override any verdict.

## Step 3 — Fix the VALID-FIX findings

- Implement per `coding-standards-frontend` / `coding-standards-backend`. If the bot
  supplied a `code_suggestion`, use it as a **starting point** and verify it compiles /
  fits the surrounding code — don't paste blindly.
- Prefer **one commit per PR-review pass** grouping related fixes, unless the developer
  wants per-finding commits. Then hand off to **`commit-workflow`** to commit + push
  (it derives the conventional prefix from the Jira ticket and adds the co-author trailer).
- Capture the resulting **commit hash** and its web URL
  (`https://bitbucket.org/$WORKSPACE/$REPO/commits/<sha>`) — you'll cite it in the reply.
  Only ever reference a commit that actually landed on `origin`; never a fabricated hash.

## Step 4 — Reply to each thread

Post a threaded reply (parent = the finding's comment id) via
`POST …/pullrequests/$PR_ID/comments` with `{"content":{"raw":"…"},"parent":{"id":<id>}}`
(see `references/bitbucket-comments-api.md`). The reply is authored by whoever owns the
credentials — confirm that's the intended account.

Reply content by verdict:
- **VALID-FIX** — "Fixed in `<sha>` — <one line on what changed>." Link the commit.
- **VALID-WONTFIX** — state the decision and why (e.g. "intentional; tracked in TE-xxxx").
- **NEEDS-INFO** — state exactly what needs confirming and from whom.
- **INVALID** — the concrete reason (quote the code that disproves it).

Keep replies short, factual, grounded. No filler.

## Step 5 — Resolve

After the reply is posted (and, for fixes, the commit is pushed):

```sh
curl -sS -u "$BITBUCKET_USERNAME:$BITBUCKET_APP_PASSWORD" -X POST \
  "https://api.bitbucket.org/2.0/repositories/$WORKSPACE/$REPO/pullrequests/$PR_ID/comments/$COMMENT_ID/resolve"
```

- Resolve VALID-FIX, VALID-WONTFIX, and INVALID threads.
- Do **not** resolve NEEDS-INFO threads unless the developer explicitly says so.
- **Never resolve a thread you haven't replied to.** Reply first, resolve second.
- Reopen with `DELETE` on the same path if a resolve was premature.

## Step 6 — Report back

A single table: comment id · file:line · finding type/importance · verdict · action
(commit hash or reason) · reply link · resolved (y/n). Then a one-line summary
(e.g. "3 fixed, 1 wontfix, 1 left open for backend confirmation").

## Guardrails

- **Outward-facing writes are gated.** Editing code, committing/pushing, replying, and
  resolving happen only after the developer approves the Step-2 plan. No silent action.
- **Reply before resolve. Fix before "fixed".** Never claim a fix without a real pushed
  commit; never resolve a thread with no reply.
- **Don't touch the summary comments.** They aren't threads and aren't yours to delete.
- **Secrets stay in env.** Read `BITBUCKET_TOKEN` / `BITBUCKET_USERNAME` /
  `BITBUCKET_APP_PASSWORD` from the environment; never echo, log, or write them into a
  reply, commit, or file. An app password appearing in any input is a credential exposure —
  flag it and recommend rotation, don't use it inline.
- **Report the API truth.** Surface Bitbucket's actual 4xx/5xx on failure; don't paper over it.
- **Bitbucket Cloud only.** A non-`bitbucket.org` remote is out of scope — stop.

## Composing with other skills

- ← `pull-request` — reuse its `references/bitbucket-api.md` for remote parsing + auth; this
  skill handles the comment lifecycle that skill doesn't.
- ← `code-review` — the validity lens for Step 2.
- ← `coding-standards-frontend` / `coding-standards-backend` — how Step 3 fixes are written.
- → `commit-workflow` — lands every fix (never hand-roll git writes here).

## What this skill is NOT

- **Not a reviewer of its own.** It reacts to EP Metrics' findings; it doesn't generate a
  fresh review (that's `code-review` / `reviewer`).
- **Not a merger.** It never merges or deploys — that's `pull-request` / `release-pipeline`.
- **Not an auto-fixer.** Every fix and every resolve passes the developer gate first.
