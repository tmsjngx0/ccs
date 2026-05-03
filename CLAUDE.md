# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repo intent

`ccs.py` — Python (stdlib only). Multi-source coding-agent session browser for Claude Code, Codex, Pi, and Opencode session logs.

ccs originated in the [`qs`](https://github.com/tmsjngx0/qs) repo and was extracted into this dedicated repo at v0.2.0. It does not import from qs and intentionally does not share a release cadence — qs is markdown KB search, ccs is session browsing, and the two were never meant to evolve together.

## Running

No build, no test suite, no lint config. The script runs directly.

```bash
./ccs.py                       # current cwd, all available sources
./ccs.py <query>               # pre-fill fzf interactive filter
./ccs.py --all                 # scan every cwd
./ccs.py --source claude       # restrict to one tool (repeat to combine)
./ccs.py --session <path>      # open JSONL or opencode://<id> directly
```

External dependencies: `fzf` (recommended; stdlib fallback picker covers fzf-less environments). Optional: `bat` (falls back to `$PAGER` / `less`). Stdlib `sqlite3` for Opencode.

## ccs.py architecture

Reading `ccs.py` end-to-end is reasonable, but four patterns are non-obvious from any single function:

### Adapter pattern (vendored)

`Adapter` subclasses (`ClaudeAdapter`, `CodexAdapter`, `PiAdapter`, `OpencodeAdapter`) each own their tool's storage format. Discovery yields `SessionMeta` records; `messages(locator)` yields `MessageRecord` records. The `ADAPTERS` registry maps tool name → class, and `adapter_for_tool()` instantiates on demand.

The adapter classes are **vendored from `~/.agents/skills/recall/scripts/recall-day.py`**, not imported. Lifecycle decoupling is intentional — that file is managed by an external skills deployer. When upstream gains capability worth pulling in, re-vendor by hand; do not introduce an import path or `sys.path` hack to that location.

### Locator as a string

`SessionMeta.locator` is a plain `str` — file path for JSONL adapters, `opencode://<session_id>` for Opencode. Every code path that consumes a locator (preview, message extraction, direct-open via `--session`) treats it as opaque and lets the adapter parse it. Do not generalize to `Path` or a `(scheme, value)` dataclass — that breaks Opencode and adds boilerplate.

### fzf hidden-column dispatch

`session_line()` emits 9 tab-delimited columns. fzf is launched with `--with-nth 4..` (display from col 4) but the returned selection still contains all 9. Cols 1–3 carry `tool` / `locator` / `session_id_full` for dispatch; the visible cols are formatted for readability and substring filtering. `_self_invoke()` is how `--preview-session`, `--preview-message`, `--copy-session`, `--copy-message`, and `--help-keys` re-enter the same script — fzf preview and bindings run as fresh processes with no module state.

If you change the column layout, update both `session_line()` *and* the `--with-nth` value, and verify `{1}`/`{2}` placeholders in the preview/copy commands still point at the right field.

### Search is fzf, not an index

Substring-on-rich-row via fzf is the search experience. Do not add a BM25 layer, do not pre-scan message bodies into a search index, do not add an `--search` flag that does content-grep across all sessions. If filtering on title/cwd/tool/timestamp is insufficient for a real use case, surface the gap before building infrastructure.

### Two-tier UX

Session picker → message picker → bat pager. The previous (pre-extraction) version had a three-tier project-first drill-down; that was removed because Claude/Codex/Pi/Opencode each define "project" differently. cwd is now a column on the session row, not a separate stage. Keep it that way.

### Banner above fzf, not in --header

The shortcut cheatsheet lives in a stderr banner printed *above* fzf via `--height=80%` inline mode, not in `--header`. Reason: with the preview pane at 65% the left list pane is too narrow for the full key list, so `--header` truncates mid-glyph. `print_picker_banner` caches the last-printed banner so it stays idempotent across the picker's loop iterations (without the cache, every `bat` exit re-prints and stacks the banner downward).

## Verifying changes to `ccs.py`

There is no test suite. After non-trivial edits, run this in-process probe — it catches adapter regressions without needing fzf:

```bash
python3 -c "
import ccs
for name, cls in ccs.ADAPTERS.items():
    a = cls()
    if not a.available():
        print(f'{name}: skipped'); continue
    sessions = list(a.discover(cwd_filter=None, all_projects=True))
    print(f'{name}: {len(sessions)} sessions')
    if sessions:
        s = next((x for x in sessions if x.user_msg_count >= 2), sessions[0])
        msgs = a.messages(s.locator)
        print(f'  first session {s.session_id} → {len(msgs)} messages')
"
```

For preview round-trip (still without fzf):

```bash
python3 ccs.py --preview-session claude <path/to/some.jsonl> | head
python3 ccs.py --preview-message claude <path> <index>
```

For clipboard/help round-trip (no fzf needed):

```bash
python3 ccs.py --copy-session opencode 'opencode://<id>' && wl-paste | head
python3 ccs.py --help-keys messages
```

The actual fzf TUI cannot be exercised non-interactively. Final sign-off requires a real terminal.

## Storage roots (env-overridable)

`ccs.py` reads from `~/.claude/projects`, `~/.codex/sessions`, `~/.pi/agent/sessions`, `~/.local/share/opencode/opencode.db`. Override via `CLAUDE_PROJECTS_DIR`, `CODEX_HOME`, `PI_SESSIONS_DIR`, `OPENCODE_DB`. Adapters auto-skip when their storage is missing — don't add availability flags or config files.

## Known limitations (do not "fix" without discussion)

- Preview re-parses the full JSONL on every fzf highlight. Acceptable; an LRU cache or message-index sidecar is a real refactor, not a quick patch.
- Codex uses cwd prefix-match; Claude/Pi/Opencode are exact-match. Documented asymmetry inherited from `recall-day.py`.
- System entries (`attachment`, `permission-mode`, `file-history-snapshot`) appear in the message list. Users filter via fzf substring (`user`/`assistant`). A `--user-only` flag would be a feature, not a bug fix.
- Native Windows: `clip.exe` and `fzf.exe` work, but Claude Code's project-directory encoding hasn't been verified on non-POSIX paths. Use `--all` to bypass cwd filtering until that's tested.
