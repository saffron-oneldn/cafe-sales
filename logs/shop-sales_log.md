# Shop Sales — Conversation Log

Rolling log of Claude sessions on the Shop Sales project. Newest entry at the top.

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
