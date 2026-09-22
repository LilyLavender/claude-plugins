---
name: ship
description: Runs the developer's end-of-build "ship" chain — commit whatever's left, push the branch, merge it into main, confirm remote main is actually up to date, then deploy if the project has a deploy mechanism. This is the step that comes after execute-plan finishes building, once the change is ready to actually go live. Use whenever the developer says to push/merge/deploy, "ship this," "merge branch into main and deploy to prod," "commit this then merge to main, then make sure remote main is up-to-date and then deploy," or gives any variant of the full sequence in one breath. Trigger on "/ship" as well as natural-language versions like "let's ship it," "merge and deploy," or "push, merge, deploy prod." Do not use this for the build step itself (that's execute-plan) or for a deploy alone with no merge involved (that's the project's own deploy skill/command) — this skill is specifically the git choreography that runs a branch into main and out to prod.
---

# Ship

This is the last mile: a finished, approved change on a branch needs to land on main and go live. The git choreography here is mechanical, but merging to main and deploying to prod are exactly the hard-to-reverse, shared-state actions worth slowing down for — so the point of this skill isn't to move fast, it's to run the same sequence correctly every time and pause at the two steps that actually matter.

## 1. Check where things stand before doing anything

```
git status
git branch --show-current
git log main..HEAD --oneline
```

- If `git status` shows uncommitted or untracked changes, that's step 3's job — don't skip past it assuming everything's already committed.
- If there's nothing to merge (`git log main..HEAD` is empty and the branch *is* main), say so and ask what's actually meant to be shipped — there's no chain to run.
- If already on main with nothing to merge from a branch, this skill has nothing to do beyond a possible deploy; confirm that's really the ask before jumping to step 7.

## 2. Figure out if a deploy applies here, and ask up front

Don't wait until step 7 to raise deploying — decide now whether it's even on the table, and if so, ask about it immediately rather than at the end of the chain.

Check what this project actually uses to deploy, in this order:

1. `vercel.json` or a `.vercel/` directory → this project deploys via Vercel; the deploy step will use the `vercel:deploy` skill.
2. A `"deploy"` script in `package.json` → the deploy step will run that script.
3. Other recognizable platform config (`netlify.toml`, `fly.toml`, `render.yaml`, `wrangler.toml`, a `Procfile`, or a GitHub Actions workflow with "deploy" in its name) → a deploy mechanism exists but isn't one to guess a command for; the deploy step will name what was found and ask how to run it.
4. None of the above → there's no deploy mechanism in this project. Treat deploy as not applicable, don't ask about it, and the chain simply ends after step 5.

If a mechanism was found (cases 1-3), ask now via `AskUserQuestion` whether this run should deploy once the merge lands — unless the original request already settled it (e.g. "push, merge, deploy prod" already covers it; don't ask again for something already stated). Getting the answer now means there's no need to interrupt again right before step 7.

## 3. Commit whatever's left

Stage deliberately — `git status` first, then add only what belongs to this change. Never `git add -A` or `git add .` (the developer's standing global instruction). If there's untracked work sitting in the tree that doesn't look like part of what's being shipped, flag it rather than sweeping it in or discarding it.

This is normally the final commit on the branch, so unlike execute-plan's incremental commits (which skip it), include the attribution footer here per the developer's global git instructions.

If everything's already committed, skip this step silently — don't manufacture an empty commit.

## 4. Push the branch

```
git push -u origin <branch>
```

If the branch has no upstream yet, `-u` sets it. If it does, a plain `git push` is fine — just don't force-push. If the push is rejected because remote has diverged, stop and report that rather than guessing whether to rebase or force.

## 5. Merge into main — confirm first

Check how this repo actually merges before picking a mechanism:

```
gh pr list --head <branch> 2>/dev/null
git log --merges -5 --oneline
```

- If there's an open PR for this branch, or the repo's recent history is full of PR merge commits, merge via `gh pr merge <branch> --merge` (or `--squash`/`--rebase` if that's what recent history shows the repo actually uses — check a few `git log --merges` messages for the pattern rather than defaulting to one).
- If the repo just merges locally (no PR habit visible), do it directly: check out main, `git pull`, `git merge <branch>`, push.

**Confirm before actually merging**, unless it's already pre-authorized in this exact request (e.g. "push, merge, deploy prod" up front already covers the merge, so don't re-ask; but a bare "ship this" with no detail on how far to go needs a check). This is the harness's standing rule for hard-to-reverse shared-state actions, and merging to main is exactly that.

## 6. Confirm remote main is actually up to date

Don't just trust that the merge succeeded — verify it:

```
git fetch origin main
git log origin/main -1 --oneline
```

Compare against what you just merged in. If origin/main doesn't reflect the merge (push failed silently, PR didn't actually merge, etc.), stop here and report it — don't proceed to deploy against a stale main.

## 7. Deploy — only if applicable, and only what step 2 already settled

If step 2 found no deploy mechanism, this is where the chain ends — nothing to do here.

If step 2 found one and it was confirmed upfront, run it now without asking again:
- **Vercel** (`vercel.json` / `.vercel/`): invoke the `vercel:deploy` skill with `prod` (or `production`) as the argument — don't reimplement the deploy step yourself.
- **`package.json` deploy script**: run it directly (`npm run deploy` or the project's actual package manager equivalent).
- **Other platform config**: use whatever step 2 said would be asked about, now that remote main is confirmed current.

Deploying to prod is exactly the kind of hard-to-reverse, shared-state action worth confirming before — that confirmation already happened in step 2, so don't ask a second time here. The only reason to stop and ask again at this point is if step 2's answer was ambiguous or deferred.

## Scope

This skill starts once there's a built, ready change to ship — it doesn't build anything itself (that's execute-plan) and it doesn't replace a standalone deploy with no merge involved (use the project's own deploy mechanism directly for that). If something fails partway (push rejected, merge conflict, deploy error), stop and report exactly where it broke rather than trying to route around it — these are the steps where guessing costs the most.
