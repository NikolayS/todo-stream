# todo-stream — SPEC v0.4

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

- **Spec version** — the version of *this document*. Appears only in the `# todo-stream — SPEC vX.Y` header and the embedded changelog. Bumps every review round. Currently **v0.4**.
- **Product version** — the version of the *published npm package and compiled binary*. Appears in `package.json`, the JSON report's `version` field, and `--version` output. Currently **0.1.0** (pre-release; will ship as 0.1.0 when the v0.1 scope below is implemented).

The JSON schema example below therefore shows `"version": "0.1.0"` (product), while this document's header is `SPEC v0.4`.

## User stories

1. **CI gatekeeper — Priya, release engineer.** Priya adds `todo-stream --format json --fail-on FIXME` to the repo's CI workflow so any PR that introduces a new `FIXME` fails the pipeline with a structured diagnostic pointing at file, line, author, and SHA.
2. **Tech-debt triage — Luis, staff engineer.** Luis runs `todo-stream --format markdown --since 2024-01-01 > DEBT.md` against the monorepo once a quarter, grouping findings by file with blame dates, and uses the Markdown report in a planning meeting to assign owners.
3. **Incoming maintainer — Dana, new OSS contributor.** Dana clones a large project (e.g. `postgres/postgres`) and runs `todo-stream src/backend --markers TODO,HACK --format markdown | less` to orient herself: she sees the oldest HACKs, who wrote them, and which files are hot-spots, without learning the project's in-house tooling.
4. **Pre-commit author — Sam, individual developer.** Sam wires `todo-stream --staged --format json` into a `lefthook` pre-commit. `--staged` limits the scan to the *set of files* listed by `git diff --cached --name-only` but reads their **working-tree** bytes (not the staged blob) — consistent with linting tools like ESLint and the pragmatics of pre-commit hooks; documented as a known limitation (see CLI surface).
5. **Pipeline consumer — automated dashboard.** A scheduled job runs `todo-stream --format json` and pipes the output into a downstream ingester that renders a historical chart of open TODO count per file — the JSON schema is stable and documented.

## Scope & non-goals (product v0.1)

**In scope (product v0.1)**
- Recursive walk of a directory with configurable include/exclude globs (defaults respect `.gitignore`).
- Extraction of configurable markers (default: `TODO`, `FIXME`, `HACK`, `XXX`) from line and block comments in common languages (see the language table below). Unknown extensions fall back to a generic regex.
- Git blame enrichment per finding: author name, author email, commit date (ISO-8601 UTC), 10-char short SHA. Non-git trees degrade gracefully (top-level `blame: null`).
- Two output formats: `json` (stable, documented schema) and `markdown` (human-readable, grouped by file).
- Filters: `--markers`, `--since`, `--author`, `--include`, `--exclude`, `--fail-on`, `--staged`.
- Single static Bun-compiled binary + `npx todo-stream` entry point.
- Platforms: macOS (arm64, x86_64) and Linux (x86_64, arm64). Windows is not a v0.1 support target (may work via WSL; unsupported natively).

**Out of scope for product v0.1 (explicitly rejected or deferred)**
- Any persistent storage (SQLite, JSON file, event log). `todo-stream` is stateless — each run re-scans.
- Long-running daemon / watch mode / stdout NDJSON streaming transport.
- Subcommands like `add` / `list` / `done` — this tool does not *manage* todos.
- Multi-device sync, multi-user collaboration.
- A todo data model with `id` / `done` / `created_at` fields — findings are *derived* from source, not stored records.
- Rich parsers (tree-sitter, language servers). Regex over extension-keyed comment syntaxes is sufficient.
- **Environment-variable configuration.** `TODO_STREAM_*` env vars are *not* a supported config source in v0.1; precedence is `CLI flags > todo-stream.config.json > defaults`.
- **String-literal tokenization.** The extractor inspects only the *comment region* of a line; it does not parse string literals. A `TODO` appearing inside a string on a line that is not otherwise a comment is *not* a finding. A `TODO` inside a string *within* a comment (e.g. `// TODO: see "FIXME" in docs`) remains a single `TODO` finding — the inner token is treated as comment text.

### Language table (authoritative; pins `language` field values)

The extractor dispatches on file extension to the following comment-syntax table. The right column is the exact string emitted as the JSON `language` field (stable public surface).

| Extensions | Comment syntax | `language` value |
|---|---|---|
| `.c` `.h` | `//`, `/* */` | `c` |
| `.cc` `.cpp` `.cxx` `.hpp` `.hh` `.hxx` | `//`, `/* */` | `cpp` |
| `.go` | `//`, `/* */` | `go` |
| `.rs` | `//`, `/* */` | `rust` |
| `.ts` `.tsx` `.mts` `.cts` | `//`, `/* */` | `typescript` |
| `.js` `.jsx` `.mjs` `.cjs` | `//`, `/* */` | `javascript` |
| `.py` | `#`, `""" """`, `''' '''` | `python` |
| `.sh` `.bash` `.zsh` | `#` | `shell` |
| `.sql` | `--`, `/* */` | `sql` |
| `.lua` | `--`, `--[[ ]]` | `lua` |
| `.rb` | `#`, `=begin =end` | `ruby` |
| any other | generic fallback | `unknown` |

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
- **CLI (`src/cli.ts`)** — argument parsing (hand-rolled, no deps), config resolution (CLI flags > `todo-stream.config.json` > defaults), dispatch. Owns process exit code.
- **Walker (`src/walker.ts`)** — streams matching file paths. Uses `Bun.Glob` for include/exclude. `.gitignore` pruning is delegated to `git check-ignore --stdin -v` when the root is inside a git worktree; outside a worktree, no `.gitignore` pruning is performed (users may still use `--exclude`). **Symlinks are not followed** (neither files nor directories) — avoids cycles and surprise traversal outside the root; symlinks are simply skipped and not reported. Emits an async iterator of absolute paths; never buffers the whole tree.
- **Extractor (`src/extract.ts`)** — pure function: `(path, bytes, config) → Finding[]`. Dispatches on extension to the language table. Returns `{path, line, column, marker, text, raw}` — no I/O, fully unit-testable from fixtures.
- **Blamer (`src/blame.ts`)** — batches findings per file and shells out to `git blame --porcelain -L a,a -L b,b -- <file>`, parses porcelain output into `{author, email, date, sha}`. Caches by absolute file path (one blame invocation per file). Degrades to top-level `blame: null` outside a git worktree, for untracked files, and for lines reported as uncommitted (see "Uncommitted lines" below).
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
7. `--fail-on <marker>[,<marker>...]` sets exit code 1 if any matching finding exists after filtering; absence of matching findings exits 0. Internal errors (I/O, bad config, spawn failure, usage errors) exit 2.

### Finding extraction algorithm
- The marker regex is **built dynamically** from `config.markers`. Each marker is regex-escaped and joined. Boundaries are symmetric and use a non-identifier character class on both sides (not `\b`, which is defined against `\w` and would misbehave for non-`\w` marker characters):
  ```
  new RegExp(
    "(^|[^A-Za-z0-9_])(" + markers.map(escape).join("|") + ")(?=[^A-Za-z0-9_]|$)[:\\s-]?\\s*(.*)"
  )
  ```
  To keep boundary semantics well-defined, **user-supplied markers must match `^[A-Za-z0-9_]+$`**. Markers outside this class (e.g. `TODO?`, `FIXME-LATER`, CJK) are rejected at config-load time with a usage error (exit 2). Match is case-sensitive; user-supplied markers are used **verbatim** (no upcasing). `--markers todo` therefore matches only lowercase `todo`, not `TODO`.
- For each line, the extractor first reduces the line to its comment region using the language's `CommentSyntax`. It does **not** tokenize string literals; see the Scope section's explicit non-goal. Markers embedded in string literals on code-only lines are not reported.
- **Line and column indexing.** `line` is 1-based (matching `git blame` and most editors). `column` is 1-based and refers to the column of the first character of the matched marker token in the original line.
- **Multi-marker block comments.** Within a single block comment, each *line* that contains a marker yields its own `Finding` at that line's line/column. Continuation lines *between* markers (lines with no marker of their own) are attached as trailing text to the most recent preceding marker in the same block. The continuation joining rule is: each continuation line is stripped of leading block-comment punctuation (`*`, `#`, `--`, `//`) and surrounding whitespace, then concatenated to the preceding marker's `text` with a single `\n` separator. Empty continuation lines are preserved as `\n` (a blank line becomes a single newline in `text`).
- Marker-line text trimming: leading comment punctuation (`*`, `#`, `--`, `//`) and the optional `:` or `-` immediately after the marker token are stripped from the captured `text`, along with surrounding whitespace.
- Unknown extensions use a generic fallback: match markers only when preceded and followed by a non-identifier character (same boundary rule as above); record with `language: "unknown"`.
- **Binary files** are skipped by the walker when any of the first 8 KiB of the file contains a NUL byte; they are not read or reported.

### Filter precedence and blame-dependent flags
- `--since` and `--author` operate on blame metadata.
- **`--since` grammar.** Accepts strictly `YYYY-MM-DD` in v0.1 (other forms, including relative `7d`, are rejected with exit 2). The date is interpreted as **UTC midnight** (`00:00:00Z`). The comparison is **inclusive of the boundary**: a finding with `blame.date >= <since>T00:00:00Z` is kept. The blame date used is the commit author date as emitted by `git blame --porcelain`, normalized to UTC.
- **`--author` matching.** Case-insensitive substring match against the concatenation `author + " <" + email + ">"` — i.e. it matches either the author name or the email, or any substring spanning the two. Multiple substrings via repeated `--author` flags are OR'd.
- If `--no-blame` is passed *together* with `--since` or `--author`, the CLI exits 2 with a usage error (these flags are mutually incompatible).
- When blame is enabled but a particular finding has `blame: null` (e.g. file is untracked, the entire tree is not a git worktree, or the line is uncommitted), applying `--since` or `--author` **drops** that finding from the output. Rationale: the user is filtering on blame data; a finding with no blame data cannot satisfy the filter.
- `--fail-on` is evaluated **after** all filters. A repo with FIXMEs that are all filtered out by `--since` exits 0.
- **`--fail-on` grammar.** Accepts a comma-separated list of markers (e.g. `--fail-on FIXME,HACK`); the flag is not repeatable. The set **must be a subset of `--markers`**; passing `--fail-on FIXME` when `--markers` does not include `FIXME` is a usage error (exit 2), not a silent no-op.

### Blame batching and uncommitted lines
- Group findings by file; run `git blame --porcelain -L a,a -L b,b -- <file>` in a single invocation per file.
- Parse porcelain headers once per unique commit.
- **Short SHA** is the first **10 hex characters** of the 40-char commit SHA returned by `git blame --porcelain`. This width is fixed — not `core.abbrev`-derived — so the JSON field is stable across environments.
- **Uncommitted lines.** When `git blame --porcelain` reports a line as `0000000000000000000000000000000000000000` with author `Not Committed Yet`, that finding's `blame` is emitted as the JSON literal `null` (not a synthesized object). Downstream consumers can distinguish "no git worktree" from "uncommitted line" by the root context; within a single run, the shape of `blame` is uniform: always either the full object or `null`.
- Outside a git worktree (detected via `git rev-parse --is-inside-work-tree`), skip blame entirely and emit `blame: null` per finding.

### JSON schema (stable surface)

The JSON renderer emits exactly this shape. `$schema` points at a hosted JSON-Schema document shipped alongside the binary; `root` is the resolved absolute path of the scan root (useful for downstream consumers correlating multiple runs). `path` is **POSIX-style and relative to `root`** (forward slashes, no leading `./`). When the scan target is a single file, `root` is set to that file's **parent directory** and `path` is the file's basename — preserving the relative-to-`root` invariant. `generated_at` is ISO-8601 UTC with second precision and a trailing `Z`: `YYYY-MM-DDTHH:MM:SSZ`.

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

The `blame` key is either the full object shown above or the JSON literal `null` (never a partially-null object). The schema is considered a stable public surface from product v0.1 onward; additive changes bump the `$schema` URL's version segment.

### Markdown output shape
- `# TODO report — <root> — <generated_at>`
- One `## <relative/path>` section per file, findings as list items: `- **TODO** L42 · Jane Doe, 2024-07-11 (a1b2c3d4e5) — rewrite with streaming parser`.
- The 10-char short SHA in the Markdown line is the same value as `blame.sha` in the JSON (consistent width across formats).

### CLI surface
```
todo-stream [path]              positional root (default: cwd)
  --format json|markdown        default: markdown when stdout isTTY, json otherwise
  --markers TODO,FIXME,HACK     default: TODO,FIXME,HACK,XXX (comma list; each token must match ^[A-Za-z0-9_]+$; used verbatim, case-sensitive)
  --include "**/*.ts"           repeatable
  --exclude "**/dist/**"        repeatable
  --since 2024-01-01            YYYY-MM-DD only; inclusive; UTC; incompatible with --no-blame
  --author <substr>             case-insensitive substring against `author <email>`; repeatable (OR); incompatible with --no-blame
  --fail-on TODO,FIXME          comma list; must be a subset of --markers; exit 1 if any matching finding remains after filters
  --no-gitignore                disable git-delegated .gitignore pruning
  --no-blame                    skip git blame enrichment; top-level `blame: null` for every finding
  --staged                      limit to FILES listed by `git diff --cached --name-only`; reads working-tree bytes (documented limitation)
  --config <path>               load JSON config
  --version | --help
```

Exit codes: `0` (success, no failing findings), `1` (`--fail-on` triggered), `2` (usage or internal error — includes unknown flag, conflicting flags, bad config JSON, invalid marker, invalid `--since` form, `--fail-on` not a subset of `--markers`, spawn failure).

## Tests plan

### Unit tests (fixture-driven, TDD — RED first)
Built **test-first** (red → green):
- **Extractor (`extract.test.ts`)** — a `tests/fixtures/` tree with one small file per supported language containing known markers in line comments, block comments, nested blocks, and code-only lines with marker-looking strings (negative case: a `const s = "TODO"` line must yield zero findings). Each assertion pins exact `(line, column, marker, text)` with 1-based indexing. Additional fixtures:
  - A block comment containing both `TODO:` and `FIXME:` on different lines with continuation text → asserts two Findings, with continuation `\n`-joined on the preceding marker.
  - A non-`\w` marker config → asserts exit 2 at config-load time.
  - `--markers todo` against a file containing `TODO` and `todo` → asserts only `todo` is matched (verbatim case).
- **Walker (`walker.test.ts`)** — fixture tree exercising `--include`/`--exclude`, symlink skipping (cycle-safe: a self-referencing symlink must not hang), and binary-file skipping (a fixture with NUL bytes in the first 8 KiB yields zero findings). `.gitignore` delegation is covered by a small integration-style test that shells out to `git check-ignore` against a tiny fixture repo (checked in as a bare tarball, extracted in the test setup).
- **Renderers (`render.test.ts`)** — golden-file tests: feed a known `Finding[]` → assert byte-exact JSON and stable Markdown. Golden-file cases enumerated:
  - Empty findings list (well-formed JSON, Markdown with header only).
  - `blame: null` finding (top-level null, not partial object).
  - Multi-line `text` with `\n` continuations (JSON escaping + Markdown rendering of the newline).
  - Non-ASCII author name and text (UTF-8 round-trip).
  - Long `text` (no truncation).
  - Two findings on the same line (same file+line, different markers from a block comment).
- **Blame parser (`blame.test.ts`)** — feed canned `git blame --porcelain` output (captured, checked in) → assert parsed records. Includes a canned output for a `0000...` uncommitted line → asserts `blame: null`. No real git invocation here.
- **CLI flags (`cli.test.ts`)** — black-box tests invoking the compiled CLI against fixture repos. Each v0.1 flag gets at least one positive and one negative case:
  - `--fail-on FIXME` with no FIXMEs → exit 0; with one FIXME → exit 1.
  - `--fail-on FOO` when `--markers` does not include `FOO` → exit 2 with a readable error.
  - `--since 2099-01-01` → empty `findings`; `--since 1970-01-01` → all findings; `--since 7d` → exit 2 (unsupported form); `--since 2024-1-1` → exit 2 (strict `YYYY-MM-DD`).
  - `--author nobody` → empty; `--author <real-name-substring>` → non-empty; `--author <email-domain>` → non-empty (covers both fields).
  - `--staged` → only staged-file findings, using working-tree bytes (fixture: stage a file containing `FIXME`, then modify the working tree to remove it — assert the finding disappears, documenting the known limitation).
  - `--config <path>` → flags loaded; bad JSON → exit 2 with readable error.
  - `--no-blame --since 2024-01-01` → exit 2, usage error.
  - `--no-gitignore` → vendored `node_modules` fixture is now visible.
  - `--markers TODO?` → exit 2 (invalid marker form).
  - Unknown flag → exit 2.
  - **No-git-worktree degradation:** scan a freshly-created `mktemp -d` directory containing one `FIXME` file → exit 0 (or 1 under `--fail-on`), every finding has top-level `blame: null`, JSON validates against the schema.

### Integration tests (real git, CI)
Built after the unit suite is green. Both integration clones are **pinned to specific commit SHAs** (not `--depth=1` of floating HEAD) so upstream drift cannot flip CI red:
- **`postgres/postgres`** — checkout at SHA `PG_PIN_SHA` (concrete SHA recorded in the test harness config, updated manually and reviewed), shallow bounded to `src/backend/access/`. Invariants: exits 0; JSON validates against the published schema; **no runtime error**; `findings.length >= 1` (sanity floor only); every finding has a non-null `blame` object; runtime well-behaved on the perf lane (see below).
- **`postgres-ai/database-lab`** — checkout at SHA `DBLAB_PIN_SHA`, full-repo scan. Invariants: exits 0; JSON validates; `findings.length >= 1`; every finding has a non-null `blame` object.
- **Count-sensitive and text-sensitive assertions live on the synthetic fixture repo**, not the real clones. The synthetic repo under `tests/fixtures/synthetic-repo/` (a checked-in bare-git tarball with deterministic authors, dates, and SHAs) pins `(path, line, marker, text, blame.sha)` quintuples; it is the authoritative coverage for end-to-end blame enrichment correctness.
- Both real-repo integration tests are gated behind `INTEGRATION=1` env so local `bun test` stays fast; CI sets it.

### Performance lane (separate, not a correctness gate)
- Runs in its own CI job on **GitHub-hosted `ubuntu-latest` (4 vCPU, 16 GB RAM)**.
- Scans the pinned `postgres/postgres` subtree; records wall-clock in a CSV artifact and **fails only if runtime exceeds 90 s** (coarse ceiling / liveness gate). This is a hard ceiling, not a trend-tracking gate — the CSV is an artifact for humans to inspect, not compared to prior runs automatically. (A baseline-regression gate is deferred to a later version.)

### CI matrix
- `bun test` (unit + black-box CLI) on every PR — must pass.
- `INTEGRATION=1 bun test` nightly and on `main` — must pass.
- Performance lane nightly only.
- `bun build --compile` smoke: compile, run `--help`, run against the repo itself.
- Typecheck: `bun tsc --noEmit`.
- Lint: `biome check` (dev-only).

### TDD call-out (explicit)
- **Test-first (RED → GREEN):** extractor (including boundary-regex edge cases and the reject-non-`\w`-marker path), walker (including symlink skip and binary skip), renderers (including every enumerated golden case), blame-porcelain parser (including the `0000...` uncommitted case), **every v0.1 CLI flag's black-box behavior and exit code** including the no-git-worktree degradation. These are deterministic and fixture-driven — write failing tests before any implementation line.
- **Test-after:** real-repo integration tests, the compile/smoke step, the perf lane. These exercise external systems where test-first yields diminishing returns.

## Team

Veteran experts to hire:
- **Veteran CLI systems engineer (1)** — owns walker, CLI surface, Bun packaging, exit-code semantics, symlink/binary-file policy.
- **Veteran parser/regex engineer (1)** — owns the extractor, comment-syntax table, boundary regex, and edge-case handling (marker-looking strings on code lines, nested blocks, multi-marker blocks, continuation joining).
- **Veteran git-plumbing engineer (1)** — owns the blamer and the `git check-ignore` delegation: porcelain parsing, batching, 10-char SHA normalization, uncommitted-line detection, graceful degradation, and worktree detection.
- **Veteran test engineer (1)** — owns fixture design, golden files, the synthetic-repo bare-git fixture, black-box CLI suite, pinned-SHA integration harness, and the postgres/postgres + database-lab integration in CI.
- **Veteran TypeScript/Bun release engineer (1, part-time)** — owns `package.json`, `bin` entry, `bun build --compile`, npm publish workflow, semver discipline, and the hosted JSON-Schema document.

Total: 4 full-time + 1 part-time.

## Implementation plan

Sprints are one week each. `⇄` = work happens in parallel; `→` = ordering dependency.

### Sprint 1 — Skeleton & red tests
- CLI engineer: scaffold repo layout, `bin/todo-stream`, `--help`, `--version`, argv parser; author red tests for flag-grammar error paths (invalid `--since`, invalid marker, unknown flag, `--fail-on` not a subset of `--markers`, `--no-blame` + `--since`). ⇄
- Parser engineer: author fixture tree under `tests/fixtures/langs/` and write **failing** extractor tests for every language in the language table, including the multi-marker-block, continuation-joining, marker-in-string-on-code-line, non-`\w`-marker-rejection, and verbatim-case cases. ⇄
- Git engineer: capture canned `git blame --porcelain` outputs (including a `0000...` uncommitted case) into `tests/fixtures/blame/` and write **failing** parser tests for 10-char SHA normalization and uncommitted→null. ⇄
- Test engineer: stand up `bun test` in CI, add the `INTEGRATION=1` gate, scaffold black-box CLI test harness (invokes the compiled binary), write **failing** CLI tests for every v0.1 flag, every exit-code case, and the no-git-worktree degradation. Also stand up the pinned-SHA synthetic fixture repo. ⇄
- Release engineer: lock Bun version, add `bun tsc --noEmit` + `biome check` to CI.
- **Gate:** all tests RED, CI green on lint/type.

### Sprint 2 — Core green
- Parser engineer → implement `extract.ts` (with dynamic marker regex, symmetric non-identifier boundaries, and the `^[A-Za-z0-9_]+$` marker validator) until fixtures pass.
- CLI engineer → implement `walker.ts` with `Bun.Glob` + `git check-ignore` delegation + symlink-skip + binary-skip, write walker tests RED→GREEN. ⇄
- Git engineer → implement blame porcelain parser (pure), including the `0000...`→null path and 10-char SHA truncation, until unit tests pass. ⇄
- Test engineer → author golden-file tests for both renderers across every enumerated case (still red; renderers not yet built); finalize the synthetic bare-git fixture repo.
- **Gate:** extractor, walker, blame parser all GREEN; renderer and CLI tests still RED.

### Sprint 3 — Render & wire
- CLI engineer → implement JSON + Markdown renderers against golden tests (POSIX-relative `path`, pinned `generated_at` format, 10-char SHA); wire end-to-end pipeline in `cli.ts`; implement exit-code semantics and all usage-error paths.
- Git engineer → implement `blame.ts` runtime (spawn `git blame`, batch per file, no-git fallback emitting `blame: null`, uncommitted→null). ⇄
- Parser engineer → extend markers config + `--markers` flag (dynamic regex, verbatim-case, marker-shape validator), add generic fallback extraction. ⇄
- Test engineer → start integration harness: pinned-SHA clone helper, postgres subtree scan with structural invariants only.
- **Gate:** renderer tests and a majority of CLI flag tests GREEN; `todo-stream` runs end-to-end against the repo itself and produces valid output.

### Sprint 4 — Integration & release
- Test engineer → finalize integration tests (pinned-SHA postgres/postgres + postgres-ai/database-lab with structural invariants only); stand up the separate perf lane on `ubuntu-latest` 4 vCPU with the 90 s ceiling. ⇄
- Release engineer → `bun build --compile` pipeline, npm publish dry-run, `npx todo-stream` smoke, README usage + JSON schema docs, publish the hosted `$schema` JSON, declare macOS + Linux support matrix. ⇄
- CLI engineer → implement `--since` (strict `YYYY-MM-DD`, UTC, inclusive), `--author` (case-insensitive substring across `author <email>`, repeatable), `--fail-on` (comma list, subset-of-markers validation), `--staged` (file-level, working-tree read), `--config`, plus the `--no-blame` + `--since`/`--author` mutual-exclusion guard. ⇄
- Git engineer → hardening: large-file guard, symlink handling in blame, uncommitted-line regression tests against live git fixtures. ⇄
- Parser engineer → extend language coverage (finalize Rust, Ruby, Lua) with fresh red→green fixtures matching the language table.
- **Gate:** all unit + black-box CLI tests GREEN; integration tests GREEN on CI against pinned SHAs; perf lane reporting; `bun run build` emits a working static binary; product v0.1.0 tagged.

### Parallelization summary
- Sprint 1 is almost fully parallel (five independent red-test authoring streams, including black-box CLI and error-path grammar).
- Sprint 2 and 3 pair the parser+git engineers in parallel while the CLI engineer advances the pipeline; the test engineer stays one step ahead writing the next red tests.
- Sprint 4 fans out: every engineer has an independent lane.

## Embedded Changelog

- **Spec v0.1 (2026-04-21)** — Initial draft. Reframed away from the "todo-list app" interview answers (SQLite, NDJSON daemon, add/list/done subcommands, id/done/created_at model) per the authoritative idea: linter for TODO/FIXME/HACK comments, not a storage tool. Locked scope to directory walk + regex extract + git blame + JSON/Markdown report. Zero runtime deps beyond Bun built-ins.
- **Spec v0.2 (2026-04-21)** — Post-review r1 refinement. Strengthened user stories (added CI gatekeeper, pre-commit, pipeline-consumer personas). Tightened the Architecture diagram and component boundaries. Added the concrete JSON schema example and Markdown output shape. Expanded the Tests plan with explicit TDD red-first call-outs and the `INTEGRATION=1`-gated postgres/postgres + database-lab harness. Specified team composition and a four-sprint parallelized plan.
- **Spec v0.3 (2026-04-21)** — Post-review r2 refinement addressing Reviewer B's findings. Introduced the spec-version vs product-version distinction; reconciled JSON schema prose and example; stated dynamic marker regex; pinned `--staged` to file-level; made `--since`/`--author` incompatible with `--no-blame`; replaced the pinned-upstream-TODO assertion with structural invariants plus a synthetic fixture; moved the perf number to a separate perf lane; declared string-literal tokenization a non-goal; specified multi-marker block handling; delegated `.gitignore` to `git check-ignore`; pinned `blame: null` wholesale.
- **Spec v0.4 (2026-04-21)** — Post-review r3 refinement addressing Reviewer B's findings. **Regex & markers:** symmetric non-identifier boundaries (no `\b`); user markers restricted to `^[A-Za-z0-9_]+$` (else exit 2); verbatim case (not upcased). **CLI grammar pinned:** `--since` is strict `YYYY-MM-DD`, UTC, inclusive; `--author` is case-insensitive substring across `author <email>`, repeatable; `--fail-on` is a comma list that must be a subset of `--markers`. **JSON surface pinned:** `path` is POSIX-style and relative to `root` (root is parent dir when scanning a single file); short SHA is fixed at 10 chars; `generated_at` is `YYYY-MM-DDTHH:MM:SSZ`; `language` values enumerated in a language table; env-var config removed from precedence. **Semantics pinned:** `--staged` reads working-tree bytes (documented limitation); uncommitted `0000...` blame → `blame: null`; symlinks skipped (not followed); binary files detected by NUL byte in first 8 KiB; block-comment continuation joined with `\n` after stripping leading punctuation; 1-based line/column. **Testing strengthened:** integration clones pinned to specific SHAs; no-git-worktree black-box test added; renderer golden-file edge cases enumerated (empty, null-blame, multi-line, non-ASCII, long text, same-line duplicates); perf lane framed as coarse ceiling, not trend-tracking. **Platforms:** macOS + Linux declared; Windows not a v0.1 support target.
