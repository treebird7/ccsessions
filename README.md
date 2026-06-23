# ccsessions

A tiny CLI session manager for [Claude Code](https://claude.com/claude-code). Lists your
sessions with project, a one-line preview, and how long ago each was active — then resumes
the one you pick. No `--resume` picker forest.

```
   1  memoak       0m  hi i need some help my terminal got stuck...
 ● 2  watsan       4m  morning watsan! for a chat bot in obsidian...
 ● 3  spidersan    4m  Install the ponytail plugin...
 ● 4  mappersan    4m  hey, you are mappersan, look for pending tasks...

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

## License

MIT
