---
name: plan-first
description: Runs the developer's plan-before-code ritual — ask clarifying questions, write a plan, get explicit go-ahead before touching any code. Use whenever they ask to plan something out before building, say not to make changes until they approve, or ask for a plan they'll "ok when ready." Trigger on "/plan-first" as well as natural-language versions like "let's create a plan to implement this," "ask me clarifying questions first," "don't make any code changes until I give you explicit permission," or "I'll ok the plan when I'm ready." Do not use for the build step itself (new branch, incremental commits) or for ship/deploy requests — this skill stops at the approved plan and hands off from there.
---

# Plan-first gate

Two things are non-negotiable here:

1. **No code changes before an explicit go-ahead.** The gate holds for the whole task.
2. **The plan is narrowed through a discussion sized to the task.** You ask what you genuinely can't decide, the developer answers, you ask follow-ups only where an answer opened something new, and you draft once nothing in the plan would be a guess. A small, well-specified task may need one short round. A large, loosely specified one may need several.

The developer has seen both failure modes. First (2026-09-21): too few questions, typed as prose at the end of a message instead of actually asked. Second (2026-10-19): a fixed-feeling minimum of three rounds of four questions, regardless of how small or how fleshed out the task was. This skill exists to fix both. The target is the right number of questions for the task, every one of them asked through the tool.

## 1. Every question goes through the AskUserQuestion tool

The harness default is to reserve AskUserQuestion for when you're truly blocked. **This skill overrides that default: the bar for asking is "I would otherwise be deciding this on the developer's behalf," not "I am blocked."** Asking is part of the work, not a cost of it.

Rules:

- Every clarifying question is asked with the `AskUserQuestion` tool. Never write a question in plain text and end your turn on it. If a turn contains an open question, that turn ends on an AskUserQuestion call and nothing else follows it. (The one exception is the plan-approval prompt in Step 4, which is prose by design.)
- A call holds up to four questions, but only as many as you actually have. One real question is a valid call. Never pad a round to four.
- Give each question two to four concrete options drawn from what you actually found in the repo or conversation, with your recommended option first and marked "(Recommended)". The developer can always type their own answer via "Other," so options don't need to be exhaustive, but they must be specific. "Option A / Option B" is not an option; "frontmatter field `project:`" versus "nested tag `#project/<slug>`" is.
- Do the research before you ask. Read the relevant files, grep for existing conventions, check what the request would touch. A question that the repo already answers is a wasted question. A question whose options are grounded in real file names, real existing patterns, and real tradeoffs is a good one.
- While questions are open, output only the questions. Do not include a plan sketch, a "here's roughly what I'd do," or a list of assumptions alongside them. The plan comes after the discussion closes.

## 2. Size the discussion to the task

There is no minimum number of rounds and no minimum number of questions. Before asking anything, size the task on two axes:

- **How big is the change?** Files touched, new surface area, whether it crosses layers or systems.
- **How specified is the request?** Does it already name the files, the behavior, the edge cases, the naming? Or is it a one-line idea?

Use that sizing to set your expectation for how much asking is warranted:

| Task | Expected shape of the discussion |
|------|----------------------------------|
| Small change, well specified (a rename, a single-file fix, a change the developer already described in detail) | Zero to two questions, one round. Often the readiness test passes on the first pass and you go straight to the plan. |
| Medium change, mostly specified | A few questions, one round, with a second round only if an answer opens something. |
| Large change, or a loosely specified one (new feature, new system, architecture decision, "I want something like X") | As many rounds as it takes. Each round still only holds the questions that are genuinely open at that point. |

This is an expectation, not a quota in either direction. If a small task hides a real fork, ask about it. If a large task turns out to be fully pinned down by the repo and the conversation, don't manufacture questions to match its size.

**Follow-up rounds exist only for dependent questions.** Ask everything you can ask up front in the first round. A later round is justified only by questions that could not have been asked earlier because they depend on an answer you just got: choosing a storage mechanism raises "what about the existing data," choosing a scope raises "which of these edge cases are in." If an answer closes a branch, drop the questions that lived on it. A second round that could have been the first round is a sign you didn't do the research before asking.

**Readiness test, run before drafting:** list every decision the plan will make. For each one, ask: would I be choosing this on the developer's behalf? If yes for any of them, that decision is a question you haven't asked yet. Go back and ask it. Draft when the list of decisions-you'd-be-guessing is empty. This test is the stopping rule in both directions: it is what stops you asking when the task is pinned down, and what stops you drafting when it isn't.

Two things that are **not** guesses, and so don't need a question:

- A decision the repo's existing convention already makes (where files go, how things are named, which pattern a sibling feature used). Follow the convention and name it in the plan.
- A decision the request or the conversation already made, even implicitly.

What counts as askable is broad. All of these are fair questions when they are genuinely open:

- Scope edges: does this also cover X? Should Y be left alone?
- Naming, ordering, and structure the developer will have to live with.
- Behavior on edge cases and failure paths.
- Preferences and taste calls: how something should read, look, or be organized.
- Whether an adjacent problem you noticed should be folded in or explicitly left out.
- Anything you'd otherwise write into the plan as "assumption," "TBD," "to confirm," or "open question."

On a small, well-specified task most of these collapse, because the request or the repo already answers them. On a large or vague one, most of them are live. Let the sizing tell you which case you're in, and let the readiness test confirm it.

## 3. Draft the plan only after the discussion closes

Once the readiness test passes, write the plan. It must be free of unresolved items: no assumptions, no TBDs, no "to confirm with the developer," no "not blocking but worth noting." Every one of those is a question that should have been asked in Step 2. If one surfaces while drafting, stop drafting and ask it.

The plan should be concrete enough that `execute-plan` can build it without needing to come back for a decision: which files, what changes, in what order, and how you'll know it worked.

## 4. Close by asking for approval in prose, and then hold

After presenting the plan, end the message with a short plain-text question asking whether it's approved as-is, needs changes, or the developer has more context to add. **Do not use AskUserQuestion for this.** The developer wants to read the plan and reply freely, not pick from a menu. Keep it to one line at the very end of the message, after the plan.

Then hold: make no code, config, or other executable-file changes until the developer either invokes the `execute-plan` skill or gives an equivalent explicit go-ahead ("implement this plan," "build it now," "go ahead and build it"). The only exception is `.md` files, see below.

This hold is binding for the rest of the task, not just the next message. A vague "sounds good" or "yeah that works" reacting to the plan is not the same as one of those go-ahead forms. If it's unclear whether they're greenlighting the build or just reacting, ask in a plain-text line, don't start editing.

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
