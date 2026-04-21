# todo-stream — SPEC v0.3

## Goal & why it's needed

`todo-stream` is a CLI **linter for outstanding in-code comments** (TODO / FIXME / HACK / XXX). It walks a directory tree, extracts these markers from source files, enriches each finding with `git blame` metadata (author, commit date, SHA), and emits a grouped report as JSON or Markdown.

**Why it's needed.** Every mature codebase accumulates `TODO`s that nobody owns. `grep -rn TODO` loses authorship and timing; GitHub's "Issues" tab loses the exact line of code. Existing tools (`leasot`, `notes`, IDE plugins) either don't run in CI, don't attach blame, or pull in heavy runtime dependencies. Engineering leads and release managers need a fast, dependency-light, pipeable report that answers: *which open TODOs does this repo carry, who wrote them, and how stale are they?* — suitable for CI gating, release-readiness dashboards, and tech-debt triage.

**Explicit non-goals (honored strictly).**
- This is **NOT a todo-list application**. It does not create, edit, or complete tasks.
- This is **NOT a CRUD storage tool**. It has no database, no persistent state, no server.
- It does not mutate source files.
- It does not replace an issue tracker; it *surfaces* comments so humans can decide whether to file issues.

## Versioning convention

Two distinct versions appear in this document and must never be conflated:

- **Spec version** — the version of *this document*. Appears only in the `# todo-stream — SPEC vX.Y` header and the embedded changelog. Bumps every review round. Currently **v0.3**.
- **Product version** — the version of the *published npm package and compiled binary*. Appears in `package.json`, the JSON report's `version` field, and `--version` output. Currently **0.1.0** (pre-release; will ship as 0.1.0 when the v0.1 scope below is implemented).

The JSON schema example below therefore shows `"version": "0.1.0"` (product), while this document's header is `SPEC v0.3`.

## User stories

1. **CI gatekeeper — Priya, release engineer.** Priya adds `todo-stream --format json --fail-on FIXME` to the repo's CI workflow so any PR that introduces a new `FIXME` fails the pipeline with a structured diagnostic pointing at file, line, author, and SHA.
2. **Tech-debt triage — Luis, staff engineer.** Luis runs `todo-stream --format markdown --since 2024-01-01 > DEBT.md` against the monorepo once a quarter, grouping findings by file with blame dates, and uses the Markdown report in a planning meeting to assign owners.
3. **Incoming maintainer — Dana, new OSS contributor.** Dana clones a large project (e.g. `postgres/postgres`) and runs `todo-stream src/backend --markers TODO,HACK --format markdown | less` to orient herself: she sees the oldest HACKs, who wrote them, and which files are hot-spots, without learning the project's in-house tooling.
4. **Pre-commit author — Sam, individual developer.** Sam wires `todo-stream --staged --format json` into a `lefthook` pre-commit. `--staged` limits the scan to *files* that appear in `git diff --cached --name-only` (file-level, not hunk-level — see CLI surface); Sam uses it to notice accidental debt in files he is about to push.
5. **Pipeline consumer — automated dashboard.** A scheduled job runs `todo-stream --format json` and pipes the output into a downstream ingester that renders a historical chart of open TODO count per file — the JSON schema is stable and documented.

## Scope & non-goals (product v0.1)

**In scope (product v0.1)**
- Recursive walk of a directory with configurable include/exclude globs (defaults respect `.gitignore`).
- Extraction of configurable markers (default: `TODO`, `FIXME`, `HACK`, `XXX`) from line and block comments in common languages (C, C++, Go, Rust, TS/JS, Python, Shell, SQL, Lua, Ruby — identified by file extension; everything else falls back to a generic regex).
- Git blame enrichment per finding: author name, author email, commit date (ISO-8601), short SHA. Non-git trees degrade gracefully (top-level `blame: null`).
- Two output formats: `json` (stable, documented schema) and `markdown` (human-readable, grouped by file).
- Filters: `--markers`, `--since`, `--author`, `--path`, `--fail-on`.
- Single static Bun-compiled binary + `npx todo-stream` entry point.

**Out of scope for product v0.1 (explicitly rejected or deferred)**
- Any persistent storage (SQLite, JSON file, event log). `todo-stream` is stateless — each run re-scans.
- Long-running daemon / watch mode / stdout NDJSON streaming transport.
- Subcommands like `add` / `list` / `done` — this tool does not *manage* todos.
- Multi-device sync, multi-user collaboration.
- A todo data model with `id` / `done` / `created_at` fields — findings are *derived* from source, not stored records.
- Rich parsers (tree-sitter, language servers). Regex over extension-keyed comment syntaxes is sufficient.
- **String-literal tokenization.** The extractor inspects only the *comment region* of a line; it does not parse string literals. A `TODO` appearing inside a string on a line that is not otherwise a comment is *not* a finding. A `TODO` inside a string *within* a comment (e.g. `// TODO: see "FIXME" in docs`) remains a single `TODO` finding — the inner token is treated as comment text.

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
- **Walker (`src/walker.ts`)** — streams matching file paths. Uses `Bun.Glob` for include/exclude. `.gitignore` pruning is delegated to `git check-ignore --stdin -v` when the root is inside a git worktree; outside a worktree, no `.gitignore` pruning is performed (users may still use `--exclude`). Emits an async iterator of absolute paths; never buffers the whole tree.
- **Extractor (`src/extract.ts`)** — pure function: `(path, bytes, config) → Finding[]`. Dispatches on extension to a comment-syntax table (`//`, `#`, `--`, `/* */`, `<!-- -->`, etc.). Returns `{path, line, column, marker, text, raw}` — no I/O, fully unit-testable from fixtures.
- **Blamer (`src/blame.ts`)** — batches findings per file and shells out to `git blame --porcelain -L a,a -L b,b -- <file>`, parses porcelain output into `{author, email, date, sha}`. Caches by file. Degrades to top-level `blame: null` outside a git worktree.
- **Renderer (`src/render/*.ts`)** — two pure renderers: `json.ts` (emits the versioned schema defined below) and `markdown.ts` (groups by file, sorts by line).
- **Reporter (in `cli.ts`)** — writes to stdout, applies `--fail-on` rules, sets exit code.

**Key abstractions**
- `Finding` — the one shared record shape crossing every boundary.
- `CommentSyntax` — `{ line?: string[]; block?: [open, close][] }`, looked up by extension.
- `Config` — frozen object assembled once at startup.

**Dependency policy.** Runtime deps = zero beyond Bun built-ins (`Bun.Glob`, `Bun.spawn`, `Bun.file`, `fs`, `path`). `git` itself is assumed on PATH (required for blame and `.gitignore` delegation; the tool degrades without it as documented). Dev deps limited to `bun test` and type definitions.

## Implementation details

### Data flow
1. `cli.ts` parses argv → `Config`.
2. `walker.ts` yields file paths matching globs, pruning via `git check-ignore` (inside a worktree) and `--exclude`.
3. For each path, `Bun.file(path).text()` → `extract.ts` → `Finding[]` (streamed, not accumulated repo-wide).
4. Findings are grouped by file and handed to `blame.ts`, which issues one `git blame` per file with all needed line ranges.
5. Filters `--since` and `--author` are applied **after** blame enrichment (see precedence rules below).
6. Enriched findings feed the chosen renderer; output is written to stdout.
7. `--fail-on <marker>` sets exit code 1 if any matching finding exists after filtering; absence of findings exits 0. Internal errors (I/O, bad config, spawn failure) exit 2.

### Finding extraction algorithm
- The marker regex is **built dynamically** from `config.markers`. Each marker is regex-escaped and joined:
  `new RegExp("(^|[^A-Za-z0-9_])(" + markers.map(escape).join("|") + ")\\b[:\\s-]?\\s*(.*)")`.
  Match is case-sensitive by default (overridable via `--markers-case=insensitive` in a later revision; v0.1 ships case-sensitive only).
- For each line, the extractor first reduces the line to its comment region using the language's `CommentSyntax`. It does **not** tokenize string literals; see the Scope section's explicit non-goal. This means markers embedded in string literals on code-only lines are not reported.
- **Multi-marker block comments.** Within a single block comment, each *line* that contains a marker yields its own `Finding` at that line's line/column. Continuation lines *between* markers (lines with no marker of their own) are attached as trailing text to the most recent preceding marker in the same block.
- Text trimming: leading comment punctuation (`*`, `#`, `--`, `//`) and surrounding whitespace are stripped from the captured `text`.
- Unknown extensions use a generic fallback: match markers only when preceded by a non-identifier character; record with `language: "unknown"`.

### Filter precedence and blame-dependent flags
- `--since` and `--author` operate on blame metadata.
- If `--no-blame` is passed *together* with `--since` or `--author`, the CLI exits 2 with a usage error (these flags are mutually incompatible).
- When blame is enabled but a particular finding has `blame: null` (e.g. file is untracked, or the entire tree is not a git worktree), applying `--since` or `--author` **drops** that finding from the output. Rationale: the user is filtering on blame data; a finding with no blame data cannot satisfy the filter.
- `--fail-on` is evaluated **after** all filters. A repo with FIXMEs that are all filtered out by `--since` exits 0.

### Blame batching
- Group findings by file; run `git blame --porcelain -L a,a -L b,b -- <file>` in a single invocation per file.
- Parse porcelain headers once per unique commit.
- Outside a git worktree (detected via `git rev-parse --is-inside-work-tree`), skip blame entirely and emit top-level `blame: null` shape per finding (the entire `blame` key is `null`, not an object with null fields).

### JSON schema (stable surface)

The JSON renderer emits exactly this shape. `$schema` points at a hosted JSON-Schema document shipped alongside the binary; `root` is the resolved absolute path of the scan root (useful for downstream consumers correlating multiple runs).

```json
{
  "$schema": "https://todo-stream.dev/schema/v1.json",
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

The `blame` key is either the object shown above or the JSON literal `null` (never a partially-null object). The schema is considered a stable public surface from product v0.1 onward; additive changes bump the `$schema` URL's version segment.

### Markdown output shape
- `# TODO report — <root> — <generated_at>`
- One `## <relative/path>` section per file, findings as list items: `- **TODO** L42 · Jane Doe, 2024-07-11 (a1b2c3d4e5) — rewrite with streaming parser`.

### CLI surface
```
todo-stream [path]              positional root (default: cwd)
  --format json|markdown        default: markdown when TTY, json otherwise
  --markers TODO,FIXME,HACK     default: TODO,FIXME,HACK,XXX (comma list, escaped into regex)
  --include "**/*.ts"           repeatable
  --exclude "**/dist/**"        repeatable
  --since 2024-01-01            drop findings whose blame date is older; incompatible with --no-blame
  --author <substr>             filter by blame author; incompatible with --no-blame
  --fail-on TODO|FIXME|...      exit 1 if any finding matches after filters
  --no-gitignore                disable git-delegated .gitignore pruning
  --no-blame                    skip git blame enrichment
  --staged                      limit to FILES listed by `git diff --cached --name-only` (file-level, not hunk-level)
  --config <path>               load JSON config
  --version | --help
```

Exit codes: `0` (success, no failing findings), `1` (`--fail-on` triggered), `2` (usage/internal error).

## Tests plan

### Unit tests (fixture-driven, TDD — RED first)
Built **test-first** (red → green):
- **Extractor (`extract.test.ts`)** — a `tests/fixtures/` tree with one small file per supported language containing known markers in line comments, block comments, nested blocks, and code-only lines with marker-looking strings (negative case: a `const s = "TODO"` line must yield zero findings). Each assertion pins exact `(line, column, marker, text)`. Additional fixture: a block comment containing both `TODO:` and `FIXME:` on different lines — asserts two Findings, one per marker line.
- **Walker (`walker.test.ts`)** — fixture tree exercising `--include`/`--exclude` and symlink handling. `.gitignore` delegation is covered by a small integration-style test that shells out to `git check-ignore` against a tiny fixture repo (checked in as a bare tarball, extracted in the test setup).
- **Renderers (`render.test.ts`)** — golden-file tests: feed a known `Finding[]` → assert byte-exact JSON and stable Markdown.
- **Blame parser (`blame.test.ts`)** — feed canned `git blame --porcelain` output (captured, checked in) → assert parsed records. No real git invocation here.
- **CLI flags (`cli.test.ts`)** — black-box tests invoking the compiled CLI against fixture repos. Each v0.1 flag gets at least one positive and one negative case:
  - `--fail-on FIXME` with no FIXMEs → exit 0; with one FIXME → exit 1.
  - `--since 2099-01-01` → empty `findings`; `--since 1970-01-01` → all findings.
  - `--author nobody` → empty; `--author <real>` → non-empty.
  - `--staged` → only staged-file findings.
  - `--config <path>` → flags loaded; bad JSON → exit 2 with readable error.
  - `--no-blame --since 2024-01-01` → exit 2, usage error.
  - `--no-gitignore` → vendored `node_modules` fixture is now visible.
  - Unknown flag → exit 2.

### Integration tests (real git, CI)
Built after the unit suite is green:
- **`postgres/postgres`** — shallow clone in CI, run `todo-stream --format json` against a bounded subtree (e.g. `src/backend/access/`). Invariants (all structural, no pinned TODO text or path): exits 0; JSON validates against the published schema; `findings.length >= 10`; every finding has a non-null `blame` object; runtime well-behaved on the perf lane (see below).
- **`postgres-ai/database-lab`** — shallow clone, full-repo scan. Invariants: exits 0; JSON validates; `findings.length >= 1`; every finding has a non-null `blame` object.
- **No pinned external TODO assertions.** Coverage for "a long-standing TODO is surfaced correctly" is delivered by a **dedicated synthetic fixture repo** under `tests/fixtures/synthetic-repo/` (a checked-in bare-git tarball with deterministic authors, dates, and SHAs). This fixture pins `(path, line, marker, text, blame.sha)` quadruples without depending on upstream drift.
- Both real-repo integration tests are gated behind `INTEGRATION=1` env so local `bun test` stays fast; CI sets it.

### Performance lane (separate, not a correctness gate)
- Runs in its own CI job on **GitHub-hosted `ubuntu-latest` (4 vCPU, 16 GB RAM)**.
- Scans `postgres/postgres` subtree; records wall-clock in a CSV artifact and fails only if runtime exceeds **90 s** (generous ceiling; tracked as a trend, not a tight gate). No time-based assertions in the correctness integration tests.

### CI matrix
- `bun test` (unit + black-box CLI) on every PR — must pass.
- `INTEGRATION=1 bun test` nightly and on `main` — must pass.
- Performance lane nightly only.
- `bun build --compile` smoke: compile, run `--help`, run against the repo itself.
- Typecheck: `bun tsc --noEmit`.
- Lint: `biome check` (dev-only).

### TDD call-out (explicit)
- **Test-first (RED → GREEN):** extractor, walker, renderers, blame-porcelain parser, **every v0.1 CLI flag's black-box behavior and exit code**. These are deterministic and fixture-driven — write failing tests before any implementation line.
- **Test-after:** real-repo integration tests, the compile/smoke step, the perf lane. These exercise external systems where test-first yields diminishing returns.

## Team

Veteran experts to hire:
- **Veteran CLI systems engineer (1)** — owns walker, CLI surface, Bun packaging, exit-code semantics.
- **Veteran parser/regex engineer (1)** — owns the extractor, comment-syntax table, and edge-case handling (marker-looking strings on code lines, nested blocks, multi-marker blocks).
- **Veteran git-plumbing engineer (1)** — owns the blamer and the `git check-ignore` delegation: porcelain parsing, batching, graceful degradation, and worktree detection.
- **Veteran test engineer (1)** — owns fixture design, golden files, the synthetic-repo bare-git fixture, black-box CLI suite, and the postgres/postgres + database-lab integration harness in CI.
- **Veteran TypeScript/Bun release engineer (1, part-time)** — owns `package.json`, `bin` entry, `bun build --compile`, npm publish workflow, semver discipline, and the hosted JSON-Schema document.

Total: 4 full-time + 1 part-time.

## Implementation plan

Sprints are one week each. `⇄` = work happens in parallel; `→` = ordering dependency.

### Sprint 1 — Skeleton & red tests
- CLI engineer: scaffold repo layout, `bin/todo-stream`, `--help`, `--version`, argv parser. ⇄
- Parser engineer: author fixture tree under `tests/fixtures/langs/` and write **failing** extractor tests for TS/JS, Go, Python, C/C++, SQL, Shell, including the multi-marker-block and marker-in-string-on-code-line cases. ⇄
- Git engineer: capture canned `git blame --porcelain` outputs into `tests/fixtures/blame/` and write **failing** parser tests. ⇄
- Test engineer: stand up `bun test` in CI, add the `INTEGRATION=1` gate, scaffold black-box CLI test harness (invokes the compiled binary), write **failing** CLI tests for every v0.1 flag and exit-code case listed in Tests plan. ⇄
- Release engineer: lock Bun version, add `bun tsc --noEmit` + `biome check` to CI.
- **Gate:** all tests RED, CI green on lint/type.

### Sprint 2 — Core green
- Parser engineer → implement `extract.ts` (with dynamic marker regex built from config) until fixtures pass.
- CLI engineer → implement `walker.ts` with `Bun.Glob` + `git check-ignore` delegation, write walker tests RED→GREEN. ⇄
- Git engineer → implement blame porcelain parser (pure) until its unit tests pass. ⇄
- Test engineer → author golden-file tests for both renderers (still red; renderers not yet built); build the synthetic bare-git fixture repo.
- **Gate:** extractor, walker, blame parser all GREEN; renderer and CLI tests still RED.

### Sprint 3 — Render & wire
- CLI engineer → implement JSON + Markdown renderers against golden tests; wire end-to-end pipeline in `cli.ts`; implement exit-code semantics.
- Git engineer → implement `blame.ts` runtime (spawn `git blame`, batch per file, no-git fallback emitting `blame: null`). ⇄
- Parser engineer → extend markers config + `--markers` flag (dynamic regex), add generic fallback extraction. ⇄
- Test engineer → start integration harness: shallow-clone helper, postgres subtree scan with structural invariants.
- **Gate:** renderer tests and a majority of CLI flag tests GREEN; `todo-stream` runs end-to-end against the repo itself and produces valid output.

### Sprint 4 — Integration & release
- Test engineer → finalize integration tests (postgres/postgres + postgres-ai/database-lab with structural invariants only); stand up the separate perf lane on `ubuntu-latest` 4 vCPU. ⇄
- Release engineer → `bun build --compile` pipeline, npm publish dry-run, `npx todo-stream` smoke, README usage + JSON schema docs, publish the hosted `$schema` JSON. ⇄
- CLI engineer → implement `--since`, `--author`, `--fail-on`, `--staged`, `--config`, plus the `--no-blame` + `--since`/`--author` mutual-exclusion guard. ⇄
- Git engineer → hardening: large-file guard, symlink handling in blame, performance pass (trend tracking, not a hard gate). ⇄
- Parser engineer → extend language coverage (Rust, Ruby, Lua) with fresh red→green fixtures.
- **Gate:** all unit + black-box CLI tests GREEN; integration tests GREEN on CI; perf lane reporting; `bun run build` emits a working static binary; product v0.1.0 tagged.

### Parallelization summary
- Sprint 1 is almost fully parallel (five independent red-test authoring streams, including black-box CLI).
- Sprint 2 and 3 pair the parser+git engineers in parallel while the CLI engineer advances the pipeline; the test engineer stays one step ahead writing the next red tests.
- Sprint 4 fans out: every engineer has an independent lane.

## Embedded Changelog

- **Spec v0.1 (2026-04-21)** — Initial draft. Reframed away from the "todo-list app" interview answers (SQLite, NDJSON daemon, add/list/done subcommands, id/done/created_at model) per the authoritative idea: linter for TODO/FIXME/HACK comments, not a storage tool. Locked scope to directory walk + regex extract + git blame + JSON/Markdown report. Zero runtime deps beyond Bun built-ins.
- **Spec v0.2 (2026-04-21)** — Post-review r1 refinement. Strengthened user stories (added CI gatekeeper, pre-commit, pipeline-consumer personas). Tightened the Architecture diagram and component boundaries. Added the concrete JSON schema example and Markdown output shape. Expanded the Tests plan with explicit TDD red-first call-outs and the `INTEGRATION=1`-gated postgres/postgres + database-lab harness. Specified team composition and a four-sprint parallelized plan.
- **Spec v0.3 (2026-04-21)** — Post-review r2 refinement addressing Reviewer B's findings. **Contradictions resolved:** introduced the explicit *spec version* vs *product version* distinction; reconciled the JSON schema block (both `$schema` and `root` are documented); stated that the marker regex is built dynamically from `config.markers`. **Ambiguities pinned:** `--staged` is file-level (user story 4 updated to match); `--since`/`--author` are mutually incompatible with `--no-blame` (exit 2) and drop findings whose blame is null; `.gitignore` pruning is delegated to `git check-ignore` inside a worktree and skipped outside; multi-marker block comments yield one Finding per marker line; top-level `blame` is either a full object or the JSON literal `null`, never partial; string-literal tokenization is an explicit non-goal (marker-looking strings on code-only lines are not findings). **Testing strengthened:** black-box CLI tests added for every v0.1 flag and every exit code (0/1/2); the brittle "pinned upstream TODO" assertion replaced with structural invariants against real repos plus a dedicated synthetic bare-git fixture repo; the `<60 s` perf number moved to a separate perf lane on pinned `ubuntu-latest` 4 vCPU hardware with a loose 90 s ceiling, not a correctness gate. Changelog now tracks every revision.
