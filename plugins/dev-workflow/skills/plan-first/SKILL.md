---
name: plan-first
description: Runs the developer's plan-before-code ritual — ask clarifying questions, write a plan, get explicit go-ahead before touching any code. Use whenever they ask to plan something out before building, say not to make changes until they approve, or ask for a plan they'll "ok when ready." Trigger on "/plan-first" as well as natural-language versions like "let's create a plan to implement this," "ask me clarifying questions first," "don't make any code changes until I give you explicit permission," or "I'll ok the plan when I'm ready." Do not use for the build step itself (new branch, incremental commits) or for ship/deploy requests — this skill stops at the approved plan and hands off from there.
---

# Plan-first gate

Two things are non-negotiable here:

1. **No code changes before an explicit go-ahead.** The gate holds for the whole task.
2. **The plan is narrowed through a discussion, not a single round of questions.** You ask, the developer answers, you ask again, until nothing in the plan would be a guess on your part. Only then do you draft.

The developer has said the failure mode they see most is too few questions, and questions typed out as prose at the end of a message instead of actually asked. This skill exists to fix both.

## 1. Every question goes through the AskUserQuestion tool

The harness default is to reserve AskUserQuestion for when you're truly blocked. **This skill overrides that default: the bar for asking is deliberately low here.** Asking is the work, not a cost of the work.

Rules:

- Every clarifying question is asked with the `AskUserQuestion` tool. Never write a question in plain text and end your turn on it. If a turn contains an open question, that turn ends on an AskUserQuestion call and nothing else follows it.
- Batch up to four questions per call. Give each question two to four concrete options drawn from what you actually found in the repo or conversation, with your recommended option first and marked "(Recommended)". The developer can always type their own answer via "Other," so options don't need to be exhaustive, but they must be specific. "Option A / Option B" is not an option; "frontmatter field `project:`" versus "nested tag `#project/<slug>`" is.
- Do the research before you ask. Read the relevant files, grep for existing conventions, check what the request would touch. A question that the repo already answers is a wasted question. A question whose options are grounded in real file names, real existing patterns, and real tradeoffs is a good one.
- While questions are open, output only the questions. Do not include a plan sketch, a "here's roughly what I'd do," or a list of assumptions alongside them. The plan comes after the discussion closes.

## 2. Keep asking until nothing would be a guess

This is a loop, not a step:

1. Research the request against the repo and conversation.
2. Ask the questions you have, via AskUserQuestion.
3. Read the answers. Answers almost always open new questions (choosing a mechanism raises "what about the existing data," choosing a scope raises "which edge cases are in"). Re-examine the request with the new information.
4. Go back to step 2 until the readiness test below passes.

**Readiness test, run before drafting:** list every decision the plan will make. For each one, ask: would I be choosing this on the developer's behalf? If the answer is yes for any of them, that decision is a question you haven't asked yet. Go back to the loop. Draft only when the list of decisions-you'd-be-guessing is empty.

What counts as askable is broad. All of these are fair questions even when the request is technically unambiguous:

- Scope edges: does this also cover X? Should Y be left alone?
- Naming, ordering, and structure the developer will have to live with.
- Behavior on edge cases and failure paths.
- Preferences and taste calls: how something should read, look, or be organized.
- Whether an adjacent problem you noticed should be folded in or explicitly left out.
- Anything you'd otherwise write into the plan as "assumption," "TBD," "to confirm," or "open question."

The only real guard is the one above: don't ask what the repo or conversation already answers. A fully specified request that touches nothing surprising can pass the readiness test on the first pass, but that's the outcome of running the test, not a shortcut around it. If you find yourself about to skip questions, run the test explicitly and be honest about what you'd be deciding alone.

## 3. Draft the plan only after the discussion closes

Once the readiness test passes, write the plan. It must be free of unresolved items: no assumptions, no TBDs, no "to confirm with the developer," no "not blocking but worth noting." Every one of those is a question that should have been asked in Step 2. If one surfaces while drafting, stop drafting and ask it.

The plan should be concrete enough that `execute-plan` can build it without needing to come back for a decision: which files, what changes, in what order, and how you'll know it worked.

## 4. Close with a question too, and then hold

After presenting the plan, ask via AskUserQuestion whether it's approved as-is, needs a change, or the developer has more context to add. Do not end the plan message with "let me know when you're ready" or any other prose prompt.

Then hold: make no code, config, or other executable-file changes until the developer either invokes the `execute-plan` skill or gives an equivalent explicit go-ahead ("implement this plan," "build it now," "go ahead and build it"). The only exception is `.md` files, see below.

This hold is binding for the rest of the task, not just the next message. A vague "sounds good" or "yeah that works" reacting to the plan is not the same as one of those go-ahead forms. If it's unclear whether they're greenlighting the build or just reacting, ask with AskUserQuestion, don't start editing.

Don't call `EnterPlanMode`/`ExitPlanMode` for this. The gate is you holding, not plan mode.

**Exception for docs:** creating or editing a `.md` file is fine during the hold when it's the plan doc itself, or another doc the task genuinely calls for. Never source code, config, or anything executable.

## 5. Save the plan to match the project's own convention, don't invent one

The developer's projects don't share a single fixed location for plan docs (some use `docs/plans/`, others `docs/planning/`, others something else entirely, and plenty don't keep a persistent plan file at all). Guessing a path and creating `docs/plans/` in a project that's never used it adds clutter they didn't ask for.

Before or while drafting, check whether this project already has a convention:

```
find . -iname '*.md' -ipath '*plan*' -not -path '*/node_modules/*' -not -path '*/.git/*'
```

(Use the platform's real equivalent if `find` isn't available, e.g. PowerShell's `Get-ChildItem -Recurse -Filter *.md`.)

- **If you find an existing plans folder with prior docs in it**, follow its established pattern: same folder, same filename shape (date-prefixed, slug-named, numbered, whatever the existing files actually do), same rough structure and header style if one is consistent across them. Write the plan there once it's approved.
- **If you find no existing convention**, don't default to `docs/plans/` or any other fixed path. Presenting the plan in the conversation is enough on its own. Only write it to a file if asked, and if so, ask where (AskUserQuestion, with a sensible spot in that project as the recommended option) rather than hardcoding one.

## Scope

This skill ends at the approved plan. Starting the branch, writing incremental commits, or deploying are separate steps the developer will ask for explicitly, typically by invoking the `execute-plan` skill, which is the expected handoff from here. Don't roll straight into implementation just because the plan was approved in the same message; the hold from Step 4 lifts only when `execute-plan` is invoked or an equivalent explicit go-ahead is given.
