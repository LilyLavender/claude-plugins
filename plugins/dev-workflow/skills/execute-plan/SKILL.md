---
name: execute-plan
description: Kicks off the build phase after the developer has an approved plan — asks upfront whether to branch or build on the current branch, and whether to make incremental commits as it goes or leave everything uncommitted for review, then implements it. Use whenever they say to start a new branch and build, to implement a plan "with incremental commits," or otherwise give the explicit go-ahead to start building after a plan (theirs or plan-first's) has been approved. Trigger on "/execute-plan" as well as natural-language versions like "put us on a new branch and implement this with incremental commits," "build it out with incremental commits," "implement it on a new branch," or "let's start building this now." Running this skill IS the go-ahead — don't wait for a separate signal once it's been invoked. Do not use this for the planning step itself (that's plan-first) or for pushing/merging/deploying (that's the ship step) — this skill starts the build and stops once it's implemented.
---

# Execute plan

The developer treats invoking this skill as the go-ahead itself — they've already decided to build, so don't re-ask "should I start?" The two things worth getting right are: don't lose or scramble any existing working-tree state while switching branches, and match how they actually want this project's history to look, which differs by context.

## 1. Ask how they want commits handled, and whether to branch — every time, before touching git

The developer works two ways depending on the project, and it's not something to guess from vibes:

- **Job/team projects:** incremental commits as you go, Conventional Commits style (`feat(scope): ...`, `fix(scope): ...`), no attribution footer — that only gets added on the final commit later, per their global git instructions.
- **Personal projects:** they like to verify changes themselves before anything lands in history, so implement the plan but leave the working tree uncommitted.

Ask which mode applies with `AskUserQuestion` right at the start (frame the two options above). Don't infer this from the repo name or "it feels like a personal project" — ask directly, since it depends on mood as much as the project.

In the same `AskUserQuestion` call, also ask whether to create a new branch or build directly on the current branch (frame it as: new branch vs. stay on current branch). Default assumption going in is a new branch — that's this skill's usual mode — but confirm rather than assume, since sometimes the developer is already on a throwaway/feature branch and building directly there is correct. If they choose to stay on the current branch, skip step 3 entirely.

## 2. Check for uncommitted work before branching

Run `git status` first. If there's existing uncommitted or untracked work that isn't part of what you're about to build, don't sweep it in and don't discard it — stash it (`git stash -u`) or ask what it is before proceeding. This matters more here than usual when branching: creating a branch from a dirty tree carries whatever's sitting there onto the new branch, silently mixing the developer's in-progress work with yours. (Still worth checking even when staying on the current branch, so you don't scoop up unrelated stray files into your commits.)

## 3. Name the branch by matching what the repo already does

Skip this step if the choice was to stay on the current branch. Don't invent a naming scheme. Check recent history first:

```
git branch -a --sort=-committerdate
git log --all --oneline --decorate -20
```

- **If a pattern is evident** (a consistent prefix like `feature/`, `fix/`, a ticket-number scheme, a username prefix, etc.), follow it.
- **If nothing is consistent** — mixed styles, or too few branches to tell — just use a short kebab-case slug describing the task (`add-csv-export`, `fix-login-redirect`). Don't force a scheme that isn't actually established.

Branch off whatever the developer is currently on (usually the base the plan was written against) unless they've said otherwise.

## 4. Implement the plan

Work through the approved plan (from plan-first's `ExitPlanMode`, a plan doc, or whatever was just described) the way you normally would — read existing patterns before writing new code, follow project conventions, keep changes scoped to what the plan actually calls for.

### If incremental commits were chosen

After each logical unit of work, decide whether it's commit-worthy with this heuristic: **can you write a commit message describing a complete, valuable change?** If yes, commit. If the honest message would be "WIP" or "partial X," keep going instead.

When you do commit:
- Run `git status` first and stage only the files that belong to this logical unit — never `git add -A` or `git add .` (this is also the developer's standing global instruction, not unique to this skill). If unrelated untracked files are sitting in the tree, leave them alone; flag them rather than guessing whether they're yours to include.
- Commit message: `type(scope): description` (Conventional Commits), no attribution footer.
- If a unit turns out bigger than expected, split it into more than one commit rather than force it into one; if two units are small and closely related, it's fine for them to land together.

### If review-before-committing was chosen

Implement the full plan (or the portion asked for) without committing anything. Don't create a "just in case" WIP commit at the end either — leave the tree exactly as the changes leave it. When you're done, say plainly what changed and where, so the developer can review before anything gets committed. If they then ask you to commit, stage deliberately (same rule as above — no blanket `git add`) rather than committing everything in one sweep.

## Scope

This skill ends when the plan is implemented (and, in incremental mode, committed). Pushing the branch, merging to main, and deploying are the ship step — a separate ask, not something to roll into automatically just because the build finished.
