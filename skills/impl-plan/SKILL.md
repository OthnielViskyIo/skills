---
name: "impl-plan"
description: "Turn the decisions about a user story into an unambiguous implementation plan, ready to be built with /tdd, and save it as an Obsidian note. Use only when the user explicitly writes /impl-plan; never for a plain mention of the word."
inputs:
  - story: "The user story: one or more Jira keys, URLs or pasted text (required)."
  - feature: "Feature folder name under luminus/ (optional). Inferred if not given."
  - project: "Project (repo or folder) the plan is for (optional). Defaults to the current working directory."
---

Write the implementation plan for [story]. It comes after `/pre-refinement` and `/research`: their job was to understand the ticket, this one turns the agreed decisions into concrete steps. Someone should be able to implement the plan without having to make a single decision of their own.

The implementation is done with `/tdd`, unless the user says otherwise. Shape the plan for it: vertical slices, each with the behavior it adds and the seam it is tested at.

This is planning only: don't edit code, change the ticket or post to Jira. Leave out git commits and pushes, e2e tests and deployment unless the user asks for them.

Two standing rules:

- **No estimates or sprints:** Never look at, or add to the note, story points or the sprint the ticket belongs to, unless the user explicitly asks.
- **Segments:** Assume the ticket covers both residential and professional. Only treat them separately if the ticket describes differences, or if the user asks.

## Gather the context

1. If [story] is a Jira key or URL, fetch it with the Atlassian MCP: description, acceptance criteria, comments, parent/epic, linked issues and subtasks. The description is often in a custom field ("US Description", "Requirements and Acceptance Criteria"), so fetch all fields with their names. If [story] is pasted text, use it as is and look up any ticket keys it mentions.
2. Search the Obsidian vault (`/Users/othnielvisky/IO_obsidian/luminus`) for the ticket key and the feature: the research note in `<feature>/research/`, the pre-refinement note in `<feature>/refinement/`, plans of sibling tickets in `<feature>/plans/`, and the user's own notes. These hold the decisions this plan builds on. Don't redo their research, and don't contradict their decisions without asking.
3. If there is no research note, say so and ask whether to run `/research` first or continue without it.
4. Read the rules for the technologies involved: the user's rules in `~/.claude/rules/`, plus the project's `CLAUDE.md`, `CONTRIBUTING.md` and lint/format configs. The plan must follow them. If a technology has no rules file, tell the user and ask for one before continuing. Never read `.ai-rulez/`, `.cursor/` or other agent rules folders.
5. Explore [project] in enough depth to name the exact files, modules and functions each step touches, and the existing patterns to follow. Look facts up in the code instead of asking the user. Stay inside [project]: other repos or teams are dependencies, not steps.

## Close the gaps

List every decision the plan needs that the sources don't make: open points from the research note, unanswered refinement questions, and design choices the code raises (where logic lives, naming, contracts, error handling, data migrations).

Close them with `/grilling`: one question at a time, each with your recommended answer, and wait for the user. Also confirm the test seams for `/tdd` this way. Skip grilling only if nothing is open.

The note has no "open points" section. If a decision can't be made yet, stop and tell the user what is blocking, instead of writing a plan with gaps.

## Agree on verification

Once the gaps are closed, propose a numbered list of checks the user can do themselves to confirm the implementation is correct. For example: which endpoint to try in Swagger, with what input and expected response; how to test the flow by hand in the browser; which data to look at in the database. Each item says what to check, how, and what result to expect. Mark the ones you recommend and say why. The user makes the final decision: wait for them to accept, drop, change or add items, and only put the agreed list in the note.

## Name and location

- **Feature:** use [feature] if given. Otherwise, match an existing folder under `luminus/` (e.g. `boiler-maintenance`, `scenario-chooser`) to the ticket's parent/epic. If none fits, propose a kebab-case name and ask the user before creating it.
- **Note name:** `<KEY> - <short summary in lowercase>.md`: the ticket key plus a few words on what the plan delivers, e.g. `DGCO-3666 - scenario chooser endpoint.md`. For several tickets in one plan: `<KEY> and <KEY> - <summary>.md`.
- **Path:** `/Users/othnielvisky/IO_obsidian/luminus/<feature>/plans/<note name>`. Create the `plans` folder if needed. Ask before overwriting an existing file.

## Write the note

No H1: Obsidian shows the file name as the title. Use this structure:

```markdown
Implementation plan for **<KEY>: <ticket title>**. Builds on [[<research note>]]. Part of [[<parent plan or note>]], if there is one.

- Repo: `<repo>`
- Branch: `<proposed branch name>`
- Status: planned
- Out of scope: what this plan deliberately leaves to other tickets or teams

## Goal
What exists when the plan is done, in 2-4 sentences: the behavior, and the contract (endpoint, component, function signature) if there is one.

## Decisions
- **Decision.** Why, and where it was made (ticket, research note, grilling on <date>).

## Steps
1. **Slice name.** The behavior this slice adds, the files it creates or changes, and the existing pattern to follow. Seam: where it is tested and the cases that prove it.
2. ...

## Verification
1. A check agreed with the user: what to check, how, and the expected result.
2. ...

## Definition of done
- [ ] A check the implementer runs to confirm their own work, e.g. a command (tests, typecheck, lint, build) and the result it must give
```

- **Verification** is for the user: manual checks to confirm the implementation is correct, as agreed in "Agree on verification".
- **Definition of done** is for the implementer: the checks to run before calling the work finished. You set these yourself; they don't need the user's agreement.
- **Steps** are ordered so each one builds on the previous and leaves the code working. Each step is one `/tdd` slice or a small group of them, at a seam confirmed during grilling. Steps that aren't test-driven (config, wiring, migrations) say how they are verified instead.
- **Precise, not verbose:** name files, functions, types, routes and values exactly. Leave out anything the implementer doesn't need to decide or find.
- **Code** only where it removes ambiguity that words can't, e.g. a contract or a signature. No implementations.
- If the user said not to use `/tdd`, drop the seams from the steps and describe how each step is verified.
- Every decision traces back to its source. Mark anything still inferred with "(inferred)" and confirm it during grilling before saving.

## Length limits

Count words, not characters: file paths and identifiers would distort a character count.

- Hard cap: 800 words for the whole note, headings and code included.
- Target: ~500 words for a typical ticket. Don't pad to fill the cap.
- Goal: at most 60 words. Each decision: at most 40 words.
- Over the limit? Cut what the implementer can find in the code or in the linked notes, merge small steps, shorten. Don't compress the wording into fragments.
- Never go over the cap. If the plan can't fit in 800 words, the ticket is probably too big: propose how to split it into separate plans.

Before saving, count the words (e.g. `wc -w`) and check every limit.

## Output

Save the note, then print its path and the list of steps in the chat.
