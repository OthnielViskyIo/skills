---
name: "code-review"
description: "Performs a structured code review on provided code or a codebase folder. Supports React, TypeScript, Next.js, NestJS, and MongoDB checklists."
inputs:
  - code: "The code snippet, file path, or folder path to review."
  - focus: "Comma-separated focus areas (optional), e.g. 'performance, best practices'."
  - severity_threshold: "Optional. Minimum severity to include in output: 'Critical', 'Warning', or 'Suggestion'. Defaults to showing all."
checklists:
- name: "React"
  path: "checklists/react.md"
- name: "TypeScript"
  path: "checklists/typescript.md"
- name: "Next.js"
  path: "checklists/nextjs.md"
- name: "NestJS"
  path: "checklists/nestjs.md"
- name: "MongoDB"
  path: "checklists/mongodb.md"
template: |
  Perform a code review on the provided code with respect to the selected stack(s).
  1. Identify the tech stack in use (React, TypeScript, Next.js, NestJS, MongoDB).
  2. If multiple stacks are detected, apply all relevant checklists. Note concerns that span multiple stacks only once, under the most relevant checklist.
  3. Read and apply each applicable checklist file verbatim before generating findings.
  4. Pay special attention to any focus areas listed by the user.
  5. Assign a severity level to each finding: **Critical** (must fix), **Warning** (should fix), or **Suggestion** (nice to have).
  6. If a severity_threshold was provided, omit findings below that threshold from the output.
  7. Output your review in the following structure:

---
## Code Review Results

**Summary:**
[Short high-level summary]

**Checklist Findings:**
- [Severity] [Checklist point: Pass/Fail/Comment]

**Focus Areas:**
- [Specific notes on requested focus areas]

**Suggested Improvements:**
- [Refactor/Improvement suggestions]
---