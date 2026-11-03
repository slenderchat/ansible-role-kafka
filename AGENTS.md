# Agent instructions

Repository-wide rules for all agents working in this repository. Nested
`AGENTS.md` files, if added, only supplement these rules for their directories.
Task-specific instructions do not belong here.

## Project context

- Read `PROJECT_CONTEXT.md` before substantial work.
- Update `PROJECT_CONTEXT.md` before handing work over and before finishing it,
  and during long work after significant changes.
- Record durable, non-obvious information:
  - decisions and their reasons;
  - constraints;
  - verification state and its scope;
  - blockers;
  - next steps;
  - useful commands.
- Do not record:
  - secrets or credentials;
  - conversation transcripts;
  - trivial changes;
  - file inventories or anything else readily discoverable from the
    repository;
  - content that duplicates `README.md` or other documentation;
  - private hostnames, addresses or inventory values.

## Working rules

- Follow the existing architecture and conventions of the role.
- Stay within the scope of the task. Do not widen the variable interface, add
  features or weaken safety checks unless the task requires it.
- Respect the existing division of responsibility. Do not change user or
  managed tool settings, or anything you were not asked to change.
- Keep `README.md` and `README.ru.md` consistent in meaning when either
  changes.

## Division of responsibility

- The user performs commands, tests, builds, Git operations, infrastructure
  operations, packaging, transfer and publication, unless a specific task
  explicitly authorizes an exception.
- Agents may use permitted file-reading and file-editing tools and read-only
  public research.
- Preparing commands or checks does not authorize executing them.

## Checks

- Plan the minimum set of checks that covers the acceptance criteria and the
  specific risks of the change. Execute checks only when the task authorizes
  it; otherwise provide them to the user.
- Reuse valid earlier results. Recheck only what the change affects.
- Report what was actually executed separately from what was established by
  reading code or documentation, and mark prepared but unexecuted checks as
  such.

## Completion

- Finish when the acceptance criteria are met and significant problems are
  resolved.
