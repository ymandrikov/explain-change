---
name: write-change-notes
description: Keeps notes on a coding task in .change-notes/ with what the diff does not show (the request, decisions, constraints, tradeoffs and checks), so that the person who gave the task can later understand the change. Use at the start of any task that changes code, tests, docs or configuration, such as a bug fix, a new feature, a refactoring or a plan to carry out, and when continuing such a task.
---

# Write Change Notes

Keep the task's current understanding in `.change-notes/<task>.md`, where `<task>` is a short kebab-case name for the task. A separate agent uses the notes to explain the change to the person who gave the task. Write them during the task, not from a finished diff, and record only what the diff does not show. Use English unless the user or the project asks for another language.

Do not stop to ask the user about a seam that the request did not ask for, a tradeoff, a blocker, or a fix that leaves the cause of a problem: record it, and the explanation shows it to the reader.

## What to record

Start with the request as a link, a verbatim quote or the path to the plan; then the task as you understand it, and where your understanding narrows, extends or interprets the request. Group the entries so that each thought stays whole: a decision, its reason, its constraint and its open question can be one entry.

- **Request.** Each part of the request that you did not do or did differently, and why.
- **Cause.** For a fix, the symptom and its cause: the nearest point in the project whose fix removes the symptom on every path that leads to it. Fix the cause by default. If the request asks to remove only the symptom, do that and record that the cause stays. If you leave the cause for another reason, record whether the fix hides the symptom, why you chose it, and what would fix the cause. Look for other paths from the same cause, and record them and whether the fix covers them.
- **Seams.** A seam is a place where a module uses another module or the outside world, together with the contract that passes through it. A dependency inside one module is not a seam; a use of another module's internals is. A library, a service or a configuration option is always a seam. For each seam that you add, change or remove, record why and whether the request asked for it. If you chose between options, record the rejected ones and why.
- **Tradeoffs.** A tradeoff is a decision where the chosen option loses something that another option, which also meets the request, would keep. It matters when a user or a caller notices the loss; the loss concerns performance, edge cases, maintenance or the project's conventions; the choice depends on the reader's priorities; the choice is expensive to reverse; or you consider the rejected option better. For each tradeoff that matters, record the options, what the chosen one loses and why you chose it. If you consider a rejected option better, say so and record what it would take.
- **Constraints.** A constraint is a condition that makes the solution more complex or limits its result. Record each one, where it came from (the request, existing code, an external contract or your assumption), and why you kept it. Record what it costs: the solution that you would write without it, the parts of the change that it added or made more complex, and the capabilities that you gave up.
- **Undoing.** A difficulty of undoing is work beyond reverting the code and releasing it again, or something that a return to the earlier state cannot restore, once users, data, other code or external consumers rely on the change. It is significant when knowing it could change whether the reader accepts the change, what they check carefully or how they prepare the release. Record each significant difficulty that you know of, and each way to recover that the change provides, such as a down migration or a feature flag.
- **Facts.** Other non-obvious facts that you found, marked as observations or hypotheses.
- **Effects.** What a user or caller should notice, and how to check it, with the expected result. Call it a proposed check until it runs.
- **Open.** What remains unverified, unclear or dependent on unavailable context.

## Verification

Record each actual run as its own entry under `## Verification`, with the state that it tested: a commit hash, or the working tree after a given edit. If an entry tests an earlier state, say which edits it does not cover. Keep the origin of results that someone else supplied.

```markdown
### V1
- Command: `python3 -m unittest discover -v`
- Result: 5 tests passed.
- State: working tree after the last edit, before the commit.
```

- **Reproduction.** Before you fix an observed problem, reproduce it at the place where it was observed: in the product, or in the test or CI command that showed it. Repeat it after the fix. Record the steps and both results, or why you could not reproduce it.
- **Guarding tests.** Run each test that you add or change for the problem on the code without the fix, and record the result. A compile or import error does not show that the test guards the fix.
- **Blockers.** A blocker is an action that you needed for the work or its verification but could not complete for a reason outside the change: the environment, missing access, or a failure that already existed in the project. Record it with the result `Blocked`: the command or action and an excerpt of its error output. If you can, run the same command on the code before your change and record the result. Record what you skipped or did instead, whether the workaround stays in the project, and what would remove the blocker.
- **Final check.** Run it after the last edit, so that it covers the result.

## Keep it current

Update the notes when an important decision changes, a constraint or non-obvious fact is found, or a check runs. Replace obsolete understanding instead of keeping a diary of reversals, and before finishing, reconcile the notes with the final state. Do not narrate routine edits, copy code into prose, classify the overall change or write the explanation. Record no passwords, tokens or unrelated private data.

Commit the notes with the code change, so that they travel with it. This does not itself authorize a commit, push or publication that the task does not authorize.
