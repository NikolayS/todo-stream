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
