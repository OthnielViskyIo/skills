---
name: pr-flow
description: "Summarize the changes on the current git branch as a short 'flow' for a PR description, so a reviewer gets an overview before reading the diff. Use when the user asks for a flow, a PR overview, or a summary of the branch changes."
inputs:
  - base: "Branch to compare against (optional). Defaults to the repo's main branch (main or master)."
---

Write a "flow": a short walk through the changes on the current branch, in the order a request, event or user action travels through them. It goes into a PR description. The reviewer should know where to start reading and how the pieces connect before opening the diff.

## Gather the changes

1. Determine [base]. If none is given, use `main` or `master`, whichever exists.
2. Run `git log --oneline <base>..HEAD` and `git diff --stat <base>...HEAD` to see the commits and touched files.
3. Read the full diff of the source files, and the new files in full. Read surrounding code where you need it to understand how a change is reached.
4. If there are uncommitted changes, say so and ask whether to include them. Never include them silently.
5. If some commits clearly belong to other work (e.g. another ticket key, or a merge from another branch), leave them out of the flow and list them after it.

## Choose the unit

Pick whatever gives the clearest overview:

- Backend: layers (controller → service/use case → domain → infrastructure) or the classes in the call chain
- Frontend: pages → components → hooks/state → API calls
- Config, infra or scripts: files or steps in the pipeline
- Mixed or scattered changes: group by concern

Start at the entry point (endpoint, page, job, CLI command) and follow the path to the response or side effect. Shared or refactored pieces get their own step where they are first used.

## Write it

- Start with the heading `Flow:`, then one step per stage, each starting with a short label: `**Request entry:** ...`, `**Orchestration:** ...`, `**Response:** ...`.
- Each step is 1–3 sentences. Name the files or classes in backticks so the reviewer can find them in the diff.
- Describe what each piece does in the flow and the decisions that matter (short-circuits, priorities, what is shared or reused, what is deliberately left alone). Leave out the details the diff already shows.
- Skip tests, lockfiles, version bumps and pure renames, unless they are the point of the change.
- State only what the code does. If something looks unintended, raise it with the user after the flow, not inside it.

## Length limits

Count words, not characters: long identifiers would distort a character count.

- Hard cap: 250 words for the whole flow.
- Target: ~150 words for small branches (under ~10 changed files). Don't pad to fill the cap.
- At most 8 steps, and at most 50 words per step.
- Over the limit? Cut detail, merge steps, or shorten them. Don't compress the wording into fragments.
- If the branch can't be described in 250 words, write the best 250-word flow you can and tell the user the PR may be worth splitting. Never go over the cap.

Before printing, count the words (e.g. `wc -w`) and check every limit.

## Output

Print the flow in the chat, ready to paste. Then, separately, list anything you left out (unrelated commits, uncommitted changes) and anything worth checking. Don't edit the PR yourself.

## Example

```markdown
Flow:

1. **Request entry:** `GET /session/scenario-chooser/validation` in `client-sessions.controller.ts` reads the language header, optional BP claim and query params, and delegates to `ClientSessionsService`. `ClientSessionQueryParamsPipe` validates the params against `ScenarioChooserValidationQueryDto` (session params plus an optional `quoteId`). The endpoint is read-only: no session is loaded or stored.
2. **Orchestration:** `ValidateScenarioChooserUseCase` returns valid/proceed straight away for any non-move scenario. For move it resolves the inputs, runs the policies and maps the result.
3. **Input resolution:** `ScenarioChooserValidationInputsService` builds the inputs from the request alone. With a `quoteId`, `OnboardingService` provides the PCF segment and products. Without one, `FunnelProductLookupService` (now shared with contact-details) maps the slugs to Salesforce IDs and `ProductsService` fetches them. `CustomersService` provides the customer segment only for known customers.
4. **Policies:** `session-validation.ts` runs the atoms in `scenario-chooser-validation.atoms.ts`, for move only. Smartflex and dynamic are blocked for prospects only. The segment check reuses `isInvalidSegment`.
5. **Result mapping:** `toScenarioChooserValidationResult` returns `{ result, action, error? }`. A segment mismatch returns `pcfModal` and wins over forbidden products, which return `redirect`.
6. **Response:** the result is validated against `ScenarioChooserValidationResultsDto`. The FE proceeds, redirects to the PCF result page, or opens the PCF modal.
```
