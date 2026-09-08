# comment-resolver-skill

A [Claude Code](https://claude.com/claude-code) skill that works through the automated
AI code-review comments left by the **"EP Metrics"** bot on a Bitbucket Cloud pull request:
it triages each finding, fixes the valid ones, replies with the outcome, and resolves the
thread — with a developer confirmation gate in front of every outward-facing write.

> Skill name: `ep-metrics-triage`

## What it does

For a given Bitbucket PR, the skill runs a triage loop:

1. **Fetch** all PR comments and filter to the EP Metrics bot, separating actionable inline
   findings (each tagged `[type, importance: n/9]`) from the non-actionable summary comments.
2. **Triage** every finding against the real code into one of four verdicts:
   - `VALID-FIX` — correct and worth fixing now
   - `VALID-WONTFIX` — real but intentional / out of scope / deferred
   - `NEEDS-INFO` — correct only under an assumption that can't be verified locally
   - `INVALID` — false positive / not applicable
3. **Fix** the `VALID-FIX` findings following the repo's coding standards.
4. **Reply** on each thread with the outcome — fixes cite the actual commit; everything else
   states a grounded reason.
5. **Resolve** the handled threads (reply first, resolve second; `NEEDS-INFO` left open by
   default).
6. **Report** a summary table of every finding, verdict, action, and resolution.

## Guardrails

- **Every write is gated.** Editing code, committing/pushing, replying, and resolving happen
  only after the developer approves the triage plan — nothing runs silently.
- **Reply before resolve. Fix before "fixed."** No thread is resolved without a reply; no fix
  is claimed without a real pushed commit.
- **Secrets stay in env.** Bitbucket credentials are read from environment variables only,
  never echoed, logged, or written into a reply, commit, or file.
- **Summary comments are left alone.** They aren't inline threads and aren't deleted.
- **Bitbucket Cloud only.**

## Composition

This skill deliberately does not reinvent PR or commit machinery. It composes existing
Talenta SDLC skills:

| Concern | Delegated to |
| --- | --- |
| Bitbucket remote parsing + auth conventions | `pull-request` |
| Validity lens for triage | `code-review` |
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
cp -R comment-resolver-skill ~/.claude/skills/ep-metrics-triage
```

Then trigger it in Claude Code with phrases like *"triage the EP Metrics comments on
PR #186"* or *"action the automated review findings on this PR."*

## Requirements

- Bitbucket Cloud credentials in the environment (a token with `pullrequest:write`, or a
  username + app password). The skill expects them via the `pull-request` skill's
  `.claude/.sdlc.env` convention.
- A local checkout on the PR's source branch when fixes are to be applied.

## Classification

INTERNAL — describes internal review tooling and workflow conventions.
