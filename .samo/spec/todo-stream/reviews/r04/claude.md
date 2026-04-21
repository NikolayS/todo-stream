# Reviewer B — Claude

## summary

Spec v0.4 is structurally complete — all nine mandatory sections are present, the Reviewer B r3 feedback appears addressed (regex boundaries, marker validation, CLI grammar, JSON surface, platform support), and the non-goals strongly honor the "NOT a todo-list/CRUD" disclaimers. The remaining issues cluster in two areas: (1) a direct contradiction between the "each line → one Finding" rule in multi-marker block comments and an enumerated golden test asserting two findings on the same line, which must be reconciled; and (2) several ambiguities around semantics that the schema/exit-code discipline depends on — `blame.date` source (author-time vs committer-time), continuation joining when blank lines are present, config auto-discovery, walker dedup, `--staged` in non-git trees, and `--fail-on` repeatability. Testing coverage is strong in the unit/black-box tier but has a real gap: `--include`/`--exclude` aren't enumerated in the CLI test list despite Sprint 1 claiming every flag is covered, and the real-repo integration invariants are so thin they may not catch a silent regression between pinned SHAs. None of these require a scope change; one more refinement round should land v0.5.

## contradiction

- (major) Multi-marker block-comment rule contradicts an enumerated golden test. Implementation details state: "Within a single block comment, each *line* that contains a marker yields its own `Finding` at that line's line/column" — implying one Finding per line. But the renderer test enumeration lists a golden case: "Two findings on the same line (same file+line, different markers from a block comment)." These cannot both be true as written. Pick one: either the rule is "each *marker occurrence* yields a Finding" (in which case clarify column handling and ordering for two markers on one line), or the golden test must be removed/reworded.

## ambiguity

- (major) The `blame.date` field is ambiguously defined. The Scope section says "commit date (ISO-8601 UTC)", the `--since` spec says "commit author date as emitted by `git blame --porcelain`", and the JSON example shows a single `date` field. `git blame --porcelain` emits both `author-time` and `committer-time`; pin exactly one (recommend `author-time`) and use the same value consistently for emission, `--since` comparison, and Markdown rendering.
- (major) Continuation-line joining is under-specified. The rule strips leading punctuation including `//`, but `//` is line-comment syntax and cannot appear as continuation punctuation inside a C-family *block* comment; listing it is confusing. Additionally, "Empty continuation lines are preserved as `\n`" collides with "concatenated… with a single `\n` separator" — a blank line between two non-empty continuation lines would produce either `\n\n` or `\n` depending on interpretation. Specify the exact algorithm with an example, and drop non-applicable punctuation tokens per language.
- (major) Config file auto-discovery is unspecified. Architecture states precedence "CLI flags > `todo-stream.config.json` > defaults", but the CLI surface only documents `--config <path>` and never says whether `todo-stream.config.json` is auto-discovered from cwd, walked up from the scan root, or requires `--config`. Pin the discovery rule (or remove the middle tier from the precedence statement).
- (minor) The walker's dedup behavior across overlapping `--include` globs is unspecified, yet `blame.ts` "caches by absolute file path (one blame invocation per file)". If the walker yields the same path twice (e.g. `--include "**/*.ts" --include "src/foo.ts"`), does the extractor run twice, producing duplicate Findings? State explicitly that paths are deduplicated (by resolved absolute path) before extraction.
- (minor) `--staged` behavior in a non-git worktree is not specified. It relies on `git diff --cached` which will fail outside a repo. Pin: does it exit 2, or silently produce an empty finding set? Also specify behavior when a staged file has been deleted from the working tree (cannot read working-tree bytes).
- (minor) `--fail-on` repeatability is ambiguous. The spec states "the flag is not repeatable" but does not say what happens on repeat: is `--fail-on FIXME --fail-on HACK` a usage error (exit 2), or does the last occurrence win? Pin one.
- (minor) The generic (unknown-extension) fallback extractor applies only the boundary regex without language-aware comment-region reduction, meaning a `TODO` inside a string literal in an unknown file *will* be reported, while the same construct in a `.ts` file will not. This asymmetry between known/unknown languages is not called out, and the test plan does not cover it. Either document the asymmetry explicitly in the Scope non-goals, or align by applying a conservative heuristic (e.g. only match when a known line-comment start precedes the marker on the same line).
- (minor) Architecture prose says blame degrades to "top-level `blame: null`" for non-git trees, but `blame` is a per-finding field (there is no top-level `blame` key in the JSON schema). This wording is misleading; say "per-finding `blame: null`" consistently.
- (minor) "Dependency policy" states "Dev deps limited to `bun test` and type definitions", but the CI matrix lists `biome check` as a lint step, implying `biome` is a dev dependency. Update the dep policy to include biome (or drop biome from CI).
- (minor) Root-path symlink handling is unspecified. "Symlinks are not followed (neither files nor directories)" is stated for traversal, but if the user passes a symlink *as the scan root* (positional `path`), is it resolved before walking, or skipped-and-exit-0? Pin the rule for the root argument explicitly.

## weak-testing

- (major) The enumerated CLI black-box test list omits `--include` and `--exclude` entirely, despite them being core v0.1 flags and Sprint 1 claiming "every v0.1 CLI flag's black-box behavior and exit code" is test-first. Also missing: interaction between `--include`/`--exclude` and `--staged`, and the ordering semantics when both are passed (exclude-wins is conventional but not stated). Add positive/negative tests for each, plus an ordering test.
- (minor) Integration-test invariants for the real clones are extremely thin: `findings.length >= 1` plus "every finding has a non-null blame object". This leaves the end-to-end pipeline under-covered against upstream drift risk that the pinned SHAs are meant to mitigate. Add at least: (a) JSON-Schema validation of the full output, (b) an assertion that at least one finding has each of the default markers present in the subtree, (c) a count stability check within a tolerance window (e.g. `±10%` of a recorded baseline) — or explicitly justify why the synthetic fixture's coverage makes these redundant.
- (minor) The no-git-worktree degradation test uses `mktemp -d`, but TMPDIR on some systems (notably certain CI runners or developer machines using Nix/sandboxing) can live inside a git worktree, causing the test to unexpectedly find git metadata. Either explicitly set up a directory guaranteed to be outside a worktree (e.g. `git init --bare` nowhere, walk up from TMPDIR and verify, or use a `GIT_CEILING_DIRECTORIES` guard), or document the assumption so flakiness can be diagnosed.
- (minor) Blame batching scalability is untested. `git blame --porcelain -L a,a -L b,b -- <file>` with thousands of `-L` ranges (plausible in a large vendored file with many TODOs) may exceed OS `ARG_MAX`. Either add a guard + test (chunk the `-L` ranges), or document an upper bound and assert the behavior when exceeded.

## suggested-next-version

v0.5

<!-- samospec:critique v1 -->
{
  "findings": [
    {
      "category": "contradiction",
      "text": "Multi-marker block-comment rule contradicts an enumerated golden test. Implementation details state: \"Within a single block comment, each *line* that contains a marker yields its own `Finding` at that line's line/column\" — implying one Finding per line. But the renderer test enumeration lists a golden case: \"Two findings on the same line (same file+line, different markers from a block comment).\" These cannot both be true as written. Pick one: either the rule is \"each *marker occurrence* yields a Finding\" (in which case clarify column handling and ordering for two markers on one line), or the golden test must be removed/reworded.",
      "severity": "major"
    },
    {
      "category": "ambiguity",
      "text": "The `blame.date` field is ambiguously defined. The Scope section says \"commit date (ISO-8601 UTC)\", the `--since` spec says \"commit author date as emitted by `git blame --porcelain`\", and the JSON example shows a single `date` field. `git blame --porcelain` emits both `author-time` and `committer-time`; pin exactly one (recommend `author-time`) and use the same value consistently for emission, `--since` comparison, and Markdown rendering.",
      "severity": "major"
    },
    {
      "category": "ambiguity",
      "text": "Continuation-line joining is under-specified. The rule strips leading punctuation including `//`, but `//` is line-comment syntax and cannot appear as continuation punctuation inside a C-family *block* comment; listing it is confusing. Additionally, \"Empty continuation lines are preserved as `\\n`\" collides with \"concatenated… with a single `\\n` separator\" — a blank line between two non-empty continuation lines would produce either `\\n\\n` or `\\n` depending on interpretation. Specify the exact algorithm with an example, and drop non-applicable punctuation tokens per language.",
      "severity": "major"
    },
    {
      "category": "ambiguity",
      "text": "Config file auto-discovery is unspecified. Architecture states precedence \"CLI flags > `todo-stream.config.json` > defaults\", but the CLI surface only documents `--config <path>` and never says whether `todo-stream.config.json` is auto-discovered from cwd, walked up from the scan root, or requires `--config`. Pin the discovery rule (or remove the middle tier from the precedence statement).",
      "severity": "major"
    },
    {
      "category": "weak-testing",
      "text": "The enumerated CLI black-box test list omits `--include` and `--exclude` entirely, despite them being core v0.1 flags and Sprint 1 claiming \"every v0.1 CLI flag's black-box behavior and exit code\" is test-first. Also missing: interaction between `--include`/`--exclude` and `--staged`, and the ordering semantics when both are passed (exclude-wins is conventional but not stated). Add positive/negative tests for each, plus an ordering test.",
      "severity": "major"
    },
    {
      "category": "weak-testing",
      "text": "Integration-test invariants for the real clones are extremely thin: `findings.length >= 1` plus \"every finding has a non-null blame object\". This leaves the end-to-end pipeline under-covered against upstream drift risk that the pinned SHAs are meant to mitigate. Add at least: (a) JSON-Schema validation of the full output, (b) an assertion that at least one finding has each of the default markers present in the subtree, (c) a count stability check within a tolerance window (e.g. `±10%` of a recorded baseline) — or explicitly justify why the synthetic fixture's coverage makes these redundant.",
      "severity": "minor"
    },
    {
      "category": "ambiguity",
      "text": "The walker's dedup behavior across overlapping `--include` globs is unspecified, yet `blame.ts` \"caches by absolute file path (one blame invocation per file)\". If the walker yields the same path twice (e.g. `--include \"**/*.ts\" --include \"src/foo.ts\"`), does the extractor run twice, producing duplicate Findings? State explicitly that paths are deduplicated (by resolved absolute path) before extraction.",
      "severity": "minor"
    },
    {
      "category": "ambiguity",
      "text": "`--staged` behavior in a non-git worktree is not specified. It relies on `git diff --cached` which will fail outside a repo. Pin: does it exit 2, or silently produce an empty finding set? Also specify behavior when a staged file has been deleted from the working tree (cannot read working-tree bytes).",
      "severity": "minor"
    },
    {
      "category": "ambiguity",
      "text": "`--fail-on` repeatability is ambiguous. The spec states \"the flag is not repeatable\" but does not say what happens on repeat: is `--fail-on FIXME --fail-on HACK` a usage error (exit 2), or does the last occurrence win? Pin one.",
      "severity": "minor"
    },
    {
      "category": "ambiguity",
      "text": "The generic (unknown-extension) fallback extractor applies only the boundary regex without language-aware comment-region reduction, meaning a `TODO` inside a string literal in an unknown file *will* be reported, while the same construct in a `.ts` file will not. This asymmetry between known/unknown languages is not called out, and the test plan does not cover it. Either document the asymmetry explicitly in the Scope non-goals, or align by applying a conservative heuristic (e.g. only match when a known line-comment start precedes the marker on the same line).",
      "severity": "minor"
    },
    {
      "category": "ambiguity",
      "text": "Architecture prose says blame degrades to \"top-level `blame: null`\" for non-git trees, but `blame` is a per-finding field (there is no top-level `blame` key in the JSON schema). This wording is misleading; say \"per-finding `blame: null`\" consistently.",
      "severity": "minor"
    },
    {
      "category": "weak-testing",
      "text": "The no-git-worktree degradation test uses `mktemp -d`, but TMPDIR on some systems (notably certain CI runners or developer machines using Nix/sandboxing) can live inside a git worktree, causing the test to unexpectedly find git metadata. Either explicitly set up a directory guaranteed to be outside a worktree (e.g. `git init --bare` nowhere, walk up from TMPDIR and verify, or use a `GIT_CEILING_DIRECTORIES` guard), or document the assumption so flakiness can be diagnosed.",
      "severity": "minor"
    },
    {
      "category": "ambiguity",
      "text": "\"Dependency policy\" states \"Dev deps limited to `bun test` and type definitions\", but the CI matrix lists `biome check` as a lint step, implying `biome` is a dev dependency. Update the dep policy to include biome (or drop biome from CI).",
      "severity": "minor"
    },
    {
      "category": "ambiguity",
      "text": "Root-path symlink handling is unspecified. \"Symlinks are not followed (neither files nor directories)\" is stated for traversal, but if the user passes a symlink *as the scan root* (positional `path`), is it resolved before walking, or skipped-and-exit-0? Pin the rule for the root argument explicitly.",
      "severity": "minor"
    },
    {
      "category": "weak-testing",
      "text": "Blame batching scalability is untested. `git blame --porcelain -L a,a -L b,b -- <file>` with thousands of `-L` ranges (plausible in a large vendored file with many TODOs) may exceed OS `ARG_MAX`. Either add a guard + test (chunk the `-L` ranges), or document an upper bound and assert the behavior when exceeded.",
      "severity": "minor"
    }
  ],
  "summary": "Spec v0.4 is structurally complete — all nine mandatory sections are present, the Reviewer B r3 feedback appears addressed (regex boundaries, marker validation, CLI grammar, JSON surface, platform support), and the non-goals strongly honor the \"NOT a todo-list/CRUD\" disclaimers. The remaining issues cluster in two areas: (1) a direct contradiction between the \"each line → one Finding\" rule in multi-marker block comments and an enumerated golden test asserting two findings on the same line, which must be reconciled; and (2) several ambiguities around semantics that the schema/exit-code discipline depends on — `blame.date` source (author-time vs committer-time), continuation joining when blank lines are present, config auto-discovery, walker dedup, `--staged` in non-git trees, and `--fail-on` repeatability. Testing coverage is strong in the unit/black-box tier but has a real gap: `--include`/`--exclude` aren't enumerated in the CLI test list despite Sprint 1 claiming every flag is covered, and the real-repo integration invariants are so thin they may not catch a silent regression between pinned SHAs. None of these require a scope change; one more refinement round should land v0.5.",
  "suggested_next_version": "v0.5",
  "usage": null,
  "effort_used": "max"
}
<!-- samospec:critique end -->
