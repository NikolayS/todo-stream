# todo-stream — SPEC v0.7

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

- **Spec version** — the version of *this document*. Appears only in the `# todo-stream — SPEC vX.Y` header and the embedded changelog. Bumps every review round. Currently **v0.7**.
- **Product version** — the version of the *published npm package and compiled binary*. Appears in `package.json`, the JSON report's `version` field, and `--version` output. Currently **0.1.0** (pre-release; will ship as 0.1.0 when the v0.1 scope below is implemented).

The JSON schema example below therefore shows `"version": "0.1.0"` (product), while this document's header is `SPEC v0.7`.

## User stories

1. **CI gatekeeper — Priya, release engineer.** Priya adds `todo-stream --format json --fail-on FIXME` to the repo's CI workflow. **`--fail-on` is a whole-repo gate**: any matching finding (pre-existing or new) trips exit 1. For a mature repo with pre-existing FIXMEs, Priya either (a) removes FIXME from the default marker set for CI while keeping TODO/HACK scanning, (b) uses `--since <baseline-date>` to restrict the gate to findings introduced after a cutoff author-date, or (c) waits for a diff/baseline mode (deferred to a later product version — see Scope). The v0.1 product intentionally does **not** provide a diff-against-base mode; the user story makes this trade-off explicit so the tool is not shipped under a false promise.
2. **Tech-debt triage — Luis, staff engineer.** Luis runs `todo-stream --format markdown --since 2024-01-01 > DEBT.md` against the monorepo once a quarter, grouping findings by file with blame dates, and uses the Markdown report in a planning meeting to assign owners.
3. **Incoming maintainer — Dana, new OSS contributor.** Dana clones a large project (e.g. `postgres/postgres`) and runs `todo-stream src/backend --markers TODO,HACK --format markdown | less` to orient herself: she sees the oldest HACKs, who wrote them, and which files are hot-spots, without learning the project's in-house tooling.
4. **Pre-commit author — Sam, individual developer.** Sam wires `todo-stream --staged --format json` into a `lefthook` pre-commit. `--staged` limits the scan to the *set of files* listed by `git diff --cached --name-only -z` (NUL-delimited; see CLI surface) but reads their **working-tree** bytes (not the staged blob) — consistent with linting tools like ESLint and the pragmatics of pre-commit hooks; documented as a known limitation.
5. **Pipeline consumer — automated dashboard.** A scheduled job runs `todo-stream --format json` and pipes the output into a downstream ingester that renders a historical chart of open TODO count per file — the JSON schema is stable and documented.

## Scope & non-goals (product v0.1)

**In scope (product v0.1)**
- Recursive walk of a directory with configurable include/exclude globs (defaults respect `.gitignore` when inside a git worktree, and **always** prune `.git/**` and `.hg/**` / `.svn/**` regardless of `.gitignore` state).
- Extraction of configurable markers (default: `TODO`, `FIXME`, `HACK`, `XXX`) from line and block comments in common languages (see the language table below). Unknown extensions fall back to a generic regex with documented caveats.
- Git blame enrichment per finding: author name, author email, commit **author-date** (ISO-8601 UTC), 10-char short SHA. Non-git trees degrade gracefully (per-finding `blame: null`).
- Two output formats: `json` (stable, documented schema) and `markdown` (human-readable, grouped by file).
- Filters: `--markers`, `--since`, `--author`, `--include`, `--exclude`, `--fail-on`, `--staged`.
- Resource guards: `--max-file-size` (default 5 MiB), `--blame-timeout` (default 30 s per invocation), `--blame-concurrency` (default 8), `--max-findings` (default 100000), `--max-output-bytes` (default 64 MiB).
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
- **Config file auto-discovery.** v0.1 does **not** auto-discover `todo-stream.config.json` from cwd or walk up from the scan root. A config file is loaded **only** when `--config <path>` is passed explicitly.
- **Diff/baseline CI mode** ("fail only on FIXMEs introduced in this PR"). Deferred to a later product version; see user story 1 for the v0.1 workarounds.
- **Python triple-quoted string handling as block comments.** Triple-quoted strings in Python are *strings*, not comments, even when used as docstrings. The extractor only recognizes `#` as a Python comment syntax in v0.1.
- **String-literal tokenization.** The extractor inspects only the *comment region* of a line (for languages in the language table); it does not parse string literals. A `TODO` appearing inside a string on a line that is not otherwise a comment is *not* a finding in known languages. A `TODO` inside a string *within* a comment (e.g. `// TODO: see "FIXME" in docs`) remains a single `TODO` finding. For the **generic (unknown-extension) fallback**, no comment-region reduction is performed; see the fallback caveat in Implementation details.
- **Name-redaction, full-text redaction, regex redaction.** Deferred; only `--redact-emails` in v0.1.
- **Trend / baseline-regression perf gate.** The perf lane is a coarse liveness ceiling only.

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

**Shebang lines.** For files dispatched as `shell` (i.e. `.sh` / `.bash` / `.zsh`) or as `python`, a line 1 that begins with `#!` is **not** treated as a comment by the extractor — it is skipped for marker extraction regardless of whether a marker token appears later on that line. Rationale: shebangs are universal and appear on files Dana orients over (user story 3); surfacing `TODO: switch to bun` in `#!/usr/bin/env bash # TODO: switch to bun` is noise. A shebang fixture pins this behavior.

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
- **CLI (`src/cli.ts`)** — argument parsing (hand-rolled, no deps), config resolution (CLI flags > `--config` file > defaults), dispatch. Owns process exit code. Owns stdout/stderr discipline (see "Output atomicity" below). Owns the `runGit` seam used by every git subprocess caller (see below).
- **Walker (`src/walker.ts`)** — streams matching file paths. Uses `Bun.Glob` for include/exclude. `.gitignore` pruning is delegated to `git check-ignore --stdin -z -v` via `runGit` when the root is inside a git worktree; outside a worktree (or when `git` is not on PATH at all), no `.gitignore` pruning is performed (users may still use `--exclude`). **`.git/`, `.hg/`, and `.svn/` directories are unconditionally pruned** at every depth (walker-level built-in exclude) regardless of `.gitignore` state, `--no-gitignore`, or user `--include` globs. **Symlinks are not followed** during traversal (neither files nor directories) — avoids cycles and surprise traversal outside the root; symlinks are skipped and not reported. **Root-path symlinks** (the positional `path` argument itself) are resolved exactly once at startup to their canonical absolute path, and traversal proceeds from that canonical path; the resolved path is used as the `root` field in the JSON output. If the resolved root does not exist or is not a regular file / directory, the CLI exits 2 with a readable error. The walker **deduplicates paths by resolved absolute path** before emitting — a path matched by multiple `--include` globs is yielded only once. The walker yields paths in **stable sorted order** (lexicographic byte-ordering over the POSIX-relative `path`), so cross-file output ordering is deterministic regardless of filesystem.
- **Extractor (`src/extract.ts`)** — pure function: `(path, bytes, config) → Finding[]`. Dispatches on extension (lowercased) then filename to the language table. Returns `{path, line, column, marker, text, raw}` — no I/O, fully unit-testable from fixtures. Takes **bytes** (Uint8Array), not decoded strings (see "Byte decoding" below).
- **Blamer (`src/blame.ts`)** — batches findings per file and shells out via the `runGit` seam (inherited from the CLI; see "The `runGit` seam" below). Issues `git blame --porcelain -L a,a -L b,b -- <file>` per file (subject to ARG_MAX chunking below), parses porcelain into `{author, email, date, sha}`. `date` is always the commit **author-date** (`author-time` + `author-tz` from porcelain), normalized to ISO-8601 UTC with second precision. Caches by absolute file path. Degrades to per-finding `blame: null` outside a git worktree, when git is absent, for untracked files, for lines reported as uncommitted, and on per-file spawn failure (see "Per-file blame spawn failures" below). Blamer concurrency across files is capped at `--blame-concurrency`.
- **Renderer (`src/render/*.ts`)** — two pure renderers: `json.ts` (emits the versioned schema defined below) and `markdown.ts` (groups by file, sorts by line then column; escapes untrusted text per "Output escaping" below).
- **Reporter (in `cli.ts`)** — **buffers the complete rendered output before writing to stdout for both JSON and Markdown** (atomicity guarantee); see "Output atomicity" below. Applies `--fail-on` rules, sets exit code. Writes diagnostics only to stderr.

**Key abstractions**
- `Finding` — the one shared record shape crossing every boundary.
- `CommentSyntax` — `{ line?: string[]; block?: [open, close][] }`, looked up by extension or filename.
- `Config` — frozen object assembled once at startup.
- `runGit` — the **single injectable spawn seam for every git invocation** the tool makes (`blame`, `check-ignore`, `rev-parse --show-toplevel`, `rev-parse --is-inside-work-tree`, `diff --cached --name-only -z`). Signature: `runGit(args: string[], opts?: {stdin?: Uint8Array; timeoutMs?: number}) → Promise<{stdout: Uint8Array; exitCode: number}>`. Tests swap this seam wholesale to cover ARG_MAX chunking, concurrency, timeouts, hardened-env assertions, and per-subcommand failure modes uniformly.

**Dependency policy.** Runtime deps = zero beyond Bun built-ins (`Bun.Glob`, `Bun.spawn`, `Bun.file`, `fs`, `path`). `git` itself is assumed on PATH at startup (resolved once; see "Subprocess hardening"); the tool degrades without it as documented. Dev deps are limited to `bun test`, `@biomejs/biome` (lint + format), and type definitions.

## Security, privacy, and robustness

This section pins operational guarantees for running `todo-stream` against untrusted repositories and emitting output into CI artifacts. All items are testable and appear in the Tests plan.

### The `runGit` seam (single git spawn boundary)
- **All** git subprocess invocations go through `runGit`. No `Bun.spawn(["git", ...])` call exists outside `runGit`'s implementation in `cli.ts`.
- `runGit` resolves the git binary **once at startup** to an absolute path (see "Subprocess hardening / PATH" below) and stores it in the frozen `Config`. Every subsequent invocation uses this absolute path.
- Tests swap `runGit` at the module boundary to capture: argv, env, stdin, and to return synthetic `(stdout, exitCode)` tuples. This lets the test suite assert the hardened env is present on `check-ignore`, `rev-parse`, `diff --cached`, and `blame` — not just blame.

### Subprocess hardening
- **No shell, ever.** `Bun.spawn` is invoked with an argv array; no shell is spawned, no interpolation performed.
- **Absolute git path.** At startup, `git` is resolved to an absolute path by scanning the parent-process `PATH` **after** PATH sanitization (see below). If no trusted absolute path is found, the tool runs in fully-degraded mode: every finding gets `blame: null`, `.gitignore` pruning is skipped, `--staged` exits 2 with a readable error, and a single stderr warning is emitted at startup.
- **PATH sanitization.** The PATH used to resolve git is built by filtering the parent-process `PATH` as follows: (a) split on `:`; (b) **drop** empty components, `.`, `..`, and any component that is not an absolute path starting with `/`; (c) reject components whose canonicalized real path resolves under the scan root (defense against a repo-local `bin/git` shadowing the system binary). The resolved git path must be an absolute path *not* under the scan root; if the only git found is under the scan root, resolution fails and the tool runs fully-degraded.
- **Hardened env (`HARDENED_GIT_ENV`)** is passed to *every* `runGit` call:
  - `GIT_OPTIONAL_LOCKS=0` (avoid touching `.git/index.lock`).
  - `GIT_TERMINAL_PROMPT=0` (never prompt).
  - `GIT_CONFIG_GLOBAL=/dev/null` and `GIT_CONFIG_SYSTEM=/dev/null` (ignore user/system config).
  - `GIT_CONFIG_NOSYSTEM=1` (belt + suspenders alongside `GIT_CONFIG_SYSTEM`).
  - `LC_ALL=C.UTF-8` and `LANG=C.UTF-8` (stable error strings).
  - `PATH` set to the sanitized PATH computed at startup (not the raw parent PATH).
- **Per-invocation config neutralization.** Every `runGit` invocation also passes these `-c key=value` flags *before* the subcommand, to neutralize dangerous settings from the repo-local `.git/config` (which is **not** covered by `GIT_CONFIG_GLOBAL`/`SYSTEM`): `core.fsmonitor=`, `core.fsmonitorHookVersion=0`, `core.hooksPath=/dev/null`, `core.sshCommand=`, `core.askPass=`, `core.editor=false`, `core.pager=cat`, `core.autocrlf=false`, `core.symlinks=false`, `diff.external=`, `diff.textconv=`, `filter.*.clean=`, `filter.*.smudge=`, `filter.*.process=`, `protocol.ext.allow=never`, `url.*.insteadOf=`, `alias.*=`, `help.autoCorrect=never`. (The `filter.*` and `alias.*` and `url.*` wildcards are set via repeated `-c` passes for the wildcard prefix — git treats an empty RHS as "clear this group" for the filter/alias/url namespaces, and an alias invocation form cannot be constructed anyway because `runGit` always passes a concrete subcommand as the first positional arg.)
- **Always `--no-pager`** is passed to every subcommand.
- **No `--textconv`** on `git blame` (textconv is off by default; stated for the record and enforced by the neutralized `diff.textconv=`).
- A fixture `tests/fixtures/malicious-config-repo/` contains a repo-local `.git/config` with `alias.*`, `pager.*`, `core.fsmonitor`, `core.sshCommand`, `filter.lfs.process`, and `diff.external` settings that would normally execute commands. Tests assert the tool runs unaffected and that **no unexpected subprocess fans out** (verified by a `Bun.spawn` call-counter wrapper).

### Path encoding and hostile filenames
- All git subprocess invocations that take or return paths use **NUL-delimited** modes: `git diff --cached --name-only -z`, `git check-ignore --stdin -z -v`, `git ls-files -z` (if used). Parsers split on `\0`, never on `\n` or whitespace.
- Paths are always passed as **argv after `--`**, never interpolated into a shell command.
- Filenames containing newlines, tabs, spaces, leading dashes, or non-ASCII bytes are supported. A fixture `tests/fixtures/hostile-names/` exercises filenames containing `\n`, `\t`, a leading `-`, a double-quote, and a 4-byte UTF-8 codepoint. The fixture is a checked-in bare-git tarball (filesystems allowing).

### Byte decoding and invalid UTF-8 (authoritative)
- Source files are read as **bytes** via `Bun.file(path).bytes()` (returns `Uint8Array`), **not** via `.text()`. The Extractor operates on the byte buffer.
- **Line splitting** is performed over bytes on the byte sequences `\x0A` (LF) and `\x0D\x0A` (CRLF); a bare `\x0D` (CR) does **not** split lines. Each split line preserves no trailing line terminator.
- **CRLF normalization.** Before the continuation algorithm runs, the Extractor normalizes each extracted segment's line terminators to LF. The `text` field therefore never contains `\r` bytes from CRLF line endings. A fixture with CRLF endings pins this.
- **Regex matching** runs over the byte buffer treated as Latin-1 (each byte is one code unit); marker tokens are restricted to `^[A-Za-z0-9_]+$` (ASCII only), so the byte view suffices for boundary matching without Unicode-aware regex.
- **JSON output decoding.** When emitting a string field, the byte slice for that field is decoded as UTF-8 using `new TextDecoder("utf-8", {fatal: false})`, which substitutes U+FFFD for any invalid byte sequence. This guarantees the JSON output is always valid UTF-8; a small stderr warning is emitted once per file (not per finding) when substitution occurred during decoding.
- **Markdown output decoding** follows the same rule (UTF-8 with U+FFFD replacement), then applies the escape rules in "Output escaping".
- A fixture `tests/fixtures/invalid-utf8/` contains a `.c` file with a `// TODO: café` comment where `café` is encoded as Latin-1 (`0xE9` for `é`); assert the finding is emitted with `é` replaced by U+FFFD in both JSON and Markdown.
- **UTF-16 / UTF-32 source files** contain NUL bytes in ASCII-range codepoints and are therefore **skipped by the binary-file heuristic** (see "Resource limits" below). This is an accepted consequence of the NUL-in-first-8KiB heuristic and is documented here so users scanning UTF-16-encoded codebases do so via a wrapper that re-encodes to UTF-8 first.

### Output escaping
- **ANSI / terminal-control grammar (authoritative).** The following byte sequences are "ANSI escape sequences" for the purposes of this spec:
  - **7-bit CSI:** `\x1b\[` followed by zero or more parameter bytes (`\x30-\x3f`), zero or more intermediate bytes (`\x20-\x2f`), and a final byte (`\x40-\x7e`).
  - **7-bit OSC:** `\x1b\]` followed by any bytes up to and including a terminator that is either `\x07` (BEL) or `\x1b\\` (ST).
  - **7-bit single-char escapes:** `\x1b` followed by a single byte in `\x40-\x5f` (excluding `[` and `]`, which are covered above).
  - **8-bit CSI and OSC** (`\x9b` and `\x9d` introducers) are also recognized and stripped.
- **Markdown renderer** applies, to `path`, `author`, `email`, `text`, and `sha` (in this order):
  1. UTF-8 decode with U+FFFD replacement (if operating on bytes).
  2. Strip every ANSI escape sequence per the grammar above.
  3. Strip every ASCII control character `< 0x20` except `\n` and `\t`, and strip `\x7f` (DEL).
  4. Escape, in text order: backticks (`` ` `` → `` \` ``), pipe (`|` → `\|`), square brackets (`[` → `\[`, `]` → `\]`), angle brackets (`<` → `\<`, `>` → `\>`), and the literal string `<!--` by escaping its leading `<`.
  - These rules are applied *after* the continuation algorithm has joined `text`, so the `\n` segments remain the separator for the multi-line rendering rule (see "Markdown output shape").
- **JSON renderer** applies, to `path`, `author`, `email`, `text`, `sha`, and `root`:
  1. UTF-8 decode with U+FFFD replacement (if operating on bytes).
  2. **Strip every ANSI escape sequence** per the grammar above. Rationale: although ANSI bytes are JSON-safe, log viewers and dashboards that re-emit decoded JSON strings to terminals would interpret them, creating log-spoofing and OSC-hyperlink phishing risks; stripping is the safer default and is symmetric with Markdown. There is no v0.1 flag to preserve ANSI in JSON; a future `--preserve-ansi` opt-out is deferred.
  3. Strip every ASCII control character `< 0x20` except `\n` and `\t`, and strip `\x7f` (DEL). (Remaining `\n` and `\t` are emitted as `\n` / `\t` JSON escapes per standard JSON rules.)
- **Stderr diagnostics** use plain ASCII, never ANSI colors.
- Fixtures verify:
  - A source comment `// TODO: [link](http://evil) \| injection <!-- hidden -->` cannot create a Markdown link, table column break, or hidden span.
  - A source comment containing `\x1b[31mRED\x1b[0m`, a 7-bit OSC hyperlink (`\x1b]8;;http://evil\x07text\x1b]8;;\x07`), an 8-bit CSI (`\x9b31mRED\x9b0m`), and a DEC private-mode escape (`\x1b[?25l`) all result in those bytes being absent from **both** JSON and Markdown output.

### Data sensitivity
- `todo-stream` output can contain: source comment text, author names, author emails, file paths, commit SHAs. This is sensitive in public CI and third-party dashboards.
- The README documents this explicitly and recommends treating `todo-stream` output as source-code-equivalent for visibility purposes.
- `--redact-emails` replaces the email local-part with a fixed opaque token. Formula: the entire local-part is replaced with `***` (three asterisks) regardless of length — so `jane@example.com` → `***@example.com` and `a@b.com` → `***@b.com`. The v0.6 "first char + ***" formula was rejected because it leaks the entire local-part for single-character usernames. Applied to both JSON (`blame.email`) and Markdown output. Name and SHA are unaffected.
- Further redaction (full-text filters, regex redaction, name redaction) is deferred.

### Resource limits
- **Per-file size cap.** Files whose `stat().size` exceeds `--max-file-size` (default 5 MiB) are skipped with a stderr warning and contribute zero findings. Configurable by flag; any value `<= 0` is a usage error (exit 2).
- **Binary-file skip.** A file is considered binary and skipped (zero findings, no warning) if any of the first 8 KiB contains a NUL byte (`\x00`). This heuristic deliberately skips UTF-16/UTF-32 source files; see "Byte decoding".
- **Blame invocation timeout.** Each `runGit` call for `blame` is killed after `--blame-timeout` seconds (default 30). A timed-out invocation is treated identically to a per-file spawn failure (see below).
- **Blame concurrency cap.** At most `--blame-concurrency` (default 8) concurrent `runGit` `blame` invocations. Files queue behind the cap.
- **Global max findings.** `--max-findings` (default 100000) caps the total number of findings retained in memory and emitted. When the cap is reached, the walker stops emitting new paths, the current batch of in-flight extractions is allowed to drain, and a single stderr warning `warning: --max-findings cap reached; output is truncated` is emitted. The process exits **2** (internal/resource error) by default — rationale: silently truncating a linter report is a correctness hazard for CI consumers. Users who want the cap to be informational can set it higher; there is no v0.1 flag to turn the exit-2-on-cap into exit-0.
- **Global max output bytes.** `--max-output-bytes` (default 64 MiB) caps the size of the *rendered* output. If the rendered JSON or Markdown buffer exceeds the cap, the tool **writes nothing to stdout**, emits `error: --max-output-bytes cap exceeded; no output written` to stderr, and exits 2. This prevents unbounded stdout pipes from OOM'ing CI runners or downstream consumers.
- **No truncation of `text`.** Finding `text` (including multi-line continuations) is emitted in full, subject only to the global output-bytes cap above. Long block comments produce long strings; this is a documented trade-off (linters must faithfully reproduce the comment to be useful). A future version may add `--max-text-bytes`.

### Per-file blame spawn failures
- If `runGit` for a specific file's blame exits non-zero, times out, or fails to parse, **all findings for that file get `blame: null`** and a single stderr warning is emitted: `warning: git blame failed for <path>: <reason>`. The run continues. Exit code is not affected by per-file blame failures.
- If `git` was not resolvable at startup (see PATH sanitization), the entire run degrades: every finding gets `blame: null`, `--staged` exits 2 with a readable error (since it cannot enumerate staged files), and `.gitignore` pruning is skipped. A single stderr warning is emitted at startup.
- Exit 2 is reserved for CLI spawn failures of non-blame calls (e.g. `git rev-parse` on startup when inside a worktree), for `--staged` without git, for resource-cap violations, and for unrecoverable errors.

### Output atomicity
- **stdout carries only the report.** **stderr carries every diagnostic** (warnings, progress, error messages).
- **Both the JSON and Markdown renderers buffer the full report in memory** and the Reporter writes the buffer as a single `stdout.write()` call after filtering, rendering, and the `--max-output-bytes` check complete. Consumers never see a partial document on stdout for either format, even if the process exits 2 mid-run. (The v0.6 allowance for streaming Markdown is removed; symmetric full-buffering is simpler to reason about, testable with the same contract, and paired with the `--max-output-bytes` ceiling to keep the memory envelope bounded.)
- If the Reporter decides to exit 2 after rendering but before the final write (e.g. the rendered buffer exceeds `--max-output-bytes`), stdout is empty. Consumers should gate on exit code before parsing — documented in the README.

## Implementation details

### Data flow
1. `cli.ts` parses argv → `Config`. If `--config <path>` is passed, the JSON file is merged under CLI flags per the precedence rule (CLI flags > --config file > defaults). No auto-discovery. `runGit` is constructed once, with the absolute git path resolved via PATH sanitization; if resolution fails, `Config.gitAvailable = false`.
2. `walker.ts` yields file paths matching globs, pruning via `git check-ignore` (inside a worktree, via `runGit`) and `--exclude`. **Unconditional prunes**: `.git/`, `.hg/`, `.svn/` at every depth — applied before and independent of `.gitignore` and `--no-gitignore`. **Ordering of `--include` and `--exclude`: exclude wins.** A path matched by any `--exclude` glob is dropped even if it is also matched by `--include`. The order in which `--include` and `--exclude` appear on the argv does not matter. Paths are yielded deduplicated by resolved absolute path and in **lexicographic byte-order** over the POSIX-relative `path`. Files exceeding `--max-file-size` are dropped with a stderr warning. Files where the first 8 KiB contains a NUL byte are dropped silently (binary skip).
3. For each path, `Bun.file(path).bytes()` → `extract.ts` → `Finding[]` (streamed, not accumulated repo-wide).
4. Findings are grouped by file and handed to `blame.ts`, which issues one or more `git blame` invocations per file (see ARG_MAX chunking below) through `runGit`, with all needed line ranges. Concurrency across files is capped at `--blame-concurrency`; per-invocation timeout is `--blame-timeout`.
5. Filters `--since` and `--author` are applied **after** blame enrichment (see precedence rules below).
6. The `--max-findings` cap is enforced during accumulation (see Resource limits).
7. Enriched findings feed the chosen renderer; the Reporter buffers the full output, checks it against `--max-output-bytes`, and writes atomically to stdout.
8. `--fail-on <marker>[,<marker>...]` sets exit code 1 if any matching finding exists after filtering; absence of matching findings exits 0. Internal errors (I/O, bad config, spawn failure of non-blame calls, usage errors, resource-cap violations) exit 2.

### Config file schema (authoritative)

`--config <path>` loads a JSON file with the following top-level keys. Unknown keys are **rejected with exit 2 and a readable error naming the offending key**. All keys are optional.

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
  "blameConcurrency": 8,
  "maxFindings": 100000,
  "maxOutputBytes": 67108864
}
```

- Field types are validated; type mismatch → exit 2 with a readable error naming the offending field and expected type.
- Values are subject to the same validation as the equivalent CLI flag (e.g. `markers` entries must match `^[A-Za-z0-9_]+$`; `since` must match `YYYY-MM-DD`; `failOn` must be a subset of effective `markers`).
- **Merge semantics (authoritative).** For each field, a value supplied on the CLI **replaces** the config-file value *wholesale*. For list-valued fields (`markers`, `include`, `exclude`, `author`, `failOn`), supplying the corresponding CLI flag (even once) overrides the entire config list — CLI values are *not* appended.
- Defaults apply only when neither the CLI nor the config file supplies a value.
- The config file does **not** support comments (it is strict JSON). Parse errors → exit 2.

### Finding extraction algorithm
- The marker regex is **built dynamically** from `config.markers`. Each marker is regex-escaped and joined. Boundaries are symmetric and use a non-identifier character class on both sides (not `\b`):
  ```
  new RegExp(
    "(^|[^A-Za-z0-9_])(" + markers.map(escape).join("|") + ")(?=[^A-Za-z0-9_]|$)[:\\s-]?\\s*(.*)",
    "g"
  )
  ```
  The regex is executed with the global flag. **User-supplied markers must match `^[A-Za-z0-9_]+$`**. Markers outside this class are rejected at config-load time (exit 2). Match is case-sensitive; markers are used **verbatim**.
- **Byte basis.** The regex runs over bytes decoded as Latin-1 code-point-by-code-point (each byte is one code unit). Because marker tokens are restricted to ASCII, this is safe for all UTF-8 input: no multi-byte UTF-8 sequence can produce a byte in `[A-Za-z0-9_]`.
- **Line splitting.** Performed over `\x0A` and `\x0D\x0A` per "Byte decoding". Line 1 is the first line.
- **Shebang skip.** If the dispatched language is `shell` or `python` and line 1 begins with bytes `0x23 0x21` (`#!`), line 1 is not scanned for markers.
- For each line of a file with a **known** extension or filename, the extractor first reduces the line to its comment region using the language's `CommentSyntax`. It does **not** tokenize string literals.
- **Generic fallback caveat.** For unknown extensions/filenames, the boundary regex runs against the full line without comment-region reduction. A `TODO` inside a string literal in an unknown file **will** be reported; the same construct in a `.ts` file will not.
- **Extension and filename lookup are case-insensitive.** Both are lowercased before table lookup.
- **Line and column indexing.** `line` is 1-based. `column` is 1-based and refers to the column of the first character of the matched marker token in the original line. Columns are counted in **UTF-8 code points** of the decoded line (replacement-safe), not in bytes.
- **Multi-marker handling (authoritative).** **Each marker *occurrence* yields its own Finding** — not each line. Findings are emitted in source order: primarily by `line` ascending, secondarily by `column` ascending.
- **Continuation-text attachment (authoritative algorithm).** Inside a *block comment* of a known language, any line between two marker occurrences — or after the final marker occurrence within that block — that contains **no** marker occurrence is a **continuation line** attached to the immediately preceding marker occurrence's `text`. The algorithm is:
  1. Normalize CRLF → LF on the block's lines.
  2. Strip the block-comment's own punctuation: `/*` / `*/` delimiters on the opening/closing lines, and leading `*` (C-family), leading `--` (SQL / Lua), and surrounding whitespace on continuation lines.
  3. If the resulting continuation line is **empty**, represent it as a single empty segment.
  4. Concatenate the preceding marker's `text` with each continuation segment using a single `\n` between segments. Consecutive empty continuations produce consecutive `\n` characters.
  5. **Empty-marker-line special case.** If the marker-line's `text` is empty after stripping (e.g. `/* TODO:\n * body\n */`), the concatenation **omits the leading `\n`** and begins with the first continuation segment (`text = "body"`, not `"\nbody"`).
  
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
- Marker-line text trimming: within the marker-line itself, the optional `:` or `-` immediately after the marker token is stripped from the captured `text`, along with surrounding whitespace and the leading block-comment punctuation of that line.
- **Binary files** are skipped by the walker when any of the first 8 KiB contains a NUL byte.

### Filter precedence and blame-dependent flags
- `--since` and `--author` operate on blame metadata.
- **`--since` grammar.** Accepts strictly `YYYY-MM-DD` in v0.1. The date is interpreted as **UTC midnight** (`00:00:00Z`). The comparison is **inclusive of the boundary**: a finding with `blame.date >= <since>T00:00:00Z` is kept. The blame date used is the commit **author-date** as emitted by `git blame --porcelain` (`author-time` + `author-tz`), normalized to UTC. A dedicated fixture commit authored at `2024-01-01T00:30:00+0900` (UTC: `2023-12-31T15:30:00Z`) with `--since 2024-01-01` must be **dropped** — this pins UTC-based comparison and prevents regressions to local-time or committer-time.
- **`--author` matching.** Matches a finding if **either** the blame `author` field **or** the blame `email` field contains the query substring (case-insensitive). Fields are matched **independently** — the query does not span the name/email boundary. Multiple substrings via repeated `--author` flags are OR'd. (Revised from v0.6, where the concatenated `author <email>` form allowed boundary-straddling substrings; a regression fixture pins that `--author "e <j"` against `Jane <jane@example.com>` no longer matches.)
- If `--no-blame` is passed *together* with `--since` or `--author`, the CLI exits 2 with a usage error.
- When blame is enabled but a particular finding has `blame: null`, applying `--since` or `--author` **drops** that finding from the output.
- `--fail-on` is evaluated **after** all filters. A repo with FIXMEs that are all filtered out by `--since` exits 0 — pinned by a dedicated test fixture.
- **`--fail-on` grammar.** Accepts a comma-separated list of markers. The flag is **not repeatable**: passing `--fail-on` more than once is a usage error (exit 2). The set must be a subset of `--markers`; a disjoint value is exit 2.

### `--staged` semantics (authoritative)
- `--staged` requires the resolved positional root to be **inside a git worktree**; otherwise exit 2.
- If the positional root is a **single file** (not a directory), worktree anchoring uses the file's parent directory: `git -C <parent-dir> rev-parse --show-toplevel` (since `git -C <file>` is invalid).
- The worktree root is discovered via `runGit(["-C", <anchor>, "rev-parse", "--show-toplevel"])`. All staged-file enumeration runs with the same `-C` anchor.
- The staged file list is produced by `runGit(["-C", <worktree-root>, "diff", "--cached", "--name-only", "-z"])`. Output is split on NUL.
- The staged file set is **intersected with the resolved scan root**: only staged files whose absolute path is at or under the resolved root are scanned.
- **Submodules.** Staged-file enumeration from the outer worktree does not recurse into submodules.
- Files staged but deleted from the working tree are silently skipped.
- **When `git` was not resolvable at startup**, `--staged` exits 2 immediately with a readable error (since enumeration is impossible).

### Blame batching, ARG_MAX, and uncommitted lines
- Group findings by file; invoke `runGit(["blame", "--porcelain", "-L", "a,a", "-L", "b,b", "--", "<file>"])` (prefixed by the neutralizing `-c` flags and any `--no-pager`) in a single call per file **when the total range count is safe**.
- **ARG_MAX guard.** If a single file accumulates more than **512 `-L` ranges**, the blamer **chunks** the ranges into batches of up to 512 and issues multiple `runGit` calls per file, merging the parsed results. The chunk size is a constant in `blame.ts`.
- Parse porcelain headers once per unique commit within a file's blame output.
- **Short SHA** is the first **10 hex characters** of the 40-char commit SHA. Fixed width.
- **Uncommitted lines.** When `git blame --porcelain` reports a line as `0000000000000000000000000000000000000000` with author `Not Committed Yet`, that finding's `blame` is emitted as `null`.
- Outside a git worktree (detected via `runGit(["rev-parse", "--is-inside-work-tree"])` returning non-zero), skip blame entirely and emit per-finding `blame: null`. When git was not resolvable at startup, the detection step is skipped and the same null-blame degradation applies.

### JSON schema (stable surface)

The JSON renderer emits exactly this shape. `path` is **POSIX-style and relative to `root`**. When the scan target is a single file, `root` is set to that file's **parent directory**. `generated_at` is ISO-8601 UTC with second precision and a trailing `Z`. `blame.date` is the commit **author-date**. `findings` is sorted by `(path, line, column)` ascending (cross-file ordering is lexicographic byte-order over `path`).

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

The `blame` key is either the full object shown above or the JSON literal `null`. The schema is a stable public surface from product v0.1 onward; additive changes bump the `$schema` URL's version segment.

### Markdown output shape (authoritative)
- `# TODO report — <root> — <generated_at>` (header; root and generated_at escaped per "Output escaping").
- One `## <relative/path>` section per file, sorted lexicographically by `path`.
- Findings within a file sort by `(line, column)` ascending and render as list items. **For a single-line `text`**:
  ```
  - **TODO** L42 · Jane Doe, 2024-07-11 (a1b2c3d4e5) — rewrite with streaming parser
  ```
- **For a multi-line `text`**: the first segment follows the `— ` separator on the bullet line; each subsequent `\n`-separated segment is rendered on its own line, **indented by 2 spaces**. Blank segments render as blank 2-space-indented lines.
  ```
  - **TODO** L42 · Jane Doe, 2024-07-11 (a1b2c3d4e5) — refactor
    first point
  
    second point
  ```
- The date shown is the **author-date**, rendered as `YYYY-MM-DD` in UTC.
- The 10-char short SHA is the same value as `blame.sha` in JSON.
- Findings with `blame: null` render as `- **TODO** L42 · (unblamed) — <text>`.
- All user-controlled fields are escaped per "Output escaping" before concatenation.

### CLI surface
```
todo-stream [path]              positional root (default: cwd); symlink roots canonicalized once; non-existent root → exit 2
  --format json|markdown        default: markdown when stdout isTTY, json otherwise (TTY check: isatty(1) via Bun.stdout.isTTY; Windows not a v0.1 target)
  --markers TODO,FIXME,HACK     default: TODO,FIXME,HACK,XXX (comma list; each token must match ^[A-Za-z0-9_]+$; verbatim, case-sensitive)
  --include "**/*.ts"           repeatable; CLI value replaces config-file list wholesale
  --exclude "**/dist/**"        repeatable; exclude wins over include; CLI value replaces config-file list wholesale
  --since 2024-01-01            YYYY-MM-DD only; inclusive; UTC; incompatible with --no-blame; compared against commit author-date
  --author <substr>             case-insensitive substring matched independently against author name OR email; repeatable (OR); incompatible with --no-blame
  --fail-on TODO,FIXME          comma list; NOT repeatable (exit 2 on repeat); must be a subset of --markers; exit 1 if any matching finding remains after filters
  --no-gitignore                disable git-delegated .gitignore pruning (.git/ is still pruned unconditionally)
  --no-blame                    skip git blame enrichment; per-finding `blame: null` for every finding
  --staged                      limit to files listed by `git diff --cached --name-only -z`, intersected with the resolved root; reads working-tree bytes; requires a git worktree (exit 2 otherwise); single-file root anchored at its parent dir; submodules not recursed
  --redact-emails               replace email local-part with `***` (entire local-part) in both JSON and Markdown output
  --max-file-size BYTES         skip files larger than this (default 5242880, i.e. 5 MiB); ≤0 → exit 2
  --blame-timeout SECONDS       per-invocation git-blame timeout (default 30); ≤0 → exit 2
  --blame-concurrency N         max concurrent git-blame invocations (default 8); ≤0 → exit 2
  --max-findings N              global cap on total findings retained and emitted (default 100000); exceeding the cap → exit 2 with stderr warning; ≤0 → exit 2
  --max-output-bytes BYTES      cap on rendered stdout size (default 67108864, i.e. 64 MiB); exceeding → exit 2, stdout empty, stderr error; ≤0 → exit 2
  --config <path>               load JSON config explicitly (no auto-discovery in v0.1); see Config file schema
  --version | --help
```

Exit codes: `0` (success, no failing findings), `1` (`--fail-on` triggered), `2` (usage or internal error — includes unknown flag, conflicting flags, bad config JSON, unknown config key, invalid marker, invalid `--since` form, `--fail-on` not a subset of `--markers`, `--fail-on` repeated, `--staged` outside a worktree, `--staged` with git unavailable, non-existent positional root, resource-limit flag out of range, `--max-findings` cap reached, `--max-output-bytes` cap exceeded, non-blame spawn failure).

## Tests plan

### Determinism seams (for golden-file tests)
The JSON and Markdown renderers accept injected values for otherwise-nondeterministic fields so golden-file tests are byte-stable:
- `generated_at` can be overridden by the env var `TODO_STREAM_NOW` (ISO-8601 UTC); defaults to `new Date()` otherwise. Documented in the README as a test/reproducibility aid.
- `root` can be rendered relativized to a test fixture dir via the test harness (the renderer takes `root` as an argument; tests pass a pinned absolute path).

### Unit tests (fixture-driven, TDD — RED first)
Built **test-first** (red → green):
- **Extractor (`extract.test.ts`)** — per-language fixtures with markers in line comments, block comments, nested blocks, and code-only lines with marker-looking strings (negative case: `const s = "TODO"` in a `.ts` file → 0 findings; in a `.unknownext` file → 1 finding). Additional fixtures:
  - Block comment with `TODO:` and `FIXME:` on different lines with continuation text, including blank-line `\n\n` case.
  - Single line with two markers → two Findings, distinct `column`, ordered ascending.
  - Empty-marker-line block → `text = "body"` (no leading `\n`).
  - Non-`\w` marker config → exit 2.
  - `--markers todo` against `TODO` and `todo` → only `todo` matched.
  - Python `"""TODO"""` docstring → 0 findings; `# TODO: real` → 1 finding.
  - `.TS` (uppercase extension), `makefile` (lowercase filename) → correct dispatch.
  - `Makefile`, `Dockerfile`, `CMakeLists.txt` → `#` comment dispatch.
  - **Shebang line fixture**: `#!/usr/bin/env bash # TODO: switch to bun` on line 1 of a `.sh` file → 0 findings from line 1, but a `TODO` on line 2 is still found.
  - **CRLF fixture**: a `.c` file with CRLF line endings and a multi-line block comment → continuation algorithm joins with `\n` (no `\r`); column numbering correct.
  - **Invalid-UTF-8 fixture**: `// TODO: caf\xE9` (Latin-1 `é`) → one finding; `text` contains U+FFFD where `0xE9` was.
- **Walker (`walker.test.ts`)** — `--include`/`--exclude` overlap, exclude-wins order-independence, root-path symlink canonicalization, internal symlink skipping (cycle-safe), binary-file skipping, hostile-filenames fixture, large-file guard, `--max-file-size=0 → exit 2`, `.git/` unconditional prune (a fixture with `.git/TODO-in-config` is scanned → 0 findings inside `.git/`, even with `--no-gitignore`), non-existent positional root → exit 2, **cross-file ordering determinism** (same fixture on a case-sensitive and a case-preserving filesystem produces byte-identical JSON and Markdown output).
- **Renderers (`render.test.ts`)** — golden-file tests with `TODO_STREAM_NOW` pinned:
  - Empty findings list.
  - `blame: null` finding (`null` in JSON; `(unblamed)` in Markdown).
  - Multi-line `text` with `\n` continuations (JSON escape + Markdown 2-space-indent).
  - Blank-continuation case producing `\n\n`.
  - Non-ASCII author name and text (UTF-8 round-trip).
  - Long `text` (no truncation below output-bytes cap).
  - Two findings on same (file, line) from two markers — distinct `column`.
  - **Injection probe**: `// TODO: [link](http://evil) | injection <!-- hidden -->` → Markdown escapes brackets/pipe/HTML-comment.
  - **ANSI probe (expanded)**: source with 7-bit CSI, 7-bit OSC (BEL- and ST-terminated), 8-bit CSI (`\x9b`), DEC private-mode — **both JSON and Markdown** stripped of all sequences. A dedicated OSC-hyperlink test asserts the hyperlink bytes are absent from JSON (not just escaped).
  - `--redact-emails` on `jane@example.com` and `a@b.com` → both emit `***@domain`.
- **Blame parser (`blame.test.ts`)** — canned porcelain → parsed records, author-time used as `date`, 10-char SHA, `0000...` → `blame: null`. No real git.
- **Blame ARG_MAX chunking (`blame_chunk.test.ts`)** — swap `runGit`; >512 lines on a single file → ≥2 invocations, merge correctness, per-invocation `-L` count ≤512.
- **Blame per-file spawn failure (`blame_fail.test.ts`)** — `runGit` stub returns non-zero for one file → that file's findings null + stderr warning; exit 0 (or 1 on `--fail-on`).
- **Blame concurrency / timeout (`blame_limits.test.ts`)** — delayed stub; concurrent in-flight ≤ `--blame-concurrency`; exceeding timeout → null + warning.
- **Subprocess hardening (`git_env.test.ts`)** — swap `runGit` with an env-capturing stub; assert `GIT_OPTIONAL_LOCKS=0`, `GIT_TERMINAL_PROMPT=0`, `GIT_CONFIG_GLOBAL=/dev/null`, `GIT_CONFIG_SYSTEM=/dev/null`, `GIT_CONFIG_NOSYSTEM=1`, `LC_ALL=C.UTF-8` on **every** git subcommand the tool issues (`blame`, `check-ignore`, `rev-parse --show-toplevel`, `rev-parse --is-inside-work-tree`, `diff --cached --name-only -z`). Also assert the `-c` neutralizing flags (`core.fsmonitor=`, `core.hooksPath=/dev/null`, `core.sshCommand=`, `diff.external=`, `protocol.ext.allow=never`, etc.) appear on every invocation.
- **PATH sanitization (`git_path.test.ts`)** — construct synthetic `PATH` values with empty elements, `.`, `..`, a relative path, and a path that resolves under the scan root. Assert:
  - Empty/relative/`.`/`..` components are dropped.
  - A repo-local `bin/git` under the scan root is **not** chosen.
  - If the only resolvable git is under the scan root, startup runs fully-degraded (no blame, `--staged` exits 2).
- **Malicious-config-repo (`malicious_config.test.ts`)** — real tiny repo with `alias.*`, `core.fsmonitor`, `core.sshCommand`, `filter.lfs.process`, `diff.external`, `pager.log` in `.git/config`. Wrap `Bun.spawn` with a call-counter; run the tool; assert only the expected git invocations happen, no extra subprocesses fan out, tool output is unaffected.
- **Config file (`config.test.ts`)** — bad JSON, nonexistent path, unknown key, wrong type, wholesale-override merge, CLI-only with no config, config-only with no CLI; includes `maxFindings` and `maxOutputBytes` fields.
- **Resource caps (`caps.test.ts`)** — fixture triggering `--max-findings` → exit 2, stderr warning, no stdout write; fixture triggering `--max-output-bytes` (e.g. `--max-output-bytes 100` against any non-trivial fixture) → exit 2, stdout empty, stderr error.
- **CLI flags (`cli.test.ts`)** — black-box tests. Each v0.1 flag gets positive and negative cases:
  - `--fail-on FIXME` with no FIXMEs → 0; with one FIXME → 1.
  - **`--fail-on` × `--since` interaction**: fixture with real FIXMEs + `--since 2099-01-01` + `--fail-on FIXME` → exit 0 AND empty `findings` (pins the post-filter evaluation order).
  - `--fail-on FOO` with no FOO in `--markers` → 2.
  - `--fail-on FIXME --fail-on HACK` (repeated) → 2.
  - `--since 2099-01-01` → empty; `--since 1970-01-01` → all; `--since 7d` → 2; `--since 2024-1-1` → 2.
  - **`--since` UTC-boundary fixture**: synthetic-repo commit authored `2024-01-01T00:30:00+0900` (UTC `2023-12-31T15:30:00Z`), `--since 2024-01-01` → finding dropped.
  - `--author nobody` → empty; `--author <real-name-substring>` → non-empty; `--author <email-domain>` → non-empty (independent-field matching).
  - **`--author` boundary-straddling regression**: query `"e <j"` against `Jane <jane@example.com>` → 0 matches (independent-field, not concatenated).
  - `--include "**/*.ts"` alone; unknown glob → empty, exit 0.
  - `--exclude "**/generated/**"` alone.
  - `--include "**/*.ts" --exclude "**/generated/**"` → exclude-wins.
  - Order-independent exclude-wins.
  - `--staged --include "**/*.ts"` → intersection.
  - `--staged subdir/` with staged files in/out of subdir.
  - `--staged <single-file>` where the file is staged → works, worktree anchored at parent dir.
  - `--staged` in fresh `mktemp -d` (under `GIT_CEILING_DIRECTORIES`) → exit 2.
  - `--staged` with staged-but-deleted file → silently skipped.
  - `--staged` with hostile-name file → scanned via `-z`.
  - `--config <path>` → good/bad/nonexistent/unknown-key/type-mismatch.
  - `--no-blame --since 2024-01-01` → exit 2.
  - `--no-gitignore` → vendored `node_modules` visible; `.git/` still pruned.
  - `--markers TODO?` → exit 2.
  - `--max-file-size 0`, `--blame-timeout -1`, `--blame-concurrency 0`, `--max-findings 0`, `--max-output-bytes 0` → all exit 2.
  - `--redact-emails` against `a@b.com` → JSON `blame.email` = `***@b.com`.
  - Unknown flag → 2.
  - Non-existent positional root → 2.
  - **Output atomicity (both formats)**: inject a post-render failure; assert stdout is empty for both JSON and Markdown (buffered write never flushed) or a complete valid document; never partial.
  - **No-git-worktree degradation**: directory under `GIT_CEILING_DIRECTORIES` with one `FIXME` file; precondition check fails fast if inside ambient worktree; assert every finding has `blame: null`, JSON validates against schema.
  - **No-git-binary degradation**: spawn the tool with sanitized `PATH` that contains no git binary; assert every finding has `blame: null`, `.gitignore` pruning is skipped, `--staged` exits 2, one startup stderr warning, exit 0 otherwise.

### Integration tests (real git, CI)
Built after the unit suite is green. Both integration clones are **pinned to specific commit SHAs**:
- **`postgres/postgres`** — checkout at `PG_PIN_SHA`, shallow-bounded to `src/backend/access/`. Invariants:
  1. Exits 0.
  2. JSON validates against published `$schema`.
  3. `findings.length >= 1`.
  4. Every finding has a non-null `blame`.
  5. **Curated marker-presence allow-list** pinned at `PG_PIN_SHA`.
  6. Runtime well-behaved on the perf lane.
- **`postgres-ai/database-lab`** — checkout at `DBLAB_PIN_SHA`, full-repo, same invariants.
- Count- and text-sensitive assertions live on `tests/fixtures/synthetic-repo/` (checked-in bare-git tarball).
- Gated behind `INTEGRATION=1`.

### Performance lane (separate, not a correctness gate)
- Runs on **GitHub-hosted `ubuntu-latest` (4 vCPU, 16 GB RAM)**.
- Scans the pinned `postgres/postgres` subtree **three times** per job; records wall-clock in a CSV artifact.
- **Fails only if all three runs exceed 90 s** (best-of-3 smoothing).

### CI matrix
- `bun test` (unit + black-box CLI) on every PR — must pass.
- `INTEGRATION=1 bun test` nightly and on `main` — must pass.
- Performance lane nightly only.
- `bun build --compile` smoke.
- `bun tsc --noEmit`.
- `biome check`.

### TDD call-out (explicit)
- **Test-first (RED → GREEN):** extractor (all edge cases including shebang, CRLF, invalid-UTF-8), walker (including `.git/` unconditional prune, cross-file ordering, non-existent root), renderers (every enumerated golden case plus expanded ANSI and injection probes and redaction), blame parser, ARG_MAX chunking, per-file failure, concurrency/timeout, **hardened env on every git subcommand via the `runGit` seam**, **PATH sanitization**, malicious-config repo, config file, **resource caps (`--max-findings`, `--max-output-bytes`)**, every CLI flag including the `--fail-on × --since` interaction, `--author` independent-field and boundary-straddling regression, `--since` UTC-boundary commit, output atomicity for both formats, no-git-worktree and no-git-binary degradation.
- **Test-after:** real-repo integration tests, compile/smoke, perf lane.

## Team

Veteran experts to hire:
- **Veteran CLI systems engineer (1)** — owns walker, CLI surface, Bun packaging, exit-code semantics, symlink/binary-file policy, include/exclude composition, `--staged` root intersection, config-file schema and merge semantics, output atomicity, resource caps (`--max-findings`, `--max-output-bytes`), cross-file ordering determinism, `.git/` unconditional prune.
- **Veteran parser/regex engineer (1)** — owns the extractor, comment-syntax table (extension + filename dispatch, case-insensitive), boundary regex, byte-basis matching, CRLF normalization, shebang skip, continuation algorithm (blank-line and empty-marker-line), generic-fallback caveat, Python triple-quote suppression, invalid-UTF-8 decoding.
- **Veteran git-plumbing engineer (1)** — owns the `runGit` seam, blamer, `git check-ignore` delegation, porcelain parsing, author-time selection, ARG_MAX chunking, 10-char SHA normalization, uncommitted-line detection, graceful degradation, worktree detection, subprocess hardening (hardened env + per-invocation `-c` neutralization), per-file spawn-failure handling, concurrency/timeout enforcement, PATH sanitization and absolute-git resolution.
- **Veteran test engineer (1)** — owns fixture design (hostile-filenames, malicious-config-repo, synthetic-repo bare-git, invalid-UTF-8, shebang, CRLF), golden files, determinism seams (`TODO_STREAM_NOW`), black-box CLI suite, pinned-SHA integration harness with curated allow-lists, JSON-Schema validator harness, best-of-3 perf lane.
- **Veteran security/ops engineer (1, part-time)** — owns the Security section: path-encoding audit, ANSI grammar + output-escaping (including JSON ANSI stripping), data-sensitivity docs, subprocess hardening review (including per-invocation `-c` config neutralization + PATH sanitization), resource-cap review, output atomicity review. Pairs with the test engineer on injection/ANSI/malicious-config/PATH-sanitization fixtures.
- **Veteran TypeScript/Bun release engineer (1, part-time)** — owns `package.json`, `bin` entry, `bun build --compile`, npm publish workflow, semver, hosted JSON-Schema document.

Total: 4 full-time + 2 part-time.

## Implementation plan

Sprints are one week each. `⇄` = parallel; `→` = ordering dependency.

### Sprint 1 — Skeleton & red tests
- CLI engineer: scaffold repo, `bin/todo-stream`, `--help`, `--version`, argv parser; red tests for flag-grammar error paths (invalid `--since`, marker, unknown flag, `--fail-on` subset/repeat, `--no-blame`+`--since`, `--staged` outside worktree / with git unavailable / with single-file root, range validation for all resource flags including `--max-findings`/`--max-output-bytes`, `--config` errors, non-existent positional root). ⇄
- Parser engineer: fixture tree, red extractor tests for every language (extension + filename, case-insensitive), multi-marker-per-line, multi-marker-block, continuation (blank-line and empty-marker-line), marker-in-string asymmetry, Python triple-quote, non-`\w` marker rejection, verbatim case, **shebang skip**, **CRLF normalization**, **invalid-UTF-8 decoding**. ⇄
- Git engineer: capture canned porcelain → red parser tests for 10-char SHA, author-time, uncommitted→null. Red tests using the `runGit` seam for ARG_MAX chunking, per-file spawn-failure, timeout, concurrency-cap, **hardened env assertions on every git subcommand** (`blame`, `check-ignore`, `rev-parse` ×2, `diff --cached`), **per-invocation `-c` neutralization**, **PATH sanitization**. ⇄
- Test engineer: `bun test` in CI, `INTEGRATION=1` gate, black-box harness, red tests for every v0.1 flag including `--fail-on × --since` interaction, `--author` boundary-straddling regression, `--since` UTC-tz-boundary fixture, `--staged` × root intersection, hostile-filename `-z`, `--redact-emails` (entire-local-part formula), output atomicity probes for **both** formats, no-git-worktree degradation under `GIT_CEILING_DIRECTORIES`, **no-git-binary degradation**, cross-file ordering determinism, `.git/` unconditional prune. Stand up synthetic-repo fixture, hostile-names fixture, malicious-config-repo fixture (with call-counter wrapper), JSON-Schema validator. Stand up `TODO_STREAM_NOW` determinism seam. ⇄
- Security/ops engineer: red tests for Markdown + JSON ANSI stripping (CSI, OSC BEL/ST, 8-bit CSI, DEC private), injection probes, subprocess-env + `-c` assertions, PATH-sanitization, large-file guard, `--max-findings` and `--max-output-bytes` cap violations. ⇄
- Release engineer: lock Bun version, add `bun tsc --noEmit` + `biome check` to CI, declare `@biomejs/biome` as dev dep.
- **Gate:** all tests RED, CI green on lint/type.

### Sprint 2 — Core green
- Parser engineer → implement `extract.ts` (dynamic marker regex, symmetric non-identifier boundaries, marker validator, per-occurrence Finding emission, continuation algorithm with empty-marker-line and CRLF, case-insensitive dispatch, shebang skip, byte-basis matching with UTF-8 decode) until fixtures pass.
- CLI engineer → implement `walker.ts` with `Bun.Glob` + `runGit` `check-ignore` delegation + root canonicalization + internal symlink-skip + binary-skip + dedup + **lexicographic ordering** + exclude-wins + `.git/` unconditional prune + `--max-file-size`. Non-existent root exit 2. ⇄
- Git engineer → implement blame porcelain parser + ARG_MAX chunking + per-file spawn-failure degradation + timeout + concurrency cap + hardened env + per-invocation `-c` neutralization + `runGit` seam + PATH sanitization and absolute-git resolution. ⇄
- Test engineer → author renderer golden files across every enumerated case (still red); finalize synthetic fixture repo and curated allow-lists for integration pins.
- Security/ops engineer → pair on injection/ANSI implementations + malicious-config call-counter harness. ⇄
- **Gate:** extractor, walker, blame parser, ARG_MAX, per-file failure, concurrency/timeout, PATH sanitization all GREEN; renderers and CLI tests still RED.

### Sprint 3 — Render & wire
- CLI engineer → implement JSON + Markdown renderers against golden tests (POSIX-relative `path`, pinned `generated_at`, 10-char SHA, author-date rendering, multi-line 2-space-indent, output escaping **including ANSI stripping in JSON**, `(unblamed)` rendering, `--redact-emails` entire-local-part formula); wire end-to-end; implement exit-code semantics and every usage-error path; implement **output atomicity for both formats** (buffered writes, stderr-only diagnostics, `--max-output-bytes` check); implement config-file loader; implement `--max-findings` accumulation cap.
- Git engineer → implement `blame.ts` runtime wiring: production `runGit` backed by `Bun.spawn`, hardened env, timeout via `AbortController`, concurrency gate. ⇄
- Parser engineer → extend markers config + `--markers` flag; finalize generic fallback. ⇄
- Test engineer → start integration harness: pinned-SHA clone helper, postgres subtree scan with schema validation + allow-list.
- Security/ops engineer → land malicious-config end-to-end against real blame runtime; ratify JSON ANSI stripping. ⇄
- **Gate:** renderer tests and a majority of CLI tests GREEN; `todo-stream` runs end-to-end against the repo itself; output-atomicity tests GREEN for both formats.

### Sprint 4 — Integration & release
- Test engineer → finalize integration tests (pinned-SHA clones + schema validation + allow-list); stand up perf lane on `ubuntu-latest` 4 vCPU with best-of-3 90 s ceiling. ⇄
- Release engineer → `bun build --compile` pipeline, npm publish dry-run, `npx todo-stream` smoke, README usage + JSON schema docs + **data-sensitivity guidance** + **`TODO_STREAM_NOW` documentation**, publish hosted `$schema`, declare macOS + Linux support matrix. ⇄
- CLI engineer → finalize `--since`, `--author` (independent-field matching), `--fail-on`, `--staged` (single-file parent anchoring, deleted-file skip, hostile-name `-z`), `--config`, plus `--no-blame` mutual-exclusion guard. ⇄
- Git engineer → hardening: large-repo shakeout, uncommitted-line regression, malicious-config repo end-to-end, PATH-sanitization regression on multi-PATH runners. ⇄
- Parser engineer → finalize Rust, Ruby, Lua, filename-dispatch, shebang, CRLF coverage. ⇄
- Security/ops engineer → finalize data-sensitivity README, sign off on injection/ANSI/hostile-name/malicious-config/PATH tests. ⇄
- **Gate:** all unit + black-box CLI tests GREEN; integration GREEN against pinned SHAs; perf lane reporting best-of-3; `bun run build` emits a working static binary; product v0.1.0 tagged.

### Parallelization summary
- Sprint 1 is almost fully parallel (six independent red-test streams).
- Sprints 2 and 3 pair parser+git in parallel while CLI advances; test engineer stays ahead with next red tests; security/ops pairs on injection/ANSI/malicious-config/PATH.
- Sprint 4 fans out: every engineer has an independent lane.

## Embedded Changelog

- **Spec v0.1 (2026-04-21)** — Initial draft. Reframed away from the "todo-list app" interview answers per the authoritative idea. Locked scope to directory walk + regex extract + git blame + JSON/Markdown report.
- **Spec v0.2 (2026-04-21)** — Post-review r1 refinement. Strengthened user stories. Tightened Architecture. Added JSON schema example and Markdown shape. Expanded Tests plan with TDD red-first call-outs and integration harness. Specified team and four-sprint plan.
- **Spec v0.3 (2026-04-21)** — Post-review r2. Introduced spec-version vs product-version distinction; reconciled JSON schema; stated dynamic marker regex; pinned `--staged` file-level; made `--since`/`--author` incompatible with `--no-blame`; replaced pinned-upstream-TODO assertion with structural invariants; moved perf number to separate lane; declared string-literal tokenization a non-goal; specified multi-marker block; delegated `.gitignore`.
- **Spec v0.4 (2026-04-21)** — Post-review r3. Pinned regex boundary, marker shape, `--since`/`--author`/`--fail-on` grammars, POSIX-relative `path`, 10-char SHA, `generated_at` format, language table, uncommitted-line handling, symlink/binary policy, continuation joining, 1-based indexing, pinned-SHA integration clones, no-git-worktree test, renderer golden enumeration, macOS+Linux platform.
- **Spec v0.5 (2026-04-21)** — Post-review r4. Resolved per-line vs per-occurrence contradiction. Pinned `blame.date` to author-date. Made continuation algorithm authoritative with blank-line handling. Removed config auto-discovery. Added `--include`/`--exclude` interaction tests. Strengthened integration invariants. Walker dedup by resolved absolute path. `--staged` outside worktree → exit 2. `--fail-on` non-repeatable. ARG_MAX chunk = 512. `GIT_CEILING_DIRECTORIES` guard. `@biomejs/biome` dev dep.
- **Spec v0.6 (2026-04-21)** — Post-review r5. Added Security, privacy, and robustness section: hostile filenames, Markdown escaping, data sensitivity with `--redact-emails`, resource limits, subprocess hardening, output atomicity. Rewrote user story 1 for whole-repo `--fail-on`. Hardened `--staged`. Pinned config file schema. Pinned Markdown multi-line rendering. Extended continuation algorithm with empty-marker-line case. Per-file blame failure handling. Removed Python triple-quote. Case-insensitive extension + exact-filename dispatch. Curated marker-presence allow-list. Best-of-3 perf lane. `runBlame` spawn seam pinned.
- **Spec v0.7 (2026-04-21)** — Post-review r6. **Subprocess hardening rewritten**: per-invocation `-c` flags now neutralize repo-local `.git/config` (`core.fsmonitor=`, `core.hooksPath=/dev/null`, `core.sshCommand=`, `core.pager=cat`, `diff.external=`, `diff.textconv=`, `filter.*`, `url.*.insteadOf=`, `alias.*=`, `protocol.ext.allow=never`, plus `GIT_CONFIG_NOSYSTEM=1`), addressing the r6A-1 finding that `GIT_CONFIG_GLOBAL`/`SYSTEM` do not cover `.git/config`. **PATH sanitization added** (r6A-2): empty/`.`/`..`/relative components dropped; components under the scan root rejected; git resolved once to an absolute path at startup; fully-degraded mode when resolution fails. **Output atomicity unified** (r6A-3, r6B-14): both JSON and Markdown buffer fully before a single stdout write; the v0.6 streaming-Markdown allowance is removed. **ANSI stripped from JSON too** (r6A-4): terminal control sequences are stripped from every string field in both renderers, with an authoritative grammar covering 7-bit CSI, 7-bit OSC (BEL and ST terminators), 8-bit CSI, and DEC private-mode. **Byte decoding pinned** (r6A-5): source read via `Bun.file().bytes()` (not `.text()`); CRLF normalized before continuation; UTF-8 decode with U+FFFD replacement; invalid-UTF-8 fixture added. **`.git/` unconditionally pruned** at every depth regardless of `.gitignore` / `--no-gitignore` (r6A-6); `.hg/` and `.svn/` too. **Global resource caps added** (r6A-8): `--max-findings` (100 000) and `--max-output-bytes` (64 MiB); both exit 2 on violation. **Scope-reduction rejected** (r6A-7): release obligations are load-bearing. **`--author` matching** changed to independent-field OR (r6B-1); v0.6 concatenation boundary-straddling regression pinned with a fixture. **ANSI grammar pinned** (r6B-2): exact byte classes for CSI/OSC/8-bit/DEC-private. **`runGit` seam generalized** (r6B-3, r6B-15): single injectable boundary for every git subcommand; hardened-env assertions test `blame`, `check-ignore`, both `rev-parse` forms, and `diff --cached`. **CRLF handling pinned** (r6B-4) and fixture added. **Cross-file ordering pinned** (r6B-5) to lexicographic byte-order; determinism test across case-sensitive and case-preserving filesystems. **git-missing-inside-worktree** (r6B-6): when git not resolvable, no `.gitignore` pruning, all `blame: null`, `--staged` exit 2. **`--fail-on × --since` interaction test added** (r6B-7). **`--redact-emails` formula changed** (r6B-8) to replace the entire local-part with `***`, closing the single-char-username leak. **Non-existent positional root → exit 2** (r6B-9). **`--staged` single-file root** anchored at parent directory (r6B-10). **Shebang lines suppressed** for `shell` and `python` dispatch (r6B-11) with fixture. **`--since` UTC-tz-boundary test added** (r6B-12). **UTF-16/32 binary-skip acknowledged** (r6B-13) as accepted consequence. Determinism seam `TODO_STREAM_NOW` added for golden-file tests. Team and sprint plan updated to reflect the new work.
