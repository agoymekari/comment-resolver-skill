---
name: pr-comment-resolver
description: >-
  Use this skill to semi-automatically work through the unresolved comments on a Bitbucket
  Cloud pull request — whether left by a review bot (e.g. the "EP Metrics" AI reviewer) or by
  a human reviewer, in English or Bahasa Indonesia. Trigger phrases include "resolve the PR
  comments", "go through the comments on this PR", "triage the review comments", "address the
  PR feedback", "handle the comments on PR #N", or pinpointing one comment by author + text
  ("assess the comment from Yoga Prasetyo that says 'dari QA expect pake data-test-id'"). It
  fetches the PR comments, drops non-actionable noise (bot lifecycle messages like "AI PR
  review started" / "Finished AI Code Review"), assesses each remaining comment as applicable
  or inapplicable using the sdlc-generic-skills relevant to the PR and repo, presents the
  assessments with reasons, and asks which are valid. For a valid + applicable comment it
  fixes the code, commits & pushes, replies with the fix summary + commit link on your behalf,
  then resolves the thread; for a valid + inapplicable comment it replies with the reason (or
  the answer, if it's a question) and resolves; for an assessment you mark invalid it does what
  you direct next. Every outward-facing write is gated behind your confirmation. Targets
  Bitbucket Cloud. Composes pull-request (Bitbucket API + auth), code-review (assessment lens),
  coding-standards-* (fixes), and commit-workflow (landing them).
---

# PR Comment Resolver

> **Classification: INTERNAL.** Operates on private repos and PRs; treat all fetched
> content as internal-confidential. Never echo Bitbucket credentials (see Guardrails).

This skill works through the open comments on a Bitbucket Cloud PR — from **any** author, bot
or human — and takes each one from **fetched → assessed → adjudicated by you → fixed/answered
→ replied → resolved**, one thread at a time, with a human gate in front of every write.

It does **not** care who wrote the comment. A finding from an automated reviewer (e.g. the
"EP Metrics" AI bot) and a review note from a teammate go through the same loop. Comments may
be in **English or Bahasa Indonesia**; the skill understands both.

It is **not tied to a specific coding agent.** This is a plain-language procedure — any
assistant that can read files, run a shell (`curl` + `git`), and edit code can execute it. The
skills it composes below are themselves agent-agnostic, so "composes X" means "run procedure X"
however your environment provides it.

It does **not** reinvent PR/commit machinery — it composes existing skills:

- **`pull-request`** → `references/bitbucket-api.md` for remote parsing + auth conventions
  (`.claude/.sdlc.env`, token/app-password precedence). This skill's
  `references/bitbucket-comments-api.md` only adds the *comment* endpoints.
- **`code-review`** → the lens for assessing whether a comment is applicable to the code.
- **`coding-standards-frontend` / `coding-standards-backend`** → how a fix is written.
- **`commit-workflow`** → landing a fix (conventional message, co-author trailer, push,
  first-push Jira self-test). This skill never hand-rolls git writes.

## Pattern: assess → you adjudicate → act, with a write gate

Reading comments and reading code is safe. Editing code, committing, replying on the PR, and
resolving threads are outward-facing and hard to reverse — so they happen only after **you**
confirm. The skill assesses every comment, shows you its reasoning, and asks which assessments
are valid. Nothing is written until you say so.

## Step 0 — Pre-flight

1. **Identify the PR.** From a URL you paste (`…/{workspace}/{repo}/pull-requests/{id}`) or,
   if none given, resolve from the current branch:
   `GET …/pullrequests?q=source.branch.name="<branch>"&state=OPEN`. Confirm the matched PR
   title with you before proceeding.
2. **Load credentials** exactly as `pull-request/references/bitbucket-api.md` describes
   (`set -a && [ -f .claude/.sdlc.env ] && source .claude/.sdlc.env; set +a`). Required
   scopes: `repository:read` (fetch), **`pullrequest:write`** (reply + resolve). If the vars
   are unset after sourcing, STOP and tell the developer what to fill in — never invent or
   print a secret.
3. **Local checkout must match the PR** to make fixes: on the PR's **source branch**, clean
   working tree, up to date with `origin`. If you're only replying/resolving (no code fix),
   a checkout isn't required — but say so.
4. **Confirm the reply account.** Replies are authored by whoever owns the credentials, and
   they are posted **on your behalf**. Confirm that's the intended account before any reply.

## Step 1 — Fetch and drop the noise

```sh
curl -sS -u "$BITBUCKET_USERNAME:$BITBUCKET_APP_PASSWORD" \
  "https://api.bitbucket.org/2.0/repositories/$WORKSPACE/$REPO/pullrequests/$PR_ID/comments?pagelen=100"
```

(Or `Authorization: Bearer $BITBUCKET_TOKEN`.) Page through `next` links if present. See
`references/bitbucket-comments-api.md` for the exact comment shapes.

**Drop** — never assess, never touch — these:

- **Automation lifecycle / summary noise.** Bot status posts that aren't a thread to act on:
  e.g. EP Metrics' `🤖 AI PR review started`, the raw-response dump (`Review for PR#…`), and
  the `✅ AI Code Review Complete` / "Finished AI Code Review" roll-up. These are top-level,
  not inline threads, and are **not resolvable**. Leave them; never delete them.
- **Already-resolved threads** (`resolution != null`) — unless the developer pinpoints one
  explicitly (Step 5b).
- **Deleted comments** (`deleted == true`).

**Keep** everything that reads as feedback or a request — inline findings *and* human review
notes, from bot or person. A bot finding may have a sibling **suggestion block** (a
```suggestion fence on the same path+line); group it into the same thread. Pure
acknowledgements with no ask ("LGTM", "makasih") aren't dropped silently — they're assessed
as inapplicable / no-action so you still see them.

## Step 2 — Assess each kept comment

For every kept comment, open the referenced file+lines (for inline comments) or the relevant
code (for top-level notes) in the local checkout, and assess it with the `code-review` lens
plus the `coding-standards-*` and any other sdlc-generic-skills relevant to this PR and repo.
Assign exactly one assessment:

| Assessment | Meaning | If you mark it VALID (Step 4) |
| --- | --- | --- |
| **APPLICABLE** | The comment calls for a real code change that is correct and worth making now. | Fix → commit+push → reply w/ summary + commit link → resolve. |
| **INAPPLICABLE** | No code change is warranted: it's intentional / out of scope / a false positive / already handled, **or** it's a question to answer rather than a change to make. | Reply with the reason (or the answer) → resolve. |

If a comment can only be judged under a fact you **can't verify from the code** (e.g. "confirm
the backend normalizes these names"), don't force a verdict — mark it **UNDECIDED**, state in
the reason exactly what must be confirmed and by whom, and leave that thread for the developer
to direct in Step 4. Never guess to fill the table.

Rules:
- Ground every assessment in the actual code you read — quote the line. Never accept or reject
  a comment on the author's say-so alone.
- Do not fabricate CWE/CVE/severity or invent APIs in a fix — if a fix needs an unverified
  fact, it's UNDECIDED, not APPLICABLE.
- Comments may be in **Bahasa Indonesia** — assess them on the same footing as English ones.
- Low-value style nits may be batched; call them out but don't over-engineer.

## Step 3 — Present the assessments and reasons

Show the developer a table: comment id · author · file:line (or "top-level") · a one-line
gist of the comment · **assessment** · **the reason behind it**. Keep it scannable.

**→ Then ask which assessments are valid, and wait.** The developer adjudicates each row:

- **Valid** — they agree with the assessment; the skill proceeds per the table in Step 4.
- **Invalid** — they disagree; they tell the skill **what to do next** for that comment
  (e.g. "actually fix it", "reply that it's deferred to TE-1234, then resolve", "leave it").
  The skill follows that instruction instead of the assessed action.

No write happens for any comment until it's been adjudicated here.

## Step 4 — Act on the adjudicated comments

For each comment, run the branch its (validity, assessment) lands on:

### 4a — Valid + APPLICABLE → fix, commit, reply, resolve

- Implement the fix per `coding-standards-frontend` / `coding-standards-backend`. If a bot
  supplied a `code_suggestion`, use it as a **starting point** and verify it compiles / fits
  the surrounding code — don't paste blindly.
- Prefer **one commit per pass** grouping related fixes, unless the developer wants per-comment
  commits. Hand off to **`commit-workflow`** to commit + push (it derives the conventional
  prefix from the Jira ticket and adds the co-author trailer). Never hand-roll git writes here.
- Capture the resulting **commit hash** and its web URL
  (`https://bitbucket.org/$WORKSPACE/$REPO/commits/<sha>`). Only ever cite a commit that
  actually landed on `origin`; never a fabricated hash.
- **Reply** on the thread (parent = the comment's id) with a short fix summary **plus the
  commit link**, posted on the developer's behalf. Then **resolve** the thread.

### 4b — Valid + INAPPLICABLE → reply (reason or answer), resolve

- **Reply** on the thread stating the concrete reason it needs no change (quote the code that
  disproves it / explains the intent), **or**, if the comment was a question, answer it.
- Then **resolve** the thread.

### 4c — Invalid assessment → do what the developer directed

- Execute the developer's Step-3 instruction for that comment (fix it after all, reply-only,
  reply-and-resolve, leave open, etc.). Same write discipline applies.

### 4d — UNDECIDED → leave open unless directed

- Reply stating what must be confirmed and by whom. **Leave the thread open** unless the
  developer explicitly says to resolve it.

**Reply language:** mirror the comment — reply in Bahasa Indonesia to an Indonesian comment,
in English to an English one. Keep replies short, factual, grounded. No filler.

**Reply before resolve. Fix before "fixed".** Never resolve a thread you haven't replied to;
never claim a fix without a real pushed commit.

Resolve / reopen endpoints (see `references/bitbucket-comments-api.md`):

```sh
# resolve
curl -sS -u "$BITBUCKET_USERNAME:$BITBUCKET_APP_PASSWORD" -X POST \
  "https://api.bitbucket.org/2.0/repositories/$WORKSPACE/$REPO/pullrequests/$PR_ID/comments/$COMMENT_ID/resolve"
```

The developer can always **reopen a resolved thread manually** in Bitbucket — without the
skill's involvement — if a resolution was premature. The skill locks nothing.

## Step 5 — Report back

A single table: comment id · author · file:line · assessment · your verdict · action taken
(commit hash / reply reason) · reply link · resolved (y/n). Then a one-line summary
(e.g. "3 fixed, 1 answered, 1 left open for backend confirmation").

## Step 5b — Pinpoint a single comment later (natural language)

New comments can land after a session. The developer can point the skill at **one specific
comment in natural language** — by author and a snippet of its text — instead of re-running
the whole PR. Example:

> Assess comment from Yoga Prasetyo that says "dari QA expect pake data-test-id"

Resolve it by matching on **`user.display_name`** (fuzzy) **+ a `content.raw` substring** (the
quoted text, language-agnostic). Confirm the matched comment (id, author, body) with the
developer before assessing, then run Steps 2 → 4 for that one thread.

## Guardrails

- **Outward-facing writes are gated.** Editing code, committing/pushing, replying, and
  resolving happen only after the developer adjudicates the Step-3 assessments. No silent
  action.
- **Reply before resolve. Fix before "fixed".** Never claim a fix without a real pushed
  commit; never resolve a thread with no reply.
- **Don't touch the automation noise.** Lifecycle/summary posts aren't threads and aren't
  yours to delete.
- **Replies are on the developer's behalf.** Confirm the credential account first; mirror the
  comment's language; keep it grounded.
- **Secrets stay in env.** Read `BITBUCKET_TOKEN` / `BITBUCKET_USERNAME` /
  `BITBUCKET_APP_PASSWORD` from the environment; never echo, log, or write them into a reply,
  commit, or file. An app password appearing in any input is a credential exposure — flag it
  and recommend rotation, don't use it inline.
- **Report the API truth.** Surface Bitbucket's actual 4xx/5xx on failure; don't paper over it.
- **Bitbucket Cloud only.** A non-`bitbucket.org` remote is out of scope — stop.

## Composing with other skills

- ← `pull-request` — reuse its `references/bitbucket-api.md` for remote parsing + auth; this
  skill handles the comment lifecycle that skill doesn't.
- ← `code-review` — the assessment lens for Step 2.
- ← `coding-standards-frontend` / `coding-standards-backend` — how Step 4a fixes are written.
- → `commit-workflow` — lands every fix (never hand-roll git writes here).

## What this skill is NOT

- **Not a reviewer of its own.** It reacts to existing comments; it doesn't generate a fresh
  review (that's `code-review` / `reviewer`).
- **Not a merger.** It never merges or deploys — that's `pull-request` / `release-pipeline`.
- **Not an auto-fixer.** Every fix and every resolve passes the developer gate first.
- **Not bot-specific.** It resolves comments from any author; a specific bot's format (e.g.
  EP Metrics' `[type, importance: n/9]` tags) is just one pattern it recognizes, not its
  reason for existing.
