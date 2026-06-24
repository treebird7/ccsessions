# ccsessions

A tiny CLI session manager for [Claude Code](https://claude.com/claude-code). Lists your
sessions with project, a one-line preview, and how long ago each was active — then resumes
the one you pick. No `--resume` picker forest.

```
   1  webapp       2m  add a dark mode toggle to the settings page
 ● 2  api          5m  the /users endpoint is returning 500s, can you...
 ● 3  scraper      1h  refactor the fetch loop to use asyncio
   4  notes-cli    3h  write tests for the tag parser

  ● = running   (4 shown)

Resume # (Enter to quit):
```

## Why

Claude Code stores every session under `~/.claude/projects/`. `claude --resume` makes you
scroll a picker; `claude -c` only grabs the newest in the current dir. If you run many
sessions across many projects (and lose them to a terminal restart), neither is fun.
`ccsessions` shows them all at a glance and resumes by exact id.

## Install

No dependencies — just Python 3 (stdlib only).

```bash
curl -fsSL https://raw.githubusercontent.com/treebird7/ccsessions/main/ccsessions \
  -o ~/.local/bin/ccsessions && chmod +x ~/.local/bin/ccsessions
```

Make sure `~/.local/bin` is on your `PATH`.

## Usage

```bash
ccsessions            # 30 most-recent sessions; type a number to resume
ccsessions -n 60      # show more
ccsessions -a         # show all
ccsessions ibn        # filter by project or preview text
```

- **● green dot** = a `claude` process is currently running in that project's directory.
- **preview** = the session's first real user message (skill/command boilerplate is skipped),
  or its saved summary if one exists.
- Picking a session runs `claude --resume <id>` in that session's working directory.

### Resume flags

By default it resumes with `--dangerously-skip-permissions`. If you don't want that, edit the
last line of the script (it's ~120 lines, all in one file) and drop the flag.

## How it works

Reads `~/.claude/projects/*/*.jsonl`, sorts by modification time, pulls `cwd` and a preview
from each, and cross-references running `claude` PIDs (`pgrep` + `lsof`) to flag live
sessions. Selecting one `exec`s `claude --resume`.

## ccstatusline integration

Show the current session number (`#1`, `#2`, …) in your [ccstatusline](https://github.com/sirmalloc/ccstatusline) statusbar.

Save this as `~/.local/bin/cc-session-num`:

```bash
curl -fsSL https://raw.githubusercontent.com/treebird7/ccsessions/main/cc-session-num \
  -o ~/.local/bin/cc-session-num && chmod +x ~/.local/bin/cc-session-num
```

Or manually:

```python
#!/usr/bin/env python3
import os, glob
sid = os.environ.get('CLAUDE_CODE_SESSION_ID', '')
files = sorted(glob.glob(os.path.expanduser('~/.claude/projects/*/*.jsonl')), key=os.path.getmtime, reverse=True)
for i, f in enumerate(files[:60], 1):
    if sid in f:
        print(f'#{i}')
        break
```

Then in `ccstatusline --tui`, add a **Custom Command** widget with command `cc-session-num`.

## License

MIT
