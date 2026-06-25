# ccsessions — Multi-Machine Sync via Dolt

**Beads:** tb-7j7h  
**Status:** Design / Pre-implementation  
**Priority:** P2

---

## Problem

`ccsessions` reads `~/.claude/projects/*/*.jsonl` locally. Sessions on other machines are invisible. A developer running Claude Code on i7, m5, and a laptop sees three disconnected session lists.

## Goal

One unified session list across all machines, with machine tags, live-status heartbeats, and local-first read performance. Resume works locally; remote sessions are visible but prompt an SSH handoff.

---

## Architecture Decision: Direct Tunnel vs Local Replica

| | Direct tunnel (m5:3307) | Local Dolt replica |
|---|---|---|
| Read speed | ~10ms over LAN | <1ms |
| Offline reads | ❌ | ✓ (last pulled state) |
| Complexity | Low | Medium (pull/push cadence) |
| Already available | ✓ (beads uses same tunnel) | Needs `dolt clone` per machine |

**Decision: Direct tunnel, with local JSON cache fallback.**  
Read from m5 Dolt on launch; if tunnel is down, fall back to `~/.cache/ccsessions/sessions.json` (last known state). Keep it simple — no local Dolt replica needed.

---

## Schema

```sql
CREATE TABLE sessions (
  machine     TEXT     NOT NULL,
  session_id  TEXT     NOT NULL,
  cwd         TEXT,
  project     TEXT,
  tldr        TEXT,
  mtime       INTEGER,          -- epoch seconds, file mtime
  last_seen   INTEGER,          -- epoch seconds, last heartbeat
  is_live     BOOLEAN  DEFAULT FALSE,
  PRIMARY KEY (machine, session_id)
);
```

**Index:**
```sql
CREATE INDEX idx_sessions_mtime ON sessions (mtime DESC);
```

Machine names come from `hostname -s` (e.g. `i7`, `m5`, `macbook`).

---

## Sync Events

### 1. Session discovered (on `ccsessions` launch)

When ccsessions scans `~/.claude/projects/` and finds a session not yet in Dolt:

```sql
INSERT INTO sessions (machine, session_id, cwd, project, tldr, mtime, last_seen, is_live)
VALUES (?, ?, ?, ?, ?, ?, ?, ?)
ON DUPLICATE KEY UPDATE tldr=VALUES(tldr), mtime=VALUES(mtime);
```

### 2. Heartbeat (while Claude Code is running)

A Claude Code `PostToolUse` hook fires `ccsessions-heartbeat` in the background:

```sql
UPDATE sessions
SET last_seen = UNIX_TIMESTAMP(), is_live = TRUE
WHERE machine = ? AND session_id = ?;
```

Stale threshold: if `last_seen` > 120s ago, treat as not live (display as ` ` not `●`).

### 3. Session end

`PostSessionEnd` hook (or detected via stale heartbeat):

```sql
UPDATE sessions SET is_live = FALSE WHERE machine = ? AND session_id = ?;
```

### 4. Pull on launch

Before displaying the list, `ccsessions` queries Dolt for all sessions from other machines:

```sql
SELECT * FROM sessions WHERE machine != ? ORDER BY mtime DESC LIMIT 200;
```

Results are merged with the local scan and written to `~/.cache/ccsessions/sessions.json`.

---

## Live Detection Across Machines

Local: `pgrep -x claude` + `lsof` (existing, keep as-is for local sessions).  
Remote: rely on `is_live` + `last_seen` from Dolt. A session is shown as live if `is_live = TRUE AND last_seen > NOW() - 120`.

The heartbeat hook needs to be wired in `~/.claude/settings.json`:

```json
{
  "hooks": {
    "PostToolUse": [
      { "matcher": "", "hooks": [{ "type": "command", "command": "ccsessions-heartbeat" }] }
    ]
  }
}
```

`ccsessions-heartbeat` is a fast script (~5ms): reads `CLAUDE_CODE_SESSION_ID` from env, fires a background MySQL UPDATE to m5, exits immediately.

---

## Display Format

```
   1  [i7]  webapp       2m  add a dark mode toggle to the settings page
 ● 2  [i7]  api          5m  the /users endpoint is returning 500s, can you...
   3  [m5]  scraper      1h  refactor the fetch loop to use asyncio
   4  [m5]  notes-cli    3h  write tests for the tag parser

  ● = running   (4 shown, 2 machines)
```

- Local machine sessions: no tag (or `[i7]` if mixed list)
- Remote sessions: `[machine]` tag in dim color
- Remote live sessions: `●` in yellow (can't be confirmed locally, trust Dolt heartbeat)

---

## Resume UX

| Session | Action |
|---|---|
| Local | `exec claude --resume <id>` in cwd (existing behavior) |
| Remote (same LAN) | Print: `ssh <machine> -t 'cd <cwd> && claude --resume <id> --dangerously-skip-permissions'` — copy to clipboard, or exec directly if `--ssh` flag passed |
| Remote (offline) | Show error: `<machine> not reachable` |

Phase 1: show-only for remote (print the SSH command, don't exec).  
Phase 2: `ccsessions --ssh` flag to exec the SSH resume directly.

---

## `cc-session-num` Behavior

Ranks **local sessions only** (current machine's `~/.claude/projects/`). The `#N` in the statusline reflects position in the local list, not the merged cross-machine list. This keeps the widget fast and deterministic regardless of tunnel state.

---

## Connection

Reuses the beads tunnel: `m5:3307` via `autossh` (already managed by `beads_tunnel()` in `.zshrc`). ccsessions checks tunnel health with a 200ms `SELECT 1` probe before querying; falls back to cache on timeout.

Database: new Dolt database `ccsessions` on m5 (separate from `beads`).

---

## Implementation Plan

### Phase 1 — Schema + pull-on-launch (read-only cross-machine view)
1. `dolt init ccsessions` on m5, create `sessions` table
2. Add `--sync` flag to `ccsessions` that pulls remote sessions and merges display
3. Local sessions pushed on launch (upsert)
4. Cache written to `~/.cache/ccsessions/sessions.json`
5. Fallback to cache when tunnel is down

### Phase 2 — Heartbeat (live status across machines)
6. `ccsessions-heartbeat` script
7. Wire into `~/.claude/settings.json` PostToolUse hook
8. `is_live` + `last_seen` staleness logic in display

### Phase 3 — Resume handoff
9. Remote session detection on pick
10. SSH command generation / exec with `--ssh` flag

### Phase 4 — `cc-session-num` multi-machine mode (optional)
11. `--global` flag: rank across all machines (for cross-machine context)

---

## Open Questions

- [ ] Should `ccsessions` default to `--sync` or require opt-in? (Network call on every launch is a tradeoff)
- [ ] Dolt database on m5: shared with beads user/password, or separate ccsessions credentials?
- [ ] Should `tldr` be updated on every heartbeat (expensive scan) or only on session start?
- [ ] Multi-user machines: scope by `$USER` in the schema?
