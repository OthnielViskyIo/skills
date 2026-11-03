# GLOSSARY.md Format

The glossary is the ubiquitous language of all luminus applications. It lives at `/Users/othnielvisky/IO_obsidian/luminus/GLOSSARY.md`.

## Structure

No H1: Obsidian shows the file name as the title. Each bounded context gets an H2, and each term an H3, so it can be linked as `[[GLOSSARY#Contract Details]]`.

```md
## Ordering

{One or two sentences on what this bounded context is and why it exists.}

### Order
{One or two sentences on what the term is.}
_Avoid_: Purchase, transaction

### Contract Details
The terms a customer agrees to when signing a contract.
_Avoid_: Contact details
_In code_: `contractDetails` (OCO UI), `contactDetails` (OCO service)

### Customer
A person or organization that places orders.
_Avoid_: Client, buyer, account
_Not to be confused with_: [[GLOSSARY#Customer (Billing)]]

## Billing

...

### Customer (Billing)
The party that is invoiced, which may differ from the one who placed the order.
_Not to be confused with_: [[GLOSSARY#Customer]]

## Open questions

- **contactDetails vs contractDetails:** same concept or two? `src/...` (OCO service) uses `contactDetails`, `src/...` (OCO UI) uses `contractDetails`.
```

## Fields

- **Definition** (required): one or two sentences.
- `_Avoid_`: other words people use for this concept that should no longer be used in speech or new code.
- `_In code_`: names the existing code uses for this concept that differ from the term, each with the application or module it appears in. These are known aliases we live with, not alternatives.
- `_Not to be confused with_`: a link to a term with the same or a similar name that means something else.

Leave out fields that don't apply.

## Rules

- **Be opinionated.** When several words exist for one concept, pick the best one and list the others under `_Avoid_` or `_In code_`.
- **Keep definitions tight.** One or two sentences max. Define what it IS, not what it does.
- **Only domain terms.** General programming concepts (timeouts, error types, utility patterns) don't belong, even if the project uses them a lot. Before adding a term, ask: is this a concept of the luminus domain, or a general programming concept? Only the former belongs.
- **One meaning per context.** A term with the same name in two contexts gets an entry in each, with the context in brackets on the second one, e.g. `### Customer (Billing)`.
- **Group terms** under an H4 within a context when natural clusters emerge. A flat list is fine otherwise.
- **Open questions** go in the last section and are removed once settled.

When the current topic could belong to more than one bounded context, infer which one. If unclear, ask.
