# Backward compatibility

The surfaces below are what users and downstream integrations depend on. `om-code-review` and the implementation skills check a change against this file; a change that breaks a surface here without the required path is a **blocker**.

## CLI flags and documented behavior

- **Surface.** Every option documented in `man/man1/fzf.1` and the README, including the `--bind` action/event grammar and the `$FZF_DEFAULT_OPTS` option set. The man page is the authoritative contract.
- **Breaking.** Removing or renaming a flag, changing a flag's meaning or default output shape, or making an option that accepted a value now require a different form.
- **Required path.** Keep the old flag working (alias it) for at least one minor release, mark it deprecated in `man/man1/fzf.1` and the README, and note the removal in `CHANGELOG.md`. A behavior change to an existing flag is a breaking change unless it only fixes clearly-buggy output.

## Exit status codes

- **Surface.** `0` normal exit, `1` no match, `2` error, `130` interrupted (documented under EXIT STATUS in `man/man1/fzf.1`).
- **Breaking.** Reusing a code for a different meaning, or returning `0` where a script relies on `1`/`2`.
- **Required path.** Do not change. A new condition must use a previously-unused code and be documented in the man page.

## Environment variables

- **Surface.** The `FZF_*` variables the binary reads and the shell integration exports (`FZF_DEFAULT_COMMAND`, `FZF_DEFAULT_OPTS`, `FZF_DEFAULT_OPTS_FILE`, `FZF_TMUX`, `FZF_TMUX_OPTS`, the `FZF_CTRL_T_*` / `FZF_ALT_C_*` / `FZF_CTRL_R_*` families, `FZF_API_KEY`, and the completion variables — see `man/man1/fzf.1`).
- **Breaking.** Removing a variable, changing its accepted format, or changing its precedence relative to flags.
- **Required path.** Accept both old and new forms for a deprecation window; document the change in the man page and `CHANGELOG.md`.

## `--bind` actions and events

- **Surface.** Action names and event names in `src/actiontype_string.go` (generated from the `actionType` constants in `src/options.go`) and documented in the `--bind` section of the man page.
- **Breaking.** Renaming or removing an action/event, or changing an action's argument syntax.
- **Required path.** Keep the old name as an alias, regenerate `actiontype_string.go` (`make generate`), update the man page and README, and note the deprecation.

## Shell integration scripts

- **Surface.** `shell/completion.bash|zsh|fish|nu` and `shell/key-bindings.bash|zsh|fish|nu` — the functions and key bindings they install into a user's interactive shell — plus `bin/fzf-tmux` and `bin/fzf-preview.sh` and their documented flags.
- **Breaking.** Renaming or removing a user-facing shell function or key binding, changing a documented flag of `fzf-tmux`/`fzf-preview.sh`, or changing the trigger variable semantics (`FZF_COMPLETION_TRIGGER`, etc.).
- **Required path.** Keep the old binding/function working (or document a migration in `CHANGELOG.md` and the man page), and regenerate the embedded copies with `shell/update.sh` so `make lint` stays green.

## Vim / Neovim plugin API

- **Surface.** The public commands, functions, and options of `plugin/fzf.vim`, documented in `doc/fzf.txt` and `README-VIM.md`.
- **Breaking.** Renaming or removing a `:FZF*` command, a public function, or a plugin option, or changing an option's default in a way that alters existing setups.
- **Required path.** Keep a deprecated alias for a release, update `doc/fzf.txt` and `README-VIM.md` together, and note it in `CHANGELOG.md`.

## `--listen` HTTP server API

- **Surface.** The request/response protocol of `fzf --listen` (`src/server.go`): the accepted `POST` action bodies (e.g. `change-query(...)`), response status codes (`200`/`400`/`401`/`503`), and `FZF_API_KEY` authentication (required for a non-localhost address, sent via the `x-api-key` header).
- **Breaking.** Changing an accepted action string, a response status meaning, or weakening/altering the `FZF_API_KEY` requirement.
- **Required path.** Treat as a network protocol: add new forms alongside old ones, never weaken the auth requirement, and document changes in the `--listen` section of the man page. Any change here is `risk-high` (see `SDLC.md`).

## Installers

- **Surface.** `install` (POSIX) and `install.ps1` (Windows) — the documented command-line options and the files they place.
- **Breaking.** Changing the meaning of a documented installer flag or where the binary/completions are installed.
- **Required path.** Keep old flags working, update the man page/README and `CHANGELOG.md`, and keep the version strings asserted by `make prerelease` consistent.
