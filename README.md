# explain-change

Agent skills that help you understand a change made by a coding agent:
whether it matches your request, how it works and why, what follows from it,
and what to check.

- **explain-change** — writes a layered report on a commit range for the person
  who gave the agent the task.
- **write-change-notes** — has the coding agent record decisions and checks
  while it works; explain-change reads these notes.

```sh
npx skills add ymandrikov/explain-change
```

## Usage

**write-change-notes** starts on its own when an agent begins a task that
changes code, tests, docs or configuration. It keeps notes in
`.change-notes/<task>.md`, to be committed with the change.

**explain-change** runs only when you call it. Call it in a new session, not
in the one that made the change: it checks what that agent claims.

```text
/explain-change
/explain-change main..my-branch, the request is in plans/passkeys.md
```

In other agents, ask for the skill by name. Without a range, it explains
`HEAD` against its merge-base with `origin/HEAD`, `main` or `master`, with
uncommitted changes. It takes the request from your message or the notes and
asks once if there is none. By default the report goes to
`.change-notes/<short head hash>.explanation.md` and is not committed.

## AGENTS.md

Add to the project's `AGENTS.md` or `CLAUDE.md`, so that every agent keeps
notes even when it would not load the skill on its own:

```markdown
## Change notes

For any task that changes code, tests, docs or configuration, use the
write-change-notes skill, and commit `.change-notes/<task>.md` with the change.
```

To keep explanation reports out of commits, add this to `.gitignore`:

```gitignore
.change-notes/*.explanation.md
```

This repository is a published mirror; changes made here directly are overwritten.

MIT License.
