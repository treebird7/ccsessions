# Changelog

## Unreleased

### Changed
- **Accurate live detection**: the `●` running marker is now per-**session** instead of per-**folder**. Previously every session that ever ran in a folder with a live `claude` lit up green (closed and ancient sessions included). Now each running `claude` process is mapped to its exact session — `--resume <id>` from the process args, or the newest recently-written jsonl in its cwd for fresh starts — so the live count matches the real number of running sessions.
- Default listing is now **100** sessions (was 30). Override with the `CCSESSIONS_LIMIT` env var, `-n <count>`, or `-a` for all.

### Added
- Repo now ships a `tbe` symlink (→ `ccsessions`) alongside the script itself, matching the
  `~/.local/bin/tbe` install symlink.
- **Freeze / thaw**: pin a session for later with a note. `tbe freeze [N] [note…]` (no N = the session you're currently in, via `CLAUDE_CODE_SESSION_ID`; with N = a number from the last listing). Frozen sessions show in a 📌 section pinned to the top of every listing — always visible even when older than the scan window — and carry your note. `tbe thaw <#|id>` unpins. State lives in `~/.claude/ccsessions-frozen.json`; `--json` rows gain a `frozen` flag.
- **TreeBirdsEye (`tbe`)**: invoke ccsessions as `tbe` to get the per-project cockpit view by default — sessions bucketed by project, freshest first, each line showing what the session is *currently doing* (latest jsonl activity, not the opening message). Same script; the entry name selects the view.
- `-g`: group any ccsessions listing by project.
- `--json`: machine-readable dump (id, proj, cwd, tldr, doing, mtime, live, ago) for orchestration tooling.
- Each row now shows the session's **latest** activity (`doing`, tailed from the jsonl) instead of only the first message — truer "what's it doing now" than a stale agent signal.
- `cc-session-num`: companion script that prints the current session's number (`#1`, `#2`, …) for use as a [ccstatusline](https://github.com/sirmalloc/ccstatusline) custom widget. Uses `CLAUDE_CODE_SESSION_ID` to match the running session against the mtime-sorted session list.
