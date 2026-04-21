# todo-stream — SPEC v0.1

## Goal & why it's needed

`todo-stream` is a CLI **linter for outstanding in-code comments** (TODO / FIXME / HACK / XXX). It walks a directory tree, extracts these markers from source files, enriches each finding with `git blame` metadata (author, commit date, SHA), and emits a grouped report as JSON or Markdown.

**Why it's needed.** Every mature codebase accumulates `TODO`s that nobody owns. `grep -rn TODO` loses authorship and timing; GitHub's "Issues" tab loses the exact line of code. Existing tools (`leasot`, `notes`, IDE plugins) either don't run in CI, don't attach blame, or pull in heavy runtime dependencies. Engineering leads and release managers need a fast, dependency-light, pipeable report that answers: *which open TODOs does this repo carry, who wrote them, and how stale are they?* — suitable for CI gating, release-readiness dashboards, and tech-debt triage.

**Explicit non-goals (honored strictly).**
- This is **NOT a todo-list application**. It does not create, edit, or complete tasks.
- This is **NOT a CRUD storage tool**. It has no database, no persistent state, no server.
- It does not mutate source files.
- It does not replace an issue tracker; it *surfaces* comments so humans can decide whether to file issues.

## User stories

1. **CI gatekeeper — Priya, release engineer.** Priya adds `todo-stream --format json --fail-on FIXME` to the repo's CI workflow so any PR that introduces a new `FIXME` fails the pipeline with a structured diagnostic pointing at file, line, author, and SHA.
2. **Tech-debt triage — Luis, staff engineer.** Luis runs `todo-stream --format markdown --since 2024-01-01 > DEBT.md` against the monorepo once a quarter, grouping findings by file with blame dates, and uses the Markdown report in a planning meeting to assign owners.
3. **Incoming maintainer — Dana, new OSS contributor.** Dana clones a large project (e.g. `postgres/postgres`) and runs `todo-stream src/backend --markers TODO,HACK --format markdown | less` to orient herself: she sees the oldest HACKs, who wrote them, and which files are hot-spots, without learning the project's in-house tooling.
4. **Pre-commit author — Sam, individual developer.** Sam wires `todo-stream --staged --format json` into a `lefthook` pre-commit to print a compact summary of TODOs touched by the diff, so he notices accidental debt before pushing.
5. **Pipeline consumer — automated dashboard.** A scheduled job runs `todo-stream --format json` and pipes the output into a downstream ingester that renders a historical chart of open TODO count per file — the JSON schema is stable and documented.

## Scope & non-goals (v0.1)

**In scope**
- Recursive walk of a directory with configurable include/exclude globs (defaults respect `.gitignore`).
- Extraction of configurable markers (default: `TODO`, `FIXME`, `HACK`, `XXX`) from line and block comments in common languages (C, C++, Go, Rust, TS/JS, Python, Shell, SQL, Lua, Ruby — identified by file extension; everything else falls back to a generic regex).
- Git blame enrichment per finding: author name, author email, commit date (ISO-8601), short SHA. Non-git trees degrade gracefully (blame fields `null`).
- Two output formats: `json` (stable, documented schema) and `markdown` (human-readable, grouped by file).
- Filters: `--markers`, `--since`, `--author`, `--path`, `--fail-on`.
- Single static Bun-compiled binary + `npx todo-stream` entry point.

**Out of scope for v0.1 (explicitly rejected or deferred)**
- Any persistent storage (SQLite, JSON file, event log). `todo-stream` is stateless — each run re-scans.
- Long-running daemon / watch mode / stdout NDJSON streaming transport.
- Subcommands like `add` / `list` / `done` — this tool does not *manage* todos.
- Multi-device sync, multi-user collaboration.
- A todo data model with `id` / `done` / `created_at` fields — findings are *derived* from source, not stored records.
- Rich parsers (tree-sitter, language servers). Regex over extension-keyed comment syntaxes is sufficient.

## Architecture

```
  ┌──────────────┐   paths   ┌──────────────┐  raw hits  ┌──────────────┐
  │  CLI / args  │──────────▶│   Walker     │───────────▶│  Extractor   │
  │  (flags.ts)  │           │ (walker.ts)  │            │ (extract.ts) │
  └──────────────┘           └──────────────┘            └──────┬───────┘
         ▲                                                      │ findings
         │ exit code                                            ▼
  ┌──────┴───────┐   report   ┌──────────────┐  enriched  ┌──────────────┐
  │   Reporter   │◀───────────│   Renderer   │◀───────────│    Blamer    │
  │ (main.ts)    │            │ json | md    │            │ (blame.ts)   │
  └──────────────┘            └──────────────┘            └──────────────┘
```

**Components & boundaries**
- **CLI (`src/cli.ts`)** — argument parsing (hand-rolled, no deps), config resolution (CLI flags > env > `todo-stream.config.json` > defaults), dispatch. Owns process exit code.
- **Walker (`src/walker.ts`)** — streams matching file paths. Uses `Bun.Glob` for include/exclude; respects `.gitignore` via a small parser. Emits an async iterator of absolute paths; never buffers the whole tree.
- **Extractor (`src/extract.ts`)** — pure function: `(path, bytes, config) → Finding[]`. Dispatches on extension to a comment-syntax table (`//`, `#`, `--`, `/* */`, `<!-- -->`, etc.). Returns `{path, line, column, marker, text, raw}` — no I/O, fully unit-testable from fixtures.
- **Blamer (`src/blame.ts`)** — batches findings per file and shells out to `git blame --porcelain -L <line>,<line> -- <file>` (or `-L` ranges), parses porcelain output into `{author, email, date, sha}`. Caches by `(file, sha-of-HEAD)` within a run. Degrades to `null` fields outside a git repo.
- **Renderer (`src/render/*.ts`)** — two pure renderers: `json.ts` (emits a versioned schema `{ $schema, tool, version, generated_at, findings: [...] }`) and `markdown.ts` (groups by file, sorts by line).
- **Reporter (in `cli.ts`)** — writes to stdout, applies `--fail-on` rules, sets exit code.

**Key abstractions**
- `Finding` — the one shared record shape crossing every boundary.
- `CommentSyntax` — `{ line?: string[]; block?: [open, close][] }`, looked up by extension.
- `Config` — frozen object assembled once at startup.

**Dependency policy.** Runtime deps = zero beyond Bun built-ins (`Bun.Glob`, `Bun.spawn`, `Bun.file`, `fs`, `path`). Dev deps limited to `bun test` and type definitions.

## Implementation details

### Data flow
1. `cli.ts` parses argv → `Config`.
2. `walker.ts` yields file paths matching globs, pruning `.gitignore` and `--exclude`.
3. For each path, `Bun.file(path).text()` → `extract.ts` → `Finding[]` (streamed, not accumulated repo-wide).
4. Findings are grouped by file and handed to `blame.ts`, which issues one `git blame` per file with all needed line ranges.
5. Enriched findings feed the chosen renderer; output is written to stdout.
6. `--fail-on <marker>` sets exit code 1 if any matching finding exists; absence of findings exits 0. Internal errors exit 2.

### Finding extraction algorithm
- For each line, strip to the comment region using the language's `CommentSyntax`.
- Match `/(^|[^A-Za-z0-9_])(TODO|FIXME|HACK|XXX)\b[:\s-]?\s*(.*)/` (markers from config).
- For block comments spanning multiple lines, continuation lines are attached to the marker on the opening line as a multi-line `text`.
- Unknown extensions use a generic fallback: match markers only when preceded by a non-identifier character; record with `language: "unknown"`.

### Blame batching
- Group findings by file; run `git blame --porcelain -L a,a -L b,b -- <file>` in a single invocation per file.
- Parse porcelain headers once per unique commit; short-SHA = first 10 chars of commit id.
- Outside a git worktree (detected via `git rev-parse --is-inside-work-tree`), skip blame entirely and emit `blame: null`.

### JSON schema (stable surface)
```json
{
  "tool": "todo-stream",
  "version": "0.1.0",
  "generated_at": "2026-04-21T00:00:00Z",
  "root": "/abs/path",
  "findings": [
    {
      "path": "src/foo.ts",
      "line": 42,
      "column": 5,
      "marker": "TODO",
      "text": "rewrite with streaming parser",
      "language": "typescript",
      "blame": {
        "author": "Jane Doe",
        "email": "jane@example.com",
        "date": "2024-07-11T14:03:22Z",
        "sha": "a1b2c3d4e5"
      }
    }
  ]
}
```

### Markdown output shape
- `# TODO report — <root> — <generated_at>`
- One `## <relative/path>` section per file, findings as list items: `- **TODO** L42 · Jane Doe, 2024-07-11 (a1b2c3d) — rewrite with streaming parser`.

### CLI surface
```
todo-stream [path]              positional root (default: cwd)
  --format json|markdown        default: markdown when TTY, json otherwise
  --markers TODO,FIXME,HACK     default: TODO,FIXME,HACK,XXX
  --include "**/*.ts"           repeatable
  --exclude "**/dist/**"        repeatable
  --since 2024-01-01            drop findings whose blame date is older
  --author <substr>             filter by blame author
  --fail-on TODO|FIXME|...      exit 1 if any finding matches
  --no-gitignore                disable .gitignore pruning
  --no-blame                    skip git blame enrichment
  --staged                      limit to files in `git diff --cached --name-only`
  --config <path>               load JSON config
  --version | --help
```

## Tests plan

### Unit tests (fixture-driven, TDD — RED first)
Built **test-first** (red → green):
- **Extractor (`extract.test.ts`)** — a `tests/fixtures/` tree with one small file per supported language containing known markers in line comments, block comments, nested blocks, strings-that-look-like-comments (negative case), and the generic fallback. Each assertion pins exact `(line, column, marker, text)`. **Write the failing tests first**, then implement `extract.ts` until green.
- **Walker (`walker.test.ts`)** — fixture tree with `.gitignore`, nested ignores, symlinks, and binary files. Assert the yielded path set. Red-first.
- **Renderers (`render.test.ts`)** — golden-file tests: feed a known `Finding[]` → assert byte-exact JSON and stable Markdown. Red-first.
- **Blame parser (`blame.test.ts`)** — feed canned `git blame --porcelain` output (captured, checked in) → assert parsed records. Red-first; no real git invocation here.

### Integration tests (real git, CI)
Built after the unit suite is green (test-second, since they depend on the full pipeline):
- **`postgres/postgres`** — shallow clone in CI (`git clone --depth=1`), run `todo-stream --format json` against a bounded subtree (e.g. `src/backend/access/`), assert: exits 0, JSON validates against the schema, `findings.length > 0`, every finding has non-null blame fields, runtime < 60 s on CI hardware.
- **`postgres-ai/database-lab`** — shallow clone, full-repo scan, same invariants, plus spot-check: at least one known long-standing `TODO` is present (pin by file path + marker, not by exact text which may rot).
- Both integration tests are gated behind `INTEGRATION=1` env so local `bun test` stays fast; CI sets it.

### CI matrix
- `bun test` (unit) on every PR — must pass.
- `INTEGRATION=1 bun test` nightly and on `main` — must pass.
- `bun build --compile` smoke: compile, run `--help`, run against the repo itself.
- Typecheck: `bun tsc --noEmit`.
- Lint: `biome check` (dev-only).

### TDD call-out (explicit)
- **Test-first (RED → GREEN):** extractor, walker, renderers, blame-porcelain parser. These are pure and fixture-driven — write failing tests before any implementation line.
- **Test-after:** CLI argument wiring, integration tests against real repos, the compile/smoke step. These exercise glue and external systems where test-first yields diminishing returns.

## Team

Veteran experts to hire:
- **Veteran CLI systems engineer (1)** — owns walker, CLI surface, Bun packaging, exit-code semantics.
- **Veteran parser/regex engineer (1)** — owns the extractor, comment-syntax table, and edge-case handling (strings-as-comments, nested blocks).
- **Veteran git-plumbing engineer (1)** — owns the blamer: porcelain parsing, batching, graceful degradation, and worktree detection.
- **Veteran test engineer (1)** — owns fixture design, golden files, and the postgres/postgres + database-lab integration harness in CI.
- **Veteran TypeScript/Bun release engineer (1, part-time)** — owns `package.json`, `bin` entry, `bun build --compile`, npm publish workflow, semver discipline.

Total: 4 full-time + 1 part-time.

## Implementation plan

Sprints are one week each. `⇄` = work happens in parallel; `→` = ordering dependency.

### Sprint 1 — Skeleton & red tests
- CLI engineer: scaffold repo layout, `bin/todo-stream`, `--help`, `--version`, argv parser. ⇄
- Parser engineer: author fixture tree under `tests/fixtures/langs/` and write **failing** extractor tests for TS/JS, Go, Python, C/C++, SQL, Shell. ⇄
- Git engineer: capture canned `git blame --porcelain` outputs into `tests/fixtures/blame/` and write **failing** parser tests. ⇄
- Test engineer: stand up `bun test` in CI, add the `INTEGRATION=1` gate (no integration tests yet). ⇄
- Release engineer: lock Bun version, add `bun tsc --noEmit` + `biome check` to CI.
- **Gate:** all tests RED, CI green on lint/type.

### Sprint 2 — Core green
- Parser engineer → implement `extract.ts` until fixtures pass. (depends on Sprint 1 fixtures)
- CLI engineer → implement `walker.ts` with `Bun.Glob` + `.gitignore`, write walker tests RED→GREEN in this sprint. ⇄
- Git engineer → implement blame porcelain parser (pure) until its unit tests pass. ⇄
- Test engineer → author golden-file tests for both renderers (still red; renderers not yet built).
- **Gate:** extractor, walker, blame parser all GREEN; renderer tests still RED.

### Sprint 3 — Render & wire
- CLI engineer → implement JSON + Markdown renderers against golden tests; wire end-to-end pipeline in `cli.ts`. (depends on Sprint 2)
- Git engineer → implement `blame.ts` runtime (spawn `git blame`, batch per file, cache, no-git fallback). ⇄
- Parser engineer → extend markers config + `--markers` flag, add generic fallback extraction. ⇄
- Test engineer → start integration harness: shallow-clone helper, postgres subtree scan (still failing E2E, since `blame.ts` may be mid-flight).
- **Gate:** renderer tests GREEN; `todo-stream` runs end-to-end against the repo itself and produces valid output.

### Sprint 4 — Integration & release
- Test engineer → finalize integration tests (postgres/postgres + postgres-ai/database-lab), tune CI runtime. (depends on Sprint 3)
- Release engineer → `bun build --compile` pipeline, npm publish dry-run, `npx todo-stream` smoke, README usage + JSON schema docs. ⇄
- CLI engineer → implement `--since`, `--author`, `--fail-on`, `--staged`, `--config`. ⇄
- Git engineer → hardening: large-file guard, symlink handling in blame, performance pass (target: scan postgres/postgres in < 60 s). ⇄
- Parser engineer → extend language coverage (Rust, Ruby, Lua) with fresh red→green fixtures.
- **Gate:** integration tests GREEN on CI; `bun run build` emits a working static binary; v0.1.0 tagged.

### Parallelization summary
- Sprint 1 is almost fully parallel (four independent red-test authoring streams).
- Sprint 2 and 3 pair the parser+git engineers in parallel while the CLI engineer advances the pipeline; the test engineer stays one step ahead writing the next red tests.
- Sprint 4 fans out: every engineer has an independent lane.

## Embedded Changelog

- **v0.1 (2026-04-21)** — Initial spec draft. Reframed away from the "todo-list app" interview answers (SQLite, NDJSON daemon, add/list/done subcommands, id/done/created_at model) per the authoritative idea: this is a **linter for TODO/FIXME/HACK comments**, not a storage tool. Locked scope to directory walk + regex extract + git blame + JSON/Markdown report. Zero runtime deps beyond Bun built-ins. Tests: fixture-driven unit suite (TDD red-first) plus CI integration tests cloning `postgres/postgres` and `postgres-ai/database-lab`.
