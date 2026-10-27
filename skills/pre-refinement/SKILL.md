---
name: "pre-refinement"
description: "Research a Jira ticket before a refinement meeting: explain what it asks for and find the gaps and unclear points to raise as questions."
disable-model-invocation: true
inputs:
  - ticket: "Jira ticket key or URL (required)."
  - feature: "Feature folder name under luminus/ (optional). Inferred if not given."
  - project: "Project (repo or folder) to scope the codebase research to (optional). Defaults to the current working directory."
---

Prepare the user for the refinement meeting of [ticket]. Explain the ticket, then find what is missing or unclear so the user can ask about it in the meeting.

This is research only. Don't edit code, write a plan, or propose an implementation: that follows in the planning phase. Don't change the ticket or post to Jira.

Two standing rules:

- **No estimates or sprints:** Never look at, or add to the note, story points or the sprint the ticket belongs to, unless the user explicitly asks.
- **Segments:** Assume the ticket covers both residential and professional, and never ask which one is in scope. Only treat them separately if the ticket describes differences, or if the user asks.

## Gather the context

1. Fetch [ticket] with the Atlassian MCP: description, acceptance criteria, comments, status, labels, parent/epic, linked issues and subtasks. The description is often in a custom field ("US Description", "Requirements and Acceptance Criteria"), so fetch all fields with their names. An empty standard description isn't a gap while one of these fields has the content.
2. Search for similar or previously completed tickets that touched the same area (JQL on the key terms, component or epic). Use them to see how comparable work was scoped and what was decided.
3. List links you can't open (Figma, Confluence, etc.) as references. Don't treat unreadable content as a gap. If a Confluence page is readable, use it.
4. Explore [project] to understand what already exists in the area the ticket touches: the relevant features, flows, models and integrations. Stay inside [project] and don't research other projects, even if the ticket mentions them: note them under Dependencies instead. Stay at the level of "what is there today". If you can't tell which repo the ticket belongs to, ask the user.

## Find the gaps

Check the ticket against these categories. Only include a category if you found something.

- **Scope:** what is in, what is out, who the user or actor is, what triggers the change. Not the segment (see the standing rules).
- **Acceptance criteria:** missing, vague, untestable, or contradicting the description or comments.
- **Business rules and edge cases:** conditions, priorities, error and empty states, existing versus new customers or data, migrations of existing data.
- **Dependencies:** other tickets, teams, services, designs, content or data the work waits on.
- **Technical gaps:** places where the ticket's assumptions don't match what the codebase does today, or where existing behavior would conflict or need a decision. Describe the gap, not the solution.

Separate what the ticket states from what you infer. If something is only your assumption, say so.

## Name and location

- **Feature:** use [feature] if given. Otherwise match an existing folder under `luminus/` (e.g. `boiler-maintenance`, `scenario-chooser`) to the ticket's parent/epic. If none fits, propose a kebab-case name and ask the user before creating it.
- **Note name:** `<KEY> - <short summary in lowercase>.md`: the ticket key plus a few words on what the ticket is about, e.g. `DGCO-3561 - set up your service page: see content of service.md`.
- **Path:** `/Users/othnielvisky/IO_obsidian/luminus/<feature>/refinement/<note name>`. Create the `refinement` folder if needed. Ask before overwriting an existing file.

## Write the note

No H1: Obsidian shows the file name as the title. Use this structure:

```markdown
## Summary
2-4 sentences: what the ticket asks for and why, in plain language.

## What exists today
Short description of the relevant current behavior in the codebase, with file paths. Skip if the ticket has no codebase footprint.

## Context
Parent/epic, linked tickets, similar past tickets and what they tell us. References that couldn't be read.

## Questions for refinement
### Scope
- **Question?** Why it matters / what in the ticket or code prompted it.
### Acceptance criteria
...

## Assumptions
What the ticket seems to take for granted and couldn't be confirmed.
```

- Each question is phrased so it can be asked out loud, followed by one line on why it matters.
- Order the questions inside a category by how much they block refinement, most blocking first.
- No padding: a short ticket gets a short note. If the ticket is clear, say so and don't invent questions.

## Length limits

Count words, not characters: file paths and identifiers would distort a character count.

- Hard cap: 400 words for the whole note, headings included.
- Target: ~250 words for a typical ticket. Don't pad to fill the cap.
- Summary: at most 60 words. At most 10 questions in total, each with a one-line reason.
- Over the limit? Keep the most blocking questions and cut the rest, shorten the reasons, merge related questions. Don't compress the wording into fragments.
- Never go over the cap. If a ticket can't be covered in 400 words, it is probably too big for one refinement: say so in the summary.

Before saving, count the words (e.g. `wc -w`) and check every limit.

## Output

Save the note, then print its path and the questions for refinement in the chat.
