# shop-sales

## Files

- `logs/shop-sales_log.md` — rolling session log, newest first. **Read the top entry
  at the start of every session** — it says exactly where things stand.

## Session Logging (always on)

Log file: `logs/shop-sales_log.md` — rolling log, newest entry at top, committed and
pushed on the session's working branch.

At the end of **every** session — or whenever asked ("save this conversation", "log this
session", "capture this", "save notes from this", "create a conversation log") — invoke
the **`save-conversation`** skill and follow it exactly. Do not skip this, even for short
sessions. The skill owns the entry template and the prepend/commit workflow — it is the
single source of truth for the log format.

The standard: a future Claude session with zero memory of this conversation must be able
to pick up and continue without asking for re-explanation. Err on the side of more
detail, not less.
