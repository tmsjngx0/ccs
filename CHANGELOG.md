# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/).

## [Unreleased]

## [0.5.1] - 2026-05-13

### Fixed (documentation)

- README now states **fzf ≥ 0.50 is required** for the `ctrl-y` / `alt-y` / `?` bindings to fire. Older fzf (notably `0.44.x` shipped by Debian / Ubuntu `apt`) silently accepts the binding strings but no-ops at runtime — yank shows the change-header confirmation but doesn't actually copy, and `?` doesn't open the help pager. Field-reported during v0.5.0 dogfood (astra on `fzf 0.44.1 (debian)` was broken; stella on `fzf 0.68.0 (Homebrew)` worked end-to-end). Install instructions added: direct binary download or `~/.fzf/install --bin`, both bypass the stale apt package
- No code change — pure docs / version bump to surface the constraint at install time

## [0.5.0] - 2026-05-13

### Changed (breaking)

- **Yank keybindings remapped: `y` → `ctrl-y`, `Y` → `alt-y`.** fzf treats every unbound letter as input to its type-to-search query, so a binding on plain `y` / `Y` blocked searching for words containing those letters (e.g. "**y**esterday" in a message). Modifier-prefixed bindings keep search free. `?` remains as the help key (universal CLI convention; `?` is rarely needed in search). Users who scripted `--copy-session` / `--copy-message` directly are unaffected — only the fzf bindings changed
- README's Quick keys line, the Message picker keybinding table, the in-app `?`-help text, and the messages-picker banner all updated to reflect the new bindings

### Fixed

- In-app `?`-help could appear invisible when invoked from the message picker — fzf's inline mode (`--height=80%`) hosts the pager output in the same screen region fzf redraws, and a user `$LESS` env containing `-F` (quit on short) or `-X` (no alternate screen, common in oh-my-zsh / prezto defaults) made `less` either flash invisibly or sit underneath fzf's redraw. `show_help_keys` now overrides `$LESS` to `-R` (raw control chars only) when invoking the pager, so the help text consistently uses the alternate screen and waits for `q`

## [0.4.5] - 2026-05-12

### Fixed

- Banner stacked above the fzf window as the user moved between the session picker and the message picker. fzf's inline mode (`--height=80%`) only redraws its own bottom-80% region; the banner zone above belongs to `ccs`, and every `print_picker_banner` call wrote a fresh line at the current stderr cursor position instead of replacing the previous banner. Now `print_picker_banner` emits `\x1b[H\x1b[2J` (cursor home + clear entire screen) before writing — fzf redraws its inline region cleanly below on the next launch, and the banner zone shows only the current picker's cheatsheet

## [0.4.4] - 2026-05-12

### Fixed

- Misleading copy-confirmation message in the message picker. After `y` / `Y` the header read `✓ Copied … — see stderr for path/method`, but `execute-silent` redirects both stdout and stderr to `/dev/null`, so there was no stderr for the user to consult. The message taught a lie. Replaced with `✓ Copied … — paste to verify` — actionable and honest. To inspect the actual fallback path/method (which clipboard tool succeeded, whether OSC 52 was sent, where the tempfile backstop landed), invoke `ccs --copy-session <tool> <locator>` directly from a regular shell where stderr is visible

## [0.4.3] - 2026-05-12

### Fixed

- `?`-help binding was broken: pressing `?` in either picker showed nothing instead of opening the help screen. Root cause: the v0.3.1 banner-reprint chain (`?:execute(help_cmd)+execute-silent(reprint_cmd)`) was added on a faulty premise — it assumed fzf uses an alternate screen, but ccs runs fzf in inline mode (`--height=80%`), where the banner above fzf's window naturally survives `?`-help round-trips because fzf only redraws its own inline region. The reprint chain raced with fzf's redraw and corrupted the help display. Reverted the chain — bindings are now just `?:execute(help_cmd)`. The banner stays visible without help. The hidden `--reprint-banner` subcommand is retained (no harm) in case a future binding pattern needs it

## [0.4.2] - 2026-05-12

### Changed

- `--upgrade` now actually upgrades from a git submodule / worktree, instead of refusing with a "wrong checkout type" prescription (the v0.4.1 behavior). Implementation switches from `git pull --rebase --tags` to `git fetch origin main --tags` + `git reset --hard origin/main`, which works identically across standalone clones and detached-HEAD submodules/worktrees. The dirty-tree guard is preserved (uncommitted changes still refuse). When the install dir is a submodule, the success message now points at the follow-up step the user owns: bumping the parent repo's submodule pointer. Reasoning: ADR-008's "prescription, not diagnosis" principle applies to actions too — when the action is unambiguous and the user has typed `--upgrade`, do the work

## [0.4.1] - 2026-05-12

### Fixed

- `--upgrade` falsely refused with "is not a git checkout" when `ccs.py` lives inside a git submodule or worktree (where `.git` is a file pointer, not a directory). Detection now uses `Path.exists()` plus a `Path.is_file()` branch that prints a submodule-specific prescription: run `git submodule update --remote` from the parent repo and commit the bumped pointer, instead of letting `git pull` fail downstream with a confusing "not currently on a branch" error

## [0.4.0] - 2026-05-12

### Added

- `pyproject.toml` (hatchling, single-module pattern) — enables `pipx install git+...`, `uvx --from git+... ccs`, and `uv tool install git+...`. `ccs.py` stays as a single file at the repo root; the wheel build packs it as the top-level `ccs` module so import paths work after install. Rationale: ADR-009 in `ccs-mgmt`. `__version__` in `ccs.py` is the single source of truth — hatchling reads it dynamically
- `For AI agents` section in README — prerequisite-check commands (bash + PowerShell), idempotent install instructions, and a table of agent-friendly invocations (`--version`, `--all --source claude`, `--copy-session`, `--upgrade`). Cross-references ADR-008 (capacity-driven self-service)
- One-line OS prerequisite install matrix in Requirements — `apt` / `pacman` / `dnf` / `brew` / `winget` commands covering `git`, `python3`, `fzf` together. The `winget install Git.Git Python.Python.3.12 junegunn.fzf` line is the most important new piece — Windows native is now a one-command bootstrap

### Changed

- **Native Windows platform support raised from Partial to Supported.** `winget` covers all prereqs end-to-end; the UnicodeEncodeError fix below removes the last blocker for Korean / emoji session titles. Caveat retained: Claude Code's project-directory encoding on non-POSIX paths is unverified — `--all` works as the bypass

### Fixed

- `UnicodeEncodeError` on Windows when piping fzf's stdin and other text-mode subprocess calls containing non-ASCII content (Korean session titles, emoji, OSC 52 base64). Python's `subprocess.run` defaulted to the legacy ANSI code page (`cp1252` on US Windows); `text=True` is now paired with explicit `encoding="utf-8"` at four call sites: fzf launcher, clipboard tool dispatch, in-app `?`-help pager, and `git status --porcelain` parse inside `--upgrade`

## [0.3.1] - 2026-05-12

### Added

- `--upgrade` flag runs `git pull --rebase --tags` in the install directory. Refuses to run if the install dir isn't a git checkout or has uncommitted changes — prevents the `git stash pop` conflict-marker scenario seen in v0.3.0 field reports where merge markers ended up inside `ccs.py` and broke the script with a `SyntaxError`

### Fixed

- Shortcut banner above the fzf window no longer disappears after `?`-help round-trip. `?` binding now chains an `execute-silent` that re-emits the banner through the new hidden `--reprint-banner SCOPE [TOOL]` subcommand, bypassing `print_picker_banner`'s same-content cache. Affects both the session picker and the message picker

## [0.3.0] - 2026-05-12

### Added

- OSC 52 clipboard escape sequence support for SSH sessions — `y` / `Y` now copy through the SSH tunnel via terminal OSC 52, with `/dev/tty` as the primary channel and stderr as fallback (matches neovim's implementation)
- Tempfile backstop written to `/tmp/ccs-copy-XXXX.md` (mode 0600) when no clipboard utility succeeds — the path is printed so the user can `cat` / `scp` it manually
- `--version` flag prints the current ccs version (sourced from `__version__` in `ccs.py`)

### Changed

- README expanded with a "Known limitations" section (preview re-parse cost, Codex prefix-match asymmetry, system-entry visibility) so users don't file them as bugs
- Adapter section header in `ccs.py` carries an attribution + re-vendoring note documenting the source skill (`recall`) and the no-import rationale
- `SessionMeta.locator` and `session_line()` carry inline documentation of the opaque-string contract and the hidden-column dispatch pattern

### Fixed

- `pbcopy` / `clip.exe` are skipped when `$SSH_CONNECTION` is set — they would silently copy to the *remote* host's pasteboard instead of the user's actual terminal
- OSC 52 sequences now use the `ST` terminator (`\e\\`) and have no Base64 size cap, matching neovim's implementation (the earlier `BEL` terminator + cap rejected large payloads silently in some terminals)
- OSC 52 writes to `/dev/tty` first so the sequence reaches the terminal even when stdout is piped; falls back to stderr if `/dev/tty` is unavailable

### Removed

- `CLAUDE.md` — agentic guidance moved upstream to the `ccs-mgmt` harness per its ADR-007 (submodules carry only user-facing docs). Architectural decisions previously narrated here are now captured as harness ADRs (vendoring, fzf-not-index, 2-tier UX); user-relevant content was absorbed into `README.md`

## [0.2.0] - 2026-05-03

First standalone release. `ccs` originated in the [qs](https://github.com/tmsjngx0/qs) repo and was extracted into this dedicated repo at this version. The version starts at 0.2.0 (not 0.1.0) because the work captured here corresponds to qs's `[Unreleased]` block plus subsequent ccs polish — qs `[0.1.0]` was the initial release of the `qs` Bash tool, never a `ccs` release. Use `git log --follow ccs.py` to see the file's full history dating back to the multi-source feat commit.

### Added

- Multi-source session browser across Claude Code, Codex, Pi, and Opencode in one fzf picker
  - Adapter pattern (`ClaudeAdapter`, `CodexAdapter`, `PiAdapter`, `OpencodeAdapter`) vendored from the [`recall`](https://github.com/tmsjngx0/agent-skills) skill so lifecycles stay decoupled
  - Opencode read-only SQLite support via `opencode://<session_id>` locators
  - Free-text positional argument pre-fills fzf's interactive filter (e.g. `ccs migration plan`)
  - `--source`, `--all`, `--cwd`, `--session` flags; storage roots overridable via `CLAUDE_PROJECTS_DIR` / `CODEX_HOME` / `PI_SESSIONS_DIR` / `OPENCODE_DB`
  - Dedup of Pi `imported-claude-*` sessions against Claude
- fzf keybindings: `y` copies the whole conversation, `Y` copies one highlighted message, `?` opens an in-app help screen
- Stdlib fallback picker for fzf-less environments — numbered list, `/text` substring filter, `q` quit, `?` minimal help
- `--copy-session`, `--copy-message`, `--help-keys` hidden CLI flags powering the fzf bindings
- Clipboard fallback chain: `pbcopy` → `wl-copy` → `xclip` → `xsel` → `clip.exe` (covers macOS / Wayland / X11 / WSL)
- Friendly hint with per-OS install commands when `fzf` is missing from PATH

### Changed

- Two-tier UX (session → message picker) replaces the earlier three-tier project drill-down; cwd is shown as a column instead of a separate stage
- Render functions unified into a single pretty-printer for consistent JSON / multi-line / tool-result rendering across adapters
- fzf layout switched to `--layout=reverse` so the prompt sits at the top and content flows downward (the rg/fzf, git log fzf convention)
- Shortcut cheatsheet floats above fzf as a full-width banner via `--height=80%` inline mode instead of being squeezed into the narrow left pane through `--header`

### Fixed

- `shutil.which` now used directly so PATHEXT (`.exe` / `.bat` / `.cmd`) resolution works on native Windows — `fzf.exe` and `bat.exe` were silently invisible before
- Banner no longer re-prints on every fzf re-launch (e.g., after each `bat` pager exit in the message picker loop) — module-level cache makes `print_picker_banner` idempotent across iterations
