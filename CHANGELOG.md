# Changelog

## Unreleased

### Changed
- **Accurate live detection**: the `●` running marker is now per-**session** instead of per-**folder**. Previously every session that ever ran in a folder with a live `claude` lit up green (closed and ancient sessions included). Now each running `claude` process is mapped to its exact session — `--resume <id>` from the process args, or the newest recently-written jsonl in its cwd for fresh starts — so the live count matches the real number of running sessions.
- Default listing is now **100** sessions (was 30). Override with the `CCSESSIONS_LIMIT` env var, `-n <count>`, or `-a` for all.

### Added
- **Agent + session-start columns**: every listing (flat, grouped, 📌 frozen) now shows when the session started — the first timestamp in its jsonl, so it survives `--resume` — and the agent label pinned to it in `~/.envoak/session-identity/<id>.env`. The agent column is dropped entirely when no listed session has a label, so non-envoak users lose no width. `--json` rows gain `started`, `started_local`, and `agent`; the text filter matches agent labels too.
- `cc-agent-session`: companion ccstatusline widget printing `<agent> · <session start>`, prefixed with 📌 when `tbe freeze` has pinned the session (same `ccsessions-frozen.json`).
- **Pending-recall indicator**: `cc-agent-session` appends 💡 when `~/.memosan/proactive-inbox.md` is non-empty, i.e. the memosan librarian has a recall queued. Passive on purpose — the statusline reports that something is waiting instead of a hook injecting it into agent context every prompt. Absent file, no indicator.
- **Per-session recall routing**: both the 💡 in `cc-agent-session` and `cc-memosan-recall` read `~/.memosan/proactive-inbox-<session>.md` — what memosan queued for *this* session — falling back to the legacy shared `proactive-inbox.md`. With several sessions running, the shared file lit the lamp for suggestions another session would collect, since memosan's delivery hook clears whatever inbox it reads.
- `cc-memosan-recall`: companion widget for a statusline row of its own, printing the *top* queued memosan suggestion (`💡 knowledge: heartwood.md (0.66) +2`) rather than only the fact that one exists. Silent — no output at all — when the inbox is empty or absent, so the row disappears rather than showing a placeholder.
- **Terminal window title**: `cc-agent-session --title` sets the window title to `<agent> · <repo> · <start>`, so windows are distinguishable from the taskbar. Run it from a `SessionStart`/`UserPromptSubmit` hook — the escape has to be written straight to `/dev/tty` (widget and hook stdout both belong to Claude Code, not the terminal), and doing that from the statusline widget writes into the middle of the TUI's frame, corrupting multi-byte glyphs into `�`. Setting it from the widget is therefore off by default; `CC_AGENT_SESSION_TITLE=1` opts in, and `CC_TITLE_TTY=<file>` redirects the escape to a file for inspection.
- Repo now ships a `tbe` symlink (→ `ccsessions`) alongside the script itself, matching the
  `~/.local/bin/tbe` install symlink.
- **Freeze / thaw**: pin a session for later with a note. `tbe freeze [N] [note…]` (no N = the session you're currently in, via `CLAUDE_CODE_SESSION_ID`; with N = a number from the last listing). Frozen sessions show in a 📌 section pinned to the top of every listing — always visible even when older than the scan window — and carry your note. `tbe thaw <#|id>` unpins. State lives in `~/.claude/ccsessions-frozen.json`; `--json` rows gain a `frozen` flag.
- **TreeBirdsEye (`tbe`)**: invoke ccsessions as `tbe` to get the per-project cockpit view by default — sessions bucketed by project, freshest first, each line showing what the session is *currently doing* (latest jsonl activity, not the opening message). Same script; the entry name selects the view.
- `-g`: group any ccsessions listing by project.
- `--json`: machine-readable dump (id, proj, cwd, tldr, doing, mtime, live, ago) for orchestration tooling.
- Each row now shows the session's **latest** activity (`doing`, tailed from the jsonl) instead of only the first message — truer "what's it doing now" than a stale agent signal.
- `cc-session-num`: companion script that prints the current session's number (`#1`, `#2`, …) for use as a [ccstatusline](https://github.com/sirmalloc/ccstatusline) custom widget. Uses `CLAUDE_CODE_SESSION_ID` to match the running session against the mtime-sorted session list.
