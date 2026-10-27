---
name: "research"
description: "Research a user story as the first step of implementing it: understand the flow from the user's perspective, all moving parts, and what this ticket covers, then save it as an Obsidian note."
disable-model-invocation: true
inputs:
  - story: "The user story: Jira key, URL or pasted text (required)."
  - feature: "Feature folder name under luminus/ (optional). Inferred if not given."
  - project: "Project (repo or folder) to look at for the current flow (optional). Defaults to the current working directory."
---

Research [story] so the user fully understands what needs to be done before implementing it. The goal is to close every gap in the user's understanding of the ticket: what the user experiences, how the flow works end to end, which parts move, and which part of that this ticket covers.

Focus on the feature and the flow, not on the technical solution. Add technical remarks only where they are needed to understand the flow or the scope. This is research only: don't edit code, write a plan, or change the ticket.

Two standing rules:

- **No estimates or sprints:** Never look at, or add to the note, story points or the sprint the ticket belongs to, unless the user explicitly asks.
- **Segments:** Assume the ticket covers both residential and professional. Only treat them separately if the ticket describes differences, or if the user asks.

## Gather the context

1. If [story] is a Jira key or URL, fetch it with the Atlassian MCP: description, acceptance criteria, comments, parent/epic, linked issues and subtasks. The description is often in a custom field ("US Description", "Requirements and Acceptance Criteria"), so fetch all fields with their names. If [story] is pasted text, use it as is and look up any ticket keys it mentions.
2. Read the parent/epic and sibling tickets to see the whole feature, and where this ticket sits in it.
3. Search the Obsidian vault (`/Users/othnielvisky/IO_obsidian/luminus`) for the ticket key and the feature, e.g. the pre-refinement note in `<feature>/refinement/` or the user's own notes. Build on what is there and don't repeat it in full.
4. Look at [project] to see how the flow works today: entry points, steps, data the user enters or sees, and the systems involved. Stay at the level of "what happens", not "how it is coded". Note other repos or teams as moving parts, but don't research them.
5. List links you can't open (Figma, Confluence, etc.) as references. Don't treat unreadable content as a gap.

If a gap can't be closed from these sources, keep it as an open point. Don't guess.

## Name and location

- **Feature:** use [feature] if given. Otherwise match an existing folder under `luminus/` (e.g. `boiler-maintenance`, `scenario-chooser`) to the ticket's parent/epic. If none fits, propose a kebab-case name and ask the user before creating it.
- **Note name:** `<KEY> - <short summary in lowercase>.md`: the ticket key plus a few words on what the ticket is about, e.g. `DGCO-3561 - set up your service page: see content of service.md`.
- **Path:** `/Users/othnielvisky/IO_obsidian/luminus/<feature>/research/<note name>`. Create the `research` folder if needed. Ask before overwriting an existing file.

## Write the note

No H1: Obsidian shows the file name as the title. Use this structure:

```markdown
## Summary
What the ticket asks for and why, in plain language, and where it sits in the feature (parent/epic).

## User's perspective
What the user does and sees, step by step, in the part of the flow this ticket touches. Include what happens before and after it, and the error or empty states.

## Flow and moving parts
The end-to-end flow, as numbered steps or a small text diagram. For each step: what happens, which system or data is involved, and which ticket covers it. Mark the steps this ticket covers.

## Scope
What this ticket does and what it leaves to other tickets or teams.

## Technical remarks
Only if needed: facts about today's code or data that change how the flow or scope should be understood, with file paths. Skip otherwise.

## Open points
What is still unclear, and who or what could answer it. Skip if nothing is open.
```

- Separate what the sources state from what you infer, e.g. "(inferred)".
- Write in plain language. A product owner should be able to follow everything except the technical remarks.
- No padding: a small ticket gets a small note.

## Length limits

Count words, not characters: file paths and identifiers would distort a character count.

- Hard cap: 500 words for the whole note, headings and diagram included.
- Target: ~350 words for a typical ticket. Don't pad to fill the cap.
- Summary: at most 60 words. Technical remarks: at most 80 words.
- Over the limit? Cut detail that doesn't help understanding, merge steps, shorten. Don't compress the wording into fragments.
- Never go over the cap. If the ticket can't be covered in 500 words, it is probably too big: say so in the summary.

Before saving, count the words (e.g. `wc -w`) and check every limit.

## Output

Save the note, then print its path and the open points in the chat.
