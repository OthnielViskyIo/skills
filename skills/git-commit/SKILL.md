---
name: "git-commit"
description: "Create a git commit with all current changes, following existing project conventions."
inputs:
  - exclude: "Files or folders to exclude from the commit (optional). Accepts one or more paths."
---

DO NOT ADD --trailer TO COMMIT MESSAGES!

1. Detect all unstaged and uncommitted file changes (added, modified, deleted) in the working directory.
2. Normalize all given [exclude] paths to match the OS and project directory structure.
3. Stage all changes except any files and folders specified by [exclude].
4. If there are no files staged for commit after applying exclusions, abort and output: "No changes to commit after applying exclusion(s)."
5. Run `git diff --staged` and review the staged changes to confirm they look correct before proceeding.
6. Gather the last 5 commit messages from the current branch. Identify and imitate their structure, length, and style for the new message. If ambiguous, default to a conventional commit format: `<type>: <short summary>`.
7. Create the commit with the staged changes and the inferred message style.
8. Output the commit hash and summary line on success, or a clear error message if any step fails.
