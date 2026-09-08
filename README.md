# comment-resolver-skill

A [Claude Code](https://claude.com/claude-code) skill that semi-automatically works through the
open comments on a Bitbucket Cloud pull request — **from any author, bot or human, in English
or Bahasa Indonesia**. It fetches the comments, assesses each one against the code, asks you
which assessments are valid, then fixes / answers / replies / resolves — with a developer
confirmation gate in front of every outward-facing write.

> Skill name: `pr-comment-resolver`

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

This skill deliberately does not reinvent PR or commit machinery. It composes existing
Talenta SDLC skills:

| Concern | Delegated to |
| --- | --- |
| Bitbucket remote parsing + auth conventions | `pull-request` |
| Assessment lens (applicable vs. not) | `code-review` |
| How a fix is written | `coding-standards-frontend` / `coding-standards-backend` |
| Landing a fix (conventional commit, co-author trailer, push) | `commit-workflow` |

Only the PR *comment* endpoints (fetch / reply / resolve) are documented locally, in
[`references/bitbucket-comments-api.md`](references/bitbucket-comments-api.md).

## Layout

```
comment-resolver-skill/
├── SKILL.md                              # the skill definition + workflow
└── references/
    └── bitbucket-comments-api.md         # PR comment fetch / reply / resolve endpoints
```

## Install

Copy the skill into your Claude Code skills directory:

```sh
cp -R comment-resolver-skill ~/.claude/skills/pr-comment-resolver
```

Then trigger it in Claude Code with phrases like *"resolve the comments on PR #186"*,
*"go through the review comments on this PR"*, or by pinpointing a single comment by author
and text.

## Requirements

- Bitbucket Cloud credentials in the environment (a token with `pullrequest:write`, or a
  username + app password). The skill expects them via the `pull-request` skill's
  `.claude/.sdlc.env` convention.
- A local checkout on the PR's source branch when fixes are to be applied.

## Classification

INTERNAL — describes internal review tooling and workflow conventions.
