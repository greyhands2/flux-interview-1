# Coding agent instructions

## Before changing files
- Inspect the repository tree, relevant source, and existing guidance before editing. Confirm the current branch and working tree state.
- Treat `sources/` as read-only reference material. Never edit, rename, move, or delete its contents.
- Do not assume commands, packages, APIs, or architecture that are not present in the repository.

## Make focused changes
- Keep changes limited to the requested outcome. Follow established repository conventions where they exist.
- Ask for clarification when a product or data contract is ambiguous; record assumptions when they do not block progress.
- Keep analytics access read-only and constrained to approved metrics and data sources. Never add arbitrary SQL execution.

## Verification and reporting
- Use repository-specific commands only after inspecting the repository and its documentation.
- Build command: `[TODO: fill in after repository inspection]`
- Test command: `[TODO: fill in after repository inspection]`
- Lint/format command: `[TODO: fill in after repository inspection]`
- Report what you changed, the verification commands run and their results, and any checks not run with a reason.
- Review the final diff for accidental scope, secrets, generated artifacts, and edits under `sources/`. Summarize the review before finishing.
