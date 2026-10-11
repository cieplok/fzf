# Add `shift-space` key name for the Kitty keyboard protocol (`CSI 32;2u`)

- Date: 2026-10-11
- Category: feature
- Priority signal: low — niche; only terminals with the Kitty keyboard protocol force-enabled emit `\e[32;2u` (stock kitty/foot/wezterm do not), and nothing is blocked without it.
- Risk signal: medium — a single-area, additive change to the input parser, unit-tested, but it touches a freshly rewritten CSI parser (see Non-goals) and the generated `eventtype_string.go`.
- Routing: `Next: om-auto-create-pr "Add shift-space key name (Kitty CSI 32;2u) — brief: .ai/specs/briefs/2026-10-11-shift-space-key-name.md"`

## Problem

fzf accepts named key chords such as `shift-tab`, `shift-up`, and `shift-delete`, but there is no `shift-space` name, so `printf 'a\n' | fzf --bind 'shift-space:accept'` exits 2 with `unsupported key: shift-space`. Terminals that force-enable the Kitty keyboard protocol (e.g. wezterm's `enable_kitty_keyboard`) emit Shift+Space as CSI `32;2u`; fzf's parser currently drops that sequence as an unimplemented CSI-u key, so the chord is silently ignored even without `--bind`. Evidence: upstream issue #4916 (exact bytes and a pty reproduction); `shift-space` appears nowhere in `src/`.

## Agreed direction

Add a `shift-space` key name that resolves to a new `ShiftSpace` event, and accept **only** the `\e[32;2u` sequence for it in the CSI parser — leaving every other CSI-u key dropped exactly as today. This mirrors how the existing `shift-*` names are wired (`case "shift-delete": add(tui.ShiftDelete)` in `src/options.go`).

"Build nothing" was rejected: the chord is genuinely unbindable for affected users, and the stated goal is to exercise the pipeline end-to-end on a real, gate-verifiable change. A docs-only workaround ("disable the Kitty protocol") was rejected as it leaves a supported terminal configuration broken.

## Resolved unknowns

| Question | Answer (from the conversation) |
|----------|--------------------------------|
| Scope: special-case vs general CSI-u support | Narrow. Accept only `\e[32;2u` as `shift-space`; all other CSI-u keys stay `Invalid`. General CSI-u support is a non-goal. |
| Does the existing `shift-*` handling transfer directly? | No. `shift-tab`/`shift-up`/`shift-delete` arrive via xterm/rxvt modified-key encodings fzf already parses; shift-space needs the CSI-u family, which fzf deliberately rejects today. Do not copy the issue's "same as the other shift chords" framing. |
| tcell path | tcell currently drops the Shift modifier on space (`src/tui/tcell.go`, `KeyRune ' '` → `default` → plain `Rune`). Treat the tcell branch as **verify-or-non-goal**: confirm whether tcell surfaces `ModShift` on space; if it does not, do not add dead code — record it as a non-goal. |
| Generated file | Adding an `EventType` constant breaks the guard in `src/tui/eventtype_string.go` until it is regenerated with `stringer`. `stringer` is **not installed** locally and `make generate` is not in the validation gate; regenerate (e.g. `go run golang.org/x/tools/cmd/stringer@latest -type=EventType`) and commit the result. Never hand-edit the generated file. |
| QA | `skip-qa` with reason: a CSI-u keystroke is a TUI input path, not reachable by the browser provider, and the tmux integration harness does not emit CSI-u. Correctness is covered by the Go unit tests added here. |
| Pipeline base branch | The setup commit (`57d402a0`, config + docs) must be on the branch the run targets. Prerequisite: push it to `origin/master` before starting the run so the worktree carries `.ai/agentic.config.json`. |

## Non-goals

- Implementing general CSI-u / Kitty keyboard-protocol key support.
- Changing the parsing or naming of any existing `shift-*` or other key chord.
- Enabling the Kitty keyboard protocol from fzf.
- A tcell branch that cannot be reached (see the verify-or-non-goal row).

## Affected areas

- `src/tui/tui.go` — add the `ShiftSpace` event constant (near `CtrlSpace`).
- `src/tui/eventtype_string.go` — regenerate via `stringer`.
- `src/options.go` — `case "shift-space": add(tui.ShiftSpace)` in `parseKeyChords`.
- `src/tui/light.go` — accept `\e[32;2u` as `ShiftSpace` in the CSI-u handling.
- `src/tui/tcell.go` — verify-or-non-goal Shift-on-space mapping.
- `src/options_test.go`, `src/tui/light_test.go` — unit tests (name parses; sequence maps to `ShiftSpace`; other CSI-u keys still `Invalid`).
