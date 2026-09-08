# comment-resolver-skill

A skill that semi-automatically works through the open comments on a Bitbucket Cloud pull
request — **from any author, bot or human, in English or Bahasa Indonesia**. It fetches the
comments, assesses each one against the code, asks you which assessments are valid, then
fixes / answers / replies / resolves — with a developer confirmation gate in front of every
outward-facing write.

> Skill name: `pr-comment-resolver`

It's a plain-language procedure, **not tied to a specific coding agent** — it runs the same
under [Claude Code](https://claude.com/claude-code), Codex, Copilot, Gemini, Cursor, or any
assistant that can read files, run a shell (`curl` + `git`), and edit code. The SDLC skills it
composes (below) are agent-agnostic too.

## What it does

For a given Bitbucket PR, the skill runs an assess → adjudicate → act loop:

1. **Fetch** all PR comments and **drop the noise** — automation lifecycle/summary posts
   (e.g. EP Metrics' "AI PR review started" / "Finished AI Code Review"), already-resolved
   threads, and deleted comments. Everything that reads as feedback — inline findings *and*
   human review notes — is kept.
2. **Assess** every kept comment against the real code, using the `code-review` lens and the
   `coding-standards-*` (and other sdlc-generic-skills) relevant to the PR and repo, into:
   - `APPLICABLE` — a correct code change worth making now
   - `INAPPLICABLE` — no change warranted (intentional / out of scope / false positive), **or**
     it's a question to answer rather than a change to make
   - `UNDECIDED` — only judgeable under a fact that can't be verified from the code
3. **Present** the assessments and the reason behind each, and **ask you which are valid.**
   - Valid → the skill proceeds per the assessment.
   - Invalid → you tell the skill what to do next for that comment.
4. **Act** on each adjudicated comment:
   - *Valid + APPLICABLE* → fix the code, commit & push, reply with the fix summary **+ commit
     link** on your behalf, then resolve.
   - *Valid + INAPPLICABLE* → reply with the reason (or answer the question), then resolve.
   - *Invalid* → do what you directed.
   - *UNDECIDED* → reply stating what to confirm and by whom; leave open unless you say resolve.
5. **Report** a summary table of every comment, assessment, verdict, action, and resolution.

Replies **mirror the comment's language** (Bahasa Indonesia in, Bahasa Indonesia out). You can
always **reopen a resolved thread manually** in Bitbucket — the skill locks nothing. And when
a new comment lands later, you can **pinpoint one comment in natural language**, e.g.
*Assess comment from Yoga Prasetyo that says "dari QA expect pake data-test-id"*, without
re-running the whole PR.

## Guardrails

- **Every write is gated.** Editing code, committing/pushing, replying, and resolving happen
  only after you adjudicate the assessments — nothing runs silently.
- **Reply before resolve. Fix before "fixed."** No thread is resolved without a reply; no fix
  is claimed without a real pushed commit.
- **Replies are posted on your behalf** by the credential owner — the account is confirmed
  first.
- **Secrets stay in env.** Bitbucket credentials are read from environment variables only,
  never echoed, logged, or written into a reply, commit, or file.
- **Automation summary posts are left alone.** They aren't inline threads and aren't deleted.
- **Bitbucket Cloud only.**

## Composition

This skill deliberately does not reinvent PR machinery. It composes existing Talenta SDLC
skills for the parts they already own:

| Concern | Delegated to |
| --- | --- |
| Bitbucket remote parsing + auth conventions | `pull-request` |
| Assessment lens (applicable vs. not) | `code-review` |
| How a fix is written | `coding-standards-frontend` / `coding-standards-backend` |
| Landing a fix (commit + push) | **inline** — plain conventional-commit recipe, no dependency on any personal commit skill |

Committing is done inline on purpose: it keeps the skill portable into a shared skill set
without pulling in anyone's personal commit workflow. Only the PR *comment* endpoints (fetch /
reply / resolve) are documented locally, in
[`references/bitbucket-comments-api.md`](references/bitbucket-comments-api.md).

## Notes from field use

A dry run against a real PR surfaced a few behaviors worth calling out — they're baked into
`SKILL.md`:

- **Assessing is read-only.** The skill reads the exact PR code with
  `git show origin/<source-branch>:<path>` — it does **not** switch your local branch or touch
  your working tree just to look. A checkout on the source branch is needed only when a fix is
  actually applied.
- **Comments are located by content, not line number.** A comment's inline line anchor goes
  stale once the PR is pushed to again, so the skill matches on the code's *content*. A finding
  that no longer matches any current code is treated as already-handled (INAPPLICABLE), not a
  fix.
- **Suggestion siblings resolve independently.** A bot finding and its code-suggestion are
  often two separate threads with their own ids — the skill assesses them as one issue but
  resolves **both**, so no duplicate thread lingers open.
- **Assessments are presented as a Markdown table** (one row per issue, with the reason) so you
  can adjudicate valid/invalid at a glance before anything is written.

## Layout

```
comment-resolver-skill/
├── SKILL.md                              # the skill definition + workflow
└── references/
    └── bitbucket-comments-api.md         # PR comment fetch / reply / resolve endpoints
```

## Install

Install it however your agent loads skills or instructions — it's just Markdown:

- **Claude Code** — `cp -R comment-resolver-skill ~/.claude/skills/pr-comment-resolver`, then
  trigger with phrases like *"resolve the comments on PR #186"* or *"go through the review
  comments on this PR"*, or by pinpointing a single comment by author and text.
- **Codex** — reference `SKILL.md` from `AGENTS.md`.
- **Cursor** — add it as a project rule under `.cursor/rules/`.
- **Copilot** — reference it from `.github/copilot-instructions.md`.
- **Other agents** — point the assistant at `SKILL.md` (and `references/`).

However it's loaded, ask the assistant to work through the PR's comments following `SKILL.md`.

## Requirements

- Bitbucket Cloud credentials in the environment (a token with `pullrequest:write`, or a
  username + app password). The skill expects them via the `pull-request` skill's
  `.claude/.sdlc.env` convention.
- A local checkout on the PR's source branch when fixes are to be applied.

## Classification

INTERNAL — describes internal review tooling and workflow conventions.
