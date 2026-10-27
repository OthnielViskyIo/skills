---
name: "code-review"
description: "Performs a structured code review on provided code or a codebase folder against the project's coding rules. Supports React, TypeScript, Next.js, NestJS, and MongoDB."
inputs:
  - code: "The code snippet, file path, or folder path to review (required)."
  - focus: "Comma-separated focus areas, e.g. 'performance, best practices' (optional)."
  - severity_threshold: "Minimum severity to include in the output: 'Critical', 'Warning', or 'Suggestion' (optional). Defaults to showing all."
rules:
- name: "TypeScript"
  path: "../../rules/typescript.md"
  applies_to: "All TypeScript code, including React, Next.js, and NestJS."
- name: "React & Next.js"
  path: "../../rules/react-next.md"
  applies_to: "React and Next.js code."
- name: "NestJS"
  path: "../../rules/nestjs.md"
  applies_to: "NestJS code."
- name: "MongoDB"
  path: "../../rules/mongodb.md"
  applies_to: "MongoDB and Mongoose code."
- name: "SOLID"
  path: "../../rules/solid.md"
  applies_to: "All code."
- name: "Clean Architecture"
  path: "../../rules/clean-architecture.md"
  applies_to: "Backend and service projects (e.g. NestJS, Nx libs with domain/application/infrastructure layers)."
---

Perform a code review on [code] with respect to the selected stack(s).

1. Identify the tech stack in use (React, TypeScript, Next.js, NestJS, MongoDB).
2. Select every rules file whose `applies_to` matches the code. If multiple apply, note concerns that span several files only once, under the most relevant one.
3. Read each selected rules file in full before generating findings. Review against the rules as written: do not substitute general best practices where a rule says otherwise.
4. Pay special attention to [focus], if given.
5. Assign a severity level to each finding:
   - **Critical** (must fix): violates a hard rule ("never", "must", "always", "no exceptions"), or causes a bug, security issue, or data loss.
   - **Warning** (should fix): goes against a "prefer"/"avoid" rule without a stated reason.
   - **Suggestion** (nice to have): improvements not covered by a rule.
6. If [severity_threshold] is given, omit findings below it from the output.
7. Output your review in the following structure, citing the rules file and section for each finding (e.g. `typescript.md §3`):

```markdown
## Code Review Results

**Summary:**
[Short high-level summary]

**Rule Findings:**
- [Severity] [Rules file §section: Pass/Fail/Comment]

**Focus Areas:**
- [Specific notes on requested focus areas]

**Suggested Improvements:**
- [Refactor/Improvement suggestions]
```
