---
name: testing-checklist
description: Produces a short, scoped checklist of what the developer should manually test after a non-trivial change — restructure, refactor, or anything that touches shared code — by reasoning about what this specific change could plausibly break, not a generic pre-ship list. Explicitly excludes anything already verified this session (tests run, typecheck passed, browser-checked) — the whole point is surfacing what still needs a human's eyes. Use whenever the developer asks for a testing checklist, what to test, what to check before shipping, or asks you to sanity-check a change is safe. Trigger on "/testing-checklist" as well as natural-language versions like "give me a testing checklist for this," "what should I test before I ship this," "what could this break," or "give me a checklist of what to check." Don't use this for every small change — it's meant for non-trivial changes (restructures, refactors, anything touching code other features depend on), not a boilerplate gate on every commit.
---

# Testing checklist

The entire value of this skill is the filtering. A generic "test the app" checklist (does it build, does lint pass, does it load) is worthless — the developer already knows to do those things, and a long list of trivially-true items just makes them scan past the one item that actually mattered. This skill exists because they were hand-crafting the prompt every time to cut a generic list down to just what's actually at risk from *this* change. Do that scoping automatically.

## 1. Figure out what actually changed

Look at the diff (uncommitted changes, or the most recent commit(s) if asked about something already committed) and the plan doc if one exists for this change. You need to understand not just *what* changed but *what depends on it* — a function that's called from five places is riskier to touch than one called from one place.

```
git diff
git diff --stat HEAD~1
```

Trace the blast radius: what else reads this state, calls this function, renders this component, hits this endpoint, depends on this schema/type. Grep for callers/importers of anything touched rather than guessing from the diff alone — the diff shows you what changed, not who's affected by the change.

## 2. Exclude what's already covered

This is the step that keeps the list short, and it's the reason this skill exists rather than a plain "list test ideas" prompt: **don't include anything you (Claude) already verified in this session.** If you ran the test suite and it passed, that's covered — don't ask for a re-check by hand. If you ran `tsc --noEmit` and it's clean, don't put "does it typecheck" on the list. If you already opened the app in a browser and clicked through a flow, don't ask for that flow to be clicked through again.

The checklist is specifically for the gap: things that could break but that neither automated checks nor anything you did yourself this session actually exercised. If you haven't run the test suite or typechecked yet and easily could, do that first — it shrinks the list and it's faster than asking someone to verify something a command could verify in seconds.

## 3. Write only the at-risk items

For each thing still uncovered, ask "could this specific change plausibly break this" — not "is this a good practice to check." If the answer is a stretch, leave it off. Concrete failure scenarios beat generic categories: not "check the API" but "check that the leads list still loads when a lead has no assigned rep, since the join changed."

Prioritize:
- Callers/consumers of anything renamed, resig'd, or restructured
- Edge cases the change plausibly touches (empty states, nulls, boundary values) if the change altered how those are handled
- Anything sharing state, a schema, or a type with the changed code
- Integration points (API contracts, external services, other features) that assume the old shape

Skip:
- Anything unrelated to the change, even if it's generally worth testing
- Anything already verified this session (see step 2)
- Vague catch-alls like "test thoroughly" or "check for regressions" — if you can't name the specific scenario, it doesn't belong on the list

## 4. Output

Flat bullet list, phrased as concrete manual checks that can be acted on directly — "does X still work when Y" — same terse, specific voice as the developer's other reports. No headers, no severity labels, no source citations. Print directly in the conversation; don't write it to a file unless asked.

If everything at-risk was already covered by tests/typecheck/your own verification this session, say that plainly instead of padding the list — "nothing left to manually check, the test suite and a browser pass covered the risk here" is a complete and useful answer.

**Line phrasing patterns:**
- "Check that X still works when Y"
- "Verify Z now that the [function/schema/component] changed"
- "Confirm [edge case] is handled the same way"
