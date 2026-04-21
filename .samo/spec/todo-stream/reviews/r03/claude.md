# Reviewer B — Claude

## summary

Spec v0.3 is in solid shape: all nine mandatory sections present, the idea's 'not a todo-list app / not a CRUD tool' disclaimer is honored throughout, and the v0.2→v0.3 refinement retired the biggest prior contradictions (spec vs product version, pinned-upstream TODO tests, `--staged` semantics of 'file-level'). Remaining issues cluster in ambiguity around the JSON public surface (path relativity, short-SHA length, language identifiers, generated_at format), CLI flag behavior edges (`--since` date form and timezone, `--author` matched fields, `--fail-on` multi-marker, `--staged` working-tree-vs-index), and a brittle integration-count invariant that should migrate onto the synthetic fixture repo. No outright contradiction with the original idea.

## ambiguity

- (major) Marker-boundary regex is inconsistent. The snippet `(^|[^A-Za-z0-9_])(` + markers.map(escape).join(`|`) + `)\b[:\s-]?\s*(.*)` uses a custom non-identifier lookbehind on the left but `\b` on the right. `\b` is defined against `\w`, so for a user-supplied marker containing a non-`\w` terminal character (e.g. `TODO?`, `FIXME-LATER`, or a CJK marker) `\b` will not anchor as intended — producing either false positives or silent no-match. The spec should pin the trailing boundary to the same class used on the left or forbid non-`\w` marker characters.
- (major) `--since` syntax is underspecified. Example is `--since 2024-01-01`, but the spec does not state (a) whether only `YYYY-MM-DD` is accepted or also full ISO-8601 / relative forms like `7d`, (b) the timezone used to compare against the blame `date` (which is ISO-8601 with offset), or (c) whether the comparison is inclusive of the boundary. This directly affects the documented behavior 'drop findings whose blame date is older' and the CLI test `--since 2099-01-01 → empty`.
- (major) `--staged` semantics vs. scan target are ambiguous. User story 4 and the CLI surface say `--staged` limits scanning to files listed by `git diff --cached --name-only`, but the extractor reads the working-tree bytes via `Bun.file(path).text()`. If a file is staged with a new `FIXME` but the working tree has since removed it (or vice versa), the tool will report working-tree findings for staged files — not staged findings. Spec should either state explicitly 'filter the file set, read working tree' or require reading the staged blob (`git show :<path>`).
- (major) `path` field in the JSON and Markdown output is not pinned as relative-to-`root` vs. absolute. The schema example shows `path: src/foo.ts` (relative) but `root` is documented as the resolved absolute path, and `--staged` / `--config` may supply paths in different forms. Downstream consumers (user story 5) need a stable rule. Recommend pinning to POSIX-style path relative to `root`, and stating behavior when the scan target is a single file.
- (minor) Short SHA length is unspecified. The example uses a 10-char SHA (`a1b2c3d4e5`) but git's default short SHA is 7 and `git blame --porcelain` returns the full 40-char SHA. The JSON schema is declared a stable public surface — consumers will index/diff on this field, so the spec must pin either full SHA or a fixed short length, not leave it to implementation.
- (minor) `--author <substr>` does not state which fields it matches (author name, author email, or both) or whether matching is case-sensitive. The Sprint-4 CLI test `--author nobody → empty; --author <real> → non-empty` cannot be written deterministically without this.
- (minor) `--fail-on` is documented as taking a single marker (`--fail-on TODO|FIXME|...`). It is not stated whether the flag is repeatable or accepts a comma list, nor what `--fail-on FIXME` does when `--markers` does not include `FIXME` (silent no-op? usage error? scan for it anyway?). This interaction is load-bearing for Priya's CI gating story.
- (minor) Uncommitted / 'Not Committed Yet' blame output is not addressed. `git blame --porcelain` on a locally modified line emits a `0000000...` SHA and synthesized author 'Not Committed Yet'. The spec's `blame` object requires `author/email/date/sha`; it should state whether such rows yield `blame: null`, a populated object with synthesized values, or are filtered out.
- (minor) Symlink handling in the walker is listed as a test target but the intended behavior is never specified in the spec body (follow symlinks? skip? skip-with-cycle-detection? follow only when inside root?). A fixture test cannot be authored red-first without this.
- (minor) Block-comment continuation text attachment is described as 'attached as trailing text to the most recent preceding marker' but the joining rule is unspecified. Concatenated with `\n`? With a single space? Are `*` / `--` / leading whitespace stripped per continuation line as they are for the marker line? The golden-file renderer tests depend on byte-exact output, so this must be pinned.
- (minor) Case sensitivity of user-supplied markers is half-specified. Spec says 'match is case-sensitive by default'. That implies `--markers todo` matches only lowercase `todo`, which will surprise users who expect the default set to be normalized to upper-case. Either document that user-supplied markers are used verbatim (and test it) or normalize.
- (minor) The `language` field in the JSON output is not enumerated. The extractor dispatches on extension to a comment-syntax table covering ~10 languages, and unknown extensions produce `language: unknown`, but the exact string for each extension (e.g. `.ts` vs `.tsx` vs `.mts`, `.cpp` vs `.cc` vs `.cxx`) is unpinned despite the JSON schema being declared stable.
- (minor) Config-resolution precedence lists `CLI flags > env > todo-stream.config.json > defaults`, but the env-var naming convention and which flags are env-addressable are not stated. At minimum spec which env var maps to which CLI flag, or state that env support is out of scope for v0.1 and remove it from the precedence list.
- (minor) `generated_at` format/timezone is shown as `2026-04-21T00:00:00Z` but not formally pinned. Since the JSON schema is a stable public surface, pin it (ISO-8601 UTC with trailing `Z`, second-precision) so downstream ingesters can rely on it.

## weak-testing

- (major) Real-repo integration invariants `findings.length >= 10` (postgres/postgres subtree) and `>= 1` (database-lab) are structurally brittle against upstream drift: a cleanup commit that removes TODOs in the pinned subtree will flip CI red even though todo-stream itself is correct. Mitigations: pin the clone to a specific commit SHA, or replace the count threshold with 'schema validates + no runtime error' and move count-sensitive assertions onto the synthetic fixture repo the spec already introduces for exactly this purpose.
- (minor) The CLI suite lacks a case for the documented graceful-degradation path 'outside a git worktree, top-level `blame` is the JSON literal `null`'. This is an explicit behavior introduced in v0.3 and is easy to write as a black-box test (scan a freshly `mkdir`ed tmpdir with a TODO file). Without it, a regression where `blame` becomes `{author:null,...}` will not be caught.
- (minor) The perf lane is described as 'fails only if runtime exceeds 90 s ... tracked as a trend, not a tight gate'. Tracking a trend requires comparing to prior runs; the spec only writes a CSV artifact. Either wire a baseline comparison (e.g. fail on >2× regression vs. last main) or drop the 'trend' framing — currently the lane is just a coarse liveness check.

## suggested-next-version

0.4

<!-- samospec:critique v1 -->
{
  "findings": [
    {
      "category": "ambiguity",
      "text": "Marker-boundary regex is inconsistent. The snippet `(^|[^A-Za-z0-9_])(` + markers.map(escape).join(`|`) + `)\\b[:\\s-]?\\s*(.*)` uses a custom non-identifier lookbehind on the left but `\\b` on the right. `\\b` is defined against `\\w`, so for a user-supplied marker containing a non-`\\w` terminal character (e.g. `TODO?`, `FIXME-LATER`, or a CJK marker) `\\b` will not anchor as intended — producing either false positives or silent no-match. The spec should pin the trailing boundary to the same class used on the left or forbid non-`\\w` marker characters.",
      "severity": "major"
    },
    {
      "category": "ambiguity",
      "text": "`--since` syntax is underspecified. Example is `--since 2024-01-01`, but the spec does not state (a) whether only `YYYY-MM-DD` is accepted or also full ISO-8601 / relative forms like `7d`, (b) the timezone used to compare against the blame `date` (which is ISO-8601 with offset), or (c) whether the comparison is inclusive of the boundary. This directly affects the documented behavior 'drop findings whose blame date is older' and the CLI test `--since 2099-01-01 → empty`.",
      "severity": "major"
    },
    {
      "category": "ambiguity",
      "text": "`--staged` semantics vs. scan target are ambiguous. User story 4 and the CLI surface say `--staged` limits scanning to files listed by `git diff --cached --name-only`, but the extractor reads the working-tree bytes via `Bun.file(path).text()`. If a file is staged with a new `FIXME` but the working tree has since removed it (or vice versa), the tool will report working-tree findings for staged files — not staged findings. Spec should either state explicitly 'filter the file set, read working tree' or require reading the staged blob (`git show :<path>`).",
      "severity": "major"
    },
    {
      "category": "weak-testing",
      "text": "Real-repo integration invariants `findings.length >= 10` (postgres/postgres subtree) and `>= 1` (database-lab) are structurally brittle against upstream drift: a cleanup commit that removes TODOs in the pinned subtree will flip CI red even though todo-stream itself is correct. Mitigations: pin the clone to a specific commit SHA, or replace the count threshold with 'schema validates + no runtime error' and move count-sensitive assertions onto the synthetic fixture repo the spec already introduces for exactly this purpose.",
      "severity": "major"
    },
    {
      "category": "ambiguity",
      "text": "`path` field in the JSON and Markdown output is not pinned as relative-to-`root` vs. absolute. The schema example shows `path: src/foo.ts` (relative) but `root` is documented as the resolved absolute path, and `--staged` / `--config` may supply paths in different forms. Downstream consumers (user story 5) need a stable rule. Recommend pinning to POSIX-style path relative to `root`, and stating behavior when the scan target is a single file.",
      "severity": "major"
    },
    {
      "category": "ambiguity",
      "text": "Short SHA length is unspecified. The example uses a 10-char SHA (`a1b2c3d4e5`) but git's default short SHA is 7 and `git blame --porcelain` returns the full 40-char SHA. The JSON schema is declared a stable public surface — consumers will index/diff on this field, so the spec must pin either full SHA or a fixed short length, not leave it to implementation.",
      "severity": "minor"
    },
    {
      "category": "ambiguity",
      "text": "`--author <substr>` does not state which fields it matches (author name, author email, or both) or whether matching is case-sensitive. The Sprint-4 CLI test `--author nobody → empty; --author <real> → non-empty` cannot be written deterministically without this.",
      "severity": "minor"
    },
    {
      "category": "ambiguity",
      "text": "`--fail-on` is documented as taking a single marker (`--fail-on TODO|FIXME|...`). It is not stated whether the flag is repeatable or accepts a comma list, nor what `--fail-on FIXME` does when `--markers` does not include `FIXME` (silent no-op? usage error? scan for it anyway?). This interaction is load-bearing for Priya's CI gating story.",
      "severity": "minor"
    },
    {
      "category": "ambiguity",
      "text": "Uncommitted / 'Not Committed Yet' blame output is not addressed. `git blame --porcelain` on a locally modified line emits a `0000000...` SHA and synthesized author 'Not Committed Yet'. The spec's `blame` object requires `author/email/date/sha`; it should state whether such rows yield `blame: null`, a populated object with synthesized values, or are filtered out.",
      "severity": "minor"
    },
    {
      "category": "ambiguity",
      "text": "Symlink handling in the walker is listed as a test target but the intended behavior is never specified in the spec body (follow symlinks? skip? skip-with-cycle-detection? follow only when inside root?). A fixture test cannot be authored red-first without this.",
      "severity": "minor"
    },
    {
      "category": "ambiguity",
      "text": "Block-comment continuation text attachment is described as 'attached as trailing text to the most recent preceding marker' but the joining rule is unspecified. Concatenated with `\\n`? With a single space? Are `*` / `--` / leading whitespace stripped per continuation line as they are for the marker line? The golden-file renderer tests depend on byte-exact output, so this must be pinned.",
      "severity": "minor"
    },
    {
      "category": "ambiguity",
      "text": "Case sensitivity of user-supplied markers is half-specified. Spec says 'match is case-sensitive by default'. That implies `--markers todo` matches only lowercase `todo`, which will surprise users who expect the default set to be normalized to upper-case. Either document that user-supplied markers are used verbatim (and test it) or normalize.",
      "severity": "minor"
    },
    {
      "category": "ambiguity",
      "text": "The `language` field in the JSON output is not enumerated. The extractor dispatches on extension to a comment-syntax table covering ~10 languages, and unknown extensions produce `language: unknown`, but the exact string for each extension (e.g. `.ts` vs `.tsx` vs `.mts`, `.cpp` vs `.cc` vs `.cxx`) is unpinned despite the JSON schema being declared stable.",
      "severity": "minor"
    },
    {
      "category": "ambiguity",
      "text": "Config-resolution precedence lists `CLI flags > env > todo-stream.config.json > defaults`, but the env-var naming convention and which flags are env-addressable are not stated. At minimum spec which env var maps to which CLI flag, or state that env support is out of scope for v0.1 and remove it from the precedence list.",
      "severity": "minor"
    },
    {
      "category": "ambiguity",
      "text": "`generated_at` format/timezone is shown as `2026-04-21T00:00:00Z` but not formally pinned. Since the JSON schema is a stable public surface, pin it (ISO-8601 UTC with trailing `Z`, second-precision) so downstream ingesters can rely on it.",
      "severity": "minor"
    },
    {
      "category": "weak-testing",
      "text": "The CLI suite lacks a case for the documented graceful-degradation path 'outside a git worktree, top-level `blame` is the JSON literal `null`'. This is an explicit behavior introduced in v0.3 and is easy to write as a black-box test (scan a freshly `mkdir`ed tmpdir with a TODO file). Without it, a regression where `blame` becomes `{author:null,...}` will not be caught.",
      "severity": "minor"
    },
    {
      "category": "weak-testing",
      "text": "The perf lane is described as 'fails only if runtime exceeds 90 s ... tracked as a trend, not a tight gate'. Tracking a trend requires comparing to prior runs; the spec only writes a CSV artifact. Either wire a baseline comparison (e.g. fail on >2× regression vs. last main) or drop the 'trend' framing — currently the lane is just a coarse liveness check.",
      "severity": "minor"
    }
  ],
  "summary": "Spec v0.3 is in solid shape: all nine mandatory sections present, the idea's 'not a todo-list app / not a CRUD tool' disclaimer is honored throughout, and the v0.2→v0.3 refinement retired the biggest prior contradictions (spec vs product version, pinned-upstream TODO tests, `--staged` semantics of 'file-level'). Remaining issues cluster in ambiguity around the JSON public surface (path relativity, short-SHA length, language identifiers, generated_at format), CLI flag behavior edges (`--since` date form and timezone, `--author` matched fields, `--fail-on` multi-marker, `--staged` working-tree-vs-index), and a brittle integration-count invariant that should migrate onto the synthetic fixture repo. No outright contradiction with the original idea.",
  "suggested_next_version": "0.4",
  "usage": null,
  "effort_used": "max"
}
<!-- samospec:critique end -->
