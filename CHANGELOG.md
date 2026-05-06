# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/).

## [Unreleased]

### Changed

- README expanded with a "Known limitations" section (preview re-parse cost, Codex prefix-match asymmetry, system-entry visibility) so users don't file them as bugs
- Adapter section header in `ccs.py` carries an attribution + re-vendoring note documenting the source skill (`recall`) and the no-import rationale
- `SessionMeta.locator` and `session_line()` carry inline documentation of the opaque-string contract and the hidden-column dispatch pattern

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
