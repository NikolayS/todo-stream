# todo-stream — SPEC v0.5

## Goal & why it's needed

`todo-stream` is a CLI **linter for outstanding in-code comments** (TODO / FIXME / HACK / XXX). It walks a directory tree, extracts these markers from source files, enriches each finding with `git blame` metadata (author, commit author-date, SHA), and emits a grouped report as JSON or Markdown.

**Why it's needed.** Every mature codebase accumulates `TODO`s that nobody owns. `grep -rn TODO` loses authorship and timing; GitHub's "Issues" tab loses the exact line of code. Existing tools (`leasot`, `notes`, IDE plugins) either don't run in CI, don't attach blame, or pull in heavy runtime dependencies. Engineering leads and release managers need a fast, dependency-light, pipeable report that answers: *which open TODOs does this repo carry, who wrote them, and how stale are they?* — suitable for CI gating, release-readiness dashboards, and tech-debt triage.

**Explicit non-goals (honored strictly).**
- This is **NOT a todo-list application**. It does not create, edit, or complete tasks.
- This is **NOT a CRUD storage tool**. It has no database, no persistent state, no server.
- It does not mutate source files.
- It does not replace an issue tracker; it *surfaces* comments so humans can decide whether to file issues.

## Versioning convention

Two distinct versions appear in this document and must never be conflated:

- **Spec version** — the version of *this document*. Appears only in the `# todo-stream — SPEC vX.Y` header and the embedded changelog. Bumps every review round. Currently **v0.5**.
- **Product version** — the version of the *published npm package and compiled binary*. Appears in `package.json`, the JSON report's `version` field, and `--version` output. Currently **0.1.0** (pre-release; will ship as 0.1.0 when the v0.1 scope below is implemented).

The JSON schema example below therefore shows `"version": "0.1.0"` (product), while this document's header is `SPEC v0.5`.

## User stories

1. **CI gatekeeper — Priya, release engineer.** Priya adds `todo-stream --format json --fail-on FIXME` to the repo's CI workflow so any PR that introduces a new `FIXME` fails the pipeline with a structured diagnostic pointing at file, line, author, and SHA.
2. **Tech-debt triage — Luis, staff engineer.** Luis runs `todo-stream --format markdown --since 2024-01-01 > DEBT.md` against the monorepo once a quarter, grouping findings by file with blame dates, and uses the Markdown report in a planning meeting to assign owners.
3. **Incoming maintainer — Dana, new OSS contributor.** Dana clones a large project (e.g. `postgres/postgres`) and runs `todo-stream src/backend --markers TODO,HACK --format markdown | less` to orient herself: she sees the oldest HACKs, who wrote them, and which files are hot-spots, without learning the project's in-house tooling.
4. **Pre-commit author — Sam, individual developer.** Sam wires `todo-stream --staged --format json` into a `lefthook` pre-commit. `--staged` limits the scan to the *set of files* listed by `git diff --cached --name-only` but reads their **working-tree** bytes (not the staged blob) — consistent with linting tools like ESLint and the pragmatics of pre-commit hooks; documented as a known limitation (see CLI surface).
5. **Pipeline consumer — automated dashboard.** A scheduled job runs `todo-stream --format json` and pipes the output into a downstream ingester that renders a historical chart of open TODO count per file — the JSON schema is stable and documented.

## Scope & non-goals (product v0.1)

**In scope (product v0.1)**
- Recursive walk of a directory with configurable include/exclude globs (defaults respect `.gitignore`).
- Extraction of configurable markers (default: `TODO`, `FIXME`, `HACK`, `XXX`) from line and block comments in common languages (see the language table below). Unknown extensions fall back to a generic regex with documented caveats.
- Git blame enrichment per finding: author name, author email, commit **author-date** (ISO-8601 UTC), 10-char short SHA. Non-git trees degrade gracefully (per-finding `blame: null`).
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
- **Environment-variable configuration.** `TODO_STREAM_*` env vars are *not* a supported config source in v0.1; precedence is `CLI flags > --config file > defaults`.
- **Config file auto-discovery.** v0.1 does **not** auto-discover `todo-stream.config.json` from cwd or walk up from the scan root. A config file is loaded **only** when `--config <path>` is passed explicitly. (Auto-discovery is deferred to a later version.)
- **String-literal tokenization.** The extractor inspects only the *comment region* of a line (for languages in the language table); it does not parse string literals. A `TODO` appearing inside a string on a line that is not otherwise a comment is *not* a finding in known languages. A `TODO` inside a string *within* a comment (e.g. `// TODO: see "FIXME" in docs`) remains a single `TODO` finding — the inner token is treated as comment text. For the **generic (unknown-extension) fallback**, no comment-region reduction is performed; see the fallback caveat in Implementation details.

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
- **CLI (`src/cli.ts`)** — argument parsing (hand-rolled, no deps), config resolution (CLI flags > `--config` file > defaults), dispatch. Owns process exit code.
- **Walker (`src/walker.ts`)** — streams matching file paths. Uses `Bun.Glob` for include/exclude. `.gitignore` pruning is delegated to `git check-ignore --stdin -v` when the root is inside a git worktree; outside a worktree, no `.gitignore` pruning is performed (users may still use `--exclude`). **Symlinks are not followed** during traversal (neither files nor directories) — avoids cycles and surprise traversal outside the root; symlinks are skipped and not reported. **Root-path symlinks** (the positional `path` argument itself) are resolved exactly once at startup to their canonical absolute path, and traversal proceeds from that canonical path; the resolved path is used as the `root` field in the JSON output. The walker **deduplicates paths by resolved absolute path** before emitting — a path matched by multiple `--include` globs is yielded only once; the extractor therefore runs at most once per file. Emits an async iterator; never buffers the whole tree.
- **Extractor (`src/extract.ts`)** — pure function: `(path, bytes, config) → Finding[]`. Dispatches on extension to the language table. Returns `{path, line, column, marker, text, raw}` — no I/O, fully unit-testable from fixtures.
- **Blamer (`src/blame.ts`)** — batches findings per file and shells out to `git blame --porcelain -L a,a -L b,b -- <file>`, parses porcelain output into `{author, email, date, sha}`. `date` is always the commit **author-date** (`author-time` + `author-tz` from porcelain), normalized to ISO-8601 UTC with second precision. Caches by absolute file path (one blame invocation per file, subject to ARG_MAX chunking below). Degrades to per-finding `blame: null` outside a git worktree, for untracked files, and for lines reported as uncommitted (see "Uncommitted lines" below).
- **Renderer (`src/render/*.ts`)** — two pure renderers: `json.ts` (emits the versioned schema defined below) and `markdown.ts` (groups by file, sorts by line then column).
- **Reporter (in `cli.ts`)** — writes to stdout, applies `--fail-on` rules, sets exit code.

**Key abstractions**
- `Finding` — the one shared record shape crossing every boundary.
- `CommentSyntax` — `{ line?: string[]; block?: [open, close][] }`, looked up by extension.
- `Config` — frozen object assembled once at startup.

**Dependency policy.** Runtime deps = zero beyond Bun built-ins (`Bun.Glob`, `Bun.spawn`, `Bun.file`, `fs`, `path`). `git` itself is assumed on PATH (required for blame and `.gitignore` delegation; the tool degrades without it as documented). Dev deps are limited to `bun test`, `@biomejs/biome` (lint + format), and type definitions.

## Implementation details

### Data flow
1. `cli.ts` parses argv → `Config`. If `--config <path>` is passed, the JSON file is merged under CLI flags per the precedence rule (CLI flags > --config file > defaults). No auto-discovery.
2. `walker.ts` yields file paths matching globs, pruning via `git check-ignore` (inside a worktree) and `--exclude`. **Ordering of `--include` and `--exclude`: exclude wins.** A path matched by any `--exclude` glob is dropped even if it is also matched by `--include`. The order in which `--include` and `--exclude` appear on the argv does not matter. Paths are yielded deduplicated by resolved absolute path.
3. For each path, `Bun.file(path).text()` → `extract.ts` → `Finding[]` (streamed, not accumulated repo-wide).
4. Findings are grouped by file and handed to `blame.ts`, which issues one or more `git blame` invocations per file (see ARG_MAX chunking below) with all needed line ranges.
5. Filters `--since` and `--author` are applied **after** blame enrichment (see precedence rules below).
6. Enriched findings feed the chosen renderer; output is written to stdout.
7. `--fail-on <marker>[,<marker>...]` sets exit code 1 if any matching finding exists after filtering; absence of matching findings exits 0. Internal errors (I/O, bad config, spawn failure, usage errors) exit 2.

### Finding extraction algorithm
- The marker regex is **built dynamically** from `config.markers`. Each marker is regex-escaped and joined. Boundaries are symmetric and use a non-identifier character class on both sides (not `\b`, which is defined against `\w` and would misbehave for non-`\w` marker characters):
  ```
  new RegExp(
    "(^|[^A-Za-z0-9_])(" + markers.map(escape).join("|") + ")(?=[^A-Za-z0-9_]|$)[:\\s-]?\\s*(.*)",
    "g"
  )
  ```
  The regex is executed with the global flag so multiple marker occurrences on a single line are all detected. To keep boundary semantics well-defined, **user-supplied markers must match `^[A-Za-z0-9_]+$`**. Markers outside this class (e.g. `TODO?`, `FIXME-LATER`, CJK) are rejected at config-load time with a usage error (exit 2). Match is case-sensitive; user-supplied markers are used **verbatim** (no upcasing). `--markers todo` therefore matches only lowercase `todo`, not `TODO`.
- For each line of a file with a **known** extension (see the language table), the extractor first reduces the line to its comment region using the language's `CommentSyntax`. It does **not** tokenize string literals; see the Scope section's explicit non-goal. Markers embedded in string literals on code-only lines of known languages are not reported.
- **Generic (unknown-extension) fallback caveat.** For files with extensions not in the language table, the extractor runs the boundary regex against the full line without comment-region reduction. This means a `TODO` appearing inside a string literal in an unknown file **will** be reported, while the same construct in a `.ts` file will not. This asymmetry is intentional (and called out in Scope) because the tool has no language model for unknown formats; users who want to suppress false positives for a specific format should either add the extension to the language table upstream or exclude those files via `--exclude`.
- **Line and column indexing.** `line` is 1-based (matching `git blame` and most editors). `column` is 1-based and refers to the column of the first character of the matched marker token in the original line.
- **Multi-marker handling (authoritative).** **Each marker *occurrence* yields its own Finding** — not each line. A single line (inside a comment, whether block or line-style) containing two markers produces two Findings with identical `path` and `line` but distinct `column` values; a block comment spanning multiple lines yields one Finding per marker occurrence across those lines. Findings are emitted in source order: primarily by `line` ascending, secondarily by `column` ascending.
- **Continuation-text attachment (authoritative algorithm).** Inside a *block comment* of a known language, any line between two marker occurrences — or after the final marker occurrence within that block — that contains **no** marker occurrence is a **continuation line** and is attached to the immediately preceding marker occurrence's `text`. The algorithm is:
  1. Strip the block-comment's own punctuation: `/*` / `*/` delimiters on the opening/closing lines, and leading `*` (C-family block comments), leading `#` (Python triple-quoted blocks), leading `--` (SQL / Lua block comments), and surrounding whitespace on continuation lines. Line-comment tokens (e.g. `//`) are *not* stripped because they cannot validly appear as block-comment continuation punctuation.
  2. If the resulting continuation line is **empty**, represent it as a single empty segment.
  3. Concatenate the preceding marker's `text` with each continuation segment using a single `\n` between segments. Consecutive empty continuations therefore produce consecutive `\n` characters (two blank continuation lines → `\n\n` between the surrounding text segments).
  
  *Example (C-family block comment):*
  ```
  /* TODO: refactor
   * first point
   *
   * second point
   * FIXME: later
   *   continued
   */
  ```
  Produces two Findings:
  - `{marker:"TODO", line:1, column:4, text:"refactor\nfirst point\n\nsecond point"}`
  - `{marker:"FIXME", line:5, column:6, text:"later\ncontinued"}`
- Marker-line text trimming: within the marker-line itself, the optional `:` or `-` immediately after the marker token is stripped from the captured `text`, along with surrounding whitespace and the leading block-comment punctuation of that line (same stripping rules as step 1 above).
- Unknown extensions use the generic fallback regex (see caveat above); record with `language: "unknown"`.
- **Binary files** are skipped by the walker when any of the first 8 KiB of the file contains a NUL byte; they are not read or reported.

### Filter precedence and blame-dependent flags
- `--since` and `--author` operate on blame metadata.
- **`--since` grammar.** Accepts strictly `YYYY-MM-DD` in v0.1 (other forms, including relative `7d`, are rejected with exit 2). The date is interpreted as **UTC midnight** (`00:00:00Z`). The comparison is **inclusive of the boundary**: a finding with `blame.date >= <since>T00:00:00Z` is kept. The blame date used is the commit **author-date** as emitted by `git blame --porcelain` (`author-time` + `author-tz`), normalized to UTC — identical to the value emitted in `blame.date`.
- **`--author` matching.** Case-insensitive substring match against the concatenation `author + " <" + email + ">"` — i.e. it matches either the author name or the email, or any substring spanning the two. Multiple substrings via repeated `--author` flags are OR'd.
- If `--no-blame` is passed *together* with `--since` or `--author`, the CLI exits 2 with a usage error (these flags are mutually incompatible).
- When blame is enabled but a particular finding has `blame: null` (e.g. file is untracked, the entire tree is not a git worktree, or the line is uncommitted), applying `--since` or `--author` **drops** that finding from the output. Rationale: the user is filtering on blame data; a finding with no blame data cannot satisfy the filter.
- `--fail-on` is evaluated **after** all filters. A repo with FIXMEs that are all filtered out by `--since` exits 0.
- **`--fail-on` grammar.** Accepts a comma-separated list of markers (e.g. `--fail-on FIXME,HACK`). The flag is **not repeatable**: passing `--fail-on` more than once on the argv is a usage error (exit 2), not last-wins. The set must be a subset of `--markers`; passing `--fail-on FIXME` when `--markers` does not include `FIXME` is a usage error (exit 2), not a silent no-op.

### Blame batching, ARG_MAX, and uncommitted lines
- Group findings by file; run `git blame --porcelain -L a,a -L b,b -- <file>` in a single invocation per file **when the total argv size is safe**.
- **ARG_MAX guard.** If a single file accumulates more than **512 `-L` ranges** (a conservative ceiling well under POSIX `ARG_MAX` on all supported platforms), the blamer **chunks** the ranges into batches of up to 512 and issues multiple `git blame` invocations per file, merging the parsed results. The chunk size is a constant in `blame.ts`, not user-configurable in v0.1. This behavior is tested (see Tests plan).
- Parse porcelain headers once per unique commit within a file's blame output.
- **Short SHA** is the first **10 hex characters** of the 40-char commit SHA returned by `git blame --porcelain`. This width is fixed — not `core.abbrev`-derived — so the JSON field is stable across environments.
- **Uncommitted lines.** When `git blame --porcelain` reports a line as `0000000000000000000000000000000000000000` with author `Not Committed Yet`, that finding's `blame` is emitted as the JSON literal `null` (not a synthesized object). Downstream consumers can distinguish "no git worktree" from "uncommitted line" by the root context; within a single run, the shape of `blame` is uniform: always either the full object or `null`.
- Outside a git worktree (detected via `git rev-parse --is-inside-work-tree`), skip blame entirely and emit per-finding `blame: null`.

### JSON schema (stable surface)

The JSON renderer emits exactly this shape. `$schema` points at a hosted JSON-Schema document shipped alongside the binary; `root` is the resolved absolute path of the scan root (useful for downstream consumers correlating multiple runs). `path` is **POSIX-style and relative to `root`** (forward slashes, no leading `./`). When the scan target is a single file, `root` is set to that file's **parent directory** and `path` is the file's basename — preserving the relative-to-`root` invariant. `generated_at` is ISO-8601 UTC with second precision and a trailing `Z`: `YYYY-MM-DDTHH:MM:SSZ`. `blame.date` is the commit **author-date** in the same ISO-8601 UTC format.

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
- The date shown in Markdown is the **author-date** (same source as `blame.date`), rendered as `YYYY-MM-DD` in UTC.
- The 10-char short SHA in the Markdown line is the same value as `blame.sha` in the JSON (consistent width across formats).

### CLI surface
```
todo-stream [path]              positional root (default: cwd); symlink roots are canonicalized once at startup
  --format json|markdown        default: markdown when stdout isTTY, json otherwise
  --markers TODO,FIXME,HACK     default: TODO,FIXME,HACK,XXX (comma list; each token must match ^[A-Za-z0-9_]+$; used verbatim, case-sensitive)
  --include "**/*.ts"           repeatable
  --exclude "**/dist/**"        repeatable; exclude wins over include
  --since 2024-01-01            YYYY-MM-DD only; inclusive; UTC; incompatible with --no-blame; compared against commit author-date
  --author <substr>             case-insensitive substring against `author <email>`; repeatable (OR); incompatible with --no-blame
  --fail-on TODO,FIXME          comma list; NOT repeatable (exit 2 on repeat); must be a subset of --markers; exit 1 if any matching finding remains after filters
  --no-gitignore                disable git-delegated .gitignore pruning
  --no-blame                    skip git blame enrichment; per-finding `blame: null` for every finding
  --staged                      limit to FILES listed by `git diff --cached --name-only`; reads working-tree bytes (documented limitation); requires a git worktree (exit 2 otherwise); files staged-but-deleted from the working tree are silently skipped
  --config <path>               load JSON config explicitly (no auto-discovery in v0.1)
  --version | --help
```

Exit codes: `0` (success, no failing findings), `1` (`--fail-on` triggered), `2` (usage or internal error — includes unknown flag, conflicting flags, bad config JSON, invalid marker, invalid `--since` form, `--fail-on` not a subset of `--markers`, `--fail-on` repeated, `--staged` outside a git worktree, spawn failure).

## Tests plan

### Unit tests (fixture-driven, TDD — RED first)
Built **test-first** (red → green):
- **Extractor (`extract.test.ts`)** — a `tests/fixtures/` tree with one small file per supported language containing known markers in line comments, block comments, nested blocks, and code-only lines with marker-looking strings (negative case: a `const s = "TODO"` line in a `.ts` file must yield zero findings; a `const s = "TODO"` line in a `.unknownext` file **must** yield one finding, documenting the generic-fallback asymmetry). Each assertion pins exact `(line, column, marker, text)` with 1-based indexing. Additional fixtures:
  - A block comment containing both `TODO:` and `FIXME:` on **different** lines with continuation text → asserts two Findings, continuation `\n`-joined on the preceding marker per the continuation algorithm (including the blank-line `\n\n` case).
  - A **single line** containing two markers (e.g. `// TODO fix this; FIXME also this`) → asserts two Findings with identical `line`, distinct `column`, ordered ascending by `column`.
  - A non-`\w` marker config → asserts exit 2 at config-load time.
  - `--markers todo` against a file containing `TODO` and `todo` → asserts only `todo` is matched (verbatim case).
- **Walker (`walker.test.ts`)** — fixture tree exercising `--include`/`--exclude` (including overlap: same path matched by two `--include` globs yields one Finding, not two), **exclude-wins ordering** (path matching both `--include "**/*.ts"` and `--exclude "**/generated/**"` is dropped regardless of flag order on argv), root-path symlink canonicalization (a symlink passed as positional root is resolved once; `root` in output is the canonical path), internal symlink skipping (cycle-safe: a self-referencing symlink must not hang), and binary-file skipping (a fixture with NUL bytes in the first 8 KiB yields zero findings). `.gitignore` delegation is covered by a small integration-style test that shells out to `git check-ignore` against a tiny fixture repo (checked in as a bare tarball, extracted in the test setup).
- **Renderers (`render.test.ts`)** — golden-file tests: feed a known `Finding[]` → assert byte-exact JSON and stable Markdown. Golden-file cases enumerated:
  - Empty findings list (well-formed JSON, Markdown with header only).
  - `blame: null` finding (top-level null on the `blame` key, not partial object).
  - Multi-line `text` with `\n` continuations (JSON escaping + Markdown rendering of the newline).
  - Blank-continuation-line case producing `\n\n` inside `text`.
  - Non-ASCII author name and text (UTF-8 round-trip).
  - Long `text` (no truncation).
  - **Two findings on the same (file, line)** from two markers on one comment line — distinct `column` values, ordered by column ascending (this is the multi-marker-per-line case, not the per-line-in-block case).
- **Blame parser (`blame.test.ts`)** — feed canned `git blame --porcelain` output (captured, checked in) → assert parsed records, including: author-time used as `date` (distinct from committer-time in the fixture to prove selection), 10-char SHA truncation, and a `0000...` uncommitted line → `blame: null`. No real git invocation here.
- **Blame ARG_MAX chunking (`blame_chunk.test.ts`)** — synthesize a `Finding[]` with **>512** distinct lines against a single fixture file; assert the blamer issues ≥2 spawn calls (mock/inject spawn), merges results correctly, and produces no duplicates. Assert the per-invocation `-L` range count never exceeds 512.
- **CLI flags (`cli.test.ts`)** — black-box tests invoking the compiled CLI against fixture repos. Each v0.1 flag gets at least one positive and one negative case:
  - `--fail-on FIXME` with no FIXMEs → exit 0; with one FIXME → exit 1.
  - `--fail-on FOO` when `--markers` does not include `FOO` → exit 2 with a readable error.
  - `--fail-on FIXME --fail-on HACK` (repeated) → exit 2 with a readable error.
  - `--since 2099-01-01` → empty `findings`; `--since 1970-01-01` → all findings; `--since 7d` → exit 2 (unsupported form); `--since 2024-1-1` → exit 2 (strict `YYYY-MM-DD`).
  - `--author nobody` → empty; `--author <real-name-substring>` → non-empty; `--author <email-domain>` → non-empty (covers both fields).
  - `--include "**/*.ts"` alone → only `.ts` findings; unknown glob (no matches) → empty findings, exit 0.
  - `--exclude "**/generated/**"` alone → default scan minus excluded tree.
  - `--include "**/*.ts" --exclude "**/generated/**"` → exclude-wins: a `.ts` file under `generated/` is dropped.
  - `--exclude X --include Y` with the same globs in opposite argv order → identical output (order-independence).
  - `--staged --include "**/*.ts"` → intersection: only staged `.ts` files scanned.
  - `--staged` in a freshly-created `mktemp -d` outside any git worktree → exit 2 with a readable error.
  - `--staged` with a file staged for addition but **deleted** from the working tree → that file silently skipped; remaining staged files processed normally.
  - `--config <path>` → flags loaded; bad JSON → exit 2 with readable error; nonexistent path → exit 2.
  - `--no-blame --since 2024-01-01` → exit 2, usage error.
  - `--no-gitignore` → vendored `node_modules` fixture is now visible.
  - `--markers TODO?` → exit 2 (invalid marker form).
  - Unknown flag → exit 2.
  - **No-git-worktree degradation:** create a test directory under a location guarded by **`GIT_CEILING_DIRECTORIES`** (set to the test harness's tmp-root) so that any ambient git worktree in TMPDIR is ignored; place one `FIXME` file inside; assert exit 0 (or 1 under `--fail-on`), every finding has per-finding `blame: null`, JSON validates against the schema. The test setup explicitly verifies `git rev-parse --is-inside-work-tree` returns non-zero before proceeding, otherwise the test fails fast with a diagnostic (not a flaky pass).

### Integration tests (real git, CI)
Built after the unit suite is green. Both integration clones are **pinned to specific commit SHAs** (not `--depth=1` of floating HEAD) so upstream drift cannot flip CI red:
- **`postgres/postgres`** — checkout at SHA `PG_PIN_SHA` (concrete SHA recorded in the test harness config, updated manually and reviewed), shallow bounded to `src/backend/access/`. Invariants:
  1. Exits 0.
  2. **JSON output validates against the published `$schema`** using a checked-in JSON-Schema validator harness.
  3. `findings.length >= 1` (sanity floor).
  4. Every finding has a non-null `blame` object.
  5. **Each default marker in `{TODO, FIXME, HACK, XXX}` that appears *at all* in the subtree (verified by an independent `grep -r` probe at test startup) has at least one finding in the output** — catches extractor silently dropping a marker kind.
  6. Runtime well-behaved on the perf lane (see below).
  - *Count-tolerance gating is deliberately not added here:* the synthetic fixture repo (below) covers end-to-end blame-enrichment count correctness deterministically, and a `±N%` tolerance on real upstream counts is either too loose to catch real regressions or too tight to avoid flakiness from manual SHA re-pinning. Justification is recorded here so future reviewers don't re-raise the suggestion.
- **`postgres-ai/database-lab`** — checkout at SHA `DBLAB_PIN_SHA`, full-repo scan. Invariants 1–5 as above.
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
- Lint: `biome check` (dev-only; `@biomejs/biome` is a declared dev dependency).

### TDD call-out (explicit)
- **Test-first (RED → GREEN):** extractor (including boundary-regex edge cases, multi-marker-per-line, continuation-blank-line, generic-fallback asymmetry, and the reject-non-`\w`-marker path), walker (including exclude-wins ordering, dedup across overlapping includes, root-path symlink canonicalization, internal symlink skip, and binary skip), renderers (including every enumerated golden case), blame-porcelain parser (including the `0000...` uncommitted case and author-time selection), blame ARG_MAX chunking, **every v0.1 CLI flag's black-box behavior and exit code** including `--include`/`--exclude` and their interactions, `--staged` outside a worktree, `--staged` with deleted-from-worktree files, and the no-git-worktree degradation under `GIT_CEILING_DIRECTORIES`. These are deterministic and fixture-driven — write failing tests before any implementation line.
- **Test-after:** real-repo integration tests, the compile/smoke step, the perf lane. These exercise external systems where test-first yields diminishing returns.

## Team

Veteran experts to hire:
- **Veteran CLI systems engineer (1)** — owns walker, CLI surface, Bun packaging, exit-code semantics, symlink/binary-file policy, include/exclude composition.
- **Veteran parser/regex engineer (1)** — owns the extractor, comment-syntax table, boundary regex, and edge-case handling (marker-looking strings on code lines, nested blocks, multi-marker per line, multi-marker blocks, continuation joining with blank lines, generic-fallback caveat).
- **Veteran git-plumbing engineer (1)** — owns the blamer and the `git check-ignore` delegation: porcelain parsing, author-time selection, batching, ARG_MAX chunking, 10-char SHA normalization, uncommitted-line detection, graceful degradation, and worktree detection.
- **Veteran test engineer (1)** — owns fixture design, golden files, the synthetic-repo bare-git fixture, black-box CLI suite, pinned-SHA integration harness, JSON-Schema validator harness, and the postgres/postgres + database-lab integration in CI.
- **Veteran TypeScript/Bun release engineer (1, part-time)** — owns `package.json`, `bin` entry, `bun build --compile`, npm publish workflow, semver discipline, and the hosted JSON-Schema document.

Total: 4 full-time + 1 part-time.

## Implementation plan

Sprints are one week each. `⇄` = work happens in parallel; `→` = ordering dependency.

### Sprint 1 — Skeleton & red tests
- CLI engineer: scaffold repo layout, `bin/todo-stream`, `--help`, `--version`, argv parser; author red tests for flag-grammar error paths (invalid `--since`, invalid marker, unknown flag, `--fail-on` not a subset of `--markers`, `--fail-on` repeated, `--no-blame` + `--since`, `--staged` outside a worktree). ⇄
- Parser engineer: author fixture tree under `tests/fixtures/langs/` and write **failing** extractor tests for every language in the language table, including multi-marker-per-line, multi-marker-block, continuation joining (including blank-line `\n\n`), marker-in-string-on-code-line (known-language suppression vs unknown-extension fallback asymmetry), non-`\w`-marker-rejection, and verbatim-case cases. ⇄
- Git engineer: capture canned `git blame --porcelain` outputs (including a `0000...` uncommitted case and one where author-time and committer-time differ) into `tests/fixtures/blame/` and write **failing** parser tests for 10-char SHA normalization, author-time selection, and uncommitted→null. Write **failing** ARG_MAX chunking tests with an injected spawn. ⇄
- Test engineer: stand up `bun test` in CI, add the `INTEGRATION=1` gate, scaffold black-box CLI test harness (invokes the compiled binary), write **failing** CLI tests for every v0.1 flag (including `--include`, `--exclude`, their interaction and ordering, `--staged` + `--include` intersection, `--staged` outside a worktree, deleted-from-worktree files), every exit-code case, and the no-git-worktree degradation under `GIT_CEILING_DIRECTORIES`. Also stand up the pinned-SHA synthetic fixture repo and the JSON-Schema validator harness. ⇄
- Release engineer: lock Bun version, add `bun tsc --noEmit` + `biome check` to CI, declare `@biomejs/biome` as a dev dep.
- **Gate:** all tests RED, CI green on lint/type.

### Sprint 2 — Core green
- Parser engineer → implement `extract.ts` (dynamic marker regex with global flag, symmetric non-identifier boundaries, `^[A-Za-z0-9_]+$` marker validator, per-occurrence Finding emission, continuation algorithm with blank-line handling) until fixtures pass.
- CLI engineer → implement `walker.ts` with `Bun.Glob` + `git check-ignore` delegation + root-path symlink canonicalization + internal symlink-skip + binary-skip + dedup-by-absolute-path + exclude-wins composition, write walker tests RED→GREEN. ⇄
- Git engineer → implement blame porcelain parser (pure) with author-time selection, including the `0000...`→null path and 10-char SHA truncation, until unit tests pass. Implement ARG_MAX chunking (constant 512) with merge logic. ⇄
- Test engineer → author golden-file tests for both renderers across every enumerated case (still red; renderers not yet built); finalize the synthetic bare-git fixture repo.
- **Gate:** extractor, walker, blame parser, ARG_MAX chunking all GREEN; renderer and CLI tests still RED.

### Sprint 3 — Render & wire
- CLI engineer → implement JSON + Markdown renderers against golden tests (POSIX-relative `path`, pinned `generated_at` format, 10-char SHA, author-date rendering); wire end-to-end pipeline in `cli.ts`; implement exit-code semantics and all usage-error paths (including `--fail-on` repeat and `--staged` outside a worktree).
- Git engineer → implement `blame.ts` runtime (spawn `git blame`, batch per file with ARG_MAX chunking, no-git fallback emitting per-finding `blame: null`, uncommitted→null). ⇄
- Parser engineer → extend markers config + `--markers` flag (dynamic regex, verbatim-case, marker-shape validator), add generic fallback extraction with documented asymmetry. ⇄
- Test engineer → start integration harness: pinned-SHA clone helper, postgres subtree scan with structural invariants + schema validation + per-marker presence probe.
- **Gate:** renderer tests and a majority of CLI flag tests GREEN; `todo-stream` runs end-to-end against the repo itself and produces valid output.

### Sprint 4 — Integration & release
- Test engineer → finalize integration tests (pinned-SHA postgres/postgres + postgres-ai/database-lab with schema validation + per-marker presence + sanity-floor invariants); stand up the separate perf lane on `ubuntu-latest` 4 vCPU with the 90 s ceiling. ⇄
- Release engineer → `bun build --compile` pipeline, npm publish dry-run, `npx todo-stream` smoke, README usage + JSON schema docs, publish the hosted `$schema` JSON, declare macOS + Linux support matrix. ⇄
- CLI engineer → implement `--since` (strict `YYYY-MM-DD`, UTC, inclusive, compared against author-date), `--author` (case-insensitive substring across `author <email>`, repeatable), `--fail-on` (comma list, non-repeatable, subset-of-markers validation), `--staged` (file-level, working-tree read, worktree check + deleted-file skip), `--config` (explicit only), plus the `--no-blame` + `--since`/`--author` mutual-exclusion guard. ⇄
- Git engineer → hardening: large-file guard, symlink handling in blame, uncommitted-line regression tests against live git fixtures. ⇄
- Parser engineer → extend language coverage (finalize Rust, Ruby, Lua) with fresh red→green fixtures matching the language table.
- **Gate:** all unit + black-box CLI tests GREEN; integration tests GREEN on CI against pinned SHAs; perf lane reporting; `bun run build` emits a working static binary; product v0.1.0 tagged.

### Parallelization summary
- Sprint 1 is almost fully parallel (five independent red-test authoring streams, including black-box CLI, error-path grammar, and ARG_MAX chunking).
- Sprint 2 and 3 pair the parser+git engineers in parallel while the CLI engineer advances the pipeline; the test engineer stays one step ahead writing the next red tests.
- Sprint 4 fans out: every engineer has an independent lane.

## Embedded Changelog

- **Spec v0.1 (2026-04-21)** — Initial draft. Reframed away from the "todo-list app" interview answers (SQLite, NDJSON daemon, add/list/done subcommands, id/done/created_at model) per the authoritative idea: linter for TODO/FIXME/HACK comments, not a storage tool. Locked scope to directory walk + regex extract + git blame + JSON/Markdown report. Zero runtime deps beyond Bun built-ins.
- **Spec v0.2 (2026-04-21)** — Post-review r1 refinement. Strengthened user stories (added CI gatekeeper, pre-commit, pipeline-consumer personas). Tightened the Architecture diagram and component boundaries. Added the concrete JSON schema example and Markdown output shape. Expanded the Tests plan with explicit TDD red-first call-outs and the `INTEGRATION=1`-gated postgres/postgres + database-lab harness. Specified team composition and a four-sprint parallelized plan.
- **Spec v0.3 (2026-04-21)** — Post-review r2 refinement addressing Reviewer B's findings. Introduced the spec-version vs product-version distinction; reconciled JSON schema prose and example; stated dynamic marker regex; pinned `--staged` to file-level; made `--since`/`--author` incompatible with `--no-blame`; replaced the pinned-upstream-TODO assertion with structural invariants plus a synthetic fixture; moved the perf number to a separate perf lane; declared string-literal tokenization a non-goal; specified multi-marker block handling; delegated `.gitignore` to `git check-ignore`; pinned `blame: null` wholesale.
- **Spec v0.4 (2026-04-21)** — Post-review r3 refinement addressing Reviewer B's findings. **Regex & markers:** symmetric non-identifier boundaries (no `\b`); user markers restricted to `^[A-Za-z0-9_]+$` (else exit 2); verbatim case (not upcased). **CLI grammar pinned:** `--since` is strict `YYYY-MM-DD`, UTC, inclusive; `--author` is case-insensitive substring across `author <email>`, repeatable; `--fail-on` is a comma list that must be a subset of `--markers`. **JSON surface pinned:** `path` is POSIX-style and relative to `root` (root is parent dir when scanning a single file); short SHA is fixed at 10 chars; `generated_at` is `YYYY-MM-DDTHH:MM:SSZ`; `language` values enumerated in a language table; env-var config removed from precedence. **Semantics pinned:** `--staged` reads working-tree bytes (documented limitation); uncommitted `0000...` blame → `blame: null`; symlinks skipped (not followed); binary files detected by NUL byte in first 8 KiB; block-comment continuation joined with `\n` after stripping leading punctuation; 1-based line/column. **Testing strengthened:** integration clones pinned to specific SHAs; no-git-worktree black-box test added; renderer golden-file edge cases enumerated (empty, null-blame, multi-line, non-ASCII, long text, same-line duplicates); perf lane framed as coarse ceiling, not trend-tracking. **Platforms:** macOS + Linux declared; Windows not a v0.1 support target.
- **Spec v0.5 (2026-04-21)** — Post-review r4 refinement addressing Reviewer B's findings (Reviewer A unavailable). **Contradiction resolved:** multi-marker handling now emits one Finding per *marker occurrence* (not per line), so two markers on one comment line yield two Findings with distinct columns; block-comment continuation attaches non-marker lines between/after markers within the same block. **Blame date pinned:** author-date (`author-time`) is the single source for `blame.date`, `--since` comparison, and Markdown date rendering. **Continuation algorithm made authoritative** with a worked example; non-applicable punctuation tokens (`//`) removed; blank-line handling specified (consecutive blanks → consecutive `\n`s). **Config discovery pinned:** no auto-discovery in v0.1; `--config` is the only way to load a config file; precedence wording tightened to `CLI flags > --config file > defaults`. **CLI tests strengthened:** `--include`/`--exclude` positive+negative cases, their interaction with `--staged`, exclude-wins ordering independence. **Semantics pinned:** walker deduplicates by resolved absolute path; root-path symlinks canonicalized once at startup; `--staged` outside a worktree exits 2; staged-but-deleted files are silently skipped; `--fail-on` is non-repeatable (repeat → exit 2); generic-fallback asymmetry documented as intentional and tested. **Blame scalability:** ARG_MAX chunk size fixed at 512 `-L` ranges per invocation with merge logic and a dedicated test. **Integration invariants strengthened:** full JSON-Schema validation and per-default-marker presence probe added; count-tolerance gating explicitly not adopted, with justification recorded. **Hygiene:** per-finding `blame: null` wording used consistently; `@biomejs/biome` declared as a dev dep; `GIT_CEILING_DIRECTORIES` guard added to the no-git-worktree test so TMPDIR-inside-worktree setups don't cause flakes.
