# Shop Sales — Conversation Log

Rolling log of Claude sessions on the Shop Sales project. Newest entry at the top.

---

# PR #45 merged; save-conversation name collides with global skill
**Date:** 2026-08-03
**Project:** Shop Sales — tooling / documentation
**Mode:** Rolling Log + Git Push
**Status:** Complete

---

## Project Context
Continuation of the same session as the 2026-08-03 "Add save-conversation skill and
session log" entry below — that entry covers the file contents; this one covers what
happened after they were pushed.

## Session Goal
Get PR #45 merged, then have Saffron say "log session" to exercise the new workflow
end-to-end and confirm the repo-scoped `save-conversation` skill actually fires.

## What Was Done
PR #45 was subscribed to, had no required CI (the repo's only workflow is the GitHub
Pages deploy, which doesn't run on PR branches) and no review comments, was marked
ready for review, then merged into `main` (merge commit `f1f5177`) — all via webhook
events, no action needed. The 1-hour check-in trigger was cancelled once the merged
notice arrived.

Saffron then said "log session," invoking the `Skill` tool by the plain name
`save-conversation`. It resolved to the **global** skill at
`/root/.claude/skills/save-conversation` — Saffron's Notion/ONE-LDN-business-context
version that saves a generic `.md` to `Documents/Claude/conversations` — not the
repo-scoped one just merged at `.claude/skills/save-conversation/SKILL.md`. The
available-skills listing shown to this session never displayed a path-prefixed variant
(e.g. `shop-sales:save-conversation`) even after the repo-scoped file existed and was
committed earlier in this same session, so the two never appeared as a disambiguated
pair. Rather than follow the global skill's output (wrong location, wrong template for
a git repo), this entry was written by hand following the repo-scoped skill's actual
process, then reset the working branch onto latest `main` (it had already been merged)
before appending it — `git fetch origin main && git checkout -B
claude/markdown-files-claude-skills-9ikmuu origin/main`.

## Artifacts Produced / Modified
| File | What it is | Status | Location |
|------|------------|--------|----------|
| logs/shop-sales_log.md | This entry, prepended | Modified | /logs/shop-sales_log.md |

## Decisions & Reasoning
- **Did not follow the launched global skill's instructions**: its save location
  (`Documents/Claude/conversations`) and template belong to Saffron's separate
  business-analysis workflow, not this git repo — following it here would have silently
  bypassed the git-native log this session had just built.
- **Restarted the branch from `origin/main` instead of reusing the merged branch**: PR
  #45 had already merged, so per the session's branch-handling convention, follow-up
  work restarts the same branch name from the latest default branch rather than
  stacking on already-merged history.

## Current State (end of session)
PR #45 merged. The repo-scoped `save-conversation` SKILL.md is correct and live on
`main`, but is **not reliably invoked by plain name** while a session is already
running — untested whether a *fresh* session on this repo resolves it correctly instead.

## Next Steps
1. In a **new** session on this repo (not a continuation of this one), invoke
   `save-conversation` and check whether the available-skills listing shows a
   path-prefixed variant (e.g. `shop-sales:save-conversation`) and whether the repo
   version fires — that tells us if it's just a mid-session discovery lag or a real
   shadowing problem.
2. If the global skill still wins in a fresh session, rename the repo-scoped one (e.g.
   `save-conversation-shop-sales`) and update `CLAUDE.md`'s reference to match, since
   same-name "most specific wins" isn't holding up in practice.
3. Once confirmed working, this manual-entry workaround is no longer needed.

## Open Questions / Blockers
Whether repo-scoped vs. global `.claude/skills/` name collisions resolve correctly in a
fresh session is unverified — see Next Steps #1.

## Environment & Config Notes
PR #45 (`ONE-LDN/shop-sales`) merged into `main` as `f1f5177`. Working branch
`claude/markdown-files-claude-skills-9ikmuu` reset onto `origin/main` for this follow-up
commit.

## Notes & Gotchas
If Saffron has this same skill name (`save-conversation`) both globally and in other
repos, the same shadowing question applies there too — worth a quick check next time
one of those repos is touched, not just this one.

---

# Add save-conversation skill and session log
**Date:** 2026-08-03
**Project:** Shop Sales — tooling / documentation
**Mode:** Rolling Log + Git Push
**Status:** Complete

---

## Project Context
New repo-level session-logging system, ported from generic `CLAUDE.md`/skill templates
Saffron extracted from another project. Distinct from Saffron's global, Notion-based
`save-conversation`/`docs-manager` workflow used for the wider ONE LDN business/data
work — this repo gets its own git-native log instead.

## Session Goal
Add a repo-scoped `save-conversation` skill and a `CLAUDE.md` "Session Logging" section
to this repo, adapting the two generic markdown files Saffron supplied, and seed
`logs/shop-sales_log.md`.

## State Before This Session
No `CLAUDE.md`, no `.claude/skills/save-conversation`, and no `logs/` directory existed
in this repo. The `.claude/skills/` directory already held several ONE LDN
Notion-oriented skills (`docs-manager`, `progress-update-writer`,
`stakeholder-update-writer`, `dashboard-doc-pair-updater`), none of which write to a
git-tracked log file.

## What Was Done
Read the two uploaded generic files: a `CLAUDE.md` snippet for "Session Logging" and a
generic `save-conversation` SKILL.md template. Adapted both for this repo: replaced
every `<project>` / `<Project Name>` placeholder with `shop-sales` / `Shop Sales`, and
rewrote the SKILL.md frontmatter — the uploaded version had malformed YAML (`---name:
save-conversationdescription: |` with no line breaks). Created `CLAUDE.md` at the repo
root with a "Files" section and the "Session Logging" section. Created
`.claude/skills/save-conversation/SKILL.md` as a repo-scoped skill. Seeded this log file
with its standard header plus this entry.

## Artifacts Produced / Modified

| File | What it is | Status | Location |
|------|------------|--------|----------|
| CLAUDE.md | Repo instructions incl. Session Logging section | Created | /CLAUDE.md |
| .claude/skills/save-conversation/SKILL.md | Repo-scoped save-conversation skill | Created | /.claude/skills/save-conversation/SKILL.md |
| logs/shop-sales_log.md | Rolling session log (this file) | Created | /logs/shop-sales_log.md |

## Decisions & Reasoning
- **Kept the skill name `save-conversation` rather than renaming it**: Saffron already
  has a global skill of the same name (Notion-based, for the ONE LDN docs-manager
  workflow). Directory-scoped skills take precedence over global ones inside a given
  repo, so keeping the name identical means this repo automatically gets the git-native
  version with no trigger-phrase changes and no risk of both firing at once.
- **Used the GitHub repo name (`shop-sales`) for the log filename, not `cafe-sales`**:
  root `README.md` says "cafe-sales", but the actual git remote, branch names, and PR
  conventions all use `shop-sales` — matched the convention that's actually
  load-bearing.
- **Seeded the log with this session as its first real entry** instead of leaving it
  empty: gives the skill's "Example of a great entry" pointer something concrete
  instead of a placeholder.

## Current State (end of session)
All three files created. This is the only entry in the log so far.

## Next Steps
1. Next session: read the top of `logs/shop-sales_log.md` at start (per the new
   `CLAUDE.md` instruction) to confirm the habit works end-to-end.
2. Once a few more real entries exist, update the "Example of a great entry" pointer in
   `.claude/skills/save-conversation/SKILL.md` to reference whichever entry is the
   strongest example, rather than this bootstrap one.
3. If Saffron wants this same pattern in other git repos she works in directly, copy the
   three-file pattern (`CLAUDE.md` section + `.claude/skills/save-conversation/SKILL.md`
   + seeded `logs/<project>_log.md`) into those repos, substituting the project name.

## Open Questions / Blockers
N/A

## Environment & Config Notes
Repo `ONE-LDN/shop-sales`, branch `claude/markdown-files-claude-skills-9ikmuu`. No open
PR existed for this branch before this session.

## Notes & Gotchas
The uploaded `save-conversation` SKILL.md template had broken YAML frontmatter (no
newlines between `name:`, `description:`, and the closing `---`) — rewritten correctly
in this repo's copy. If Saffron pastes the same source template into another repo,
check for the same formatting issue before using it as-is.

---
