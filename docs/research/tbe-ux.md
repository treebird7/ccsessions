# tbe UX research: making the picker friendlier for a human

2026-11-08. Scope: the curses picker (`run_picker`, `picker_items`), the scan loop in
`main` (`take`, `closed_rows`, `pinned_rows`) and search (`search_session`, `hit`).
These notes compare tbe with Claude Code's own `/resume` picker and with fzf, atuin, zoxide,
lazygit, tig, k9s, ccmanager and claude-squad, using primary sources only. Sizes:
**S** is under ~30 lines, **M** is under ~100, **L** is more than that. Every item is stdlib-only.

## What was measured on this machine

- `~/.claude/history.jsonl`: 21,773 prompts across 1,207 session ids. **1,054 of those ids
  have no transcript** (the same number as tbe's earlier session total, by coincidence) under `~/.claude/projects/*/` (none deeper either). Their last prompts
  run from 2025-11 to 2026-10-16, with 100–180 per month from March to September.
- Surviving transcripts all have mtimes in 2026-10 or 2026-11 (989 files). That fits the
  30-day default sweep. `cleanupPeriodDays` is **120 now**; the 2026-11-01 settings backup
  has no such key, so 30 applied until recently. Some ghosts were last used in October, inside
  the window a 30-day sweep should still have kept, so their cause is **not determined**.
- `history.jsonl` holds **only the owner's prompts** (`display`, `pastedContents`,
  `timestamp`, `project`, `sessionId`), never Claude's replies. `pastedContents` values are
  inline (`{"id","type":"text","content":…}`), not paste-cache references.
- `ccsessions-closed.json` has 296 entries: **127** still have a transcript and aren't
  frozen (what `c` shows), and 168 are already gone, so they are ghosts with a close note.
- Transcripts carry `"type":"ai-title"` (10,991 lines) and `"custom-title"` entries.
  `scan()` reads neither and uses `summary` or the first user message instead.
- Pasted-text search fails in two measured ways. `tbe search '"date"'` matches **1**
  session, while plain `date` matches 143. 171 transcripts contain `\"date\"` (JSON-escaped),
  some of them temp-dir sessions that tbe hides. `tbe search '⏺ date'`
  matches **0**; plain `date` matches 143. The glyph `⏺` is something Claude Code draws on
  screen, so it is in a paste from the terminal but never in the jsonl.

## External reference points (cited)

- **Claude Code `/resume` picker**: `/` "or any printable character" enters search, and you
  can paste a PR URL. `Space` previews, `Ctrl+R` renames, `Ctrl+A` toggles all projects,
  `Ctrl+W` toggles worktrees, `Ctrl+B` toggles the current branch. Rows show the name, else the
  AI-generated title, summary or first prompt, plus "time since last activity, git branch,
  and file size". Unnamed sessions get a generated title.
  https://code.claude.com/docs/en/sessions#use-the-session-picker and
  https://code.claude.com/docs/en/sessions#name-your-sessions
- **Retention**: files under `projects/<project>/<session>.jsonl` and others are deleted
  "once they're older than `cleanupPeriodDays`"; "the default is 30 days and the minimum is 1".
  `history.jsonl` is under "Kept until you delete them" (except under HIPAA config).
  https://code.claude.com/docs/en/claude-directory#cleaned-up-automatically. The page does not
  say how age is measured. The settings-reference page was truncated when fetched, so it isn't
  cited for these details.
- **fzf**: extended-search syntax `'exact`, `^prefix`, `suffix$`, `!negate`, and `|` as OR;
  `--preview` runs a command on the current line. https://github.com/junegunn/fzf (the README
  doesn't document a matched/total counter, so that isn't cited.)
- **atuin**: `filter_mode` takes `global`, `host`, `session`, `directory` or `workspace`,
  and "you can still cycle through the rest with ctrl-r". `search_mode` is
  `prefix`/`fulltext`/`fuzzy`/`daemon-fuzzy`, and `show_preview` exists.
  https://docs.atuin.sh/latest/configuration/config/ (the older `/configuration/config/` path
  returns a 404). `atuin search --before/--after`, with the example `--after "yesterday 3pm"`;
  the page shows examples but not a list of accepted formats.
  https://docs.atuin.sh/latest/reference/search/
- **zoxide**: `zi` is "cd with interactive selection (using fzf)", with `_ZO_FZF_OPTS`
  passed through. https://github.com/ajeetdsouza/zoxide
- **lazygit**: `?` opens the keybindings menu; `/` filters or searches per panel, not globally.
  https://github.com/jesseduffield/lazygit/blob/master/docs/keybindings/Keybindings_en.md
- **k9s**: `?` shows "active keyboard mnemonics and help"; `/` filters, `/!` inverts,
  `/-f` fuzzy. https://github.com/derailed/k9s
- **tig**: `/` `?` `n` `N` search; the title bar shows position, for example
  "commit 1 of 61 (1%)"; `h` opens help. https://jonas.github.io/tig/doc/manual.html
- **claude-squad**: "The menu at the bottom of the screen shows available commands", `?` for
  help, and `tab` switches between the preview and diff tabs. https://github.com/smtg-ai/claude-squad
- **ccmanager**: per-session states Waiting/Busy/Idle, per-project counts
  `[active/busy/waiting]`, and `/` to filter projects. It does not describe a browsable
  history. https://github.com/kbwo/ccmanager

## P0: fix these first

### P0-1. Search hides closed matches, and nothing says closed sessions exist (S)
**Problem.** `take()` skips closed ids, and `closed_rows()` runs only when `show_closed`.
So `/foo` in the picker says "no session matches" even when a closed session matches. This
is how the owner lost a session (pain point 3). The title bar shows `+closed` only once the
toggle is already on.
**Precedent.** Claude Code's picker keeps scope toggles one key away (`Ctrl+A`/`Ctrl+W`/`Ctrl+B`),
atuin cycles filter modes with ctrl-r, and tig always shows "N of M" in its title.
**Change.**
- In `main`'s picker loop, when `filt` is set, always compute `closed_rows()` (it already
  respects `hit()`). If `show_closed` is off, pass only the count into `run_picker`.
  Without a filter, the count is the cheap `sum(1 for s in closed if s in by_id and s not in frozen)`,
  not `len(closed)`.
- In `run_picker.draw`, have the title say `tbe  42 sessions · 127 closed (c)`. Have the
  empty state say "no open session matches; 3 closed match, press c".
- Simpler alternative: while a search is active, show closed rows by default (in the closed
  section that already exists in `picker_items`). A user typing a query wants every match.

### P0-2. Pasting into `/` can fire picker keys (S; found by reading the code, not reproduced)
**Problem.** `prompt()` uses `scr.getstr(…, 200)`, which returns at the first newline and
keeps 200 characters. A multi-line paste of Claude's reply submits line 1 only, and the
rest stays in the tty buffer. After `main` rescans and reopens the picker, `loop()`'s
`getch()` reads that leftover text as keys: `x` closes the highlighted row, `f` freezes it,
`c`/`m` toggle, `q` quits, and a `\n` **resumes the highlighted session** with
`--dangerously-skip-permissions`. In `--plain` mode the leftover lines feed the next
`input()`, where a digit line resumes too.
**Precedent.** Claude Code's picker accepts pasted PR URLs in search (sessions doc above), so
pasting into a session search is an expected use.
**Change.** In `prompt()`, after `getstr` returns, drain the pending input:
`scr.nodelay(True)`, read with `getch()` until it returns -1, then `nodelay(False)`. Append
the drained text to the query with newlines turned into spaces. Raise the 200-character cap
to around 2000. At minimum, call `curses.flushinp()` before returning to `loop()`. In
`--plain`, read the query from `sys.stdin` with `select` the same way.

### P0-3. Make pasted snippets match (S–M)
**Problem.** The owner finds sessions by pasting a fragment of Claude's reply (pain point 5).
`search_session` ANDs every whitespace token, and its byte prefilter compares raw tokens
against **JSON-escaped** file bytes. So a token with `"` or `\` never matches (1 vs 171
measured). One terminal glyph (`⏺ ⎿ • →`) or markdown punctuation sinks the whole query
(0 vs 143 measured).
The quote case was measured with `tbe search`. The picker uses the same `search_session`.
**Precedent.** fzf has explicit modes (`'exact`, `!negate`, `|`) instead of guessing, and
atuin offers `fulltext` and `fuzzy` modes (both cited above).
**Change.**
- `search_session`: build the byte needles from `json.dumps(t, ensure_ascii=False)[1:-1].encode()`.
  Use `ensure_ascii=False` because transcripts store `—` raw: 0 `\u` escapes and 1,277 em
  dashes in one file. Use the needles in both raw checks (the whole blob and `raw.lower()`).
  Only the `texts_of` match after `json.loads` keeps the unescaped term.
- New `query_terms(q)`, called from `hit()` and the ranked path: lower-case, strip leading
  and trailing punctuation from each token, drop tokens with no alphanumeric character, and
  drop duplicates. For long pastes (say more than 8 tokens), keep the 8 longest distinct
  words. Long words carry the signal, and fewer terms are cheaper.
- Optional exact-phrase mode: a query in `'…'` matches the whitespace-collapsed phrase
  against whitespace-collapsed text, following fzf's `'` convention.
- Optional ranked fallback in `tbe search`: when nothing matches every term, return sessions
  matching ≥ 75% of them, labelled "partial". `search_session` already counts `found`.

### P0-4. Show "ghost" sessions from `history.jsonl` (M)
**Problem.** 1,054 session ids exist only in `history.jsonl` (pain point 4). The owner found
the July ones by hand. Claude Code's own picker can't show them, because the transcripts are
deleted (retention doc above), and `claude --resume <id>` would print
"No conversation found with session ID" (https://code.claude.com/docs/en/sessions#resume-by-session-id-or-name).
**Limits to state in the UI.** Ghosts **cannot be resumed**, and their search covers only
the owner's own prompts and pastes, **not Claude's replies**. So a pasted reply snippet
can't find a ghost.
**Change.**
- New `ghost_rows(known_ids)`: stream `history.jsonl` and group by `sessionId`, keeping
  `project`, first and last `timestamp`, the prompt count and the first non-`/` `display`
  as the tldr. Drop ids in `by_id` and temp-dir projects. `project` is a raw path, so encode
  it first: `re.sub(r'[^A-Za-z0-9]', '-', project).startswith(TMP_DIRS)`. Merge in the 168
  closed ids with no transcript, and use their close note as the tldr.
  Search text is `display` plus each `pastedContents[*].content`, through the same
  `query_terms`. About 5 MB, so a single pass is fine.
- Picker: a `gone (1054)` section after `closed`, toggled with `o` ("old"; `g` is taken by
  Home). The title shows the count
  even when hidden, as in P0-1. Rows are dimmed and marked `†`.
- Enter on a ghost never calls `exec claude`. It opens the preview with all of that
  session's prompts (redacted, as `recent()` does) and the line
  `transcript deleted; prompts only. cwd: <project>`.
- `tbe search` and `--json`: include ghosts with `"gone": true` and no `file`, ranked below
  live matches.

## P1: worth doing next

### P1-1. A hint bar that shows what is hidden, and `?` for help (S)
**Problem.** The fixed key line doesn't show state. The owner pressed `m` and saw nothing
change (pain point 2, since fixed), and nothing says closed or ghost sessions exist.
**Precedent.** lazygit `?`, k9s `?` ("active keyboard mnemonics"), tig `h`, and
claude-squad's bottom menu with `?` (all cited above).
**Change.** In `run_picker.draw`, build the key line from state, for example
`c closed(127)`, `o gone(1054)`, and `m more (scanned 412/989)`. Drop `m` once `fiter` is
exhausted, which needs a flag passed in from `main`. Add a `?` overlay listing every key and
the search syntax, drawn in `draw` when a `help` flag is set.

### P1-2. Date filters and a date jump (M)
**Problem.** The owner remembers "roughly July" (pain point 5). `ago()` only shows `37d`,
so old rows give no calendar anchor, and nothing can filter by date.
**Precedent.** atuin `--before/--after`; Claude Code's picker shows time since last activity.
**Change.**
- New `parse_when(s)`, in a stdlib form only: `2026-07`, `2026-07-14`, `jul`, `7d`, `3w`,
  `yesterday`. Accept `after:`/`before:` tokens in the query; `query_terms` strips them and
  `hit()` checks `mtime`, or the last `timestamp` for ghosts. Mirror them as
  `tbe search --after/--before`.
- In `ago()`, print an absolute date beyond 7 days (`Jul14`); `when()` already formats it.
- Picker key `@`: prompt for a date and move the cursor to the first row at or before it.
  In ungrouped mode, `picker_items` can add `("h", "July 2026")` headers on month change.

### P1-3. Show Claude Code's own titles (S)
**Problem.** Rows show the first prompt or the last paragraph. Claude Code generates a title
for unnamed sessions and lets the owner set one with `/rename` (sessions doc above). These
are in the transcripts as `ai-title`/`custom-title` entries but unused.
**Change.** In `scan()`, read the last `custom-title` (`customTitle`), else `ai-title`
(`aiTitle`), and prefer it to `summary`/`first_user` for `tldr`. Include it in `hit()` meta
so it is searchable. The docs say the entry format is internal and can change, so fall back
silently.

### P1-4. A view toggle: by project or by time (S)
**Problem.** `tbe` always groups by project. The owner's memory is "content + date", which is
a chronological view, and that exists only as `ccsessions` without `-g`.
**Precedent.** atuin cycles filter modes with one key; Claude Code toggles scope with
`Ctrl+A`/`Ctrl+B`.
**Change.** Key `t` in `loop()` returns `("view", None)`, and `main` flips `grouped`. Show
the mode in the title bar.

### P1-5. A retention warning (S)
**Problem.** Sessions disappear silently.
**Change.** New `retention_days()`: read `cleanupPeriodDays` from `~/.claude/settings.json`,
default 30 (documented default). Show it in the `?` overlay and in the ghost section header:
`gone (1054) · Claude Code deletes transcripts after 120 days`. Optional: a footer warning
on frozen sessions within 7 days of the cutoff, since freezing doesn't protect them from the
sweep. The docs don't say how age is measured, so word it as "may be deleted after".

## P2: nice to have

- **Search progress (S).** Picker search reads every transcript. In `main`, before `take()`
  under a filter, draw `searching 989 transcripts…`. Today the screen freezes for the scan.
- **Rank picker search like `tbe search` (S).** When `filt` is set, sort `body_rows` with
  `print_ranked`'s key (score capped, then mtime), so a conversation hit beats a tool-output
  hit. The picker currently lists matches in mtime order.
- **`n`/`N` between hits (S).** Following tig, cycle the preview's `hit` among a session's
  matches. `search_session` keeps only the newest per weight, so keep a short list.
- **A full preview pane (M).** Claude Code uses `Space` to preview and claude-squad uses `tab`
  to switch panes. Key `Space` toggles a full-height preview of `recent(path, n=40)` with
  scrolling.
- **Live state like ccmanager (L, skip for now).** Waiting/Busy/Idle would need hooks or
  transcript-tail parsing. The `●` dot covers most of the need.

## Not recommended

- **Depending on fzf or `zi`-style piping.** Rewriting on fzf would add a binary dependency
  and lose the preview's redaction, the freeze/close keys and the grouped headers. The gaps
  above are query handling and discoverability, which fzf wouldn't fix.
- **Restoring transcripts or keeping copies of them.** That is outside a picker. The owner
  has already raised `cleanupPeriodDays`, and ghosts cover finding sessions.

## Order of work
P0-1 and P0-2 together (one PR, both in `run_picker`/`main`), then P0-3 (`search_session`,
`query_terms`, with tests in `_selftest` for the quote and glyph cases), then P0-4, then the
P1 items in the order listed.
