---
name: open-items
description: Scans the current project for everything that's still outstanding — unfinished plan items, uncommitted or unmerged git state, and loose threads from the current conversation — and reports them as a flat list the developer can act on. Distinct from a recap of what's done. Use whenever the developer asks for open items, what's left, what hasn't been finished, what's still outstanding, or asks to review the current state of the repo/plan and wants to know what's not done yet. Trigger on "/open-items" as well as natural-language versions like "give me a list of open items," "what's left in this repo," "what hasn't been actually finished," "are there any open items," or "what all do the current changes do, and is anything still open."
---

# Open items list

This is for finding out what's not done, not for reviewing what is. Don't reuse this for an end-of-day recap (that's a separate skill, framed the opposite way: what shipped, not what's left). If both are being asked for, run both and keep them as two separate lists — don't blend them into one.

An "open item" is anything real work is still needed on: a plan step not yet built, an unresolved bug, a change sitting uncommitted, a question raised earlier that never got answered. It is not a status report and not a changelog — skip anything already done, even if it was done recently.

## 1. Confirm scope

Run in the current project directory:

```
git rev-parse --show-toplevel
```

If this fails, there's no git repo here — say so and ask which project is meant rather than guessing.

## 2. Gather from all three sources

Pull from all three every time unless the request scoped it explicitly (e.g. "just the plan doc" or "just what's uncommitted"). An item can legitimately show up from more than one source (a plan step that's also an unresolved conversation thread) — when that happens, list it once, not twice.

### A. Plan / doc-derived items

Don't rely on a fixed set of filename keywords to find candidate docs — the developer's projects keep working documents under folders like `docs/plans/`, `docs/planning/`, `docs/audits/`, `docs/research/`, and the naming isn't consistent enough to guess in advance (a keyword list that includes "plan" but not "audit" will silently skip a whole audit doc).

**Don't use `git ls-files` for this.** These working docs are often gitignored on purpose (scratch planning material not meant to ship) — a git-tracked-only search will silently come back empty in exactly the projects that have the most open items to report. Search the actual filesystem instead:

```
find . -iname '*.md' -not -path '*/node_modules/*' -not -path '*/.git/*'
```

Use whatever the platform's real equivalent of `find` is (PowerShell's `Get-ChildItem -Recurse -Filter *.md` if `find` isn't available) — the point is walking the working tree directly, not asking git what it's tracking.

**Triage before reading anything in full — this is the step that keeps the skill fast on a project with months of daily planning docs.** Reading every doc cover to cover and verifying every item against the code is fine for a handful of files; it doesn't scale to a project where docs get written multiple times a day for months. So:

1. Count the candidate `.md` files first. If it's a small set (roughly 15 or fewer), just read them all in full as described below — the triage below is overhead you don't need.
2. If it's a larger set, narrow before reading:
   - Sort by modified time and treat anything old relative to the project's pace as low-priority — a plan doc from four months ago in a project that's shipped dozens of commits since is much more likely to be fully resolved or superseded than something from this week. Use judgment on the cutoff based on the project's actual commit cadence rather than a fixed day count.
   - Within that recent set, grep for the same open-item markers (`- [ ]`, "Next steps," "Open questions," "TODO," "Status: Not yet started," findings-list language) across all candidates at once before opening any of them individually — skip straight past docs with zero hits, since a doc with no open-item markers at all is very unlikely to be worth a full read.
   - Only fully read and verify the docs that both are recent-ish *and* grep-matched.
3. Whatever you skipped, say so plainly in the output rather than silently pretending you covered everything: name (or count) the older/no-hit docs you didn't open, and invite a follow-up about a specific one by name if it needs checking. A short, honest "I only checked recent docs, didn't dig into the 40 from before June" is worth far more than quietly producing an incomplete list that looks exhaustive.

For each doc you do read in full, skim it enough to tell whether it's a plan, audit, research note, or anything else tracking work, then pull out:
- Unchecked `- [ ]` items
- Items under headers like "Next steps," "Open questions," "TODO," "Not yet implemented," or an audit-style findings list (severity-ranked issues are open items even without checkbox syntax)
- Anything phrased as deferred or explicitly punted ("we decided to hold off on X," "revisit after Y")

**Then verify against the actual codebase before trusting any of it.** A doc's own status line or checkbox state is a claim, not ground truth — plans and audits go stale the moment work happens without the doc being updated. For each item that looks open, spend a moment checking whether it's actually still open: grep for the function/class/flag the item names, check whether the described bug pattern still exists in the current code, or cross-reference against CLAUDE.md or another doc that reflects current reality.

When you find one that's marked open (unchecked box, "Status: Not yet started," listed as a finding) but is actually already done in the code, don't just quietly drop it from the list — call it out separately as a stale-doc item, e.g. "Marked open in the migration plan but already done: X." This is worth as much as the genuinely-open items, since it flags that the doc itself needs updating. Keep these visually distinct from the real open items (a short separate group at the end works fine) so it's easy to tell apart "still need to build this" from "just need to check this off."

If a doc has a long list (an audit with dozens of findings, say), spot-check a representative handful rather than verifying every single one — but say what you did: how many total, which ones you actually confirmed one way or the other, and that the rest should be treated as still open until checked. Don't silently under-report a long list just because full verification would take a while. This budget is per-run, not per-doc: if triage above already surfaced several docs each with their own long list, spread a similar handful of spot-checks across all of them rather than exhaustively verifying the first doc and rushing or skipping the rest.

Don't flag something as open just because it's phrased tentatively in a doc that's otherwise a finished spec — read enough surrounding context to tell a genuinely open item from a rhetorical question or a resolved discussion that was left in the doc as history.

### B. Git-state-derived items

```
git status --short
git branch --no-merged main 2>/dev/null || git branch --no-merged master
git stash list
```

- Uncommitted or untracked changes: list what they touch, not just "there are uncommitted changes" — say what file/area, the way the developer would describe it themselves.
- Branches with commits not yet merged to main/master: name the branch and, if the branch name or its last commit message doesn't make it obvious, glance at `git log main..<branch> --oneline` to say what it's for.
- Stashes: only worth surfacing if there's no more obvious explanation lying around (a stash from months ago tied to an already-shipped feature isn't an open item).

Also check for inline markers in tracked source, but keep this scoped — don't dump every `TODO` in a large codebase indiscriminately. If the count is large (a few dozen+), say how many there are and show a handful of examples rather than listing all of them, and note that the rest exist so the exact term can be grepped for directly if the full list is wanted:

```
git grep -niE 'TODO|FIXME|XXX' -- ':!*.md' ':!node_modules'
```

### C. Conversation-derived items

Scan the current conversation (not other sessions — you don't have those) for things that were raised but never resolved: a question that got deferred, a "let's come back to this," a bug mentioned in passing that was never fixed, a follow-up either side flagged and then moved past without closing the loop. Don't include things that were just discussed and then genuinely finished later in the same conversation.

If nothing in the conversation qualifies, that's fine — don't force an item onto the list just to have something from this source.

## 3. Write the list

Flat bullet list, one line per item, same terse and specific voice as the developer's other reports — state what's open, not why it's open or how you found it. No headers or source labels needed; this is for their own use, not a formal report, so don't over-format it. If grouping by source would make a long list easier to scan, that's fine, but don't add it as ceremony when a flat list reads fine on its own.

**Don't cite which file or doc an item came from.** Describe the item itself (the feature, the bug, the decision) the way the developer would say it out loud — not "docs/audits/2026-09-06-codebase-audit.md — X." They know their own repo; the file path is noise, not signal.

Skip an item type entirely if that source had nothing (e.g. don't write "Conversation: none" — just omit it).

If genuinely nothing is open across all three sources, say so plainly rather than padding the list.

**Line phrasing patterns:**
- "X still needs Y" / "X isn't built yet"
- "Uncommitted: X" (for working-tree changes)
- "Branch `X` has Y not yet merged to main"
- "Never resolved: X" (for a conversation loose end)
- "Open question: X" (for something explicitly unanswered in a plan doc)

## 4. Output

Print the list directly in the conversation. Don't write it to a file unless asked.
