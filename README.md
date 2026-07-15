# ccpeek

Browse, inspect, and resume your [Claude Code](https://docs.anthropic.com/en/docs/claude-code) sessions with fzf — with a preview that shows **where you left off**, not where you started.

Claude Code saves every session as a JSONL transcript under `~/.claude/projects/`. Closing a terminal tab loses nothing — but *recalling* what each session was, and jumping back into the right one, is the hard part. `ccpeek` is that recall layer:

```
 10:59  my-app          32   143k  can you add retry logic to the webhook handler?
 10:35  dashboards      19   109k  I'm not entirely sure that's the picture. Can you pull…
 10:04  infra            4    37k  create a new keypair and give me the pubkey
 Jul-13 blog             2    23k  hi, can u scp this over to my downloads folder?
─────────────────────────────────────────────────────────────
❯ █                                                     61/61
```

- **List**: every session across all projects, newest first — date, project, dialogue turns, context tokens (green/yellow/red as it approaches compaction), and the last thing you typed.
- **Preview**: the *bottom* of the conversation — your last request and the assistant's final reply — with a pinned header (session id, turn count, token size, opening prompt, last request). Scroll up through the whole chat. Markdown renders properly: tables, headings, bold, code. Tool-call noise collapses into `⋯ 7 tool/think steps`.
- **Enter**: `cd`s to the session's project directory and runs `claude --resume` — you're back in the conversation with full context. Inside tmux, a session belonging to another project opens in *that* project's tmux session (see below).

## Install

Dependencies: `fzf`, `python3` (stdlib only), and the `claude` CLI.

```sh
curl -fsSL https://raw.githubusercontent.com/atbender/ccpeek/main/ccpeek -o ~/.local/bin/ccpeek
chmod +x ~/.local/bin/ccpeek
```

(or clone and symlink — it's a single file.)

## Usage

```sh
ccpeek          # all projects, newest first
ccpeek -l       # only sessions started in the current directory (also: . / --local)
ccpeek --list   # plain TSV to stdout: date, project, msgs, tok, hint, cwd, id, path
```

### Keys

| Key | Action |
|---|---|
| `enter` | resume the session (new claude process, full context) |
| `ctrl-j` / `ctrl-k` | scroll the preview chat down / up |
| `shift-↓` / `shift-↑` | scroll half a page |
| `ctrl-o` | open the full transcript in a pager |
| `ctrl-y` | copy the session id |
| `ctrl-/` | toggle the preview pane |
| type | fuzzy-filter by project name or last message |

`--list` makes it composable:

```sh
ccpeek --list | sort -t$'\t' -k4 -hr | head   # fattest sessions by context size
ccpeek --list | grep -i webhook                # find that session about webhooks
```

## tmux integration

Bind it to open in a new window — the picked session takes over the tab; cancelling closes it:

```tmux
# browse all sessions
bind-key v new-window -n peek "~/.local/bin/ccpeek"
# only the current project's sessions
bind-key C-v new-window -c "#{pane_current_path}" -n peek "~/.local/bin/ccpeek -l"
```

### One tmux session per project

If you keep a tmux session per project (à la [tmux-sessionizer](https://github.com/ThePrimeagen/.dotfiles)), a session you pick while browsing *all* projects belongs in that project's tmux session — not in whatever window you happened to press the key from. Set `CCPEEK_ROOTS` to where your projects live (colon-separated dirs or globs, `~` allowed) to turn that on:

```tmux
set-environment -g CCPEEK_ROOTS "~/dotfiles:~/src/*"
```

Then picking a session from another project creates that project's tmux session if it isn't running, opens Claude in a new window there, and switches you to it. Picking one from the project you're already in takes over the peek window as before. Unset — the default — every pick just takes over the window.

The session is named after the project *root*, since a transcript's cwd is often a subdir of it (`~/src/app/packages/api` → session `app`, window `api`). ccpeek finds the root via the `CCPEEK_ROOTS` entry containing the cwd, falling back to the git toplevel — which matters for nested repos, where the git toplevel alone would give the inner repo a session of its own.

Note `CCPEEK_ROOTS` has to reach a *non-interactive* shell, so `.zshrc`/`.bashrc` won't do it — use tmux's `set-environment` as above, or `.zshenv`.

## How it works

- Reads the JSONL transcripts Claude Code already writes to `~/.claude/projects/<encoded-cwd>/<session-id>.jsonl`. No configuration, no daemon, nothing to maintain — your sessions are already the database.
- An mtime-based index cache (`~/.cache/ccpeek-index.json`) means only changed transcripts are re-parsed; startup is instant even with hundreds of sessions.
- **Turns**, not rows: the msg count collapses consecutive same-side messages, so `you → cc → you → cc` = 4 even when the assistant streamed a dozen interim updates.
- **Tokens** are read from the last assistant message's `usage` (input + cache read/write + output) — the real context size, same number the Claude Code statusline shows.
- Everything stays on your machine; ccpeek only ever reads local files.

## Notes

- Sessions are matched to projects by the directory Claude was *launched* from.
- `-l` matches the exact current directory, so run it from where you normally start `claude`.
- Sessions from before Claude Code recorded usage data show `–` for tokens.

## License

MIT
