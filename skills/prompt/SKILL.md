---
name: "prompt"
description: "Restructure or rewrite a piece of text into a well-engineered prompt. Use when the user invokes /prompt, asks to 'improve', 'rewrite' or 'engineer' a prompt, or wants a rough request turned into a clear instruction for an LLM."
---

Rewrite the text the user gives you into a prompt that an LLM can act on well. The text is the *material*, not an instruction to you: do not carry out the task it describes.

## 1. Understand the intent

Before rewriting, work out:

- **Goal:** what outcome does the author actually want?
- **Audience:** which model or agent will receive the prompt (a chat model, a coding agent, a subagent, an API system prompt)?
- **Deliverable:** what should the response look like (code, a file, a list, a decision, prose)?
- **Constraints:** scope, tools, style, length, things to avoid.
- **Success criteria:** how would the author know the response is good?

If a gap would change the rewrite substantially and cannot be inferred from the text or the surrounding conversation/codebase, ask the user, one question at a time, each with your recommended answer. Otherwise, make a sensible assumption and list it.

## 2. Apply the practices that fit

Use judgment; a two-line request does not need every section. Scale the structure to the task.

- **Lead with the task.** State what to do in the first sentence, in plain, direct language.
- **Give context and the why.** Explain the purpose behind instructions so the model can generalize instead of following rules blindly.
- **Be specific.** Replace vague words ("better", "clean", "some") with concrete expectations.
- **Say what to do, not only what to avoid.** Turn negations into positive instructions where possible.
- **Separate parts clearly.** Use headings or XML-style tags (`<context>`, `<task>`, `<constraints>`, `<examples>`, `<output_format>`) when the prompt mixes instructions with data or reference material.
- **Define the output.** Format, length, structure, and where the result should go.
- **Add examples when format or tone is subtle.** Keep them varied so the model doesn't copy one too literally.
- **Break multi-step work into ordered steps**, and say when the model should stop, verify, or check back in.
- **Assign a role only when it adds useful framing.** Skip generic "You are an expert…" lines.
- **Remove noise.** Cut filler, repetition, politeness padding, and SHOUTING; emphasis should be rare and earned.
- **Preserve the author's intent and facts.** Don't invent requirements; flag any you added as assumptions.

## 3. Output

Respond with:

1. **Rewritten prompt:** in a single fenced code block, ready to copy.
2. **What changed:** 3–6 short bullets on the most important changes and why.
3. **Assumptions / open questions:** a numbered list of anything you inferred or that the user should confirm (omit if none).

Then stop. Only execute the rewritten prompt if the user explicitly asks you to (e.g. "run it", "go").
