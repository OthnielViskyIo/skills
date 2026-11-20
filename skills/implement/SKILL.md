---
name: "implement"
description: "Implement a piece of work based on a spec or set of tickets. Use only when the user explicitly writes /implement; never for a plain mention of the word."
---

Implement the work described in the spec or tickets.

## Break points

Before writing code, propose the break points: the PRs and the commits of each. Take them from the plan's Split section if it has one. Otherwise propose them yourself: subtasks that can be reviewed and merged on their own become separate PRs, and each PR gets several commits where the change naturally splits. Keep PRs small enough to review (not 60+ changed files), but don't split work that doesn't split naturally. Wait for the user to agree.

Work one PR at a time, on its own branch. Never create or check out a branch yourself: when the next PR needs a new branch, ask the user to create it.

At each break point, stop:

1. Run typechecking and the tests of the touched files.
2. Summarize what the commit holds, how the user can verify it, and propose a commit message.
3. Ask the user to verify the work. They commit it themselves or ask you to. Never push.

Don't continue until the commit is made. If a commit grows well beyond what was planned, propose an extra break point instead of carrying on.

## Building

Use `/tdd` where possible, at pre-agreed seams.

Run typechecking regularly, single test files regularly, and the full test suite once at the end of each PR.

When a PR's work is done, use `/code-review` to review it.
