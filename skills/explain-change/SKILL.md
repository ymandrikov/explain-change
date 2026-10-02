---
name: explain-change
description: Use when the person who gave a coding agent a task needs to understand the resulting change — whether it matches the request, how it works, what follows from it and what to check.
---

# Explain Change

The reader gave a coding agent a task. They know the project and their request, but they did not write the code. Write a report that lets them understand the change with less effort than reading the diff: whether the result matches the request, how the change works and why, what follows from it, which decisions and constraints to judge, and what to check. An explanation compresses meaning. It keeps what matters and what is unknown, and leaves out the rest.

## Inputs

- **Range.** Use the range the user gives. Otherwise use the merge-base of `HEAD` with `origin/HEAD`, `main` or `master` as base and `HEAD` as head. If that range is empty, ask. Record both ends as commit hashes and base every claim on the code at them (`git show <hash>:<path>`), not on memory. If tracked files have uncommitted changes, use the hash from `git stash create` as head, and name untracked files that belong to the change.
- **Request.** The task that the reader gave the agent: text, a plan file or an issue. Use the one the user gives, or the one that coding notes quote. If there is none, ask once.
- **Author statements.** Commit messages and coding notes under `.change-notes/` are claims of the author. Check them against the code. If notes leave out a significant change or claim something that the code does not show, say so under Review first.
- **Running code.** Read diffs with `git diff --no-ext-diff --no-textconv`. Do not run the project's tests, build, hooks or dependency installs. To confirm a claim about behavior, you may copy a changed function from `git show` into a temporary file and run it on a small sample input. The script must write only to a temporary directory and use no network. Give its command and result in the section that it supports.

Save the report to the path the user gives, or to `.change-notes/<short head hash>.explanation.md`. Do not commit it.

## Understand the change

Put most of the work here. A report cannot be better than your understanding of the change.

1. Read every change and the code around it. Know what each change does and why it is there.
2. Work from the outside in: find the actions whose result changes, such as commands, endpoints, public APIs, UI actions and configuration. Work from the inside out: follow changed code to its callers, unchanged ones included, until you reach a result that a user or caller can observe, or a named gap in your knowledge.
3. Find the reason for each change in the request, the notes, commit messages, a changelog or docs. If no source states it, infer it and mark it as inferred, or say that it is unknown.
4. Ask the question of each kind of change. A restructuring: which behavior must stay the same, and does it? A new capability: what happens in its ordinary failure cases?
5. Keep evidence apart from belief. Mark inferred reasons as inferred and facts from notes only as `per notes`. A check that you propose is not a result. Report a run only when it ran, with its command, result and the code state that it tested.

Then work through the topics below. Each topic says when it applies, what to find and what the reader must learn. The topics are what you look for, not sections of the report: [Write the report](#write-the-report) says where each finding goes. Find everything yourself. Statements of the agent, such as notes, commit messages or its final message, are claims: check them against the code.

### The request

The request is every instruction that the reader gave for the task. A later instruction replaces an earlier one. A plan that the reader approved is part of the request. A purpose that the request states is a requirement too: a result that does what the request says but does not reach its stated purpose does not cover it. A part of the request is covered when the changed behavior can be reached from its entry point and gives the requested result. Code that is never called, or a setting that is read but has no effect, does not cover it.

1. Divide the request into requirements. If the reader approved a plan, also find where it narrows or changes the original message. With notes, also compare their Task with the request.
2. Decide from the code whether the result covers each requirement: done, done partly, done differently or not done.
3. For each change, find the requirement that it serves, or note that it goes beyond the request. Note where the agent chose one reading of an ambiguous request.
4. If there is no request, say that coverage cannot be determined. Do not reconstruct the request from the change, because the change always covers a request reconstructed from it.

The reader must learn each part of the request that the result does not do, does only partly or does differently; each place where the approved plan narrows or changes the original message; and that the rest of the request is done. The reader must not miss a part of the request that the result does not cover.

### Cause or symptom

This topic applies to any change, or part of a change, that responds to an observed problem, whatever its label. The cause is the nearest point in the project whose fix removes the symptom on every path that leads to it. Do not go deeper, to design or process. If the cause is outside the project, name it as a boundary.

1. Decide whether the change, or a part of it, responds to an observed problem.
2. Trace the cause in the code yourself, even when the notes name it: the notes can repeat the mistake of the agent.
3. Find whether the fix removes the cause. If it does not, find whether it hides the symptom: for example, it catches and ignores an error, retries, loosens or skips a test, or special-cases the input from the report. Name the mechanism: what exactly is no longer seen.
4. Look in unchanged code for other paths from the same cause, and find whether the fix covers them.
5. Never say that the cause is removed without showing how the cause leads to the symptom and that the fix acts on the cause.
6. If the request asked to remove only the symptom, say that the cause stays, but do not present it as a difference from the request.

The reader must learn the symptom and its cause; whether the fix removes the cause, and if it does not, whether it hides the symptom; other paths from the same cause and whether the fix covers them; the evidence that the symptom is gone; and if the cause stays, why, what remains open and what would fix the cause. The reader must not miss a fix that leaves the cause.

### Checking a fix

This topic applies to the same changes as the previous one. A check shows that a fix works only if the same check showed the problem before the fix.

1. Take the steps to see that the problem is gone from the notes, and check them against the code. The steps work at the place where the problem was observed: in the product, or in the test or CI command that showed it. They give the input and the result to expect. If the notes have no steps, derive them from the code and say that no one ran them.
2. For each test that covers the problem, decide whether it fails without the fix, from a recorded run on the code without the fix or from the code. Look for a test that checks a neighbouring case or mocks the part that failed. A compile or import error does not show that a test guards the fix. Do not run the tests. If you cannot decide, say so and give the reader the steps to check.

The reader must learn the steps, with the input, the result to expect and whether someone ran them; and for each test that covers the problem, what it checks and whether it fails without the fix. If no test covers the problem, say so. The reader must not miss a test added for the problem that passes without the fix.

### Seams and architectural consequences

This topic applies to any change, including a small fix. A seam is a place where a module uses another module or the outside world, together with the contract that passes through it. Take module boundaries from the structure of the project. A dependency inside one module is not a seam. A use of another module's internals is a seam. A dependency on the outside world, such as a library, a service or a configuration option, is always a seam. An architectural consequence is what follows from a change in seams for other code and later changes: where decisions now belong, what modules must know about one another, and which later changes now need coordinated changes in several modules. It concerns the structure of the project, not how existing callers behave.

1. Find the seams that the change adds, changes or removes, and mark which ones the request asked for.
2. Find what relies on each seam, and trace each consequence from its seam. A list of dependencies is not enough.
3. Assess each added or changed seam: is it necessary, does it fit how the project already connects these modules, and what will it cost to remove later. Do not ask whether a requested seam is necessary to meet the request. If the assessment is negative, give a concrete alternative.
4. Read the modules on both sides of a seam, not the whole project.

The reader must learn each architectural consequence and the seam that it follows from; for each such seam, its contract and whether the request asked for it; and your assessment of each added or changed seam, with its grounds. If the change has no architectural consequences, the reader must learn that. The reader must not miss an architectural consequence of a seam that the request did not ask for.

### Constraints

A constraint is a condition that makes the solution more complex than it would be without it, or limits its result. It comes from the request, existing code, an external contract or an assumption of the agent. To relax a constraint is to remove it or make it weaker.

1. Assess a constraint that the request states too: an explicit request does not exempt it.
2. Know what keeping each constraint costs: extra work in the change, later maintenance, testing, operation, or capabilities that the change gives up.
3. Separate the cost of the constraint from the cost of the implementation that the agent chose. An implementation that is more complex than necessary does not show that the constraint costs too much.
4. For a constraint whose value is in question, find what relaxing it would make simpler or possible, and what benefit would be lost. The notes can record the solution that the agent would write without the constraint; it is a claim too.
5. If information is missing, say what is known and what is needed to judge the constraint. If the benefit is unknown, do not say that there is no benefit. If the assessment depends on an assumption, name it.

For each constraint whose value is in question, the reader must learn the constraint and its benefit; what keeping it costs; what would become simpler or possible if it were relaxed; and your assessment of whether the benefit is worth the cost, with its grounds and what is unknown. A constraint whose value is not in question does not need a separate account. The reader must not miss a grounded concern that a constraint costs more than it is worth.

### Tradeoffs

This topic applies to any change. A tradeoff is a decision of the agent where the chosen option loses something that another viable option would keep. A viable option meets the request within the project. A choice that the request prescribes is not a decision of the agent. Options that differ only in style, or between which the project's conventions already decide, are not a choice.

A tradeoff matters when at least one of these is true:

- A user or a caller notices the loss, or the loss concerns performance, edge cases, maintenance or the project's conventions.
- The choice depends on the reader's priorities rather than on a technical fact, so the reader could choose differently.
- The choice is expensive to reverse: other code, stored data, a file format, a protocol, a public API or a configuration option starts to rely on it.
- The agent considers the rejected option better.

Name an option that the agent did not record only when the project supports it: the project already does it this way elsewhere, the library is already a dependency, or a suitable mechanism is nearby. Say that you found it, not the agent.

For each tradeoff that matters, the reader must learn the decision and its options; what the chosen option gives, what it loses, and who or what notices the loss; why the agent chose it, or that the reason is unknown; your assessment, with its grounds; and what it costs to switch to another option now. The reader must not miss a tradeoff that is expensive to reverse, or where the agent considers the rejected option better.

### Undoing the change

A difficulty of undoing is work beyond reverting the code and releasing it again, or something that a return to the earlier state cannot restore. It appears once users, data, other code or external consumers rely on the change. A difficulty is significant when knowing it could change what the reader does: accept the change or ask for another approach, check a part more carefully, or prepare the release of the change. Judge this by what cannot be restored, who else must take part in undoing, and how much work undoing takes compared with the change itself. This holds also when the request asked for the part. A difficulty is not significant when the project undoes such changes as a routine and nothing is lost.

1. Check each difficulty against the code, including the ones that the notes record.
2. Decide which difficulties are significant. For each, say why it deserves attention. Do not give an account of simple or insignificant cases.
3. Judge a way to recover, such as `git revert`, a down migration or a feature flag, by the work that it actually does. Its presence alone does not make undoing easy. Where it matters for your assessment, say what work it does and what work remains.
4. If something unknown stops you from judging a difficulty, say what is unknown.
5. Do not give numbers for time or cost without grounds.

For each significant difficulty, the reader must understand what undoing the change would take, such as reverting the code, restoring data, reworking code that depends on the change, or agreeing on the change with external consumers; what makes this work hard; the event after which undoing becomes harder, if there is one; and what cannot be restored, or what nobody knows how to restore. These points explain one difficulty. They are not a form to fill in for each part of the change. A change without significant difficulties needs no separate account. The reader must not miss a significant difficulty.

### Blockers

A blocker is an action that the agent needed for the work or its verification but could not complete for a reason outside the change: the environment, missing access, or a failure that already existed in the project. For example, the sandbox does not let the tests run, or the build tool fails in the sandbox.

1. A check that has no record of a run, or a check made in a different way from how the project works, is a sign of a blocker.
2. From the recorded output, the code and the configuration, judge whether the blocker is real or the change caused it. Do not run the commands again. If there is no evidence, say that it is unknown and give the command that the reader can run.

The reader must learn each blocker that changed the result or left part of it unverified, and what it changed; whether the blocker is real or the change caused it, as you judge it; and what the reader can do to remove the blocker and complete the work or the check. Do not report blockers that the agent got past without an effect. The reader must not miss a blocker that changed the result or left part of it unverified.

## Write the report

Follow the shape of [the example](example.md):

````markdown
# <What the change does>

Range `<base>..<head>`.

## Summary

**Request and result.** <Whether the result matches the request. Each difference once.>

### What changed

[<Section title>](#section-title)

- <Entity> previously <old behavior>. It now <new behavior>.

<Whether the change has architectural consequences, with a section link.>

**Review first.** <Findings that the reader must not miss, then other items, each with a reason and a section link.>

**What ran.** <What ran, who ran it, on which state. Each blocker.>

## <Section title>

<How the change works, why it is done this way, what follows. Unknowns and checks.>

### Implementation details

<How the code produces this behavior, with a short real diff where it helps.>
````

The shape sets the reading order, not a quota. Leave out a block that has nothing to say.

**Sections.** Give each purpose that the reader can accept or reject on its own a section: a fix, a capability, a restructuring, or a meaningful change of tests, docs or configuration. Changes that make sense only together share a section. Unrelated changes get separate sections, even in one file. Tests go in the section of the behavior that they test. Put the main goal first, then what the reader needs before later sections.

**Summary.** Write it last. A reader who stops after the Summary knows whether the result matches the request, what changed, where to look first, and what did not run. It holds every finding that the reader must not miss.

- *Request and result.* The reader wrote the request, so do not retell it. Say in one sentence what was done as asked. Then state each difference once, with a section link: not done, done partly, done differently, beyond the request, one reading of an ambiguous request, or a place where the approved plan departs from the original message. Say what the reader must decide and the consequence that the decision depends on. Without a request, say that the result cannot be compared with one.
- *What changed.* Put each section's linked title on its own line, with bullets under it. Each bullet states one difference in behavior: what the entity did previously and what it does now. Then say in one sentence whether the change has architectural consequences, with a section link. Link to sections with GitHub-style anchors.
- *Review first.* Each finding that a topic marks as one that the reader must not miss, except differences from the request and blockers, which have their own places. Then the other decisions, risks, unknowns and constraints that most need attention. Give each item its reason and a section link. Give a finding that several topics mark as one item, and do not repeat the differences from the request.
- *What ran.* What actually ran, who reported it and on which state. If no run is known, say so. Then each blocker: what did not run or ran differently, what it changed or left unverified, whether it is real or the change caused it, and what the reader can do to complete the work or the check.

**Section text.** The reader comes from the Summary, so start with what it does not say. Give the local context that the reader needs, but do not explain the project's architecture or the language. Explain how the parts now work together, why this solution was chosen, and what follows for users, callers, existing data and installations, and for docs that the change makes wrong. Put each finding from the topics in the section of the purpose that it concerns; a seam goes in the section of the purpose that added it. Name each unknown and what would resolve it. Give the steps to check each fix, as [Checking a fix](#checking-a-fix) says. Elsewhere, propose a check only when it resolves an unknown, or when the code predicts a result other than the intended one. Say what to do and what result to expect; in the second case, state both results.

**Assessments.** Write each assessment that a topic asks for as a sentence that starts with "In my judgment,", followed by its grounds in the code and what you do not know. Keep facts out of these sentences, so that the reader can tell your judgment from what the code shows.

**Implementation details.** Show how the code produces the behavior above: the conditions, the data flow and the dependencies on unchanged code. Do not retell the section. A fact that users or callers can observe belongs in the section text, even when only the code shows it. When seeing code explains a point, put a real, shortened `diff` directly below the sentence that it supports. Keep identifiers and significant conditions, leave out only whole lines, and mark each omission with a line `...`. For a mechanism spread across files, use labelled pseudocode or a small table of cases instead. Refer to code as `path:line` at head.

**Language.** Write in ASD-STE100 Simplified Technical English, unless the user or the project asks for another language.

- Write short sentences with one statement each, in active voice. Start each paragraph and bullet with the entity and the main fact.
- Use the project's terms from its code, docs and glossary, one term for one concept. Keep identifiers.
- Connect facts with words that state the relation, such as "because", "so" and "only if".
- Do not use jargon or buzzwords, such as "leverage", "robust", "surface" as a verb, "harden", "plumbing" or "load-bearing".
- Do not use arrows or symbol chains, such as `A → B → C`, in place of sentences.
- Do not use abstract nouns in place of verbs: write "validates", not "performs validation of".
- Do not invent names for things that the project does not name.
- Do not list the inputs that you read, describe your method or tell the history of your work.

## Read it as the reader

Read the whole report as the reader, and revise it until each answer is yes:

1. From the Summary alone, can the reader tell whether the result matches the request, what changed, what to look at first and why, and what did not run? Does the Summary hold every finding that the reader must not miss?
2. Does each section explain how its change works and what follows? Do its implementation details add how the code does it, instead of retelling the section?
3. Can the reader tell an inferred reason from a stated one, a proposed check from a run, and your judgment from a fact?
4. Is every sentence one that the reader needs? Remove repeated, obvious and method sentences.

Length is not a goal: do not drop significant facts or unknowns to be short. If this shape stops you from writing a useful explanation, say so.

Return the report path, the range and any material limits.
