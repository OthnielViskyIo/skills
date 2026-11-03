---
name: "domain-modeling"
description: "Build the Domain-Driven Design model of the luminus applications step by step: the glossary (ubiquitous language), the model (MODEL.md) and ADRs, kept in Obsidian. Use when discussing domain terminology, finding naming inconsistencies in the code, writing or editing the glossary or model, or recording or editing an ADR."
---

Build the domain model of the existing luminus applications, using **Domain-Driven Design (DDD)**. This is reverse engineering: the applications already exist, and the job is to find the model they carry, name it, and write it down. The code is evidence, not the final word: it often uses different words for the same thing, or the same word for different things.

This is the *active* work: questioning terms, digging through the code for inconsistencies, inventing edge-case scenarios, and writing the glossary, the model and decisions down as soon as they are settled. (Just *reading* the glossary for vocabulary is not this skill. Any skill can do that.)

## Words used in this skill

- **Glossary:** always means the DDD **ubiquitous language**: the shared words that domain experts, developers, the UI and the code should all use, each with one meaning within a bounded context.
- **Model:** the DDD **domain model**: the concepts behind those words (bounded contexts, aggregates, entities, value objects, domain events, rules and lifecycles) and how they relate.
- **Bounded context:** the area within which a word has one meaning. The same word may mean different things in two contexts. That is allowed, as long as it is written down.

The glossary gives the words. The model shows how the concepts behind them fit together. Every name in the model must be a glossary term.

## Files

All files live in the Obsidian vault, not in the project repo:

```
/Users/othnielvisky/IO_obsidian/luminus/
├── GLOSSARY.md        ← ubiquitous language, all bounded contexts
├── MODEL.md           ← domain model: text + mermaid diagrams
└── adr/
    ├── 0001-slug.md
    └── 0002-slug.md
```

- [GLOSSARY.md](./GLOSSARY-FORMAT.md): one file for all luminus applications, grouped by bounded context.
- [MODEL.md](./MODEL-FORMAT.md): one file, grouped by bounded context, with mermaid diagrams (Obsidian renders them).
- [adr/](./ADR-FORMAT.md): decisions about the model.

Create files lazily, when you have something to write. Obsidian conventions apply: no H1 (Obsidian shows the file name as the title), and link between notes with `[[GLOSSARY#Term]]` / `[[MODEL#Section]]`.

## Working step by step

The glossary and the model grow over many sessions, one area at a time (a feature, a flow, an application). Never try to model everything at once.

1. **Start from what exists.** Read `GLOSSARY.md`, `MODEL.md` and `adr/` before anything else. Use their terms, and build on them.
2. **Agree on the area.** Agree with the user on which area this session covers, and stick to it. Note anything outside it as a follow-up instead of chasing it.
3. **Dig into the code.** Read the code for the area across every layer (see below), and collect candidate terms, concepts and inconsistencies.
4. **Settle with the user.** Go through the findings with the user. Write each term or model fact as soon as it is settled. Don't save them up for the end.
5. **Record coverage.** At the end, update the `Coverage` section of `MODEL.md`: what this session covered, and what is still open.

Mark anything you derived from the code that the user has not confirmed as `(inferred)`. Remove the marker once the user confirms it.

## During the session

### Hunt for inconsistencies in the code

Compare how one concept is named and shaped in every place it shows up: UI components and copy/translations, API contracts, services, database fields, events, and Jira tickets. Look for:

- **Synonyms:** different words for one concept (`contactDetails` in the OCO service vs `contractDetails` in the OCO UI).
- **Homonyms:** one word for different concepts ("customer" meaning the account holder in one place and the person on site in another).
- **Diverging shapes:** the same concept with different fields, states or rules in different places.
- **Code vs speech:** the code says one thing, the user or the tickets say another.

Show every inconsistency to the user, with file paths. Never silently pick a winner. Then settle it as described in [Handling inconsistencies](#handling-inconsistencies).

### Challenge against the glossary

When the user uses a term that conflicts with the glossary, say so right away: "The glossary defines 'cancellation' as X, but you seem to mean Y. Which is it?"

### Sharpen fuzzy language

When the user uses vague or overloaded terms, propose a precise canonical term: "You're saying 'account': do you mean the Customer or the User? Those are different things."

### Discuss concrete scenarios

When concepts and their relations are discussed, stress-test them with specific scenarios. Invent scenarios that probe edge cases and force a precise answer about where one concept ends and another begins.

### Cross-reference with code

When the user states how something works, check whether the code agrees. If it doesn't, say so: "Your code cancels whole Orders, but you just said partial cancellation is possible. Which is right?"

### Update the glossary and model inline

When a term is settled, update `GLOSSARY.md` right there, using [GLOSSARY-FORMAT.md](./GLOSSARY-FORMAT.md). When a concept, relation, lifecycle or rule is settled, update `MODEL.md`, using [MODEL-FORMAT.md](./MODEL-FORMAT.md).

Keep them apart:

- The **glossary** is words and definitions only: no implementation details, no diagrams, no specs.
- The **model** is the concepts and how they relate. It may point to where a concept lives in the code, but it is not a technical design.

## Handling inconsistencies

Some inconsistencies can't be fixed in the code (other teams own it, or changing it is too costly). The ubiquitous language must still be clear. First find out with the user which case it is, then record it as follows:

1. **Synonyms: same concept, different names.** Pick one canonical term for the glossary and for all speech. List the other spoken words under `_Avoid_`, and list the names the code actually uses under `_In code_`, saying where. Example: the canonical term is **Contract Details**, with `_In code_: contractDetails (OCO UI), contactDetails (OCO service)`. The code name is recorded as a known alias, not as a second term. Ask the user which name should win. If they can't say yet, put it under `Open questions` in the glossary.
2. **Homonyms: same word, different concepts.** These are different concepts. If they belong to different bounded contexts, define the term in each context, and add a `_Not to be confused with_` link between them. Show the translation between the two in the context map in `MODEL.md`. If they are in the same context, give at least one of them a new, more precise name, and record the overloaded code name under `_In code_`.
3. **Diverging shapes: same concept, different data or rules.** Keep one glossary term. In `MODEL.md`, describe the concept once, and list the differences per application under that concept (e.g. "The OCO UI has no `Cancelled` state."). If one application has its own view of the concept on purpose, model it as its own bounded context with a translation, and don't force the two into one shape.
4. **Not settled yet.** Add it to the `Open questions` section at the end of `GLOSSARY.md` (for words) or `MODEL.md` (for concepts), with the options and the file paths. Remove it once settled.

Offer an ADR when the choice of how to settle it meets the criteria below, e.g. "We keep `contactDetails` in the OCO service and treat it as an alias of Contract Details, because the service is owned by another team."

## Offer ADRs sparingly

Only offer to create an ADR when all three are true:

1. **Hard to reverse**: the cost of changing your mind later is meaningful
2. **Surprising without context**: a future reader will wonder "why did they do it this way?"
3. **The result of a real trade-off**: there were genuine alternatives, and you picked one for specific reasons

If any of the three is missing, skip the ADR. Use the format in [ADR-FORMAT.md](./ADR-FORMAT.md).
