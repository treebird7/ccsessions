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

### TreeBirdsEye (`tbe`)

The repo also ships a `tbe` symlink to the same script. Invoking it as `tbe` instead of
`ccsessions` switches the default view to the per-project cockpit (grouped, freshest first)
and enables `tbe freeze`/`tbe thaw`. Symlink it alongside `ccsessions`:

```bash
ln -s ~/.local/bin/ccsessions ~/.local/bin/tbe
```

## Usage

```bash
ccsessions            # 30 most-recent sessions; type a number to resume
ccsessions -n 60      # show more
ccsessions -a         # show all
ccsessions ibn        # filter by project or preview text
```

- **● green dot** = a `claude` process is currently running in that project's directory.
- **agent + start columns** = the agent label pinned to that session (envoak/dawn users; the
  column disappears entirely when no listed session has one) and when the session began.
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

### Agent label + session start (`cc-agent-session`)

A second widget prints the session's agent label and when the session started, plus a 📌 when
`tbe freeze` has it pinned:

```
sherlock-m5 · Jul31 13:36
```

```bash
curl -fsSL https://raw.githubusercontent.com/treebird7/ccsessions/main/cc-agent-session \
  -o ~/.local/bin/cc-agent-session && chmod +x ~/.local/bin/cc-agent-session
```

Add it as another **Custom Command** widget.

#### Terminal window title

As a side effect the widget also sets the **terminal window title** to
`<agent> · <repo> · <start>`, so you can tell your windows apart from the taskbar:

```
sherlock-m5 · Toak · Jul31 13:36
```

The escape has to go straight to `/dev/tty`, because both widget and hook stdout belong to
Claude Code, not to your terminal. **Set it from a hook, not from the widget** — a widget
writes on every statusline render, which lands in the middle of the TUI's own frame and
shreds multi-byte glyphs into `�`. A hook fires between turns instead:

```json
{ "hooks": { "SessionStart": [
    { "hooks": [{ "type": "command", "command": "~/.local/bin/cc-agent-session --title" }] } ] } }
```

- `cc-agent-session --title` — title only, nothing on stdout. The hook form above.
- `CC_AGENT_SESSION_TITLE=1` — also set the title from the statusline widget. Off by default
  for the frame-corruption reason above; fine in terminals that tolerate it.
- `CC_TITLE_TTY=/tmp/x` — write the escape to a file instead, to see what it would set.

#### Pending-recall indicator

A trailing 💡 means `~/.memosan/proactive-inbox.md` is non-empty — the
[memosan](https://github.com/treebird7/memosan) librarian has a recall queued for you. Passive by
design: the statusline says something is waiting, rather than a hook injecting it into the agent's
context on every prompt. No memosan, no file, no 💡.

### What memosan queued (`cc-memosan-recall`)

`cc-agent-session`'s 💡 says *something* is waiting. This widget says *what* — the top queued
suggestion, on a statusline row of its own:

```
💡 knowledge: heartwood.md (0.66) +2
```

```bash
curl -fsSL https://raw.githubusercontent.com/treebird7/ccsessions/main/cc-memosan-recall \
  -o ~/.local/bin/cc-memosan-recall && chmod +x ~/.local/bin/cc-memosan-recall
```

Add it as a **Custom Command** widget — a row of its own suits it, since it prints nothing at
all when the inbox is empty, and an empty row is the right rendering of "the librarian has
nothing for you". `+N` counts the suggestions below the top one; titles longer than 40 chars
are truncated. It reads the same inbox the 💡 does, written by memosan's
`scripts/proactive-scorer.ts` and cleared by its delivery hook — so the row empties itself
once the suggestion has actually reached the agent.

Both indicators read `~/.memosan/proactive-inbox-<session>.md`, i.e. only what memosan queued
for *this* session, falling back to the legacy shared `proactive-inbox.md` for older memosan
installs. The distinction matters when you run several sessions at once: the delivery hook
clears whatever inbox it reads, so with one shared file the first session to prompt collects
a suggestion scored from another session's topic, and the lamp lights in sessions that will
never receive it.

#### Where the values come from

The start time is the first timestamp in the
session's jsonl, so it survives `--resume`. The agent label comes from
`~/.envoak/session-identity/<session-id>.env` (written by the `dawn` skill), falling back to
`~/.envoak/current-identity.env` and then `$TOAK_AGENT_ID` — without envoak it just prints
`$TOAK_AGENT_ID` or `?`.

## License

MIT
