# todo-stream — SPEC v0.6

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

- **Spec version** — the version of *this document*. Appears only in the `# todo-stream — SPEC vX.Y` header and the embedded changelog. Bumps every review round. Currently **v0.6**.
- **Product version** — the version of the *published npm package and compiled binary*. Appears in `package.json`, the JSON report's `version` field, and `--version` output. Currently **0.1.0** (pre-release; will ship as 0.1.0 when the v0.1 scope below is implemented).

The JSON schema example below therefore shows `"version": "0.1.0"` (product), while this document's header is `SPEC v0.6`.

## User stories

1. **CI gatekeeper — Priya, release engineer.** Priya adds `todo-stream --format json --fail-on FIXME` to the repo's CI workflow. **`--fail-on` is a whole-repo gate**: any matching finding (pre-existing or new) trips exit 1. For a mature repo with pre-existing FIXMEs, Priya either (a) removes FIXME from the default marker set for CI while keeping TODO/HACK scanning, (b) uses `--since <baseline-date>` to restrict the gate to findings introduced after a cutoff author-date, or (c) waits for a diff/baseline mode (deferred to a later product version — see Scope). The v0.1 product intentionally does **not** provide a diff-against-base mode; the user story makes this trade-off explicit so the tool is not shipped under a false promise.
2. **Tech-debt triage — Luis, staff engineer.** Luis runs `todo-stream --format markdown --since 2024-01-01 > DEBT.md` against the monorepo once a quarter, grouping findings by file with blame dates, and uses the Markdown report in a planning meeting to assign owners.
3. **Incoming maintainer — Dana, new OSS contributor.** Dana clones a large project (e.g. `postgres/postgres`) and runs `todo-stream src/backend --markers TODO,HACK --format markdown | less` to orient herself: she sees the oldest HACKs, who wrote them, and which files are hot-spots, without learning the project's in-house tooling.
4. **Pre-commit author — Sam, individual developer.** Sam wires `todo-stream --staged --format json` into a `lefthook` pre-commit. `--staged` limits the scan to the *set of files* listed by `git diff --cached --name-only -z` (NUL-delimited; see CLI surface) but reads their **working-tree** bytes (not the staged blob) — consistent with linting tools like ESLint and the pragmatics of pre-commit hooks; documented as a known limitation.
5. **Pipeline consumer — automated dashboard.** A scheduled job runs `todo-stream --format json` and pipes the output into a downstream ingester that renders a historical chart of open TODO count per file — the JSON schema is stable and documented.

## Scope & non-goals (product v0.1)

**In scope (product v0.1)**
- Recursive walk of a directory with configurable include/exclude globs (defaults respect `.gitignore`).
- Extraction of configurable markers (default: `TODO`, `FIXME`, `HACK`, `XXX`) from line and block comments in common languages (see the language table below). Unknown extensions fall back to a generic regex with documented caveats.
- Git blame enrichment per finding: author name, author email, commit **author-date** (ISO-8601 UTC), 10-char short SHA. Non-git trees degrade gracefully (per-finding `blame: null`).
- Two output formats: `json` (stable, documented schema) and `markdown` (human-readable, grouped by file).
- Filters: `--markers`, `--since`, `--author`, `--include`, `--exclude`, `--fail-on`, `--staged`.
- Resource guards: `--max-file-size` (default 5 MiB), `--blame-timeout` (default 30 s per invocation), `--blame-concurrency` (default 8).
- Privacy guard: `--redact-emails` (replaces the email local-part with `***` in both JSON and Markdown output).
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
- **Diff/baseline CI mode** ("fail only on FIXMEs introduced in this PR"). Deferred to a later product version; see user story 1 for the v0.1 workarounds.
- **Python triple-quoted string handling as block comments.** Triple-quoted strings in Python are *strings*, not comments, even when used as docstrings. The extractor only recognizes `#` as a Python comment syntax in v0.1 — markers inside docstrings are not reported. (Removed from v0.5's scope where it was listed; see language table.)
- **String-literal tokenization.** The extractor inspects only the *comment region* of a line (for languages in the language table); it does not parse string literals. A `TODO` appearing inside a string on a line that is not otherwise a comment is *not* a finding in known languages. A `TODO` inside a string *within* a comment (e.g. `// TODO: see "FIXME" in docs`) remains a single `TODO` finding — the inner token is treated as comment text. For the **generic (unknown-extension) fallback**, no comment-region reduction is performed; see the fallback caveat in Implementation details.

### Language table (authoritative; pins `language` field values)

The extractor dispatches on **file extension first**, then on **exact filename** if the extension match fails. Both dispatches are **case-insensitive** on the extension/filename token (`Foo.TS` → `typescript`, `makefile` → `make`). The right column is the exact string emitted as the JSON `language` field (stable public surface).

**Extension dispatch:**

| Extensions | Comment syntax | `language` value |
|---|---|---|
| `.c` `.h` | `//`, `/* */` | `c` |
| `.cc` `.cpp` `.cxx` `.hpp` `.hh` `.hxx` | `//`, `/* */` | `cpp` |
| `.go` | `//`, `/* */` | `go` |
| `.rs` | `//`, `/* */` | `rust` |
| `.ts` `.tsx` `.mts` `.cts` | `//`, `/* */` | `typescript` |
| `.js` `.jsx` `.mjs` `.cjs` | `//`, `/* */` | `javascript` |
| `.py` | `#` (triple-quoted strings are NOT comments; see Scope) | `python` |
| `.sh` `.bash` `.zsh` | `#` | `shell` |
| `.sql` | `--`, `/* */` | `sql` |
| `.lua` | `--`, `--[[ ]]` | `lua` |
| `.rb` | `#`, `=begin =end` | `ruby` |
| any other extension with no filename match | generic fallback | `unknown` |

**Exact-filename dispatch** (applied only when the file has no extension or its extension is not in the table above):

| Filename | Comment syntax | `language` value |
|---|---|---|
| `Makefile` `GNUmakefile` | `#` | `make` |
| `Dockerfile` | `#` | `dockerfile` |
| `Jenkinsfile` | `//`, `/* */` | `groovy` |
| `CMakeLists.txt` | `#` | `cmake` |
| `.gitignore` `.dockerignore` | `#` | `ignore` |

Any file not matched by extension or filename uses the generic fallback and is reported with `language: "unknown"`.

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
- **CLI (`src/cli.ts`)** — argument parsing (hand-rolled, no deps), config resolution (CLI flags > `--config` file > defaults), dispatch. Owns process exit code. Owns stdout/stderr discipline (see "Output atomicity" below).
- **Walker (`src/walker.ts`)** — streams matching file paths. Uses `Bun.Glob` for include/exclude. `.gitignore` pruning is delegated to `git check-ignore --stdin -z -v` when the root is inside a git worktree; outside a worktree, no `.gitignore` pruning is performed (users may still use `--exclude`). **Symlinks are not followed** during traversal (neither files nor directories) — avoids cycles and surprise traversal outside the root; symlinks are skipped and not reported. **Root-path symlinks** (the positional `path` argument itself) are resolved exactly once at startup to their canonical absolute path, and traversal proceeds from that canonical path; the resolved path is used as the `root` field in the JSON output. The walker **deduplicates paths by resolved absolute path** before emitting — a path matched by multiple `--include` globs is yielded only once; the extractor therefore runs at most once per file. Emits an async iterator; never buffers the whole tree.
- **Extractor (`src/extract.ts`)** — pure function: `(path, bytes, config) → Finding[]`. Dispatches on extension (lowercased) then filename to the language table. Returns `{path, line, column, marker, text, raw}` — no I/O, fully unit-testable from fixtures.
- **Blamer (`src/blame.ts`)** — batches findings per file and shells out via a single injectable seam `runBlame(args: string[]): Promise<{stdout: string; exitCode: number}>` (the *only* spawn caller in `blame.ts`; tests swap it). The production implementation of `runBlame` calls `Bun.spawn(["git", ...args], {env: HARDENED_GIT_ENV, stderr: "pipe"})` (see "Subprocess hardening" below), with a per-invocation timeout from `--blame-timeout`. Issues `git blame --porcelain -L a,a -L b,b -- <file>` per file (subject to ARG_MAX chunking below), parses porcelain into `{author, email, date, sha}`. `date` is always the commit **author-date** (`author-time` + `author-tz` from porcelain), normalized to ISO-8601 UTC with second precision. Caches by absolute file path. Degrades to per-finding `blame: null` outside a git worktree, for untracked files, for lines reported as uncommitted, and on per-file spawn failure (see "Per-file blame spawn failures" below). Blamer concurrency across files is capped at `--blame-concurrency`.
- **Renderer (`src/render/*.ts`)** — two pure renderers: `json.ts` (emits the versioned schema defined below) and `markdown.ts` (groups by file, sorts by line then column; escapes untrusted text per "Output escaping" below).
- **Reporter (in `cli.ts`)** — buffers complete output before writing to stdout (atomicity guarantee), applies `--fail-on` rules, sets exit code. Writes diagnostics only to stderr.

**Key abstractions**
- `Finding` — the one shared record shape crossing every boundary.
- `CommentSyntax` — `{ line?: string[]; block?: [open, close][] }`, looked up by extension or filename.
- `Config` — frozen object assembled once at startup.
- `runBlame` — injectable spawn seam in `blame.ts`; pinned as the architectural boundary for ARG_MAX-chunking tests.

**Dependency policy.** Runtime deps = zero beyond Bun built-ins (`Bun.Glob`, `Bun.spawn`, `Bun.file`, `fs`, `path`). `git` itself is assumed on PATH (required for blame and `.gitignore` delegation; the tool degrades without it as documented). Dev deps are limited to `bun test`, `@biomejs/biome` (lint + format), and type definitions.

## Security, privacy, and robustness

This section pins operational guarantees for running `todo-stream` against untrusted repositories and emitting output into CI artifacts. All items are testable and appear in the Tests plan.

### Path encoding and hostile filenames
- All git subprocess invocations that take or return paths use **NUL-delimited** modes: `git diff --cached --name-only -z`, `git check-ignore --stdin -z -v`, `git ls-files -z` (if used). Parsers split on `\0`, never on `\n` or whitespace.
- Paths are always passed as **argv after `--`**, never interpolated into a shell command. `Bun.spawn` is invoked with an argv array; no shell is ever spawned.
- Filenames containing newlines, tabs, spaces, leading dashes, or non-ASCII bytes are supported. A fixture `tests/fixtures/hostile-names/` exercises filenames containing `\n`, `\t`, a leading `-`, a double-quote, and a 4-byte UTF-8 codepoint. The fixture is a checked-in bare-git tarball (filesystems allowing).
- **Invalid UTF-8 in source bytes** is read as bytes, run through the extractor (which treats the regex input as Latin-1 byte-preserving), and emitted in JSON using U+FFFD replacement where JSON requires valid UTF-8. The `raw` field preserves the original bytes in a hex-escaped form when replacement would be lossy (not emitted in v0.1 output, but reserved).

### Output escaping
- **Markdown renderer** escapes, in `path`, `author`, `email`, `text`, and `sha`:
  - Backticks (`` ` ``) → `` \` ``
  - Pipe (`|`) → `\|`
  - Square brackets (`[` `]`) → `\[` `\]`
  - Angle brackets (`<` `>`) → `\<` `\>`
  - HTML comment openers (`<!--`) → escape the `<`.
  - All ASCII control characters < 0x20 except `\n` and `\t` are stripped; ANSI escape sequences (`\x1b[...m` and similar CSI/OSC sequences) are stripped.
  - The escape list is applied *after* the continuation algorithm has joined `text`, so the `\n` segments remain the separator for the multi-line rendering rule (see "Markdown output shape").
- **JSON renderer** relies on standard JSON string escaping for all string fields. Control characters are `\u00XX`-escaped. No additional sanitization is needed for JSON, but ANSI escape sequences in source text are preserved byte-for-byte in the JSON string (they are JSON-safe) — consumers that re-emit JSON `text` into terminals must handle them.
- **Stderr diagnostics** use plain ASCII, never ANSI colors.
- A fixture verifies that a source comment `// TODO: [link](http://evil) \| injection` cannot create a Markdown link, table column break, or hidden span in the rendered report.

### Data sensitivity
- `todo-stream` output can contain: source comment text, author names, author emails, file paths, commit SHAs. This is sensitive in public CI and third-party dashboards.
- The README documents this explicitly and recommends treating `todo-stream` output as source-code-equivalent for visibility purposes.
- `--redact-emails` replaces `jane@example.com` with `j***@example.com` (local-part → first char + `***`) in both JSON (`blame.email`) and Markdown output. Name and SHA are unaffected.
- Further redaction (full-text filters, regex redaction, name redaction) is deferred.

### Resource limits
- **Per-file size cap.** Files whose `stat().size` exceeds `--max-file-size` (default 5 MiB) are skipped with a stderr warning and contribute zero findings. Rationale: `Bun.file(path).text()` loads the file fully; the cap bounds memory envelope. Configurable by flag; hard ceiling enforced even if `--max-file-size=0` is passed (any value `<= 0` is a usage error, exit 2).
- **Blame invocation timeout.** Each `runBlame` spawn is killed after `--blame-timeout` seconds (default 30). A timed-out invocation is treated identically to a per-file spawn failure (see below).
- **Blame concurrency cap.** At most `--blame-concurrency` (default 8) concurrent `runBlame` invocations. Files queue behind the cap; findings from un-blamed files are not held up past the cap.
- **No truncation of `text`.** Finding `text` (including multi-line continuations) is emitted in full. Long block comments produce long strings; this is a documented trade-off (linters must faithfully reproduce the comment to be useful). A future version may add `--max-text-bytes`.

### Subprocess hardening
- All `git` invocations use the hardened env `HARDENED_GIT_ENV`:
  - `GIT_OPTIONAL_LOCKS=0` (avoid touching `.git/index.lock`).
  - `GIT_TERMINAL_PROMPT=0` (never prompt).
  - `GIT_CONFIG_GLOBAL=/dev/null` and `GIT_CONFIG_SYSTEM=/dev/null` (ignore user/system config; avoid injecting aliases or hooks via malicious config).
  - `LC_ALL=C.UTF-8` and `LANG=C.UTF-8` (stable error strings).
  - `PATH` inherited from the parent process *minus* any trailing `.` component.
- All invocations pass `--no-pager` (though `git blame --porcelain` and `git check-ignore` are not pagers anyway).
- `git blame` is invoked **without** `--textconv` (textconv is off by default; stated for the record).
- `git` is resolved via PATH; no hard-coded path. If `git` is not found at startup and `--no-blame` was not passed, every finding gets `blame: null` and a single stderr warning is emitted.
- Hooks are irrelevant (blame/check-ignore/diff don't run hooks), but for defense in depth, `core.hooksPath=/dev/null` is passed via `-c` to every invocation.
- A fixture `tests/fixtures/malicious-config-repo/` contains a repo-local `.git/config` with alias, pager, and `core.fsmonitor` settings that would normally execute commands; the test asserts the tool runs unaffected.

### Per-file blame spawn failures
- If `runBlame` for a specific file exits non-zero, times out, or fails to parse, **all findings for that file get `blame: null`** and a single stderr warning is emitted: `warning: git blame failed for <path>: <reason>`. The run continues. Exit code is not affected by per-file blame failures.
- If `git` is not on PATH at all, the entire run degrades to `--no-blame` behavior with a single stderr warning (not exit 2).
- Exit 2 is reserved for CLI spawn failures of non-blame calls (e.g. `git rev-parse` on startup) or for unrecoverable errors.

### Output atomicity
- **stdout carries only the report.** **stderr carries every diagnostic** (warnings, progress, error messages).
- The JSON renderer **buffers the full report in memory** and writes it as a single `stdout.write()` call after filtering and rendering complete. Consumers never see a partial JSON document on stdout, even if the process exits 2 mid-run.
- The Markdown renderer may stream per-file sections as they complete (findings within a file are buffered together), but the `# TODO report` header is emitted first and a file section is never split across stdout writes.
- If the Reporter decides to exit 2 after partial output could have been written (e.g. a late-detected internal error), the already-written stdout is not truncated; the exit code + stderr diagnostic are the consumer's signal. JSON consumers should therefore gate on exit code before parsing — documented in the README.

## Implementation details

### Data flow
1. `cli.ts` parses argv → `Config`. If `--config <path>` is passed, the JSON file is merged under CLI flags per the precedence rule (CLI flags > --config file > defaults). No auto-discovery.
2. `walker.ts` yields file paths matching globs, pruning via `git check-ignore` (inside a worktree) and `--exclude`. **Ordering of `--include` and `--exclude`: exclude wins.** A path matched by any `--exclude` glob is dropped even if it is also matched by `--include`. The order in which `--include` and `--exclude` appear on the argv does not matter. Paths are yielded deduplicated by resolved absolute path. Files exceeding `--max-file-size` are dropped with a stderr warning before reaching the extractor.
3. For each path, `Bun.file(path).text()` → `extract.ts` → `Finding[]` (streamed, not accumulated repo-wide).
4. Findings are grouped by file and handed to `blame.ts`, which issues one or more `git blame` invocations per file (see ARG_MAX chunking below) through the `runBlame` seam, with all needed line ranges. Concurrency across files is capped at `--blame-concurrency`; per-invocation timeout is `--blame-timeout`.
5. Filters `--since` and `--author` are applied **after** blame enrichment (see precedence rules below).
6. Enriched findings feed the chosen renderer; the Reporter buffers the full output and writes atomically to stdout.
7. `--fail-on <marker>[,<marker>...]` sets exit code 1 if any matching finding exists after filtering; absence of matching findings exits 0. Internal errors (I/O, bad config, spawn failure of non-blame calls, usage errors) exit 2.

### Config file schema (authoritative)

`--config <path>` loads a JSON file with the following top-level keys. Unknown keys are **rejected with exit 2 and a readable error naming the offending key** (no silent ignore, no forward-compat). All keys are optional.

```json
{
  "markers": ["TODO", "FIXME"],
  "include": ["**/*.ts"],
  "exclude": ["**/dist/**"],
  "format": "json",
  "since": "2024-01-01",
  "author": ["jane", "bob@example.com"],
  "failOn": ["FIXME"],
  "noBlame": false,
  "noGitignore": false,
  "redactEmails": false,
  "maxFileSize": 5242880,
  "blameTimeout": 30,
  "blameConcurrency": 8
}
```

- Field types are validated (strings, arrays of strings, booleans, integers as named); type mismatch → exit 2 with a readable error naming the offending field and expected type.
- Values are subject to the same validation as the equivalent CLI flag (e.g. `markers` entries must match `^[A-Za-z0-9_]+$`; `since` must match `YYYY-MM-DD`; `failOn` must be a subset of effective `markers`).
- **Merge semantics (authoritative).** For each field, a value supplied on the CLI **replaces** the config-file value *wholesale*. For list-valued fields (`markers`, `include`, `exclude`, `author`, `failOn`), supplying the corresponding CLI flag (even once) overrides the entire config list — CLI values are *not* appended. Rationale: CLI-first precedence must be predictable; merge-by-append would make it impossible to narrow a list from the CLI.
- Defaults apply only when neither the CLI nor the config file supplies a value.
- The config file does **not** support comments (it is strict JSON). Parse errors → exit 2.

### Finding extraction algorithm
- The marker regex is **built dynamically** from `config.markers`. Each marker is regex-escaped and joined. Boundaries are symmetric and use a non-identifier character class on both sides (not `\b`, which is defined against `\w` and would misbehave for non-`\w` marker characters):
  ```
  new RegExp(
    "(^|[^A-Za-z0-9_])(" + markers.map(escape).join("|") + ")(?=[^A-Za-z0-9_]|$)[:\\s-]?\\s*(.*)",
    "g"
  )
  ```
  The regex is executed with the global flag so multiple marker occurrences on a single line are all detected. To keep boundary semantics well-defined, **user-supplied markers must match `^[A-Za-z0-9_]+$`**. Markers outside this class (e.g. `TODO?`, `FIXME-LATER`, CJK) are rejected at config-load time with a usage error (exit 2). Match is case-sensitive; user-supplied markers are used **verbatim** (no upcasing). `--markers todo` therefore matches only lowercase `todo`, not `TODO`.
- For each line of a file with a **known** extension or filename (see the language table), the extractor first reduces the line to its comment region using the language's `CommentSyntax`. It does **not** tokenize string literals; see the Scope section's explicit non-goal. Markers embedded in string literals on code-only lines of known languages are not reported.
- **Generic (unknown-extension/filename) fallback caveat.** For files with extensions or filenames not in the language table, the extractor runs the boundary regex against the full line without comment-region reduction. This means a `TODO` appearing inside a string literal in an unknown file **will** be reported, while the same construct in a `.ts` file will not. This asymmetry is intentional (and called out in Scope) because the tool has no language model for unknown formats; users who want to suppress false positives for a specific format should either add the extension to the language table upstream or exclude those files via `--exclude`.
- **Extension case sensitivity.** The extension is lowercased before table lookup: `Foo.TS` → `typescript`. Filename lookup is also case-insensitive: `makefile`, `MAKEFILE`, `Makefile` all → `make`. A walker fixture pins this on both a case-sensitive (Linux tmpfs) and a case-preserving (macOS) filesystem.
- **Line and column indexing.** `line` is 1-based (matching `git blame` and most editors). `column` is 1-based and refers to the column of the first character of the matched marker token in the original line.
- **Multi-marker handling (authoritative).** **Each marker *occurrence* yields its own Finding** — not each line. A single line (inside a comment, whether block or line-style) containing two markers produces two Findings with identical `path` and `line` but distinct `column` values; a block comment spanning multiple lines yields one Finding per marker occurrence across those lines. Findings are emitted in source order: primarily by `line` ascending, secondarily by `column` ascending.
- **Continuation-text attachment (authoritative algorithm).** Inside a *block comment* of a known language, any line between two marker occurrences — or after the final marker occurrence within that block — that contains **no** marker occurrence is a **continuation line** and is attached to the immediately preceding marker occurrence's `text`. The algorithm is:
  1. Strip the block-comment's own punctuation: `/*` / `*/` delimiters on the opening/closing lines, and leading `*` (C-family block comments), leading `--` (SQL / Lua block comments), and surrounding whitespace on continuation lines. Line-comment tokens (e.g. `//`) are *not* stripped because they cannot validly appear as block-comment continuation punctuation. (Python has no block comments in v0.1, so no Python-specific stripping applies.)
  2. If the resulting continuation line is **empty**, represent it as a single empty segment.
  3. Concatenate the preceding marker's `text` with each continuation segment using a single `\n` between segments. Consecutive empty continuations therefore produce consecutive `\n` characters (two blank continuation lines → `\n\n` between the surrounding text segments).
  4. **Empty-marker-line special case.** If the marker-line's `text` is the empty string after punctuation/colon/dash stripping (e.g. `/* TODO:\n * body\n */`), the concatenation **omits the leading `\n`** and begins with the first continuation segment. The final `text` in this case is `"body"` (not `"\nbody"`).
  
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
  
  *Example (empty marker line):*
  ```
  /* TODO:
   * body
   */
  ```
  Produces one Finding: `{marker:"TODO", line:1, column:4, text:"body"}` (no leading `\n`).
- Marker-line text trimming: within the marker-line itself, the optional `:` or `-` immediately after the marker token is stripped from the captured `text`, along with surrounding whitespace and the leading block-comment punctuation of that line (same stripping rules as step 1 above).
- Unknown extensions/filenames use the generic fallback regex (see caveat above); record with `language: "unknown"`.
- **Binary files** are skipped by the walker when any of the first 8 KiB of the file contains a NUL byte; they are not read or reported.

### Filter precedence and blame-dependent flags
- `--since` and `--author` operate on blame metadata.
- **`--since` grammar.** Accepts strictly `YYYY-MM-DD` in v0.1 (other forms, including relative `7d`, are rejected with exit 2). The date is interpreted as **UTC midnight** (`00:00:00Z`). The comparison is **inclusive of the boundary**: a finding with `blame.date >= <since>T00:00:00Z` is kept. The blame date used is the commit **author-date** as emitted by `git blame --porcelain` (`author-time` + `author-tz`), normalized to UTC — identical to the value emitted in `blame.date`.
- **`--author` matching.** Case-insensitive substring match against the concatenation `author + " <" + email + ">"` — i.e. it matches either the author name or the email, or any substring spanning the two. Multiple substrings via repeated `--author` flags are OR'd.
- If `--no-blame` is passed *together* with `--since` or `--author`, the CLI exits 2 with a usage error (these flags are mutually incompatible).
- When blame is enabled but a particular finding has `blame: null` (file is untracked, the tree is not a git worktree, the line is uncommitted, or per-file blame spawn failed), applying `--since` or `--author` **drops** that finding from the output. Rationale: the user is filtering on blame data; a finding with no blame data cannot satisfy the filter.
- `--fail-on` is evaluated **after** all filters. A repo with FIXMEs that are all filtered out by `--since` exits 0.
- **`--fail-on` grammar.** Accepts a comma-separated list of markers (e.g. `--fail-on FIXME,HACK`). The flag is **not repeatable**: passing `--fail-on` more than once on the argv is a usage error (exit 2), not last-wins. The set must be a subset of `--markers`; passing `--fail-on FIXME` when `--markers` does not include `FIXME` is a usage error (exit 2), not a silent no-op.

### `--staged` semantics (authoritative)
- `--staged` requires the current working directory (or the resolved positional root) to be **inside a git worktree**; otherwise exit 2 with a readable error.
- The worktree root is discovered via `git rev-parse --show-toplevel` executed with `-C <resolved-root>` and the hardened env (see Subprocess hardening). All staged-file enumeration runs with the same `-C` anchor.
- The staged file list is produced by `git -C <worktree-root> diff --cached --name-only -z`. Output is split on NUL, never on `\n`. Paths are interpreted as relative to the worktree root.
- The staged file set is then **intersected with the resolved scan root**: only staged files whose absolute path is at or under the resolved root are scanned. `todo-stream subdir --staged` therefore scans only staged files under `subdir/` (relative to the worktree). An empty intersection exits 0 with an empty `findings` array (not an error).
- **Submodules.** The staged-file enumeration from the outer worktree does not recurse into submodules. Findings inside submodules are not surfaced by `--staged` in v0.1; documented limitation.
- Files staged but deleted from the working tree are silently skipped (the tool reads working-tree bytes).
- Filenames with spaces, newlines, or non-ASCII characters are correctly handled by the `-z` parse path; a fixture pins this behavior.

### Blame batching, ARG_MAX, and uncommitted lines
- Group findings by file; invoke `runBlame(["blame", "--porcelain", "-L", "a,a", "-L", "b,b", "--", "<file>"])` in a single call per file **when the total range count is safe**.
- **ARG_MAX guard.** If a single file accumulates more than **512 `-L` ranges** (a conservative ceiling well under POSIX `ARG_MAX` on all supported platforms), the blamer **chunks** the ranges into batches of up to 512 and issues multiple `runBlame` calls per file, merging the parsed results. The chunk size is a constant in `blame.ts`, not user-configurable in v0.1. Tests swap `runBlame` to assert chunking behavior without real git.
- Parse porcelain headers once per unique commit within a file's blame output.
- **Short SHA** is the first **10 hex characters** of the 40-char commit SHA returned by `git blame --porcelain`. This width is fixed — not `core.abbrev`-derived — so the JSON field is stable across environments.
- **Uncommitted lines.** When `git blame --porcelain` reports a line as `0000000000000000000000000000000000000000` with author `Not Committed Yet`, that finding's `blame` is emitted as the JSON literal `null` (not a synthesized object). Downstream consumers can distinguish "no git worktree" from "uncommitted line" by the root context; within a single run, the shape of `blame` is uniform: always either the full object or `null`.
- Outside a git worktree (detected via `git rev-parse --is-inside-work-tree`), skip blame entirely and emit per-finding `blame: null`.
- **Per-file spawn failure handling:** see "Per-file blame spawn failures" in the Security section.

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

### Markdown output shape (authoritative)
- `# TODO report — <root> — <generated_at>` (header; root and generated_at are both escaped per "Output escaping").
- One `## <relative/path>` section per file (path escaped).
- Findings within a file are rendered as list items. **For a single-line `text`** (no embedded `\n`):
  ```
  - **TODO** L42 · Jane Doe, 2024-07-11 (a1b2c3d4e5) — rewrite with streaming parser
  ```
- **For a multi-line `text`** (containing one or more `\n`): the first segment follows the `— ` separator on the bullet line; each subsequent `\n`-separated segment is rendered on its own line, **indented by 2 spaces** (so the list item continues per CommonMark loose-list continuation rules). Blank segments (consecutive `\n`s) render as blank continuation lines (2-space indent, no content).
  ```
  - **TODO** L42 · Jane Doe, 2024-07-11 (a1b2c3d4e5) — refactor
    first point
  
    second point
  ```
- The date shown in Markdown is the **author-date** (same source as `blame.date`), rendered as `YYYY-MM-DD` in UTC.
- The 10-char short SHA in the Markdown line is the same value as `blame.sha` in the JSON (consistent width across formats).
- Findings with `blame: null` render as `- **TODO** L42 · (unblamed) — <text>` — no author, no date, no SHA; the literal token `(unblamed)` is stable and documented.
- All user-controlled fields (`path`, `author`, `email`, `sha`, `text`) are escaped per "Output escaping" before concatenation.

### CLI surface
```
todo-stream [path]              positional root (default: cwd); symlink roots are canonicalized once at startup
  --format json|markdown        default: markdown when stdout isTTY, json otherwise
  --markers TODO,FIXME,HACK     default: TODO,FIXME,HACK,XXX (comma list; each token must match ^[A-Za-z0-9_]+$; used verbatim, case-sensitive)
  --include "**/*.ts"           repeatable; CLI value replaces config-file list wholesale
  --exclude "**/dist/**"        repeatable; exclude wins over include; CLI value replaces config-file list wholesale
  --since 2024-01-01            YYYY-MM-DD only; inclusive; UTC; incompatible with --no-blame; compared against commit author-date
  --author <substr>             case-insensitive substring against `author <email>`; repeatable (OR); incompatible with --no-blame
  --fail-on TODO,FIXME          comma list; NOT repeatable (exit 2 on repeat); must be a subset of --markers; exit 1 if any matching finding remains after filters
  --no-gitignore                disable git-delegated .gitignore pruning
  --no-blame                    skip git blame enrichment; per-finding `blame: null` for every finding
  --staged                      limit to FILES listed by `git diff --cached --name-only -z`, intersected with the resolved root; reads working-tree bytes (documented limitation); requires a git worktree (exit 2 otherwise); files staged-but-deleted from the working tree are silently skipped; submodules not recursed
  --redact-emails               replace email local-part with `***` in both JSON and Markdown output
  --max-file-size BYTES         skip files larger than this (default 5242880, i.e. 5 MiB); 0 or negative → exit 2
  --blame-timeout SECONDS       per-invocation git-blame timeout (default 30); 0 or negative → exit 2
  --blame-concurrency N         max concurrent git-blame invocations (default 8); 0 or negative → exit 2
  --config <path>               load JSON config explicitly (no auto-discovery in v0.1); see Config file schema
  --version | --help
```

Exit codes: `0` (success, no failing findings), `1` (`--fail-on` triggered), `2` (usage or internal error — includes unknown flag, conflicting flags, bad config JSON, unknown config key, invalid marker, invalid `--since` form, `--fail-on` not a subset of `--markers`, `--fail-on` repeated, `--staged` outside a git worktree, resource-limit flag out of range, non-blame spawn failure).

## Tests plan

### Unit tests (fixture-driven, TDD — RED first)
Built **test-first** (red → green):
- **Extractor (`extract.test.ts`)** — a `tests/fixtures/` tree with one small file per supported language containing known markers in line comments, block comments, nested blocks, and code-only lines with marker-looking strings (negative case: a `const s = "TODO"` line in a `.ts` file must yield zero findings; a `const s = "TODO"` line in a `.unknownext` file **must** yield one finding, documenting the generic-fallback asymmetry). Each assertion pins exact `(line, column, marker, text)` with 1-based indexing. Additional fixtures:
  - A block comment containing both `TODO:` and `FIXME:` on **different** lines with continuation text → asserts two Findings, continuation `\n`-joined on the preceding marker per the continuation algorithm (including the blank-line `\n\n` case).
  - A **single line** containing two markers (e.g. `// TODO fix this; FIXME also this`) → asserts two Findings with identical `line`, distinct `column`, ordered ascending by `column`.
  - An **empty-marker-line block** (`/* TODO:\n * body\n */`) → asserts `text = "body"` (no leading `\n`).
  - A non-`\w` marker config → asserts exit 2 at config-load time.
  - `--markers todo` against a file containing `TODO` and `todo` → asserts only `todo` is matched (verbatim case).
  - A Python file with `"""TODO: in a docstring"""` at module level → asserts **zero findings** (Python triple-quotes are strings, not comments); a Python file with `# TODO: real` → asserts one finding.
  - A `.TS` file (uppercase extension) and a `makefile` (lowercase filename) on Linux → asserts correct language dispatch via case-insensitive lookup.
  - A `Makefile`, `Dockerfile`, and `CMakeLists.txt` fixture → asserts `#` comment dispatch and correct `language` field.
- **Walker (`walker.test.ts`)** — fixture tree exercising `--include`/`--exclude` (including overlap: same path matched by two `--include` globs yields one Finding, not two), **exclude-wins ordering** (path matching both `--include "**/*.ts"` and `--exclude "**/generated/**"` is dropped regardless of flag order on argv), root-path symlink canonicalization (a symlink passed as positional root is resolved once; `root` in output is the canonical path), internal symlink skipping (cycle-safe: a self-referencing symlink must not hang), binary-file skipping (a fixture with NUL bytes in the first 8 KiB yields zero findings), and **hostile filenames** (fixture `tests/fixtures/hostile-names/` with filenames containing `\n`, `\t`, a leading `-`, a double-quote, and a 4-byte UTF-8 codepoint → all scanned and correctly path-rendered in output). `.gitignore` delegation is covered by a small integration-style test that shells out to `git check-ignore -z` against a tiny fixture repo (checked in as a bare tarball, extracted in the test setup). **Large-file guard**: a fixture > 5 MiB → skipped with stderr warning, zero findings; `--max-file-size=0` → exit 2.
- **Renderers (`render.test.ts`)** — golden-file tests: feed a known `Finding[]` → assert byte-exact JSON and stable Markdown. Golden-file cases enumerated:
  - Empty findings list (well-formed JSON, Markdown with header only).
  - `blame: null` finding (top-level null on the `blame` key in JSON; literal `(unblamed)` in Markdown).
  - Multi-line `text` with `\n` continuations (JSON `\n` escaping + Markdown 2-space-indent continuation rendering).
  - Blank-continuation-line case producing `\n\n` inside `text` (Markdown renders a blank indented line between segments).
  - Non-ASCII author name and text (UTF-8 round-trip).
  - Long `text` (no truncation).
  - **Two findings on the same (file, line)** from two markers on one comment line — distinct `column` values, ordered by column ascending.
  - **Injection probe**: a source comment `// TODO: [link](http://evil) | injection <!-- hidden -->` — assert the rendered Markdown escapes brackets, pipe, and the HTML-comment opener so the report cannot forge a link, break the table-free bullet structure, or hide content.
  - **ANSI escape probe**: a source comment containing `\x1b[31mRED\x1b[0m` — assert Markdown output strips the escapes; JSON output preserves them but escapes the `\x1b` control byte as `\u001b`.
  - `--redact-emails` on a finding with `jane@example.com` → JSON and Markdown both emit `j***@example.com`; name and SHA unchanged.
- **Blame parser (`blame.test.ts`)** — feed canned `git blame --porcelain` output (captured, checked in) → assert parsed records, including: author-time used as `date` (distinct from committer-time in the fixture to prove selection), 10-char SHA truncation, and a `0000...` uncommitted line → `blame: null`. No real git invocation here.
- **Blame ARG_MAX chunking (`blame_chunk.test.ts`)** — swap the `runBlame` seam for a stub; synthesize a `Finding[]` with **>512** distinct lines against a single fixture file; assert ≥2 invocations, merge correctness, no duplicates, and per-invocation `-L` range count ≤512.
- **Blame per-file spawn failure (`blame_fail.test.ts`)** — swap `runBlame` to return non-zero for one file; assert that file's findings get `blame: null`, a stderr warning is emitted, and the run continues with exit 0 (or 1 if `--fail-on` triggers on the unblamed findings).
- **Blame concurrency and timeout (`blame_limits.test.ts`)** — swap `runBlame` with a delayed stub; assert concurrent in-flight count never exceeds `--blame-concurrency`; assert a stub that exceeds `--blame-timeout` is killed and degrades to `blame: null` + warning.
- **Subprocess hardening (`blame_env.test.ts`)** — swap `runBlame` with a stub that captures its `env`; assert `GIT_OPTIONAL_LOCKS=0`, `GIT_TERMINAL_PROMPT=0`, `GIT_CONFIG_GLOBAL=/dev/null`, `GIT_CONFIG_SYSTEM=/dev/null`, `LC_ALL=C.UTF-8` are all present. A slower fixture-based end-to-end test under `tests/fixtures/malicious-config-repo/` (a real tiny repo with alias/pager/fsmonitor in `.git/config`) asserts the run is unaffected.
- **Config file (`config.test.ts`)** — bad JSON → exit 2; nonexistent path → exit 2; unknown key → exit 2 with key named in error; wrong type for a known key → exit 2 with field + expected type; valid config with all fields → merged into Config with CLI-override precedence; CLI `--include` supplied once → fully replaces config-file `include` list (not appended); config-file-only invocation (no CLI flags) → respects every field; `--config ./cfg.json --markers FOO` with config `markers: ["BAR"]` → effective markers are `[FOO]`.
- **CLI flags (`cli.test.ts`)** — black-box tests invoking the compiled CLI against fixture repos. Each v0.1 flag gets at least one positive and one negative case:
  - `--fail-on FIXME` with no FIXMEs → exit 0; with one FIXME → exit 1.
  - `--fail-on FOO` when `--markers` does not include `FOO` → exit 2.
  - `--fail-on FIXME --fail-on HACK` (repeated) → exit 2.
  - `--since 2099-01-01` → empty `findings`; `--since 1970-01-01` → all findings; `--since 7d` → exit 2 (unsupported form); `--since 2024-1-1` → exit 2 (strict `YYYY-MM-DD`).
  - `--author nobody` → empty; `--author <real-name-substring>` → non-empty; `--author <email-domain>` → non-empty (covers both fields).
  - `--include "**/*.ts"` alone → only `.ts` findings; unknown glob (no matches) → empty findings, exit 0.
  - `--exclude "**/generated/**"` alone → default scan minus excluded tree.
  - `--include "**/*.ts" --exclude "**/generated/**"` → exclude-wins: a `.ts` file under `generated/` is dropped.
  - `--exclude X --include Y` with the same globs in opposite argv order → identical output (order-independence).
  - `--staged --include "**/*.ts"` → intersection: only staged `.ts` files scanned.
  - `--staged subdir/` with staged files both inside and outside `subdir/` → only files under the resolved `subdir/` are scanned.
  - `--staged` in a freshly-created `mktemp -d` outside any git worktree → exit 2.
  - `--staged` with a file staged for addition but **deleted** from the working tree → that file silently skipped.
  - `--staged` with a hostile-name file (contains `\n`) staged → correctly scanned via `-z` parse; assert at least one finding with the unusual path preserved in JSON.
  - `--config <path>` → flags loaded; bad JSON → exit 2; nonexistent path → exit 2; unknown key → exit 2; type mismatch → exit 2.
  - `--no-blame --since 2024-01-01` → exit 2.
  - `--no-gitignore` → vendored `node_modules` fixture is now visible.
  - `--markers TODO?` → exit 2.
  - `--max-file-size 0` → exit 2; `--blame-timeout -1` → exit 2; `--blame-concurrency 0` → exit 2.
  - `--redact-emails` against a fixture with `a@b.com` author → JSON `blame.email` is `a***@b.com`.
  - Unknown flag → exit 2.
  - **Output atomicity**: inject a post-render failure (via a fixture that triggers an internal assertion) with `--format json` → assert stdout is empty (buffered render was never flushed) or a complete valid JSON (if render completed before failure); never partial JSON. Diagnostics always on stderr.
  - **No-git-worktree degradation:** create a test directory under a location guarded by **`GIT_CEILING_DIRECTORIES`** (set to the test harness's tmp-root) so that any ambient git worktree in TMPDIR is ignored; place one `FIXME` file inside; assert exit 0 (or 1 under `--fail-on`), every finding has per-finding `blame: null`, JSON validates against the schema. The test setup explicitly verifies `git rev-parse --is-inside-work-tree` returns non-zero before proceeding, otherwise the test fails fast.

### Integration tests (real git, CI)
Built after the unit suite is green. Both integration clones are **pinned to specific commit SHAs** so upstream drift cannot flip CI red:
- **`postgres/postgres`** — checkout at SHA `PG_PIN_SHA` (recorded in the test harness config), shallow bounded to `src/backend/access/`. Invariants:
  1. Exits 0.
  2. **JSON output validates against the published `$schema`** using a checked-in JSON-Schema validator harness.
  3. `findings.length >= 1` (sanity floor).
  4. Every finding has a non-null `blame` object.
  5. **Curated marker-presence allow-list**: a small list of `(marker, path-glob)` pairs pinned at `PG_PIN_SHA` is asserted to produce ≥1 finding. The list is curated by hand at SHA pin time from markers known to sit in actual comments (not string literals), avoiding the `grep -r` false-positive trap entirely. When `PG_PIN_SHA` is updated, the allow-list is re-curated alongside it. (Replaces v0.5's `grep -r` probe.)
  6. Runtime well-behaved on the perf lane (see below).
- **`postgres-ai/database-lab`** — checkout at SHA `DBLAB_PIN_SHA`, full-repo scan. Invariants 1–5 as above with its own curated allow-list.
- **Count-sensitive and text-sensitive assertions live on the synthetic fixture repo** under `tests/fixtures/synthetic-repo/` (checked-in bare-git tarball with deterministic authors, dates, and SHAs). It pins `(path, line, marker, text, blame.sha)` quintuples and is the authoritative coverage for end-to-end blame enrichment correctness.
- Both real-repo integration tests are gated behind `INTEGRATION=1`.

### Performance lane (separate, not a correctness gate)
- Runs in its own CI job on **GitHub-hosted `ubuntu-latest` (4 vCPU, 16 GB RAM)**.
- Scans the pinned `postgres/postgres` subtree **three times** per job; records wall-clock for each run in a CSV artifact.
- **Fails only if all three runs exceed 90 s** (best-of-3 smoothing, hard ceiling / liveness gate). A single slow run due to noisy-neighbor runner variance does not flip CI red. The CSV is an artifact for humans to inspect, not compared to prior runs automatically. A baseline-regression gate remains deferred.

### CI matrix
- `bun test` (unit + black-box CLI) on every PR — must pass.
- `INTEGRATION=1 bun test` nightly and on `main` — must pass.
- Performance lane nightly only.
- `bun build --compile` smoke: compile, run `--help`, run against the repo itself.
- Typecheck: `bun tsc --noEmit`.
- Lint: `biome check`.

### TDD call-out (explicit)
- **Test-first (RED → GREEN):** extractor (including boundary-regex edge cases, multi-marker-per-line, continuation blank-line case, **empty-marker-line case**, generic-fallback asymmetry, Python triple-quote negative case, extension case-insensitivity, filename-dispatch for Makefile/Dockerfile/CMakeLists, and the reject-non-`\w`-marker path), walker (including exclude-wins ordering, dedup across overlapping includes, root-path symlink canonicalization, internal symlink skip, binary skip, large-file skip, hostile-filename fixture), renderers (including every enumerated golden case plus injection and ANSI probes and `--redact-emails`), blame-porcelain parser (including the `0000...` uncommitted case and author-time selection), blame ARG_MAX chunking via the `runBlame` seam, blame per-file spawn-failure degradation, blame concurrency cap and timeout, subprocess-hardening env assertions, config file schema (unknown-key rejection, type validation, merge semantics), **every v0.1 CLI flag's black-box behavior and exit code** including `--include`/`--exclude`, `--staged` root intersection, hostile-filename `-z` parse, `--max-file-size`/`--blame-timeout`/`--blame-concurrency` range validation, output-atomicity (no partial JSON on stdout), and the no-git-worktree degradation under `GIT_CEILING_DIRECTORIES`. These are deterministic and fixture-driven — write failing tests before any implementation line.
- **Test-after:** real-repo integration tests, the compile/smoke step, the perf lane. These exercise external systems where test-first yields diminishing returns.

## Team

Veteran experts to hire:
- **Veteran CLI systems engineer (1)** — owns walker, CLI surface, Bun packaging, exit-code semantics, symlink/binary-file policy, include/exclude composition, `--staged` root intersection, config-file schema and merge semantics, output atomicity.
- **Veteran parser/regex engineer (1)** — owns the extractor, comment-syntax table (including filename dispatch and case-insensitive lookup), boundary regex, and edge-case handling (marker-looking strings on code lines, nested blocks, multi-marker per line, multi-marker blocks, continuation joining with blank lines and empty-marker-line case, generic-fallback caveat, Python triple-quote suppression).
- **Veteran git-plumbing engineer (1)** — owns the blamer and the `git check-ignore` delegation: porcelain parsing, author-time selection, batching, ARG_MAX chunking via the `runBlame` seam, 10-char SHA normalization, uncommitted-line detection, graceful degradation, worktree detection, subprocess hardening (env, no-shell, hardened `GIT_*`), per-file spawn-failure handling, concurrency/timeout enforcement.
- **Veteran test engineer (1)** — owns fixture design (including hostile-filenames, malicious-config-repo, synthetic-repo bare-git fixture), golden files, black-box CLI suite, pinned-SHA integration harness with curated marker-presence allow-lists, JSON-Schema validator harness, best-of-3 perf lane, and the postgres/postgres + database-lab integration in CI.
- **Veteran security/ops engineer (1, part-time)** — owns the Security, privacy, and robustness section end-to-end: path-encoding audit, output-escaping/injection tests, data-sensitivity documentation, subprocess hardening review, output atomicity review. Pairs with the test engineer to land the injection/ANSI/malicious-config fixtures.
- **Veteran TypeScript/Bun release engineer (1, part-time)** — owns `package.json`, `bin` entry, `bun build --compile`, npm publish workflow, semver discipline, and the hosted JSON-Schema document.

Total: 4 full-time + 2 part-time.

## Implementation plan

Sprints are one week each. `⇄` = work happens in parallel; `→` = ordering dependency.

### Sprint 1 — Skeleton & red tests
- CLI engineer: scaffold repo layout, `bin/todo-stream`, `--help`, `--version`, argv parser; author red tests for flag-grammar error paths (invalid `--since`, invalid marker, unknown flag, `--fail-on` not a subset of `--markers`, `--fail-on` repeated, `--no-blame` + `--since`, `--staged` outside a worktree, `--max-file-size`/`--blame-timeout`/`--blame-concurrency` range validation, `--config` unknown-key and type-mismatch). ⇄
- Parser engineer: author fixture tree under `tests/fixtures/langs/` and write **failing** extractor tests for every language in the language table (extension + filename dispatch, case-insensitive lookup), including multi-marker-per-line, multi-marker-block, continuation joining (blank-line `\n\n` and empty-marker-line cases), marker-in-string-on-code-line (known-language suppression vs unknown-extension fallback asymmetry), Python triple-quote suppression, non-`\w`-marker-rejection, and verbatim-case cases. ⇄
- Git engineer: capture canned `git blame --porcelain` outputs into `tests/fixtures/blame/` and write **failing** parser tests for 10-char SHA normalization, author-time selection, uncommitted→null. Write **failing** ARG_MAX chunking tests using the `runBlame` seam. Write **failing** per-file spawn-failure, timeout, concurrency-cap, and hardened-env tests. ⇄
- Test engineer: stand up `bun test` in CI, add the `INTEGRATION=1` gate, scaffold black-box CLI test harness, write **failing** CLI tests for every v0.1 flag (including `--include`/`--exclude` interaction with `--staged`, hostile-filename `-z` parse, `--staged` root intersection, `--redact-emails`, output-atomicity probe) and the no-git-worktree degradation under `GIT_CEILING_DIRECTORIES`. Stand up the pinned-SHA synthetic fixture repo, the hostile-names fixture, the malicious-config-repo fixture, and the JSON-Schema validator harness. ⇄
- Security/ops engineer: author red tests for the Markdown injection probes, ANSI-escape probes, subprocess env assertions, and large-file guard. ⇄
- Release engineer: lock Bun version, add `bun tsc --noEmit` + `biome check` to CI, declare `@biomejs/biome` as a dev dep.
- **Gate:** all tests RED, CI green on lint/type.

### Sprint 2 — Core green
- Parser engineer → implement `extract.ts` (dynamic marker regex with global flag, symmetric non-identifier boundaries, `^[A-Za-z0-9_]+$` marker validator, per-occurrence Finding emission, continuation algorithm with blank-line and empty-marker-line handling, case-insensitive extension + filename dispatch) until fixtures pass.
- CLI engineer → implement `walker.ts` with `Bun.Glob` + `git check-ignore -z` delegation + root-path symlink canonicalization + internal symlink-skip + binary-skip + dedup-by-absolute-path + exclude-wins composition + `--max-file-size` enforcement, write walker tests RED→GREEN. ⇄
- Git engineer → implement blame porcelain parser (pure) with author-time selection, `0000...`→null, 10-char SHA truncation. Implement ARG_MAX chunking (constant 512) with merge logic behind the `runBlame` seam. Implement per-file spawn-failure degradation, timeout, concurrency cap, hardened env. ⇄
- Test engineer → author golden-file tests for both renderers across every enumerated case (still red; renderers not yet built); finalize the synthetic bare-git fixture repo; build the curated marker-presence allow-lists for `PG_PIN_SHA` and `DBLAB_PIN_SHA`.
- Security/ops engineer → pair with test engineer on injection/ANSI fixture implementations. ⇄
- **Gate:** extractor, walker, blame parser, ARG_MAX chunking, per-file failure, concurrency/timeout all GREEN; renderer and CLI tests still RED.

### Sprint 3 — Render & wire
- CLI engineer → implement JSON + Markdown renderers against golden tests (POSIX-relative `path`, pinned `generated_at` format, 10-char SHA, author-date rendering, **multi-line `text` → 2-space-indent continuation in Markdown**, output escaping rules, `(unblamed)` rendering, `--redact-emails`); wire end-to-end pipeline in `cli.ts`; implement exit-code semantics and all usage-error paths (including `--fail-on` repeat, `--staged` outside a worktree, `--staged` root intersection); implement output-atomicity (buffered JSON write, stderr-only diagnostics); implement config-file loader with unknown-key rejection, type validation, and wholesale-override merge semantics.
- Git engineer → implement `blame.ts` runtime wiring: production `runBlame` backed by `Bun.spawn`, hardened env, timeout via `AbortController`, concurrency gate. ⇄
- Parser engineer → extend markers config + `--markers` flag (dynamic regex, verbatim-case, marker-shape validator), finalize generic fallback extraction. ⇄
- Test engineer → start integration harness: pinned-SHA clone helper, postgres subtree scan with structural invariants + schema validation + curated marker-presence allow-list.
- Security/ops engineer → land the malicious-config-repo end-to-end test against the real blame runtime. ⇄
- **Gate:** renderer tests and a majority of CLI flag tests GREEN; `todo-stream` runs end-to-end against the repo itself and produces valid output; output-atomicity tests GREEN.

### Sprint 4 — Integration & release
- Test engineer → finalize integration tests (pinned-SHA postgres/postgres + postgres-ai/database-lab with schema validation + curated marker-presence + sanity-floor invariants); stand up the separate perf lane on `ubuntu-latest` 4 vCPU with **best-of-3** 90 s ceiling. ⇄
- Release engineer → `bun build --compile` pipeline, npm publish dry-run, `npx todo-stream` smoke, README usage + JSON schema docs + **data-sensitivity guidance**, publish the hosted `$schema` JSON, declare macOS + Linux support matrix. ⇄
- CLI engineer → finalize `--since`, `--author`, `--fail-on`, `--staged` (file-level, working-tree read, worktree check + root intersection + deleted-file skip + hostile-name `-z` parse), `--config` (explicit only), plus the `--no-blame` + `--since`/`--author` mutual-exclusion guard. ⇄
- Git engineer → hardening: large-repo shakeout, uncommitted-line regression tests against live git fixtures, malicious-config repo end-to-end. ⇄
- Parser engineer → extend language coverage (finalize Rust, Ruby, Lua, filename-dispatch suite) with fresh red→green fixtures matching the language table.
- Security/ops engineer → finalize data-sensitivity README section, sign off on injection/ANSI/hostile-name/malicious-config tests. ⇄
- **Gate:** all unit + black-box CLI tests GREEN; integration tests GREEN on CI against pinned SHAs; perf lane reporting best-of-3; `bun run build` emits a working static binary; product v0.1.0 tagged.

### Parallelization summary
- Sprint 1 is almost fully parallel (six independent red-test authoring streams, including security/injection fixtures, config-file schema, error-path grammar, ARG_MAX chunking, and resource-limit validation).
- Sprint 2 and 3 pair the parser+git engineers in parallel while the CLI engineer advances the pipeline; the test engineer stays one step ahead writing the next red tests; the security/ops engineer pairs on injection/ANSI/malicious-config fixtures.
- Sprint 4 fans out: every engineer has an independent lane.

## Embedded Changelog

- **Spec v0.1 (2026-04-21)** — Initial draft. Reframed away from the "todo-list app" interview answers (SQLite, NDJSON daemon, add/list/done subcommands, id/done/created_at model) per the authoritative idea: linter for TODO/FIXME/HACK comments, not a storage tool. Locked scope to directory walk + regex extract + git blame + JSON/Markdown report. Zero runtime deps beyond Bun built-ins.
- **Spec v0.2 (2026-04-21)** — Post-review r1 refinement. Strengthened user stories (added CI gatekeeper, pre-commit, pipeline-consumer personas). Tightened the Architecture diagram and component boundaries. Added the concrete JSON schema example and Markdown output shape. Expanded the Tests plan with explicit TDD red-first call-outs and the `INTEGRATION=1`-gated postgres/postgres + database-lab harness. Specified team composition and a four-sprint parallelized plan.
- **Spec v0.3 (2026-04-21)** — Post-review r2 refinement addressing Reviewer B's findings. Introduced the spec-version vs product-version distinction; reconciled JSON schema prose and example; stated dynamic marker regex; pinned `--staged` to file-level; made `--since`/`--author` incompatible with `--no-blame`; replaced the pinned-upstream-TODO assertion with structural invariants plus a synthetic fixture; moved the perf number to a separate perf lane; declared string-literal tokenization a non-goal; specified multi-marker block handling; delegated `.gitignore` to `git check-ignore`; pinned `blame: null` wholesale.
- **Spec v0.4 (2026-04-21)** — Post-review r3 refinement. Pinned regex boundary semantics, marker shape, `--since`/`--author`/`--fail-on` grammars, POSIX-relative `path`, 10-char SHA, `generated_at` format, language table, uncommitted-line handling, symlink/binary policy, continuation joining, 1-based indexing, pinned-SHA integration clones, no-git-worktree test, renderer golden enumeration, and macOS+Linux platform declaration.
- **Spec v0.5 (2026-04-21)** — Post-review r4 refinement. Resolved the per-line vs per-occurrence contradiction (one Finding per marker occurrence). Pinned `blame.date` to commit author-date everywhere. Made the continuation algorithm authoritative with a worked example and blank-line handling. Removed config-file auto-discovery. Added `--include`/`--exclude` interaction, order-independence, and `--staged` intersection tests. Strengthened integration invariants with schema validation and a grep-based marker-presence probe. Walker deduplicates by resolved absolute path; root-path symlinks canonicalized. `--staged` outside a worktree → exit 2; staged-deleted files silently skipped. `--fail-on` non-repeatable. ARG_MAX chunk size fixed at 512. `GIT_CEILING_DIRECTORIES` guard on the no-git-worktree test. `@biomejs/biome` declared as dev dep.
- **Spec v0.6 (2026-04-21)** — Post-review r5 refinement addressing both reviewers. **New Security, privacy, and robustness section** covering: hostile filenames (NUL-delimited git I/O, argv-only path passing, UTF-8 handling, dedicated fixture); Markdown/terminal output escaping (backticks, pipes, brackets, HTML-comment openers, ANSI-escape stripping); data sensitivity with `--redact-emails` flag and README guidance; resource limits (`--max-file-size` default 5 MiB, `--blame-timeout` default 30 s, `--blame-concurrency` default 8, with range validation); subprocess hardening (`GIT_OPTIONAL_LOCKS=0`, `GIT_TERMINAL_PROMPT=0`, `GIT_CONFIG_GLOBAL=/dev/null`, `GIT_CONFIG_SYSTEM=/dev/null`, `LC_ALL=C.UTF-8`, hooksPath=/dev/null, malicious-config-repo fixture); output atomicity (stdout buffered for JSON, stderr-only diagnostics). **CI gatekeeper user story 1 rewritten** to acknowledge `--fail-on` is a whole-repo gate and document the v0.1 workarounds; diff/baseline mode explicitly deferred. **`--staged` hardened**: uses `-z` NUL-delimited output, intersects with the resolved scan root, pins worktree discovery via `git -C`, documents submodule non-recursion. **Config file schema pinned**: authoritative key list, wholesale-override merge semantics for list-valued fields, unknown-key and type-mismatch rejection with exit 2. **Markdown multi-line `text` rendering pinned**: 2-space-indent continuation lines per CommonMark loose-list rules; `(unblamed)` token for null-blame findings. **Continuation algorithm extended** with the empty-marker-line case (leading `\n` omitted). **Per-file blame spawn failure** degrades that file's findings to `blame: null` + stderr warning; run continues. **Python triple-quote handling removed**: Python comments are `#` only in v0.1; docstrings are strings. **Extension dispatch is case-insensitive**; added **exact-filename dispatch** for `Makefile`, `Dockerfile`, `Jenkinsfile`, `CMakeLists.txt`, `.gitignore`, `.dockerignore`. **Integration invariant 5 replaced**: curated marker-presence allow-list pinned at each integration SHA instead of `grep -r` probe (avoids false positives from markers in string literals). **Perf lane smoothed**: best-of-3 runs per job, fails only if all three exceed 90 s. **ARG_MAX spawn seam pinned**: `runBlame(args)` is the sole spawn boundary in `blame.ts`, swappable in tests. **Sprint-4 large-file guard specified**: flag + range validation + test. Added a part-time security/ops engineer to the team. Rejected over-scope-reduction: the postgres/postgres + database-lab clones, hosted schema, npm packaging, macOS+Linux binaries, and perf lane are all load-bearing for the stated user stories and release obligations.
