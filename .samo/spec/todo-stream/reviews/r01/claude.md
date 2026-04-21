# Reviewer B — Claude

## summary

All nine mandatory sections are present and the spec faithfully honors the 'NOT a todo-list / NOT a CRUD storage tool' disclaimers — no contradictions with the original idea. The dominant weaknesses are (a) under-specified semantics at several boundaries (gitignore subset, symlink/binary handling, column/line indexing, block-comment multi-marker behavior, marker-regex composition, `--author` match field, blame-cache key), (b) a short-SHA width contradiction between the algorithm and the Markdown example, and (c) weak integration testing: floating HEAD on shallow clones, `findings.length > 0` style invariants, a vague 60 s perf gate, and no unit-test coverage listed for filters (`--since`, `--author`, `--fail-on`, `--staged`) or CLI error paths. Golden JSON tests also need a determinism story for `generated_at`/`root`. Tightening these will significantly strengthen v0.2.

## ambiguity

- (major) JSON renderer golden-file tests are described as 'byte-exact JSON', but the documented JSON schema includes a dynamic `generated_at` timestamp (and `root` absolute path). Byte-exact golden tests will fail on every run unless time and root are injected/mocked. The spec does not describe this injection seam — state how `generated_at` and `root` are made deterministic in tests.
- (major) The extractor algorithm hardcodes the regex alternation `(TODO|FIXME|HACK|XXX)` while `--markers` is a configurable list. The spec never says how user-provided markers are composed into the regex, how they are escaped, or whether they are matched case-sensitively. Specify.
- (minor) `--fail-on` is shown as `TODO|FIXME|...` in the CLI surface — unclear whether it accepts a single marker, a comma-separated list, or can be repeated. User story 1 uses a single marker, which does not disambiguate. Pin the exact grammar.
- (minor) `--format` defaults to 'markdown when TTY, json otherwise' but the spec does not say which stream's TTY-ness is checked (stdout vs stderr vs stdin) or how this interacts with piping into CI log collectors. Also unclear how Windows consoles are handled. Specify the exact predicate.
- (major) Interaction of `--since` / `--author` with `--no-blame` is undefined. Both filters require blame metadata; with blame disabled, does the CLI error out, silently drop the filter, or treat all findings as unfiltered? Specify the precedence/validation rules.
- (minor) The `Finding` record uses `line` and `column` but neither field's indexing convention (0-based vs 1-based) is specified. Renderer goldens and downstream consumers will diverge without this. Pin the convention in the schema section.
- (major) Block-comment handling says continuation lines are 'attached to the marker on the opening line as a multi-line text'. Behavior is undefined when a single block comment contains multiple markers (e.g. a TODO on line 3 and a FIXME on line 5 of the same `/* … */`), and whether text is trimmed of the leading `*`/indentation. Specify.
- (minor) Blame cache key is `(file, sha-of-HEAD)` 'within a run', but blame results are per-line, not per-file, and HEAD is constant for the whole run — so the cache key is effectively just `file`. Either the key is misdescribed or the cache granularity is wrong. Clarify.
- (major) The walker 'respects `.gitignore` via a small parser' and tests mention 'nested ignores', but the spec does not define the supported subset of gitignore semantics (negation `!`, `**` globs, directory-only `foo/`, `$GIT_DIR/info/exclude`, global excludes). 'Small parser' is a known source of silent divergence from git's behavior. Pin the supported subset or delegate to `git check-ignore`.
- (major) Walker test fixtures mention symlinks and binary files, but the spec does not specify behavior: are symlinks followed, skipped, or followed with cycle detection? How are binary files detected (extension list, NUL-byte sniff, `.gitattributes`)? Specify so tests can be meaningful.
- (minor) The JSON schema text mentions a `$schema` field in prose ('emits a versioned schema `{ $schema, tool, version, generated_at, findings: [...] }`') but the example block omits `$schema`. Either include it in the example with a concrete URL/path, or drop the prose mention.
- (minor) `--author <substr>` does not specify whether the substring is matched against author name, email, or both, nor whether matching is case-sensitive. Two users reading this will implement different filters. Specify.
- (minor) `git blame` is invoked via `Bun.spawn` but the spec does not address Windows support (shell differences, path separators, `git.exe` discovery) despite targeting an `npx`-distributed CLI. State the supported platforms for v0.1 explicitly, or describe the Windows path.
- (minor) The spec says 'Non-git trees degrade gracefully (blame fields `null`)' but the schema shows `blame` as an object. Clarify whether `blame` itself is nullable, or whether its inner fields are individually nullable — downstream JSON consumers need a definite shape.

## contradiction

- (major) Short-SHA length is inconsistent. Implementation details say 'short-SHA = first 10 chars of commit id', but the Markdown output example shows a 7-character SHA (`a1b2c3d`). Pin a single canonical width and update both sections.

## weak-testing

- (major) Integration-test invariants are too loose to detect regressions: `findings.length > 0`, 'every finding has non-null blame fields', and 'runtime < 60 s on CI hardware'. A broken extractor that returns one finding still passes. Add lower-bounds (e.g. `findings.length ≥ N` for a pinned clone SHA), distribution checks across markers, and a small set of exact-match assertions against pinned (path, line, marker) triples.
- (major) Integration tests clone `postgres/postgres` and `postgres-ai/database-lab` with `--depth=1`, meaning HEAD drifts between runs. Pinning 'by file path + marker, not by exact text' does not fix this — files are renamed or deleted too. Pin each integration scan to a specific upstream SHA (or a cached fixture tarball) so assertions are reproducible and failures are attributable.
- (minor) 'runtime < 60 s on CI hardware' is an unstable perf gate: 'CI hardware' is not pinned, and shallow-clone time alone can fluctuate. Either drop the perf assertion from correctness tests and move it to a separate perf lane with a pinned runner, or express it as a ratio against a baseline captured in the same run.
- (major) Filters (`--since`, `--author`, `--fail-on`, `--staged`, `--config`) are implemented in Sprint 4 but the tests plan lists no unit tests for them — only the extractor, walker, renderer, and blame-parser are called out as fixture-driven TDD. Add explicit red-first test targets for each filter and for `--fail-on` exit-code semantics.
- (minor) The spec does not describe tests for the CLI's error paths: unknown flag, conflicting flags (e.g. `--no-blame` with `--since`), invalid `--config` JSON, unreadable files, non-git roots. The only exit codes mentioned (0/1/2) need test coverage to be a stable contract.
- (minor) Renderer golden-file tests are described only as 'a known Finding[] → byte-exact JSON and stable Markdown'. No fixtures are enumerated for edge cases: empty findings, findings with null blame, multi-line text, non-ASCII content, very long text, duplicate findings on the same line. List the concrete golden fixtures to ensure coverage.

## suggested-next-version

0.2

<!-- samospec:critique v1 -->
{
  "findings": [
    {
      "category": "ambiguity",
      "text": "JSON renderer golden-file tests are described as 'byte-exact JSON', but the documented JSON schema includes a dynamic `generated_at` timestamp (and `root` absolute path). Byte-exact golden tests will fail on every run unless time and root are injected/mocked. The spec does not describe this injection seam — state how `generated_at` and `root` are made deterministic in tests.",
      "severity": "major"
    },
    {
      "category": "contradiction",
      "text": "Short-SHA length is inconsistent. Implementation details say 'short-SHA = first 10 chars of commit id', but the Markdown output example shows a 7-character SHA (`a1b2c3d`). Pin a single canonical width and update both sections.",
      "severity": "major"
    },
    {
      "category": "ambiguity",
      "text": "The extractor algorithm hardcodes the regex alternation `(TODO|FIXME|HACK|XXX)` while `--markers` is a configurable list. The spec never says how user-provided markers are composed into the regex, how they are escaped, or whether they are matched case-sensitively. Specify.",
      "severity": "major"
    },
    {
      "category": "ambiguity",
      "text": "`--fail-on` is shown as `TODO|FIXME|...` in the CLI surface — unclear whether it accepts a single marker, a comma-separated list, or can be repeated. User story 1 uses a single marker, which does not disambiguate. Pin the exact grammar.",
      "severity": "minor"
    },
    {
      "category": "ambiguity",
      "text": "`--format` defaults to 'markdown when TTY, json otherwise' but the spec does not say which stream's TTY-ness is checked (stdout vs stderr vs stdin) or how this interacts with piping into CI log collectors. Also unclear how Windows consoles are handled. Specify the exact predicate.",
      "severity": "minor"
    },
    {
      "category": "ambiguity",
      "text": "Interaction of `--since` / `--author` with `--no-blame` is undefined. Both filters require blame metadata; with blame disabled, does the CLI error out, silently drop the filter, or treat all findings as unfiltered? Specify the precedence/validation rules.",
      "severity": "major"
    },
    {
      "category": "ambiguity",
      "text": "The `Finding` record uses `line` and `column` but neither field's indexing convention (0-based vs 1-based) is specified. Renderer goldens and downstream consumers will diverge without this. Pin the convention in the schema section.",
      "severity": "minor"
    },
    {
      "category": "ambiguity",
      "text": "Block-comment handling says continuation lines are 'attached to the marker on the opening line as a multi-line text'. Behavior is undefined when a single block comment contains multiple markers (e.g. a TODO on line 3 and a FIXME on line 5 of the same `/* … */`), and whether text is trimmed of the leading `*`/indentation. Specify.",
      "severity": "major"
    },
    {
      "category": "ambiguity",
      "text": "Blame cache key is `(file, sha-of-HEAD)` 'within a run', but blame results are per-line, not per-file, and HEAD is constant for the whole run — so the cache key is effectively just `file`. Either the key is misdescribed or the cache granularity is wrong. Clarify.",
      "severity": "minor"
    },
    {
      "category": "ambiguity",
      "text": "The walker 'respects `.gitignore` via a small parser' and tests mention 'nested ignores', but the spec does not define the supported subset of gitignore semantics (negation `!`, `**` globs, directory-only `foo/`, `$GIT_DIR/info/exclude`, global excludes). 'Small parser' is a known source of silent divergence from git's behavior. Pin the supported subset or delegate to `git check-ignore`.",
      "severity": "major"
    },
    {
      "category": "ambiguity",
      "text": "Walker test fixtures mention symlinks and binary files, but the spec does not specify behavior: are symlinks followed, skipped, or followed with cycle detection? How are binary files detected (extension list, NUL-byte sniff, `.gitattributes`)? Specify so tests can be meaningful.",
      "severity": "major"
    },
    {
      "category": "weak-testing",
      "text": "Integration-test invariants are too loose to detect regressions: `findings.length > 0`, 'every finding has non-null blame fields', and 'runtime < 60 s on CI hardware'. A broken extractor that returns one finding still passes. Add lower-bounds (e.g. `findings.length ≥ N` for a pinned clone SHA), distribution checks across markers, and a small set of exact-match assertions against pinned (path, line, marker) triples.",
      "severity": "major"
    },
    {
      "category": "weak-testing",
      "text": "Integration tests clone `postgres/postgres` and `postgres-ai/database-lab` with `--depth=1`, meaning HEAD drifts between runs. Pinning 'by file path + marker, not by exact text' does not fix this — files are renamed or deleted too. Pin each integration scan to a specific upstream SHA (or a cached fixture tarball) so assertions are reproducible and failures are attributable.",
      "severity": "major"
    },
    {
      "category": "weak-testing",
      "text": "'runtime < 60 s on CI hardware' is an unstable perf gate: 'CI hardware' is not pinned, and shallow-clone time alone can fluctuate. Either drop the perf assertion from correctness tests and move it to a separate perf lane with a pinned runner, or express it as a ratio against a baseline captured in the same run.",
      "severity": "minor"
    },
    {
      "category": "weak-testing",
      "text": "Filters (`--since`, `--author`, `--fail-on`, `--staged`, `--config`) are implemented in Sprint 4 but the tests plan lists no unit tests for them — only the extractor, walker, renderer, and blame-parser are called out as fixture-driven TDD. Add explicit red-first test targets for each filter and for `--fail-on` exit-code semantics.",
      "severity": "major"
    },
    {
      "category": "weak-testing",
      "text": "The spec does not describe tests for the CLI's error paths: unknown flag, conflicting flags (e.g. `--no-blame` with `--since`), invalid `--config` JSON, unreadable files, non-git roots. The only exit codes mentioned (0/1/2) need test coverage to be a stable contract.",
      "severity": "minor"
    },
    {
      "category": "ambiguity",
      "text": "The JSON schema text mentions a `$schema` field in prose ('emits a versioned schema `{ $schema, tool, version, generated_at, findings: [...] }`') but the example block omits `$schema`. Either include it in the example with a concrete URL/path, or drop the prose mention.",
      "severity": "minor"
    },
    {
      "category": "ambiguity",
      "text": "`--author <substr>` does not specify whether the substring is matched against author name, email, or both, nor whether matching is case-sensitive. Two users reading this will implement different filters. Specify.",
      "severity": "minor"
    },
    {
      "category": "ambiguity",
      "text": "`git blame` is invoked via `Bun.spawn` but the spec does not address Windows support (shell differences, path separators, `git.exe` discovery) despite targeting an `npx`-distributed CLI. State the supported platforms for v0.1 explicitly, or describe the Windows path.",
      "severity": "minor"
    },
    {
      "category": "ambiguity",
      "text": "The spec says 'Non-git trees degrade gracefully (blame fields `null`)' but the schema shows `blame` as an object. Clarify whether `blame` itself is nullable, or whether its inner fields are individually nullable — downstream JSON consumers need a definite shape.",
      "severity": "minor"
    },
    {
      "category": "weak-testing",
      "text": "Renderer golden-file tests are described only as 'a known Finding[] → byte-exact JSON and stable Markdown'. No fixtures are enumerated for edge cases: empty findings, findings with null blame, multi-line text, non-ASCII content, very long text, duplicate findings on the same line. List the concrete golden fixtures to ensure coverage.",
      "severity": "minor"
    }
  ],
  "summary": "All nine mandatory sections are present and the spec faithfully honors the 'NOT a todo-list / NOT a CRUD storage tool' disclaimers — no contradictions with the original idea. The dominant weaknesses are (a) under-specified semantics at several boundaries (gitignore subset, symlink/binary handling, column/line indexing, block-comment multi-marker behavior, marker-regex composition, `--author` match field, blame-cache key), (b) a short-SHA width contradiction between the algorithm and the Markdown example, and (c) weak integration testing: floating HEAD on shallow clones, `findings.length > 0` style invariants, a vague 60 s perf gate, and no unit-test coverage listed for filters (`--since`, `--author`, `--fail-on`, `--staged`) or CLI error paths. Golden JSON tests also need a determinism story for `generated_at`/`root`. Tightening these will significantly strengthen v0.2.",
  "suggested_next_version": "0.2",
  "usage": null,
  "effort_used": "max"
}
<!-- samospec:critique end -->
