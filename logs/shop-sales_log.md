# Shop Sales — Conversation Log

Rolling log of Claude sessions on the Shop Sales project. Newest entry at the top.

---

# Fix CSV upload errors: missing anon UPDATE policy + duplicate conflict targets
**Date:** 2026-08-14
**Project:** Shop Sales — Supabase RLS / CSV upload pipeline
**Mode:** Rolling Log + Git Push
**Status:** Complete (two sequential bugs found and fixed in one session — see below)

---

## Project Context
User reported a browser console screenshot showing errors while uploading a WodBoard
Shop Sales CSV: a `401` resource-load failure and a `Supabase upload error: Error: DB
upsert failed (shop_sales): {"code":"42501", ... "new row violates row-level security
policy (USING expression) for table \"shop_sales\""}`, thrown from `sbUpsert`
(index.html:1261) called by `handleFiles` (index.html:1911). This traces back to commit
`0eca66f` ("Make CSV re-upload correct existing rows instead of skipping them"), which
changed the upsert resolution to `merge-duplicates` so re-uploads correct existing rows
instead of being skipped.

## Session Goal
Diagnose and fix the RLS error blocking CSV sales uploads.

## State Before This Session
Repo clean on `claude/sales-csv-upload-error-y12v26` at commit `5dc6a71`. No pending
code changes. The bug was purely on the Supabase (database) side, not in `index.html`.

## What Was Done
Connected to the Supabase MCP server and identified the project backing this app:
`ljjwssicvvyyueyznmou` ("PT Dashboard", matches `SUPABASE_URL` in index.html:1219).
Queried `pg_policies` for `shop_sales`, `shop_invoice_aliases`, and `shop_product_lookup`.

Found `shop_sales` had only three RLS policies for role `anon`:
- `anon_insert_shop_sales` (INSERT, with_check true)
- `anon_select_shop_sales` (SELECT, qual true)
- `auth_all_shop_sales` (ALL, but only for role `authenticated`, which this
  client-side-only anon-key app never authenticates as)

**No `anon` UPDATE policy existed.** `sbUpsert` posts to
`/rest/v1/shop_sales?on_conflict=order_no,order_line` with
`Prefer: resolution=merge-duplicates`, which PostgREST executes as
`INSERT ... ON CONFLICT (order_no, order_line) DO UPDATE`. The DO UPDATE branch is
gated by the table's UPDATE RLS policy, not the INSERT policy. With no UPDATE policy
for `anon`, any upload whose rows collided with an existing `(order_no, order_line)`
pair (i.e. any CSV re-upload, or any file overlapping a previous upload's date range)
hit the RLS default-deny and threw the exact 42501 error in the screenshot. First-time
uploads of entirely new order lines worked fine (INSERT-only path), which is why this
wasn't caught immediately after `0eca66f` shipped.

Confirmed this was an oversight specific to `shop_sales` by comparing sibling tables:
`shop_deliveries` has `anon_update_shop_deliveries` (migration `20260521121301`) and
`shop_product_lookup` has `anon_update_shop_product_lookup` (migration
`20260517094108`) — both added when those tables needed anon-writable UPDATE paths.
`shop_sales` never got the equivalent when `0eca66f` introduced its own UPDATE
requirement via `merge-duplicates`.

Applied a migration directly to the Supabase project via `mcp__Supabase__apply_migration`
(name: `anon_update_shop_sales`):
```sql
CREATE POLICY anon_update_shop_sales ON public.shop_sales
  FOR UPDATE TO anon
  USING (true)
  WITH CHECK (true);
```
Verified via `pg_policies` that `shop_sales` now lists INSERT/SELECT/UPDATE for `anon`
plus the existing `authenticated` ALL policy.

The `401` line in the screenshot is almost certainly the browser network panel's status
label for the same failing POST request (Chrome renders failed-resource lines as
`URL:1` regardless of resource type) — not a separate auth/token problem. No JWT or
`SUPABASE_ANON` key issue was found; the anon key in index.html:1220 is valid (role
`anon`, far-future `exp`).

That first fix alone did **not** fully resolve the user's problem — see the second bug
below, found when the user retried the upload after PR #49 was opened.

### Bug 2 (found on retry): duplicate conflict targets within one upload batch
After the RLS fix, the user retried the upload and got a *different* error:
Postgres `21000` — `"ON CONFLICT DO UPDATE command cannot affect row a second time"`,
with hint `"Ensure that no rows proposed for insertion within the same command have
duplicate constrained values."` — again from `sbUpsert` (index.html:1261) via
`handleFiles` (index.html:1911).

This is a Postgres limitation, not an RLS/permissions issue: a single
`INSERT ... ON CONFLICT (order_no, order_line) DO UPDATE` statement cannot target the
same `(order_no, order_line)` row twice — if two rows in the *same* upsert batch share
a key, Postgres errors instead of silently overwriting. Reading `handleFiles` /
`processRows` (index.html:1839, 1697): `order_line` is assigned as `idx + 1` per item
within each CSV row's `Items` field, scoped to that one row. If the same `order_no`
appears more than once across the file(s) selected in one upload — e.g. two CSV
exports with overlapping date ranges uploaded together, or a duplicate row within a
single export — `processRows` (called once per file, concatenated into `allDbRows`)
produces two or more rows with an identical `(order_no, order_line)` pair, and whichever
chunk of 500 they land in throws this error.

**Fix:** in `handleFiles`, after all files are parsed and concatenated into
`allDbRows` but before chunking/upserting, dedupe by `` `${order_no}::${order_line}` ``
using a `Map` (last occurrence wins — consistent with "re-upload corrects existing
data"). This guarantees no chunk sent to Postgres can ever contain a duplicate
conflict target, regardless of how many files or how much date-range overlap the user
selects in one go.

## Artifacts Produced / Modified

| File | What it is | Status | Location |
|------|------------|--------|----------|
| (Supabase migration `anon_update_shop_sales`) | RLS policy adding anon UPDATE on `shop_sales` | Created (DB-side, not in repo) | Supabase project `ljjwssicvvyyueyznmou` |
| index.html | Dedupe `allDbRows` by `(order_no, order_line)` before upserting, in `handleFiles` | Modified | /index.html (`handleFiles`, ~line 1894) |
| logs/shop-sales_log.md | Rolling session log | Modified (this entry prepended, then amended in-session as bug 2 was found) | /logs/shop-sales_log.md |

Migrations for this repo's Supabase project aren't tracked as files in the repo (none
exist under version control; they're managed directly against the project via
MCP/dashboard) — only the RLS policy addition above is DB-side; the dedupe fix is a
normal code change committed to this branch.

## Decisions & Reasoning
- **Fixed via a direct Supabase RLS policy addition rather than an app-code workaround**:
  the app's upsert logic (merge-duplicates on conflict) is correct and intentional per
  `0eca66f`; the database was simply missing the RLS grant that logic depends on.
  Changing the app to fall back to insert-only would silently reintroduce the "re-upload
  skips existing rows" bug that `0eca66f` was written to fix.
- **Modeled the new policy on `anon_update_shop_deliveries` / `anon_update_shop_product_lookup`**:
  both are existing, working precedents for anon-writable UPDATE on this schema's
  tables, so matching their `USING (true) WITH CHECK (true)` shape keeps RLS policy
  style consistent across the shop schema rather than inventing new semantics.
- **Deduped client-side (in `handleFiles`) rather than changing the upsert to
  per-row requests or a different conflict strategy**: batching stays intact (still one
  request per 500-row chunk), the fix is a single `Map` pass with no behavior change
  for the common case (no duplicate keys), and "last occurrence wins" matches the
  existing "re-upload corrects data" intent rather than introducing new semantics
  (e.g. silently dropping the duplicate instead would be surprising/inconsistent with
  that intent).

## Current State (end of session)
Both fixes are live: the RLS policy is applied directly to the Supabase project
(commit `0142317`, PR #49), and the dedupe fix is committed and pushed to
`claude/sales-csv-upload-error-y12v26` (commit `a114d87`, added to the same PR #49).
The favicon 404 visible in the second screenshot is unrelated cosmetic noise (no
`favicon.ico` file in the repo) and not part of this fix. Not yet manually re-tested
against a live CSV upload in a browser (no browser/UI access in this session) — see
Next Steps.

## Next Steps
1. Re-upload the CSV(s) that triggered the errors through the app's UI and confirm no
   console errors and that `rows updated` reflects correctly. Pay particular attention
   to any upload involving multiple files or overlapping date ranges, since that's the
   scenario that produced bug 2.
2. If uploads are still failing after this, get the exact console error again —
   don't assume it's a variant of either bug above without seeing the new message,
   since the error code (42501 vs 21000 vs something else) determines whether it's an
   RLS/permissions issue or an application-logic issue.
3. If any other repo/task ever needs to track Supabase migrations as files, consider
   adding a `supabase/migrations/` directory to this repo so schema/RLS changes are
   code-reviewable — currently they exist only inside the Supabase project itself.

## Open Questions / Blockers
N/A — root cause identified and fixed.

## Environment & Config Notes
- Supabase project: `ljjwssicvvyyueyznmou` ("PT Dashboard", org `ohwlndcukzphjbtjjpin`).
- Table affected: `public.shop_sales` (RLS enabled).
- New policy: `anon_update_shop_sales` (UPDATE, role `anon`, `USING (true) WITH CHECK (true)`).
- Branch: `claude/sales-csv-upload-error-y12v26`; PR: `ONE-LDN/shop-sales#49` (draft).
- Commits this session: `0142317` (session log / RLS fix note), `a114d87` (dedupe fix
  in `index.html`).

## Notes & Gotchas
- PostgREST's upsert-via-`on_conflict` needs **both** INSERT and UPDATE RLS policies
  for whatever role is calling it — an INSERT-only policy is enough for first-time
  rows but silently fails (RLS denies, not a graceful skip) the moment a row collides
  with an existing key. Any future table that adds `merge-duplicates`/upsert behavior
  should get this checked as part of that change, not discovered later via a user
  bug report.
- Separately, `ON CONFLICT DO UPDATE` in Postgres cannot touch the same conflict-target
  row twice within one statement (error `21000`) — this is unrelated to RLS and will
  happen even with correct policies if the *client* sends two rows with the same
  conflict key in one batch. Any code building batched upsert payloads from
  concatenated/multi-source input (multiple files, multiple pages, etc.) needs to
  dedupe by the conflict key before sending, not rely on the DB to reconcile it.
- These are genuinely two independent bugs that happened to surface back-to-back on
  the same upload flow — don't assume fixing one implies the other is also fixed.
- This repo has no in-repo migration files — all schema/RLS state lives only in the
  Supabase project. Anyone auditing schema history needs Supabase MCP/dashboard
  access, not `git log`.


**Date:** 2026-08-03
**Project:** Shop Sales — cross-tool session (Notion progress tracking, not shop-sales code)
**Mode:** Rolling Log + Git Push
**Status:** Complete (no-op on both sides)

---

## Project Context
Same repo-native logging system introduced in the 2026-08-03 "Add save-conversation
skill and session log" entry below. This session is otherwise unrelated to shop-sales
code — it's logged here only because the git-native logging habit applies regardless of
what the session's actual subject was.

## Session Goal
Started as an attempt to post a drafted weekly update to the ONE LDN Notion "Progress
Document." Partway through, Saffron redirected: no Notion write, log the session in this
repo per `save-conversation` instead.

## State Before This Session
Repo was clean on branch `claude/practical-euler-2whi78` (matches the branch assigned
for this session per the task's git instructions), tip commit `6d1a26d` ("Update
SKILL.md"). No shop-sales code, data, or docs work was pending.

## What Was Done
Loaded the Notion MCP tools and fetched the ONE LDN "📈 Progress Document" page
(`https://app.notion.com/p/3660164f7fc681d79e14e2085e8c6ed6`) to check its structure
before writing a weekly update entry to it — read-only, no edits made. The content of
the draft update and whatever question/selection preceded this (referenced obliquely at
the top of this session as "your selection") was lost to context summarization before
being captured here, so it is **not** reconstructable from this session — see Open
Questions below. Saffron then said "nothing to notion," cancelling the planned Notion
write, and asked instead to log the session per this repo's `save-conversation` skill.
Re-verified git state (`git log`, `git status`, `git branch --show-current`) and read
the existing log before writing this entry.

## Artifacts Produced / Modified

| File | What it is | Status | Location |
|------|------------|--------|----------|
| logs/shop-sales_log.md | Rolling session log | Modified (this entry prepended) | /logs/shop-sales_log.md |

No other files touched. No Notion pages were created or edited — the fetch was
read-only.

## Decisions & Reasoning
- **Did not attempt to reconstruct or re-post the Notion draft from memory**: the
  original draft text and the user's selection on it were not present in this session's
  visible context (summarized away before this entry was written). Fabricating either
  would risk posting incorrect content to a shared team doc. Deferred entirely to
  Saffron's explicit "nothing to notion."
- **Logged this session in shop-sales' git log despite it being about a different
  system (Notion progress tracking)**: Saffron's instruction was explicit
  ("log session according to .../save-conversation/SKILL.md"), and this repo's
  `CLAUDE.md` says session logging is "always on" — didn't second-guess the request even
  though the subject matter is cross-project.

## Current State (end of session)
No Notion changes made. No shop-sales code changes. This log entry is the only output
of the session, about to be committed and pushed.

## Next Steps
1. If the weekly Progress Document update still needs to go up, that has to be
   re-drafted from scratch in a fresh session — nothing from this session's earlier
   draft is recoverable. Start from the current "Active Projects" table state, which was
   last updated 09/06/2026 per the page content read this session.
2. Otherwise, no shop-sales-specific follow-up from this session.

## Open Questions / Blockers
What was actually in the pre-summarization Notion draft, and what the user selected in
response to whatever question preceded "Taking your selection as the green light" — both
unknown. If this matters, ask Saffron directly rather than guessing; do not assume the
Progress Document table in the fetched snapshot (last updated 09/06/2026) is still
current.

## Environment & Config Notes
Repo `ONE-LDN/shop-sales`, branch `claude/practical-euler-2whi78`, tip commit at session
start `6d1a26d`. Notion page touched (read-only): "📈 Progress Document"
(`3660164f-7fc6-81d7-9e14-e2085e8c6ed6`), parented under "ONE LDN — Digital
Transformation" → "Teamspace Home".

## Notes & Gotchas
This is a good illustration of why the "handoff document, not reference file" standard
in this skill matters: mid-session context summarization dropped the Notion draft
content entirely, and the only honest move was to say so rather than reconstruct it
from a plausible-sounding guess.

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
