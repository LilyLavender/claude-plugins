# claude-plugins

A Claude Code plugin marketplace holding a few workflow skills.

## What's in it

| Plugin | Skills | What it does |
|--------|--------|--------------|
| `dev-workflow` | `plan-first`, `execute-plan`, `ship` | The plan -> build -> ship chain. `plan-first` asks clarifying questions and holds off on code changes until the user explicitly approves the plan. `execute-plan` builds the approved plan with some questions beforehand. `ship` commits what's left, pushes, merges to main, then deploys if applicable. |
| `open-items` | `open-items` | Lists what is still outstanding in the current project: unfinished plan items, uncommitted or unmerged git state, loose threads from the conversation. |
| `testing-checklist` | `testing-checklist` | After a change, produces a short checklist of only the things that change could plausibly break. Skips anything already verified in the session. |

## Install

Inside a Claude Code session:

```
/plugin marketplace add LilyLavender/claude-plugins
/plugin install dev-workflow@lilylavender
/plugin install open-items@lilylavender
/plugin install testing-checklist@lilylavender
```
