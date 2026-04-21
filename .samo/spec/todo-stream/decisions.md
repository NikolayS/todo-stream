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
