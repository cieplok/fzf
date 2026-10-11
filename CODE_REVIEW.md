# Code review

What reviewers check in this repository, on top of the general `om-code-review` checklist. Read together with `AGENTS.md` (layout and routing) and `BACKWARD_COMPATIBILITY.md` (protected surfaces).

## Priorities

1. **Correctness of matching and event flow.** fzf's core value is the fuzzy match and the reader/matcher/terminal pipeline. A scoring or ordering change needs a test that pins the new behavior, and any change to an assembly/`unsafe` fast path in `src/algo` must keep the reference implementation and the fuzz/equivalence tests (`fastpath_equiv_test.go`, `runeprefilter_test.go`) green.
2. **Concurrency safety.** The engine runs multiple goroutines. New shared state must go through `src/util` primitives (`atomicbool`, `eventbox`, `concurrent_set`, `slab`) or a mutex; a review finding for an unsynchronized read/write is a blocker. `go test -race` is worth running for engine changes even though it is not in the gate.
3. **CLI / script contract.** Flags, `--bind` actions, environment variables, shell integration, and the Vim plugin are the product's public API. A change to any of them must update `man/man1/fzf.1` and the README in the same PR, and is judged against `BACKWARD_COMPATIBILITY.md`.
4. **Security.** fzf intentionally runs shell commands (`--preview`, `--bind …:execute(...)`, `$FZF_DEFAULT_COMMAND`); that is by design and not a finding. Real findings are: injection introduced into the shell integration scripts, an auth bypass in the `--listen` HTTP server (`src/server.go`, `FZF_API_KEY` must gate non-localhost access and every request), or unsafe handling of `FZF_API_KEY`/env values. `shell/*` is sourced into a user's shell — quoting and eval safety matter there.

## Repo-specific checks

- **Formatting and lint.** `gofmt -s` clean (`make lint` fails otherwise). Ruby files under `test/` must pass `rubocop` (rubocop-minitest, rubocop-performance) and shell scripts must be in sync with their embedded copies (`shell/update.sh --check`).
- **Generated files are not hand-edited.** `src/actiontype_string.go` and `src/eventtype_string.go` come from `stringer` via `make generate`; edit the source constants and regenerate. The embedded shell scripts are produced by `shell/update.sh` — edit the source script and regenerate, never the embedded copy alone.
- **Platform parity.** Changes touching `_unix`/`_windows` files or build-tagged code must keep both platforms compiling and behaviorally equivalent; note in the PR what was verified and on which platform.
- **Tests.** Behavior changes need a Go unit test (`*_test.go` next to the code). End-to-end behavior changes should extend the Ruby suite in `test/`; call out if integration tests were not run locally and why.
- **Docs in lockstep.** New flag → man page + README. New plugin option → `doc/fzf.txt` + `README-VIM.md`. Version-sensitive release files are asserted by `make prerelease`.
- **No build artifacts.** `bin/fzf`, `target/`, `dist/`, `tmp/` are gitignored; reject a diff that adds a binary.

## Severity guidance

- **Blocker** — data race, broken match ordering without a pinning test, fast-path/reference divergence, an auth bypass in `--listen`, a public-contract break without the `BACKWARD_COMPATIBILITY.md` path, unformatted Go, a red required check.
- **Major** — a flag/action/env change missing its man-page and README update, missing tests for a behavior change, `_unix`/`_windows` drift, hand-edited generated file.
- **Minor** — style nits beyond `gofmt`, comment wording, a missing benchmark for a perf-only change.

A `risk-high` change here is one touching the `--listen` HTTP server, auth, or the shell-integration scripts sourced into users' shells; it needs the extra scrutiny in `SDLC.md` (a second reviewer, no self-QA).
