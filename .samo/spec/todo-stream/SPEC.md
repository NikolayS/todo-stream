# todo-stream — SPEC v0.1

## 1. Goal & Why It's Needed

**Goal:** Ship a single-binary CLI, `todo-stream`, that walks a directory tree, extracts `TODO` / `FIXME` / `HACK` (and user-configurable) comment markers from source files, enriches each finding with git blame metadata (author, commit date, SHA, line), and emits the findings as a machine-readable stream (JSON Lines by default, TSV, or Markdown).

**Why it's needed:** Engineering teams accumulate in-code debt markers that rot silently — nobody knows who wrote them, when, or why, and they never surface in code review or CI. Existing options are inadequate:

- `grep -rn TODO` gives you hits but zero provenance (no author, no age, no aggregation).
- IDE plugins (VS Code "TODO Tree", JetBrains) are per-developer, not CI-friendly, not scriptable, not portable.
- Full linters (golangci-lint, eslint) have `godox`/`no-warning-comments` rules but they fail builds on *any* hit; teams disable them rather than triage.
- Issue trackers (Jira, Linear) diverge from code reality — TODOs in code outlive their tracker entries.

`todo-stream` fills the gap: a **linter-style, CI-friendly, provenance-enriched** view of in-code debt that is trivially pipeable into dashboards, Slack digests, PR comments, or issue-backfill scripts. It is **not** a todo-list app, **not** a CRUD store, **not** a task manager — it is a read-only reporter over source + git history.

## 2. Scope

### In scope (v0.1)
- Recursive directory walk with include/exclude globs (defaults aligned with common source trees).
- Comment-marker extraction for `TODO`, `FIXME`, `HACK` (configurable list, case-insensitive by default).
- Per-finding git blame enrichment: author name, author email, commit SHA, commit ISO-8601 date, line number, original line text.
- Output formats: **JSON Lines (default, streaming)**, **TSV**, **Markdown table/summary**.
- Respect `.gitignore` automatically; honor an optional `.todo-stream-ignore` for finding-level suppression (a plain-text baseline file).
- Exit codes suitable for CI: `0` no findings, `1` findings present, `2` tool error.
- Single static Go binary distributed via GitHub Releases + Homebrew tap; an npm wrapper package republishes the same binary under `todo-stream` so npm-centric repos can `npx todo-stream`.

### Out of scope (v0.1)
- Interactive TUI, watch mode, LSP server, web UI.
- Writing to issue trackers (Jira/Linear/GitHub Issues) — punted to v0.2 as an optional sink.
- Language-aware AST parsing — v0.1 uses lexical comment detection with per-language comment-syntax rules, not a full parser.
- Cross-repo aggregation / daemon mode.

### Reinterpretation of interview answers
The interview elicited answers consistent with a "todo-list app," which conflicts with the stated Idea (a linter for in-code markers). The SPEC honors the Idea and re-maps the interview answers as follows, all of which the user can overturn in the next round:
- `storage-backend: plain-text` → the `.todo-stream-ignore` baseline file (plain text), not a todo persistence store.
- `streaming-model: append-only event log` → the JSON-Lines output stream, one finding per line, append-only in the sense that the tool never mutates source.
- `core-commands: add / list / done` → re-mapped to `scan` (list), `baseline add` (accept a finding into baseline = "add to ignore"), `baseline prune` (drop entries whose underlying TODO is gone = "done").
- `implementation-language: Go static binary` → **accepted as-is**, overriding the Bun hint in the Idea. Go gives a smaller, dependency-free binary, native git integration via `os/exec`, and trivial cross-compilation.
- `tui-scope: pure CLI, JSON/TSV` → **accepted as-is**.

## 3. User Stories

1. **CI gate (persona: platform engineer Priya).** As a platform engineer wiring a new repo's CI, Priya runs `todo-stream --format jsonl --baseline .todo-stream-ignore` in a GitHub Action so that **any new TODO/FIXME added in a PR fails the build, while grandfathered ones stay green**, giving her a ratchet against debt growth without a big-bang cleanup.
2. **Weekly debt digest (persona: tech lead Tomás).** As a tech lead of a 12-person team, Tomás cron-runs `todo-stream --format markdown --group-by author` against the main branch so that **every Monday he gets a Markdown table of open in-code TODOs grouped by author with commit age**, which he pastes into Slack to keep debt visible without nagging people individually.
3. **Onboarding map (persona: new hire Nadia).** As a new hire joining a 200k-LOC codebase, Nadia runs `todo-stream --format jsonl | jq 'select(.marker == "HACK")'` to **surface known rough edges with their original author and date** so she can ask the right person the right question instead of silently stumbling into a minefield.
4. **Issue backfill (persona: SRE Sam).** As an SRE preparing a quarterly cleanup sprint, Sam pipes `todo-stream --format jsonl --since 2023-01-01` into a script that opens GitHub issues for each finding older than a year, so **stale debt becomes trackable work** without manual hunting.

## 4. Architecture

```
 ┌────────────┐   ┌──────────────┐   ┌──────────────┐   ┌──────────────┐   ┌──────────────┐
 │   config   │──▶│    walker    │──▶│   extractor  │──▶│   enricher   │──▶│   emitter    │
 │ (flags +   │   │ (fs traverse,│   │ (per-file    │   │ (git blame,  │   │ (jsonl/tsv/  │
 │ .ts-config)│   │  gitignore)  │   │  comment re) │   │  baseline)   │   │  markdown)   │
 └────────────┘   └──────────────┘   └──────────────┘   └──────────────┘   └──────────────┘
                        │                    │                  │                  │
                        ▼                    ▼                  ▼                  ▼
                   file paths          raw findings       enriched findings    stdout bytes
                   (chan string)       (chan Finding)    (chan Finding)
```

### Components (Go packages)
- `cmd/todo-stream` — Cobra-based CLI entrypoint; parses flags, wires pipeline, handles exit codes.
- `internal/config` — merges defaults, `.todo-stream.yml` if present, and flags. Pure, no I/O beyond reading the config file.
- `internal/walker` — channel-producing directory walker that honors `.gitignore` via `go-git`'s matcher. Pluggable `FS` interface for test fakes.
- `internal/extractor` — given a file path + language classification (by extension), scans for configured markers inside comments using a language-aware lexer table (not a full parser; recognizes `//`, `/* */`, `#`, `--`, `<!-- -->`, triple-quoted strings are **not** searched). Emits `RawFinding{path, line, col, marker, text}`.
- `internal/enricher` — runs `git blame --porcelain -L n,n -- path` per finding (batched per file), parses porcelain output, attaches `{author, email, sha, date}`. Applies baseline filter.
- `internal/baseline` — reads/writes `.todo-stream-ignore` (plain text, one `sha:relpath:line:marker` tuple per line, `#` comments allowed).
- `internal/emitter` — three implementations behind an `Emitter` interface: `jsonl`, `tsv`, `markdown`.
- `internal/gitexec` — thin wrapper around `os/exec` for `git` invocations; isolates this boundary for testing and for the "no git repo" fallback (enricher degrades gracefully: emits findings with null blame fields).

### Boundaries
- **No network.** The tool never dials out.
- **Git is optional.** If the target directory is not a git repo, enrichment fields are null and the tool still runs.
- **Streaming by default.** Findings are emitted as they are produced; the tool does not buffer the full result set for `jsonl`/`tsv`. Markdown output buffers (requires sorting/grouping).
- **Concurrency.** Walker → extractor → enricher form a bounded worker-pool pipeline (`GOMAXPROCS` workers for extraction, serialized git-blame batching per file to avoid git lock contention).

### Key abstractions
- `type Finding struct { Path string; Line int; Marker string; Text string; Blame *Blame }`
- `type Blame struct { Author, Email, SHA string; Date time.Time }`
- `type Emitter interface { Emit(Finding) error; Close() error }`
- `type Baseline interface { Contains(Finding) bool; Entries() []Entry }`

## 5. Implementation Details

### Data flow
1. `cmd` loads config → constructs pipeline.
2. `walker` pushes file paths into `paths chan string` (capacity 256); respects `.gitignore` and `--include` / `--exclude` globs.
3. N extractor workers read `paths`, open each file, scan line-by-line with a precompiled regex per language family (`(?i)\b(TODO|FIXME|HACK)\b[:(]?\s*(.*)`), produce `RawFinding`s into `raw chan RawFinding`.
4. Enricher worker pool groups raw findings by path, invokes `git blame --porcelain -L a,b --incremental -- path` once per file covering the union of finding lines, zips blame info onto findings, applies baseline filter, emits to `out chan Finding`.
5. Emitter drains `out` and writes to stdout. For `markdown`, it buffers into a slice, sorts by (author, date), groups per user flag, then writes.
6. Exit code: `0` if `out` empty after baseline filter, `1` if any finding emitted, `2` if pipeline errored.

### Language/comment table (initial)
| Language group | Extensions | Line comment | Block comment |
|---|---|---|---|
| C-family | .c .h .cc .cpp .hpp .go .rs .java .kt .swift .cs .js .jsx .ts .tsx .scala | `//` | `/* */` |
| Shell-family | .sh .bash .zsh .py .rb .pl .toml .yml .yaml .tf .mk Makefile | `#` | — |
| SQL-family | .sql | `--` | `/* */` |
| Markup | .html .xml .vue .svelte .md | — | `<!-- -->` |
| Lisp-family | .el .lisp .clj | `;` | — |

Strings are **not** parsed; false positives inside string literals are accepted as a known v0.1 limitation (documented). A user can suppress specific findings via baseline.

### Baseline file format (`.todo-stream-ignore`)
```
# auto-generated by: todo-stream baseline add
abc1234:internal/foo/bar.go:42:TODO
def5678:pkg/x/y.js:17:FIXME
```
Match key is `(sha, relpath, line, marker)`. If the blame SHA changes, the baseline entry no longer matches — i.e., *modifying the line invalidates the grandfathering*, which is intentional: the ratchet should catch re-touched debt.

### State transitions for baseline CLI
- `todo-stream baseline add` — runs a scan, appends every current finding to `.todo-stream-ignore` (dedupe, sort).
- `todo-stream baseline prune` — runs a scan, drops baseline entries that no longer match any finding (the "done" case).
- `todo-stream baseline list` — prints baseline entries enriched with blame (for review).

### Performance targets
- 100k LOC repo, warm FS cache: scan completes in < 3 s on an M2 laptop.
- Memory: O(findings) not O(LOC); streaming emitters hold ≤ 1 MB resident for jsonl.
- Integration test target: `postgres/postgres` (~2.5M LOC) completes in < 60 s with blame disabled and < 5 min with blame enabled (on CI).

## 6. Tests Plan

### TDD (red/green) — built test-first
- **Extractor** (`internal/extractor`): exhaustive table tests over (language × marker × comment-syntax × edge cases: URL fragments, trailing punctuation, marker inside string literal documented as false positive). **Test-first, red-green-refactor.**
- **Baseline matcher** (`internal/baseline`): tuple match/mismatch, malformed lines, comment lines. **Test-first.**
- **Emitters** (`internal/emitter`): golden-file tests for jsonl, tsv, markdown over a fixed finding set. **Test-first.**
- **Config merge** (`internal/config`): flag > file > default precedence. **Test-first.**

### Built test-after (behaviour-first, tests pin the shape)
- `internal/walker` — exercised via integration tests against real fixture trees; unit tests only for the gitignore matcher boundary.
- `internal/enricher` — integration tests against a small ephemeral git repo created in `t.TempDir()`; unit tests for porcelain parser only.
- `cmd/todo-stream` — end-to-end tests shelling out to the built binary.

### CI test matrix
1. **Unit tests** (`go test ./...`) on Linux, macOS, Windows, Go 1.22 and latest.
2. **Fixture integration** — a committed `testdata/fixtures/` tree with known findings; assert exact JSONL output.
3. **Ephemeral-repo integration** — create a repo in `t.TempDir()`, commit files with TODOs under distinct authors/dates, run binary, assert blame fields.
4. **External-repo smoke (nightly only, not PR-blocking)** — shallow-clone `postgres/postgres` and `postgres-ai/database-lab` into a CI cache, run `todo-stream`, assert non-zero finding count and < 5 min runtime. Lives in a separate nightly workflow so PR CI stays under 3 min.
5. **Lint** — `golangci-lint run`.
6. **Race** — `go test -race ./...`.
7. **Release dry-run** — GoReleaser `--snapshot` on tag PRs.

### Coverage target
- `internal/extractor`, `internal/baseline`, `internal/emitter`, `internal/config`: ≥ 90 % line coverage (these are the pure-logic cores).
- Overall repo: ≥ 75 %.

## 7. Team

Veteran experts to hire for the build:
- **Veteran Go CLI systems engineer (1)** — primary owner; Cobra, channels, goroutines, cross-compilation, GoReleaser.
- **Veteran Git internals engineer (1)** — owns the `enricher` + `gitexec` boundary; deep `git blame --porcelain` experience, handles submodule/worktree edge cases.
- **Veteran language-tooling / lexer engineer (1)** — owns the extractor's per-language comment rules; prior work on linters or syntax highlighters.
- **Veteran CI/release engineer (0.5)** — GitHub Actions, Homebrew tap, npm binary-wrapper package, GoReleaser config, SBOM/signing.
- **Veteran QA / integration-test engineer (0.5)** — fixture design, ephemeral-repo test harness, nightly external-repo smoke workflow.

Total: 4.0 FTE-weeks worth of specialists for the v0.1 cut.

## 8. Implementation Plan

Three short sprints. Parallel tracks are labeled `[A]`, `[B]`, `[C]`, `[D]`, `[E]` corresponding to the five hires above.

### Sprint 1 — Skeleton + pure cores (week 1)
Goal: a binary that compiles and runs against a fixture, emitting findings *without* blame.
- `[A]` Scaffold repo layout, Cobra CLI, `scan` subcommand, exit-code wiring, Makefile, CI boot.
- `[C]` **TDD**: build `internal/extractor` against fixture files; finalize language/comment table; golden tests green.
- `[A]` **TDD**: build `internal/emitter` (jsonl/tsv/markdown) against a fixed `[]Finding` slice; golden tests green.
- `[A]` **TDD**: build `internal/config` merge logic; golden tests green.
- `[D]` GitHub Actions workflow: unit + race + lint on Linux/macOS/Windows.
- **Dependency:** `[A]` config scaffold must land by day 2 so `[C]` can import config types; otherwise `[C]` stubs them.
- **Parallelizable:** `[A]` emitter, `[A]` config, `[C]` extractor can run concurrently after day 2.

**Exit criterion:** `todo-stream scan ./testdata/fixtures` emits correct JSONL with null blame; CI green.

### Sprint 2 — Enrichment + baseline (week 2)
Goal: full v0.1 behaviour.
- `[B]` **TDD** on `internal/gitexec` porcelain parser; then integration-tested `internal/enricher` against ephemeral repos.
- `[A]` Wire the walker → extractor → enricher → emitter pipeline with bounded worker pools; add `--include`/`--exclude`/`.gitignore` support.
- `[A]` **TDD**: `internal/baseline` matcher and file format; `baseline add|prune|list` subcommands.
- `[E]` Fixture tree + ephemeral-repo integration tests. Author/date assertions.
- **Dependency:** `[B]` enricher depends on `[A]` pipeline scaffold from Sprint 1; `[A]` baseline can land in parallel with `[B]` enricher (independent packages).
- **Parallelizable:** `[B]` enricher, `[A]` baseline, `[E]` integration tests all run concurrently after Sprint 1 exit.

**Exit criterion:** all v0.1 user stories executable end-to-end against `testdata/fixtures`; ephemeral-repo integration tests green.

### Sprint 3 — Release hardening (week 3)
Goal: shippable binary on GitHub Releases + Homebrew + npm wrapper.
- `[D]` GoReleaser config (darwin/linux/windows × amd64/arm64), signed checksums, Homebrew tap, npm binary-wrapper package.
- `[E]` Nightly workflow: clone `postgres/postgres` + `postgres-ai/database-lab`, run binary, assert runtime/finding-count budgets.
- `[A]` Performance pass: pprof, reduce allocations in extractor hot path, document numbers in README.
- `[A]`+`[C]` Docs: README usage, `.todo-stream.yml` example, CI recipe snippet (the Sprint-1 platform-engineer story).
- **Dependency:** release work `[D]` depends on a stable CLI surface from Sprint 2 (flag freeze at Sprint-2 exit).
- **Parallelizable:** `[D]` release, `[E]` nightly smoke, `[A]`/`[C]` docs all run concurrently.

**Exit criterion:** `brew install todo-stream` and `npx todo-stream` both work; nightly smoke green two nights in a row; SPEC v0.1 tagged `v0.1.0`.

## 9. Non-goals / Deferred

- Interactive TUI, watch mode, LSP — deferred to v0.3+.
- Issue-tracker sinks (GitHub/Jira/Linear) — deferred to v0.2.
- AST-level parsing to eliminate string-literal false positives — deferred; baseline suppression is the v0.1 escape hatch.
- SARIF output for code-scanning dashboards — deferred to v0.2.
- Age-based severity (`--fail-older-than 180d`) — deferred to v0.2.

## 10. Risks & Mitigations

- **Git blame cost on huge repos.** Mitigation: batch blame per file, `--no-blame` flag, document perf budget, exercise in nightly smoke.
- **False positives in string literals.** Mitigation: baseline file; document limitation; revisit with AST in v0.2.
- **Windows path / line-ending quirks.** Mitigation: CI matrix includes Windows from day one; path handling via `filepath.ToSlash` at boundaries.
- **Interview/Idea conflict (see §2).** Mitigation: explicit reinterpretation documented above; user can reject in the next review round and the SPEC pivots.

## 11. Embedded Changelog

- v0.1 (2026-04-21) — initial draft scaffold; reconciled Idea (TODO-comment linter) with interview answers (Go single binary, pure CLI, JSON/TSV), re-mapping `add/list/done` onto `scan` + `baseline add|prune|list`; committed to Go + Cobra + GoReleaser; defined three-sprint plan with five specialist roles.
