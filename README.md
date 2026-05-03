# ccs

Multi-source session browser for coding-agent JSONL logs — Claude Code, Codex, Pi, and Opencode in one fzf picker. Type to filter, drill into messages, render with `bat`. Stdlib-only Python.

```bash
ccs                              # current cwd, all available sources
ccs migration plan               # pre-fill fzf query (substring filter on tool/cwd/title/time)
ccs --all                        # scan every cwd across all tools
ccs --source claude              # restrict to one tool (repeat to combine)
ccs --cwd /path/to/project       # specific cwd, all tools
ccs --session ~/.claude/projects/.../foo.jsonl   # open a JSONL file directly
ccs --session opencode://ses_2786e7db…           # open an Opencode session by id
```

**Quick keys (fzf):** `Enter` open · `y` copy session · `Y` copy one msg · `?` in-app help · `Esc` back

## Install

```bash
# Clone
git clone https://github.com/tmsjngx0/ccs.git ~/.local/share/ccs

# Add to PATH (or symlink)
ln -s ~/.local/share/ccs/ccs.py ~/.local/bin/ccs

# Or alias (zsh / bash)
echo 'alias ccs="~/.local/share/ccs/ccs.py"' >> ~/.zshrc
```

That's it. No `pip install`, no virtualenv, no extra packages. `python3` and one optional binary (fzf) are all you need.

## Requirements

Realistically you install **zero or one thing** — `fzf` (optional, recommended). Everything else is Python stdlib or auto-detected with graceful fallback.

| | Tool | Notes |
|---|---|---|
| Required | `python3` (≥3.9) | Stdlib only — no `pip install`. `sqlite3` is bundled. |
| Recommended | [`fzf`](https://github.com/junegunn/fzf) | Fuzzy filter + live preview + key bindings. `brew install fzf` · `scoop install fzf` · `apt install fzf`. **Without it, ccs runs in a stdlib fallback picker** (numbered list, `/text` substring filter, `q` quit). |
| Pager (optional, one of) | `bat` → `$PAGER` → `less` | First one found wins. Plain `less` is fine. |
| Clipboard (optional, one of) | `pbcopy` · `wl-copy` · `xclip` · `xsel` · `clip.exe` | macOS / Wayland / X11 / X11 / WSL+Windows. Your OS already ships one. |

## Platform support

| Platform | Status |
|---|---|
| macOS | ✅ |
| Linux (X11 / Wayland) | ✅ |
| WSL2 on Windows | ✅ |
| Native Windows | ⚠️ Partial — `clip.exe`/`fzf.exe` work, but Claude Code's project-directory encoding hasn't been verified on non-POSIX paths. Use `--all` to bypass cwd filtering. |

## Keybindings

Session picker:

| Key | Action |
|-----|--------|
| `Enter` | Open the message picker for the session |
| `?` | Show the keybinding help screen |
| `Esc` | Quit |

Message picker (after picking a session):

| Key | Action |
|-----|--------|
| `Enter` | Render the message in `bat` |
| `y` | Copy the **whole conversation** (all messages, joined) to clipboard |
| `Y` | Copy **only the highlighted message** to clipboard |
| `?` | Show the keybinding help screen |
| `Esc` | Back to session picker |

Clipboard fallback chain: `pbcopy` → `wl-copy` → `xclip` → `xsel` → `clip.exe`.

## How it works

```
┌────────────────────────────┐
│ adapters discover sessions │      claude / codex / pi / opencode
│  (auto-skip if missing)    │      → SessionMeta records
└─────────────┬──────────────┘
              ▼
┌────────────────────────────┐
│ fzf session picker         │      one row per session, hidden cols carry
│  (substring filter)        │      tool + locator for dispatch
└─────────────┬──────────────┘
              ▼
┌────────────────────────────┐
│ fzf message picker         │      message list w/ live preview
│  (per session)             │
└─────────────┬──────────────┘
              ▼
┌────────────────────────────┐
│ bat pager on selected msg  │
└────────────────────────────┘
```

Each adapter walks its native storage (JSONL files for Claude/Codex/Pi, read-only SQLite for Opencode). The picker uses `fzf`'s built-in interactive filter on a rich row (`tool · time · msgs · size · cwd · title`), so search is just typing.

## Sources

| Tool        | Storage                                          | Format      |
|-------------|--------------------------------------------------|-------------|
| Claude Code | `~/.claude/projects/<encoded-cwd>/*.jsonl`       | JSONL       |
| Codex       | `~/.codex/sessions/YYYY/MM/DD/rollout-*.jsonl`   | JSONL       |
| Pi          | `~/.pi/agent/sessions/<encoded-cwd>/*.jsonl`     | JSONL       |
| Opencode    | `~/.local/share/opencode/opencode.db`            | SQLite (RO) |

Adapters auto-skip when their storage doesn't exist on the machine. Pi `imported-claude-*` files are deduped against Claude.

## Configuration

| Env var                | Default                                                  |
|------------------------|----------------------------------------------------------|
| `CLAUDE_PROJECTS_DIR`  | `~/.claude/projects`                                     |
| `CODEX_HOME`           | `~/.codex`                                               |
| `PI_SESSIONS_DIR`      | `~/.pi/agent/sessions`                                   |
| `OPENCODE_DB`          | `~/.local/share/opencode/opencode.db`                    |

## Notes

Adapter classes are vendored from the [`recall`](https://github.com/tmsjngx0/agent-skills) skill so `ccs` has no runtime dependency on that skill's deploy location. Upstream changes can be re-vendored as needed.

ccs originated inside the [`qs`](https://github.com/tmsjngx0/qs) repo and was extracted into this standalone repo at v0.2.0. Commit history before that point lives here too — `git log --follow ccs.py` shows the full lineage.

## License

MIT
