# decisions

- No review-loop decisions yet.

## Round 1 — 2026-04-21T13:22:07.607Z

- deferred ambiguity#?: Determinism seam for `generated_at`/`root` in golden JSON tests lives in the locked Tests plan / Implementation details sections — will specify an injection point (e.g. `--now`/`NOW=` env and root-relativization) next round.
- deferred contradiction#?: Short-SHA width mismatch (10 vs 7) sits across the locked Implementation details and Markdown output sections; will pin a single canonical width next round.
- deferred ambiguity#?: How `--markers` composes into the extractor regex (escaping, case-sensitivity) lives in the locked Implementation details / CLI surface; will specify composition and `\b` boundary rules next round.
- deferred ambiguity#?: `--fail-on` grammar (single vs comma-list vs repeatable) belongs to the locked CLI surface; will pin exact grammar next round.
- deferred ambiguity#?: `--format` TTY predicate (which stream, Windows consoles) belongs to the locked CLI surface; will state predicate explicitly next round.
- deferred ambiguity#?: Interaction of `--since`/`--author` with `--no-blame` requires edits to the locked CLI surface and Implementation details; will pin validation precedence next round.
- deferred ambiguity#?: Line/column indexing convention (1-based vs 0-based) belongs to the locked JSON schema section; will pin in next round.
- deferred ambiguity#?: Block-comment multi-marker handling and trimming rules belong to the locked Finding extraction algorithm; will specify one-marker-per-line and star-strip rules next round.
- deferred ambiguity#?: Blame cache key description belongs to the locked Blame batching section; will reconcile key granularity next round.
- deferred ambiguity#?: Supported `.gitignore` subset (or delegation to `git check-ignore`) belongs to the locked Walker/Components section; will pin the subset next round.
- deferred ambiguity#?: Symlink/binary-file policy belongs to the locked Walker/Implementation details; will specify cycle-safe symlink handling and NUL-byte binary detection next round.
- deferred weak-testing#?: Tightening integration invariants with pinned (path,line,marker) triples and lower-bound counts requires edits to the locked Tests plan; will add next round.
- deferred weak-testing#?: Pinning integration scans to specific upstream SHAs (instead of floating `--depth=1` HEAD) requires edits to the locked Tests plan; will address next round.
- deferred weak-testing#?: Moving the `<60 s` perf gate to a separate perf lane with a pinned runner requires edits to the locked Tests plan; will restructure next round.
- deferred weak-testing#?: Adding red-first unit tests for `--since`/`--author`/`--fail-on`/`--staged`/`--config` requires edits to the locked Tests plan and Implementation plan; will add next round.
- deferred weak-testing#?: CLI error-path test coverage (unknown flag, conflicting flags, bad config, non-git root) belongs to the locked Tests plan; will add explicit cases next round.
- deferred ambiguity#?: `$schema` field presence/URL belongs to the locked JSON schema block; will reconcile prose and example next round.
- deferred ambiguity#?: `--author` match target (name vs email vs both, case-sensitivity) belongs to the locked CLI surface / Implementation details; will specify next round.
- deferred ambiguity#?: Windows support posture for v0.1 belongs to the locked Scope and Implementation details; will declare supported platforms explicitly next round.
- deferred ambiguity#?: Nullable shape of `blame` (object-null vs per-field-null) belongs to the locked JSON schema; will pin to `blame: null` wholesale next round.
- deferred weak-testing#?: Enumerating renderer golden-fixture edge cases (empty, null-blame, multi-line, non-ASCII, long text, duplicates) belongs to the locked Tests plan; will enumerate next round.

## Round 2 — 2026-04-21T13:22:07.607Z

- accepted contradiction#version: Added an explicit 'Versioning convention' section separating spec version (v0.3) from product version (0.1.0), bumped the header, and filled in v0.2 and v0.3 changelog entries.
- accepted contradiction#json-schema: Reconciled the JSON renderer prose and example: both now include `$schema` and `root`, and the example is declared the canonical stable surface.
- accepted contradiction#marker-regex: Finding extraction algorithm now states the regex is built dynamically from `config.markers` with per-marker escaping, making `--markers` functional by construction.
- accepted ambiguity#staged: Pinned `--staged` to file-level semantics (files listed by `git diff --cached --name-only`) and rewrote user story 4 to match the CLI surface.
- accepted ambiguity#since-author-blame: Added an explicit precedence block: `--no-blame` combined with `--since`/`--author` is a usage error (exit 2); when blame is enabled but null, those findings are dropped by blame-dependent filters.
- accepted weak-testing#pinned-todo: Replaced the brittle 'pinned long-standing TODO' assertion with structural invariants against the real repo plus a dedicated synthetic bare-git fixture that pins (path, line, marker, text, sha) deterministically.
- accepted weak-testing#cli-flags: Added a black-box CLI test section covering every v0.1 flag with positive and negative cases and all three exit codes (0/1/2), and made those tests red-first in Sprint 1.
- accepted weak-testing#strings: Declared string-literal tokenization an explicit non-goal and added a fixture assertion that `const s = "TODO"` on a code-only line yields zero findings.
- accepted ambiguity#block-multi-marker: Extraction algorithm now specifies that each marker line within a block comment yields its own Finding, with non-marker continuation lines attached to the preceding marker.
- accepted ambiguity#gitignore: Delegated `.gitignore` pruning to `git check-ignore --stdin` inside a worktree and declared it a no-op outside, eliminating the need to specify a custom parser subset.
- accepted ambiguity#perf-hardware: Moved the runtime number out of correctness tests into a dedicated perf lane pinned to GitHub-hosted `ubuntu-latest` (4 vCPU, 16 GB RAM) with a loose 90 s ceiling for trend tracking.
- accepted missing-requirement#changelog: Added v0.2 and v0.3 changelog entries summarizing the refinements made in each round.

## Round 3 — 2026-04-21T13:22:07.607Z

- accepted ambiguity#?: Replaced the asymmetric `\b` with a symmetric non-identifier class on both sides and restricted user markers to `^[A-Za-z0-9_]+$` (rejected at config-load with exit 2), eliminating the boundary ambiguity.
- accepted ambiguity#?: Pinned `--since` to strict `YYYY-MM-DD`, interpreted as UTC midnight, with inclusive boundary; other forms (including `7d`) exit 2.
- accepted ambiguity#?: Pinned `--staged` to file-set filter + working-tree read semantics and documented the limitation in the CLI surface and user story 4.
- accepted weak-testing#?: Integration clones are now pinned to specific commit SHAs and count-sensitive assertions moved to the synthetic fixture repo; the real-repo invariants are now schema-validates + non-empty-sanity-floor only.
- accepted ambiguity#?: Pinned `path` to POSIX-style and relative to `root`, with the single-file scan case specified (root becomes the parent dir).
- accepted ambiguity#?: Fixed short SHA at 10 hex characters, independent of `core.abbrev`, and made the Markdown renderer use the same width for consistency.
- accepted ambiguity#?: Pinned `--author` to case-insensitive substring matching against the `author <email>` concatenation, repeatable and OR'd.
- accepted ambiguity#?: Pinned `--fail-on` to a comma list (non-repeatable) that must be a subset of `--markers`; a disjoint value is exit 2, not a silent no-op.
- accepted ambiguity#?: Uncommitted `0000...` lines emit `blame: null` (not a synthesized object), keeping the blame shape uniform (full object or null).
- accepted ambiguity#?: Declared symlinks skipped (not followed) in the walker, avoiding cycles and surprise out-of-root traversal.
- accepted ambiguity#?: Pinned continuation-line joining: strip leading block-comment punctuation, concatenate with `\n`, preserve blank lines as a single `\n`.
- accepted ambiguity#?: Stated markers are used verbatim (no upcasing) and match is case-sensitive; documented in the CLI surface so `--markers todo` matching only lowercase is explicit.
- accepted ambiguity#?: Added an authoritative language table mapping extensions to comment syntax and exact `language` field values.
- accepted ambiguity#?: Dropped env-var configuration from v0.1 scope and from the precedence list; precedence is now `CLI flags > todo-stream.config.json > defaults` only.
- accepted ambiguity#?: Pinned `generated_at` format to ISO-8601 UTC with second precision and a trailing `Z` (`YYYY-MM-DDTHH:MM:SSZ`).
- accepted weak-testing#?: Added an explicit black-box CLI test for the no-git-worktree degradation path (scan a fresh tmpdir with a FIXME file; assert top-level `blame: null` for every finding).
- rejected weak-testing#?: Did not add automated baseline-regression gating; reframed the perf lane as a coarse 90 s ceiling / liveness gate with a human-inspected CSV artifact, and explicitly deferred trend comparison to a later version — the 'trend' framing was removed.

## Round 4 — 2026-04-21T13:22:07.607Z

- accepted r4#1: Resolved the per-line vs per-occurrence contradiction by pinning the rule to one Finding per marker occurrence, with a same-line two-marker case retained as a golden test alongside the block-comment-across-lines case.
- accepted r4#2: Pinned blame.date to commit author-date (author-time+author-tz) consistently across the Scope, Components, JSON schema, Markdown output, --since comparison, and a new blame-parser test that distinguishes it from committer-time.
- accepted r4#3: Rewrote the continuation-joining rule as an explicit three-step algorithm with a worked C-family example; removed `//` from the stripped-punctuation list and specified blank lines as empty segments so consecutive blanks produce consecutive `\n`s.
- accepted r4#4: Removed auto-discovery from v0.1 scope: `--config <path>` is the only config-file entry point, and precedence now reads `CLI flags > --config file > defaults`.
- accepted r4#5: Added positive and negative tests for `--include` and `--exclude`, their interaction with `--staged`, and an order-independence test proving exclude-wins regardless of argv order; wired into Sprint 1 red tests.
- accepted r4#6: Strengthened real-repo integration invariants with full JSON-Schema validation and a per-default-marker presence probe; explicitly rejected count-tolerance gating and recorded the justification inline so future reviewers don't re-raise it.
- accepted r4#7: Walker now explicitly deduplicates by resolved absolute path before extraction so overlapping --include globs cannot produce duplicate Findings.
- accepted r4#8: Pinned --staged outside a git worktree to exit 2 with a usage error; staged-but-deleted-from-worktree files are silently skipped; both behaviors are tested.
- accepted r4#9: Made --fail-on non-repeatable with exit 2 on repeat (rather than last-wins), and added a black-box test for the repeat case.
- accepted r4#10: Documented the known-vs-unknown-extension asymmetry (string literals suppressed only for languages in the language table) as intentional in Scope and the Implementation details, with a paired fixture test in the extractor suite.
- accepted r4#11: Replaced the misleading "top-level blame: null" wording with "per-finding blame: null" throughout the Architecture, Implementation details, and CLI surface.
- accepted r4#12: Hardened the no-git-worktree test with a `GIT_CEILING_DIRECTORIES` guard and an explicit precondition check that fails fast if the test directory is inside an ambient worktree, preventing silent flakes on Nix/sandboxed CI runners.
- accepted r4#13: Dependency policy updated to declare `@biomejs/biome` as a dev dep alongside `bun test` and type definitions, reconciling it with the CI lint step.
- accepted r4#14: Root-path symlinks are now canonicalized exactly once at startup; the canonical path is used as `root` in the JSON, and a walker test pins this behavior.
- accepted r4#15: Added a fixed 512-range ARG_MAX chunk size in the blamer with merge logic, plus a dedicated `blame_chunk.test.ts` that injects spawn and asserts multi-invocation behavior on >512-range inputs.

## Round 5 — 2026-04-21T13:22:07.607Z

- accepted A1: Added a Security section mandating NUL-delimited git modes (-z), argv-only path passing after --, UTF-8 handling rules, and a hostile-filenames fixture with \n/\t/leading-dash/quote/4-byte UTF-8 cases.
- accepted A2: Added an Output-escaping subsection with explicit Markdown escape rules (backticks, pipes, brackets, HTML-comment openers), ANSI-escape stripping, and injection + ANSI golden probes in the renderer tests.
- accepted A3: Added a Data-sensitivity subsection plus a --redact-emails flag (local-part → first char + ***) and README guidance; full-text/name redaction deferred.
- accepted A4: Added --max-file-size (5 MiB default), --blame-timeout (30 s default), --blame-concurrency (8 default), explicit no-truncation-of-text trade-off, range validation, and tests for each.
- accepted A5: Rewrote user story 1 to make the whole-repo-gating behavior explicit, documented the v0.1 workarounds (marker set reduction, --since baseline), and explicitly deferred diff/baseline mode.
- accepted A6: Added a dedicated --staged section: git -C anchoring, worktree discovery via `git rev-parse --show-toplevel`, NUL-delimited enumeration, intersection with the resolved scan root, explicit submodule non-recursion, and new tests covering subdir + hostile-name staged files.
- accepted A7: Added Output-atomicity rules: JSON renderer buffers full output before a single stdout.write; diagnostics go exclusively to stderr; partial-output behavior documented; test probe verifies stdout never contains partial JSON on failure.
- accepted A8: Added Subprocess hardening: no shell, argv-only, GIT_OPTIONAL_LOCKS=0, GIT_TERMINAL_PROMPT=0, GIT_CONFIG_GLOBAL/SYSTEM=/dev/null, LC_ALL=C.UTF-8, hooksPath=/dev/null, PATH-resolved git, plus a malicious-config-repo fixture.
- accepted A9: Removed Python triple-quoted-string handling from the language table; Python comments are # only in v0.1; added a negative fixture asserting markers in docstrings yield zero findings.
- rejected A10: The postgres/postgres + database-lab clones, hosted schema, npm packaging, and macOS+Linux binaries are all load-bearing for the stated user stories (release engineer, staff engineer, OSS maintainer, pipeline consumer) and non-negotiable release obligations; shrinking them would undermine the spec's purpose.
- accepted B1: Added an authoritative Config-file-schema subsection: enumerated keys, per-field types, validation mirror of CLI validation, wholesale-override merge semantics for list fields, unknown-key rejection (exit 2), and dedicated config.test.ts coverage.
- accepted B2: Pinned the Markdown multi-line rendering: first segment on the bullet line, subsequent \n-separated segments as 2-space-indented continuation lines per CommonMark loose-list rules; blank segments render as blank indented lines; golden test enumerated.
- accepted B3: Replaced the `grep -r` probe with a curated (marker, path-glob) allow-list pinned at each integration SHA, avoiding false positives from markers inside string literals on code-only lines.
- accepted B4: Added step 4 to the continuation algorithm: when the marker-line text is empty after stripping, the leading \n is omitted; added a worked example and a dedicated extractor fixture.
- accepted B5: Added Per-file-blame-spawn-failures subsection: non-zero exit or timeout for one file → all that file's findings get blame:null + a single stderr warning; run continues with exit 0/1 (not 2); blame_fail.test.ts covers this.
- accepted B6: Added --max-file-size (default 5 MiB) to the CLI surface with range validation and a walker fixture that exceeds the cap; stderr warning + zero findings emitted.
- accepted B7: Extension dispatch is now explicitly case-insensitive (lowercased before lookup); filename dispatch is also case-insensitive; extractor fixtures pin .TS and lowercase makefile behavior.
- accepted B8: --staged now uses `git diff --cached --name-only -z` and the hostile-names fixture is routed through the staged path to prove correct NUL-delimited parsing for whitespace/non-ASCII filenames.
- accepted B9: Added an exact-filename dispatch table (Makefile, Dockerfile, Jenkinsfile, CMakeLists.txt, .gitignore, .dockerignore) applied when extension lookup misses; extractor fixtures cover each.
- accepted B10: Perf lane now runs three scans per job and fails only if all three exceed 90 s (best-of-3 smoothing), acknowledging hosted-runner variance; trend-tracking remains deferred.
- accepted B11: Pinned the spawn seam as `runBlame(args): Promise<{stdout, exitCode}>` — the sole spawn caller in blame.ts — and made it the architectural boundary tests swap for ARG_MAX chunking, concurrency, timeout, env, and per-file-failure tests.
