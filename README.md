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

**Quick keys (fzf):** `Enter` open · `ctrl-y` yank conversation · `alt-y` yank one message · `?` in-app help · `Esc` back

## Install

Three options — pick whichever fits your habits:

```bash
# Option 1 — git clone (no Python packaging needed; self-upgradeable via `ccs --upgrade`)
git clone https://github.com/tmsjngx0/ccs.git ~/.local/share/ccs
ln -s ~/.local/share/ccs/ccs.py ~/.local/bin/ccs

# Option 2 — pipx (isolated install, standard Python tool)
pipx install git+https://github.com/tmsjngx0/ccs.git

# Option 3 — uvx (uv's one-shot runner — try without installing)
uvx --from git+https://github.com/tmsjngx0/ccs.git ccs --version
# permanent install with uv:
uv tool install git+https://github.com/tmsjngx0/ccs.git
```

`npx` does not work — `ccs` is a Python single-file tool, not a Node package. `pipx run` and `uvx` are the Python equivalents for one-shot execution.

`python3` (≥3.9) and an optional `fzf` are all you need at runtime. No `pip install` of dependencies is required for Option 1 (stdlib only).

## For AI agents

`ccs` is built so an AI agent (Claude Code, Codex, etc.) can install and operate it on the user's machine without human intervention. The install commands above are **idempotent and non-interactive** — safe to re-run.

**Prerequisite check** (run before install):

```bash
# Linux / macOS / WSL
for t in git python3; do command -v "$t" >/dev/null 2>&1 || echo "missing: $t"; done

# Windows (PowerShell)
foreach ($t in 'git','python','fzf') { if (-not (Get-Command $t -ErrorAction SilentlyContinue)) { Write-Output "missing: $t" } }
```

If anything is missing, install it via the OS package manager (see [Requirements](#requirements) below). `fzf` is optional — without it, `ccs` falls back to a stdlib numbered picker.

**Agent-friendly commands**:

| Command | Purpose |
|---|---|
| `ccs --version` | Verify install / pre-flight check |
| `ccs --all --source claude` | All Claude sessions across all cwds (one row per line; tab-delimited) |
| `ccs --session <jsonl_path>` | Open a specific session by path |
| `ccs --copy-session claude <jsonl>` | Copy session content (clipboard → OSC52 → tempfile fallback chain) |
| `ccs --upgrade` | In-place upgrade via `git pull` (refuses if uncommitted changes; agent should `git -C <install_dir> status` first) |

Rationale for the agent-first design lives in [ADR-008 (Solo-maintainer capacity drives self-service feature priority)](https://github.com/tmsjngx0/ccs-mgmt/blob/main/docs/decisions/ADR-008-solo-maintainer-capacity-drives-self-service.md) in the `ccs-mgmt` harness.

## Requirements

Realistically you install **zero or one thing** — `fzf` (optional, recommended). Everything else is Python stdlib or auto-detected with graceful fallback.

| | Tool | Notes |
|---|---|---|
| Required | `python3` (≥3.9) | Stdlib only — no `pip install` for Option 1. `sqlite3` is bundled. |
| Required (Option 1) | `git` | Any version; used by Option 1 and by `ccs --upgrade`. |
| Recommended | [`fzf`](https://github.com/junegunn/fzf) **≥ 0.50** | Fuzzy filter + live preview + key bindings. **Without it, ccs runs in a stdlib fallback picker** (numbered list, `/text` substring filter, `q` quit). **Debian / Ubuntu `apt` ships an outdated 0.44.x where ccs's `ctrl-y` / `alt-y` / `?` bindings silently no-op — install the latest binary directly:** `curl -L https://github.com/junegunn/fzf/releases/latest/download/fzf-linux_amd64.tar.gz \| tar xz -C ~/.local/bin/` or use `git clone --depth 1 https://github.com/junegunn/fzf.git ~/.fzf && ~/.fzf/install --bin`. |
| Pager (optional, one of) | `bat` → `$PAGER` → `less` | First one found wins. Plain `less` is fine. |
| Clipboard (optional, one of) | `pbcopy` · `wl-copy` · `xclip` · `xsel` · `clip.exe` | macOS / Wayland / X11 / X11 / WSL+Windows. Your OS already ships one. OSC 52 + tempfile fallback covers SSH sessions. |

**One-line prereq install** (covers `git`, `python3`, `fzf` together):

| OS | Command |
|---|---|
| Linux (Debian/Ubuntu) | `sudo apt install git python3 fzf` |
| Linux (Arch) | `sudo pacman -S git python fzf` |
| Linux (Fedora) | `sudo dnf install git python3 fzf` |
| macOS | `brew install git python fzf` |
| Windows | `winget install Git.Git Python.Python.3.12 junegunn.fzf` |

## Platform support

| Platform | Status |
|---|---|
| macOS | ✅ |
| Linux (X11 / Wayland) | ✅ |
| WSL2 on Windows | ✅ |
| Native Windows | ✅ Supported (v0.4.0+) — `winget` covers all prereqs (see [Requirements](#requirements)). UTF-8 forced on fzf stdin so Korean / emoji session titles render correctly. Caveat: Claude Code's project-directory encoding on non-POSIX paths is unverified — use `--all` to bypass cwd filtering if `--here` returns empty. |

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
| `ctrl-y` | Copy the **whole conversation** (all messages, joined) to clipboard |
| `alt-y` | Copy **only the highlighted message** to clipboard |
| `?` | Show the keybinding help screen |
| `Esc` | Back to session picker |

Modifier-prefixed (`ctrl-y` / `alt-y`) instead of plain letters because fzf treats every unbound letter as input to its type-to-search query — a binding on plain `y` / `Y` would block searching for words containing those letters (e.g. "**y**esterday" in a message).

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

Each adapter checks its storage path at startup and silently skips when the path is missing — no flags, no config files. Install one of the four tools and ccs picks it up; uninstall and ccs stops listing it.

## Known limitations

These are intentional trade-offs documented so you don't file them as bugs:

- **Preview re-parses the full session JSONL on every fzf highlight.** Acceptable for typical session sizes; an LRU cache or sidecar message-index would be a real refactor, not a quick patch.
- **Codex sessions match cwd by prefix; Claude / Pi / Opencode match exactly.** Documented asymmetry inherited from the upstream `recall` adapter — Codex stores rollouts under date directories rather than per-cwd.
- **System entries appear in the message list.** `attachment`, `permission-mode`, `file-history-snapshot` and similar are visible alongside user/assistant turns. Filter via fzf substring (`user` / `assistant`) when you want only conversational content. A `--user-only` flag would be a feature, not a bug fix.

## Notes

Adapter classes are vendored from the [`recall`](https://github.com/tmsjngx0/agent-skills) skill so `ccs` has no runtime dependency on that skill's deploy location. Upstream changes can be re-vendored as needed.

ccs originated inside the [`qs`](https://github.com/tmsjngx0/qs) repo and was extracted into this standalone repo at v0.2.0. Commit history before that point lives here too — `git log --follow ccs.py` shows the full lineage.

## License

MIT
