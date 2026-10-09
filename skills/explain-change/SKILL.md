---
name: explain-change
description: Writes a report that explains a coding agent's change to the person who gave the task, so that they can tell whether it matches the request, how it works, what follows from it and what to check.
disable-model-invocation: true
---

# Explain Change

The reader gave a coding agent a task. They know the project and their request, but did not write the code. Write a report that lets them understand the change with less effort than reading the diff: whether the agent understood the request, why the change is made, how it works and what follows, which decisions and constraints to judge, and where to look deeper or what to check. Keep what is significant and what is unknown; leave out the rest.

## Inputs

- **Range.** Use the range the user gives. Otherwise use the merge-base of `HEAD` with `origin/HEAD`, `main` or `master` as base and `HEAD` as head; if that range is empty, ask. Record both ends as commit hashes and base every claim on the code at them (`git show <hash>:<path>`), not on memory. If tracked files have uncommitted changes, use the hash from `git stash create` as head, and name untracked files that belong to the change.
- **Request.** The task that the reader gave the agent: text, a plan file or an issue. Use the one the user gives, or the one that the notes quote. If there is none, ask once.
- **Claims.** Commit messages, the agent's final message and the notes under `.change-notes/` are claims of the agent. Find everything yourself and check each claim against the code. If the notes leave out a significant change or claim what the code does not show, say so under Review first.
- **Running code.** Read diffs with `git diff --no-ext-diff --no-textconv`. Do not run the project's tests, build, hooks or dependency installs, and do not repeat the agent's commands. To confirm a claim about behavior, you may copy a changed function from `git show` into a temporary file and run it on a small input, with no network and with writes only to a temporary directory. Give the command and result in the section that it supports.

Save the report to the path the user gives, or to `.change-notes/<short head hash>.explanation.md`. Do not commit it.

## Understand the change

Put most of the work here: a report cannot be better than your understanding.

1. Read every change and the code around it. Find its reason in the request, the notes, commit messages, a changelog or docs; if no source states it, mark your inference as inferred, or say that the reason is unknown.
2. Work from the outside in: find the actions whose result changes, such as commands, endpoints, public APIs, UI actions and configuration. Work from the inside out: follow changed code to its callers, unchanged ones included, until you reach a result that a user or caller can observe, or a named gap in your knowledge.
3. For a restructuring, ask which behavior must stay the same, and whether it does. For a new capability, ask what happens in its ordinary failure cases.
4. Mark facts that only the notes give as `per notes`. A proposed check is not a result: report a run only when it ran, with its command, result and the code state that it tested.

Then work through the topics below; [Write the report](#write-the-report) says where each finding goes. The reader must not miss these findings, so each goes in the Summary:

- a part of the request that the result does not cover;
- a fix that leaves the cause of the problem;
- a test added for the problem that passes without the fix;
- an architectural consequence of a seam that the request did not ask for;
- a grounded concern that a constraint costs more than it is worth;
- a tradeoff that is expensive to reverse, or where the agent considers the rejected option better;
- a significant difficulty of undoing the change;
- a blocker that changed the result or left part of it unverified.

### The request

The request is every instruction that the reader gave for the task, including a plan that the reader approved; a later instruction replaces an earlier one. A purpose that the request states is a requirement too: a result that does what the request says but misses its purpose does not cover it. A requirement is covered when the changed behavior can be reached from its entry point and gives the requested result. Code that is never called, or a setting that is read but has no effect, does not cover it.

Divide the request into requirements and decide from the code whether the result covers each one. Compare the task as the notes restate it with the request. Tell the reader each requirement that the result does not do, does only partly or does differently; each change beyond the request; each reading of an ambiguous request that the agent chose; each place where the approved plan narrows or changes the original message; and that the rest is done. If there is no request, say that coverage cannot be determined. Do not reconstruct the request from the change: the change always covers a request reconstructed from it.

### Cause or symptom

This applies to any change, or part of one, that responds to an observed problem, whatever its label. The cause is the nearest point in the project whose fix removes the symptom on every path that leads to it, not a deeper point in design or process. A cause outside the project is a boundary: name it. Tell the reader:

1. The symptom and its cause. Say that the fix removes the cause only after you show how the cause leads to the symptom and that the fix acts on the cause.
2. If the fix leaves the cause, whether it hides the symptom, and what exactly is no longer seen: for example, it catches and ignores an error, retries, loosens or skips a test, or special-cases the input from the report.
3. Other paths from the same cause in unchanged code, and whether the fix covers them.
4. The evidence that the symptom is gone.
5. If the cause stays: why, what remains open and what would fix it. If the request asked to remove only the symptom, say that the cause stays, but not as a difference from the request.

### Checking a fix

This applies to the same changes. A check shows that a fix works only if the same check showed the problem before the fix. Tell the reader:

1. Steps to see that the problem is gone, at the place where it was observed: in the product, or in the test or CI command that showed it. Give the input, the result to expect and whether someone ran them. If the notes give no steps, derive them from the code and say that no one ran them.
2. For each test that covers the problem, what it checks and whether it fails without the fix, or that no test covers it. Decide from a recorded run on the code without the fix, or from the code. Look for a test that checks a neighbouring case or mocks the part that failed. A compile or import error does not show that a test guards the fix. If you cannot decide, say so and give the steps to check.

### Seams and architectural consequences

This applies to any change, including a small fix. A seam is a place where a module uses another module or the outside world, together with the contract that passes through it. Take module boundaries from the structure of the project. A dependency inside one module is not a seam; a use of another module's internals is. A library, a service or a configuration option is always a seam. An architectural consequence is what follows from a change in seams for other code and later changes: where decisions now belong, what modules must know about one another, and which later changes now need coordinated changes in several modules. It concerns the structure of the project, not how existing callers behave.

Find the seams that the change adds, changes or removes, and read the modules on both sides of each, not the whole project. Tell the reader:

1. Each architectural consequence, traced from its seam through what relies on it. A list of dependencies is not enough.
2. For each such seam, its contract and whether the request asked for it.
3. Your assessment of each added or changed seam: whether it is necessary, whether it fits how the project already connects these modules, and what it will cost to remove later. Do not ask whether a requested seam is necessary to meet the request. If the assessment is negative, give a concrete alternative.
4. That the change has no architectural consequences, when that is so.

### Constraints

A constraint is a condition that makes the solution more complex or limits its result. It comes from the request, existing code, an external contract or an assumption of the agent. To relax a constraint is to remove it or make it weaker. A constraint that the request states needs assessment too; one whose value is not in question needs no account. For each constraint whose value is in question, tell the reader:

1. The constraint and its benefit.
2. What keeping it costs: extra work in the change, later maintenance, testing, operation, or capabilities that the change gives up. Separate it from the cost of the agent's implementation: an implementation that is more complex than necessary does not show that the constraint costs too much.
3. What relaxing it would make simpler or possible, and what benefit would be lost. The notes can record the solution that the agent would write without it.
4. Your assessment of whether the benefit is worth the cost, with its grounds. If information is missing, say what is known and what is needed; if the benefit is unknown, do not say that there is none. Name each assumption that the assessment depends on.

### Tradeoffs

This applies to any change. A tradeoff is a decision of the agent where the chosen option loses something that another viable option would keep. A viable option meets the request within the project. A choice that the request prescribes is not a decision of the agent, and options that differ only in style, or between which the project's conventions already decide, are not a choice. A tradeoff matters when at least one of these is true:

- A user or a caller notices the loss, or the loss concerns performance, edge cases, maintenance or the project's conventions.
- The choice depends on the reader's priorities rather than on a technical fact.
- The choice is expensive to reverse: other code, stored data, a file format, a protocol, a public API or a configuration option starts to rely on it.
- The agent considers the rejected option better.

Name an option that the agent did not record only when the project supports it: the project already does it this way elsewhere, the library is already a dependency, or a suitable mechanism is nearby; say that you found it, not the agent. For each tradeoff that matters, tell the reader the decision and its options; what the chosen option gives, what it loses, and who or what notices the loss; why the agent chose it, or that the reason is unknown; your assessment, with its grounds; and what it costs to switch now.

### Undoing the change

A difficulty of undoing is work beyond reverting the code and releasing it again, or something that a return to the earlier state cannot restore. It appears once users, data, other code or external consumers rely on the change. A difficulty is significant when knowing it could change what the reader does: accept the change or ask for another approach, check a part more carefully, or prepare the release. Judge this by what cannot be restored, who else must take part in undoing, and how much work undoing takes compared with the change itself, also for a part that the request asked for. A difficulty that the project undoes as a routine, with nothing lost, is not significant; give no account of it.

For each significant difficulty, tell the reader why it deserves attention: what undoing would take, such as reverting the code, restoring data, reworking code that depends on the change, or agreeing on the change with external consumers; what makes this work hard; the event after which undoing becomes harder, if there is one; and what cannot be restored, or what nobody knows how to restore. These points explain one difficulty; they are not a form for each part of the change. Judge a way to recover, such as `git revert`, a down migration or a feature flag, by the work that it actually does: where it matters, say what work it does and what work remains. If something unknown stops you from judging a difficulty, say what. Do not give numbers for time or cost without grounds.

### Blockers

A blocker is an action that the agent needed for the work or its verification but could not complete for a reason outside the change: the environment, missing access, or a failure that already existed in the project. A check without a record of a run, or one made differently from how the project works, is a sign of a blocker. Leave out blockers that the agent got past without an effect. For each blocker that changed the result or left part of it unverified, tell the reader what it changed or left unverified; whether it is real or the change caused it, as you judge from the recorded output, the code and the configuration; if there is no evidence, say that it is unknown and give the command that the reader can run; and what the reader can do to remove it and complete the work or the check.

## Write the report

Follow this shape:

````markdown
# <What the change does>

Range `<base>..<head>`.

## Summary

<The idea of the change.>

<Whether the result matches the request.>

### What changed

[<Section title>](#section-title)

- <Entity> previously <old behavior>. It now <new behavior>. <What follows.>

<Whether the change has architectural consequences.>

**Review first.** <Items, each with a reason and a section link.>

**What ran.** <Runs and blockers.>

## <Section title>

<How the change works, why, what follows.>

### Implementation details

<How the code produces this behavior.>
````

The shape sets the reading order, not a quota: leave out a block that has nothing to say.

**Sections.** Give a section to each purpose that the reader can accept or reject on its own: a fix, a capability, a restructuring, or a meaningful change of tests, docs or configuration. Changes that make sense only together share a section; unrelated changes get separate sections, even in one file. Tests go with the behavior that they test. Put the main goal first, then what the reader needs before later sections.

**Summary.** Write it last. The reader reads it first, decides from it whether to accept the change and where to look deeper, and may stop after it. The reader wrote the request, so the Summary does not retell it.

- *Idea.* What was wrong or missing, and how the change deals with it, in the project's concepts. Without it, the reader links only neighbouring facts and cannot judge the change as a whole.
- *Request.* If the result does everything as asked, say so in one sentence. Otherwise state each difference from [The request](#the-request) once, with a section link, and say what the reader must decide and the consequence that the decision depends on. Without a request, say that the result cannot be compared with one.
- *What changed.* Each section's linked title on its own line, with bullets under it. Each bullet states one difference in behavior: what the entity did previously and what it does now. Add what follows for users, callers, stored data or installations when that could change whether the reader accepts the change, what they check or how they prepare the release. Then say in one sentence whether the change has architectural consequences, with a section link. Use GitHub-style anchors.
- *Review first.* Each finding that the reader must not miss, except differences from the request and blockers, which have their own places. Then the other decisions, risks, unknowns and constraints that most need attention. Give each item once, with its reason and a section link.
- *What ran.* What actually ran, who reported it and on which state, or that no run is known. Then each blocker.

Each conclusion that the reader draws from the Summary alone must hold: whether the request is met, how the change works, what follows, whether to accept it, what to check and where to look. A statement can be literally true and still fail: "Users can now sign in with a passkey" is true when only a user who already has a password can add a passkey, but the reader concludes that a new user can sign up without a password. A reader who reads on often keeps reasoning from the Summary, so a section adds detail but never corrects it. Leave out detail, but name in a few words each limit that changes such a conclusion; do not hide it in a more general statement, because the limit tells the reader where to look.

The Summary describes behavior and the project's concepts: what users and callers observe, and the commands, endpoints, settings and modules that the reader knows. It does not name the code that implements them, such as functions, conditions or files.

**Section text.** Start with what the Summary does not say. Give the local context that the reader needs, but do not explain the project's architecture or the language. Explain how the parts now work together, why this solution was chosen, and what follows for users, callers, existing data and installations, and for docs that the change makes wrong. Put each finding in the section of the purpose that it concerns; a seam goes with the purpose that added it. Name each unknown and what would resolve it. Give the steps to check each fix. Elsewhere, propose a check only when it resolves an unknown, or when the code predicts a result other than the intended one; say what to do and what result to expect, and in the second case give both results.

**Assessments.** State the facts first, in their own sentences. Then write each assessment that a topic asks for as a sentence that starts with "In my judgment,", with its grounds and what you do not know. The grounds refer only to facts already stated, so that the reader can tell your judgment from what the code shows.

**Implementation details.** Show how the code produces the behavior: the conditions, the data flow and the dependencies on unchanged code. Do not retell the section; a fact that users or callers can observe belongs in the section text. When seeing code explains a point, put a real, shortened `diff` directly below the sentence that it supports: keep identifiers and significant conditions, leave out only whole lines, and mark each omission with a line `...`. For a mechanism spread across files, use labelled pseudocode or a small table of cases. Refer to code as `path:line` at head.

**Language.** Write in ASD-STE100 Simplified Technical English, unless the user or the project asks for another language: short sentences with one statement each, active voice, verbs instead of abstract nouns. Start each paragraph and bullet with the entity and the main fact. Use the project's terms, one term for one concept, and keep identifiers; do not invent names for things that the project does not name. Connect facts with words such as "because", "so" and "only if", not with arrows or symbol chains. Do not use jargon or buzzwords, such as "leverage", "robust", "surface" as a verb, "harden", "plumbing" or "load-bearing". Do not list the inputs that you read, describe your method or tell the history of your work.

## Read it as the reader

Read the whole report as the reader, and revise it until each answer is yes:

1. For each statement in the Summary, does the conclusion that the reader draws from it without the sections hold against the sections and the code? Does the Summary hold every finding that the reader must not miss? If a section narrows or contradicts the Summary, change the Summary.
2. Does each section explain how its change works and what follows, and do its implementation details add how the code does it?
3. Can the reader tell an inferred reason from a stated one, a proposed check from a run, and your judgment from a fact?
4. Is every sentence one that the reader needs? Remove repeated, obvious and method sentences, but not significant facts or unknowns.

If this shape stops you from writing a useful explanation, say so. Return the report path, the range and any material limits.
