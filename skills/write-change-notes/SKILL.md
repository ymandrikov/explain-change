---
name: write-change-notes
description: Use when starting or continuing a coding task whose decisions and verification need to accompany the resulting change for a later explanation or review.
---

# Write Change Notes

Keep the coding task's current understanding in `.change-notes/<task>.md`, where `<task>` is a short kebab-case name for the task. Notes are optional input for a separate explaining agent; write them during the task, not by reconstructing intent from a finished diff.

The purpose is to help the person who gave the task understand the change with less reconstruction effort. Record what the diff alone does not show. Use English unless the user or project instructions ask for another language.

## How to work

1. Do not stop to ask the user about a seam that the request did not ask for, a tradeoff, a blocker, or a fix that leaves the cause of a problem. Record it: the explanation shows it to the reader.
2. Before you fix an observed problem, reproduce it at the place where it was observed: in the product, or in the test or CI command that showed it. If you cannot, record why.
3. Find the cause of the problem: the nearest point in the project whose fix removes the symptom on every path that leads to it. Fix the cause by default. If the request asks to remove only the symptom, do that.
4. Look for other paths from the same cause.
5. After the fix, repeat the reproduction.
6. Run each test that you add or change for the problem on the code without the fix. A compile or import error does not show that the test guards the fix.
7. If an action that you need for the work or its verification fails for a reason outside the change, run the same command on the code before your change, if you can.

## What to record

Start with the task: the original request as a link, a verbatim quote or the path to the plan, then the task as you understand it, and where your understanding narrows, extends or interprets the request.

Then record the following. Group the entries so that each thought stays whole: a decision, its reason, the constraint behind it and its open question can be one entry.

- **Request.** Each part of the request that you did not do or did differently, and why.
- **Cause.** For a fix: the symptom and its cause. If the fix does not remove the cause, whether it hides the symptom, why you chose it, and what would fix the cause. If the request asked to remove only the symptom, that the cause stays. Other paths from the same cause that you found, and whether the fix covers them.
- **Seams.** A seam is a place where a module uses another module or the outside world, together with the contract that passes through it. A dependency inside one module is not a seam. A use of another module's internals is a seam. A library, a service or a configuration option is always a seam. For each seam that you add, change or remove, record why, and whether the request asked for it. If you chose between options, record the rejected ones and why.
- **Tradeoffs.** A tradeoff is a decision where the chosen option loses something that another option, which also meets the request, would keep. Record each tradeoff that matters: a user or a caller notices the loss, the loss concerns performance, edge cases, maintenance or the project's conventions, the choice depends on the reader's priorities, the choice is expensive to reverse, or you consider the rejected option better. Record the options, what the chosen one loses, and why you chose it. If you consider a rejected option better, say so and record what it would take.
- **Constraints.** A constraint is a condition that makes the solution more complex than it would be without it, or limits its result. Record each constraint, where it came from (the request, existing code, an external contract or your assumption), and why you kept it. Record what it costs: the solution that you would write without it, the parts of the change that it added or made more complex, and the capabilities that you gave up.
- **Undoing.** Each significant difficulty of undoing the change that you know of. A difficulty is work beyond reverting the code and releasing it again, or something that a return to the earlier state cannot restore, once users, data, other code or external consumers rely on the change. It is significant when knowing it could change whether the reader accepts the change, what they check carefully or how they prepare the release. Also record each way to recover that the change provides, such as a down migration or a feature flag.
- **Facts.** Other non-obvious facts that you found. Distinguish observations from hypotheses.
- **Effects.** What a user or caller should notice, and how someone can check it, with the expected result. Call it a proposed check until it runs.
- **Open.** What remains unverified, unclear or dependent on unavailable context.

## Verification

Record each actual run as its own entry under `## Verification`:

```markdown
### V1
- Command: `python3 -m unittest discover -v`
- Result: 5 tests passed.
- State: working tree after the last edit, before the commit.
```

Record these runs too:

- The reproduction of the problem before the fix and after it: the steps and the results.
- Each test that you add or change for the problem, on the code without the fix, and its result.
- Each blocker, with the result `Blocked`. A blocker is an action that you needed for the work or its verification but could not complete for a reason outside the change: the environment, missing access, or a failure that already existed in the project. Record the command or action, an excerpt of its error output, and the result on the code before your change if you ran it. Record what you skipped or did instead, whether the workaround stays in the project, and what would remove the blocker.

Run the final check after the last edit, so that it covers the result. Name the state it tested: a commit hash, or the working tree after the last edit. If an entry tests an earlier state, say which edits it does not cover. Keep the origin of results that someone else supplied.

## Keep it current

Update the notes when an important decision changes, a constraint or non-obvious fact is found, or a check runs. Replace obsolete understanding instead of keeping a diary of reversals. Before finishing, reconcile the notes with the final state. Keep each entry that adds context the diff does not show, including significant uncertainty and real verification evidence; do not narrate routine edits or copy code into prose.

Do not classify the overall change, organize it into report sections, or write the human-facing explanation. Record no passwords, tokens or unrelated private data.

Commit the notes with the code change so that they travel with it; the explainer finds them among the files that the range changes. This instruction does not itself authorize a commit, push or publication that the task does not authorize.
