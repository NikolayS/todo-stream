# Reviewer B — Claude

## summary

Spec v0.5 is mature and all nine mandatory baseline sections are present; the big-ticket items from r4 (multi-marker-per-occurrence, continuation algorithm, ARG_MAX chunking, GIT_CEILING_DIRECTORIES guard, integration schema validation) are landed cleanly. Remaining issues cluster in two buckets. (1) Public-surface ambiguity that will silently fragment conforming implementations: the `--config` file schema is declared as a precedence tier but its keys, merge semantics for list-valued fields, and unknown-key handling are unspecified; Markdown rendering of embedded `\n` in `text` is called out as a golden-test case but not pinned in the output-shape section that otherwise mandates a single-line list item per finding; the continuation algorithm does not cover the empty-marker-line case that realistic block comments routinely produce. (2) Weak testing: integration invariant 5's `grep -r` probe can falsely flag markers that appear only inside string literals on code-only lines (which the extractor correctly suppresses), flipping CI red on spec-conformant behavior; the `--staged` flag is specified against `git diff --cached --name-only` without `-z`, leaving filenames with whitespace or non-ASCII characters exposed to mis-parsing with no test fixture; the perf-lane 90 s ceiling does not acknowledge hosted-runner variance and has no smoothing. Minor gaps: per-file `git blame` spawn-failure handling is undefined (degrade vs abort), the Sprint-4 "large-file guard" has no spec or test, extension dispatch case sensitivity is unstated, extensionless conventional filenames (`Makefile`, `Dockerfile`) fall silently into the generic fallback with its documented asymmetry, and the ARG_MAX test's spawn-injection seam is not pinned in the architecture. No idea-contradiction findings — the non-goals (not a todo-list app, not a CRUD storage tool) are consistently honored throughout.

## ambiguity

- (major) The `--config <path>` JSON file is declared a stable config source in the precedence rule (CLI flags > --config file > defaults) but its schema is nowhere specified. Which keys are accepted (markers, include, exclude, fail-on, since, author, no-blame, no-gitignore, format)? How do list-valued fields compose — does a CLI `--include X` override config `include: [Y,Z]` wholesale, or is it appended? Are unknown keys rejected (exit 2) or ignored? Is the file validated? The tests plan exercises `--config <path>` (bad JSON → exit 2; nonexistent → exit 2) but does not pin field shape or merge semantics, so implementers could diverge in a way that immediately breaks any user relying on the config file.
- (minor) The continuation algorithm does not define the case where the marker line itself has empty `text` after punctuation-stripping. For `/* TODO:\n * body\n */`, step 3 says "concatenate the preceding marker's `text` with each continuation segment using a single `\n` between segments." With marker-line `text` = `""` and one continuation `"body"`, the concatenation produces `"\nbody"` (leading newline), but a reader might expect `"body"` (no leading newline when marker text is empty). The same ambiguity applies to `// TODO` at end-of-line in a line comment context (not a block). Worked examples cover only non-empty marker lines. Without pinning this, two conforming implementations will diverge on realistic comment shapes.
- (minor) Spawn-failure handling for per-file `git blame` is undefined. Reporter exit codes list `2` for "spawn failure" at the CLI level, and the Blamer explicitly degrades to `blame: null` for three cases (non-git worktree, untracked file, uncommitted line) — but says nothing about a `git blame` invocation that exits non-zero for a specific file (e.g., corrupted pack, transient fs error, submodule boundary). Is that file's findings dropped, set to `blame: null`, or does the whole run abort with exit 2? This is a meaningful robustness cliff on large repos where one bad file should not tank the report.
- (minor) Extension dispatch case sensitivity is unspecified. The language table uses lowercase extensions (`.ts`, `.cpp`, `.hpp`), but files on case-preserving filesystems can appear as `.TS` or `.Hpp`. Does the extractor lowercase the extension before dispatch, or does `Foo.TS` fall into the generic `unknown` fallback — triggering the deliberately-asymmetric string-in-code behavior? This lights up differently across macOS (case-insensitive default) and Linux tests and is not covered in the walker or extractor fixtures.
- (minor) Extensionless files with conventional names (`Makefile`, `Dockerfile`, `Jenkinsfile`, `.gitignore`, `CMakeLists.txt`) are not addressed. The language table dispatches strictly on extension, so these fall to the `unknown` generic fallback, inheriting the documented string-literal asymmetry. Given that these files frequently carry `TODO` in idiomatic `#` comments, either (a) the table should include a filename-based entry for common shell-comment files, or (b) the spec should explicitly note they use the fallback and accept the precision loss. Leaving it silent guarantees divergence between implementations that hardcode special cases and those that don't.
- (minor) ARG_MAX chunking is tested by "mock/inject spawn," but the architecture section does not specify the spawn dependency-injection seam (function boundary, module-level variable, constructor arg?). Without pinning the seam, the test harness and the blamer can drift — the chunking test will pass against a mock that matches the seam it assumed, not the seam the production blamer actually uses. Pin the injection point (e.g., `blame.ts` exports a `runBlame(args): Promise<string>` that is the only spawn caller and is swappable in tests).

## contradiction

- (major) The Markdown output shape renders every finding as a single list item on one line — `- **TODO** L42 · Jane Doe, 2024-07-11 (a1b2c3d4e5) — rewrite with streaming parser` — but the `text` field can legitimately contain `\n` (including `\n\n` for blank continuation lines) per the authoritative continuation algorithm, and the Renderers golden-file enumeration explicitly includes a case for "Multi-line `text` with `\n` continuations (JSON escaping **+ Markdown rendering of the newline**)." The spec never pins what "Markdown rendering of the newline" means — literal `\n` in the string, hard line break (two spaces + newline), soft wrap with indented continuation, replaced with a separator? A renderer implementing any of these would pass the written prose while producing non-interoperable output.

## weak-testing

- (major) Integration invariant 5 ("Each default marker in {TODO, FIXME, HACK, XXX} that appears at all in the subtree — verified by an independent `grep -r` probe at test startup — has at least one finding in the output") is vulnerable to false failures. `grep -r TODO` matches the token anywhere on a line, including inside string literals on code-only lines, whereas the extractor correctly suppresses those in known languages. A subtree where, e.g., `HACK` appears only inside a `const s = "HACK"` C-string is detected by the probe but legitimately produces zero findings — flipping CI red on an extractor that is behaving per spec. Either (a) restrict the probe to comment regions (same logic as the extractor, which defeats independence), or (b) use a probe over git-tracked comment lines only, or (c) scope invariant 5 to markers known-present in comments via a curated allow-list pinned at the integration SHA.
- (minor) `--staged` is specified to operate on files listed by `git diff --cached --name-only`, but the spec does not require `-z` (NUL-delimited) output. Filenames containing newlines, quotes, or non-ASCII characters get quoted/escaped by default git output and will parse incorrectly as file paths. The CLI test matrix covers "staged-but-deleted" and "`--staged` outside a worktree" but has no fixture exercising awkward filenames, so a conformant implementation can silently mis-scan staged files with whitespace or non-ASCII names.
- (minor) The performance lane's single 90 s ceiling on GitHub-hosted `ubuntu-latest` is named "coarse ceiling / liveness gate," but GitHub's hosted runners exhibit documented wall-clock variance well over 20% and occasional spikes above 2×, and postgres/postgres' `src/backend/access/` has grown over time against a pinned-SHA scan that is *manually* updated. A single fixed-number threshold without a smoothing mechanism (e.g., best-of-3, N consecutive-failures policy) will eventually flake on infrastructure noise alone. The spec explicitly waves off a trend-tracking gate but does not acknowledge or mitigate the runner-variance risk for the one gate it does keep.

## missing-requirement

- (minor) Sprint 4 lists "hardening: large-file guard" as a deliverable, but the Implementation details have no large-file guard specification: no maximum file size, no documented behavior when one is exceeded (skip silently? emit warning? emit synthetic finding?), no test in the Tests plan. For a tool that calls `Bun.file(path).text()` (whole-file read into memory), this is a load-bearing limit that affects memory envelope and should be pinned in v0.1 alongside the binary-detection (8 KiB NUL) rule it sits next to.

## suggested-next-version

v0.6

<!-- samospec:critique v1 -->
{
  "findings": [
    {
      "category": "ambiguity",
      "text": "The `--config <path>` JSON file is declared a stable config source in the precedence rule (CLI flags > --config file > defaults) but its schema is nowhere specified. Which keys are accepted (markers, include, exclude, fail-on, since, author, no-blame, no-gitignore, format)? How do list-valued fields compose — does a CLI `--include X` override config `include: [Y,Z]` wholesale, or is it appended? Are unknown keys rejected (exit 2) or ignored? Is the file validated? The tests plan exercises `--config <path>` (bad JSON → exit 2; nonexistent → exit 2) but does not pin field shape or merge semantics, so implementers could diverge in a way that immediately breaks any user relying on the config file.",
      "severity": "major"
    },
    {
      "category": "contradiction",
      "text": "The Markdown output shape renders every finding as a single list item on one line — `- **TODO** L42 · Jane Doe, 2024-07-11 (a1b2c3d4e5) — rewrite with streaming parser` — but the `text` field can legitimately contain `\\n` (including `\\n\\n` for blank continuation lines) per the authoritative continuation algorithm, and the Renderers golden-file enumeration explicitly includes a case for \"Multi-line `text` with `\\n` continuations (JSON escaping **+ Markdown rendering of the newline**).\" The spec never pins what \"Markdown rendering of the newline\" means — literal `\\n` in the string, hard line break (two spaces + newline), soft wrap with indented continuation, replaced with a separator? A renderer implementing any of these would pass the written prose while producing non-interoperable output.",
      "severity": "major"
    },
    {
      "category": "weak-testing",
      "text": "Integration invariant 5 (\"Each default marker in {TODO, FIXME, HACK, XXX} that appears at all in the subtree — verified by an independent `grep -r` probe at test startup — has at least one finding in the output\") is vulnerable to false failures. `grep -r TODO` matches the token anywhere on a line, including inside string literals on code-only lines, whereas the extractor correctly suppresses those in known languages. A subtree where, e.g., `HACK` appears only inside a `const s = \"HACK\"` C-string is detected by the probe but legitimately produces zero findings — flipping CI red on an extractor that is behaving per spec. Either (a) restrict the probe to comment regions (same logic as the extractor, which defeats independence), or (b) use a probe over git-tracked comment lines only, or (c) scope invariant 5 to markers known-present in comments via a curated allow-list pinned at the integration SHA.",
      "severity": "major"
    },
    {
      "category": "ambiguity",
      "text": "The continuation algorithm does not define the case where the marker line itself has empty `text` after punctuation-stripping. For `/* TODO:\\n * body\\n */`, step 3 says \"concatenate the preceding marker's `text` with each continuation segment using a single `\\n` between segments.\" With marker-line `text` = `\"\"` and one continuation `\"body\"`, the concatenation produces `\"\\nbody\"` (leading newline), but a reader might expect `\"body\"` (no leading newline when marker text is empty). The same ambiguity applies to `// TODO` at end-of-line in a line comment context (not a block). Worked examples cover only non-empty marker lines. Without pinning this, two conforming implementations will diverge on realistic comment shapes.",
      "severity": "minor"
    },
    {
      "category": "ambiguity",
      "text": "Spawn-failure handling for per-file `git blame` is undefined. Reporter exit codes list `2` for \"spawn failure\" at the CLI level, and the Blamer explicitly degrades to `blame: null` for three cases (non-git worktree, untracked file, uncommitted line) — but says nothing about a `git blame` invocation that exits non-zero for a specific file (e.g., corrupted pack, transient fs error, submodule boundary). Is that file's findings dropped, set to `blame: null`, or does the whole run abort with exit 2? This is a meaningful robustness cliff on large repos where one bad file should not tank the report.",
      "severity": "minor"
    },
    {
      "category": "missing-requirement",
      "text": "Sprint 4 lists \"hardening: large-file guard\" as a deliverable, but the Implementation details have no large-file guard specification: no maximum file size, no documented behavior when one is exceeded (skip silently? emit warning? emit synthetic finding?), no test in the Tests plan. For a tool that calls `Bun.file(path).text()` (whole-file read into memory), this is a load-bearing limit that affects memory envelope and should be pinned in v0.1 alongside the binary-detection (8 KiB NUL) rule it sits next to.",
      "severity": "minor"
    },
    {
      "category": "ambiguity",
      "text": "Extension dispatch case sensitivity is unspecified. The language table uses lowercase extensions (`.ts`, `.cpp`, `.hpp`), but files on case-preserving filesystems can appear as `.TS` or `.Hpp`. Does the extractor lowercase the extension before dispatch, or does `Foo.TS` fall into the generic `unknown` fallback — triggering the deliberately-asymmetric string-in-code behavior? This lights up differently across macOS (case-insensitive default) and Linux tests and is not covered in the walker or extractor fixtures.",
      "severity": "minor"
    },
    {
      "category": "weak-testing",
      "text": "`--staged` is specified to operate on files listed by `git diff --cached --name-only`, but the spec does not require `-z` (NUL-delimited) output. Filenames containing newlines, quotes, or non-ASCII characters get quoted/escaped by default git output and will parse incorrectly as file paths. The CLI test matrix covers \"staged-but-deleted\" and \"`--staged` outside a worktree\" but has no fixture exercising awkward filenames, so a conformant implementation can silently mis-scan staged files with whitespace or non-ASCII names.",
      "severity": "minor"
    },
    {
      "category": "ambiguity",
      "text": "Extensionless files with conventional names (`Makefile`, `Dockerfile`, `Jenkinsfile`, `.gitignore`, `CMakeLists.txt`) are not addressed. The language table dispatches strictly on extension, so these fall to the `unknown` generic fallback, inheriting the documented string-literal asymmetry. Given that these files frequently carry `TODO` in idiomatic `#` comments, either (a) the table should include a filename-based entry for common shell-comment files, or (b) the spec should explicitly note they use the fallback and accept the precision loss. Leaving it silent guarantees divergence between implementations that hardcode special cases and those that don't.",
      "severity": "minor"
    },
    {
      "category": "weak-testing",
      "text": "The performance lane's single 90 s ceiling on GitHub-hosted `ubuntu-latest` is named \"coarse ceiling / liveness gate,\" but GitHub's hosted runners exhibit documented wall-clock variance well over 20% and occasional spikes above 2×, and postgres/postgres' `src/backend/access/` has grown over time against a pinned-SHA scan that is *manually* updated. A single fixed-number threshold without a smoothing mechanism (e.g., best-of-3, N consecutive-failures policy) will eventually flake on infrastructure noise alone. The spec explicitly waves off a trend-tracking gate but does not acknowledge or mitigate the runner-variance risk for the one gate it does keep.",
      "severity": "minor"
    },
    {
      "category": "ambiguity",
      "text": "ARG_MAX chunking is tested by \"mock/inject spawn,\" but the architecture section does not specify the spawn dependency-injection seam (function boundary, module-level variable, constructor arg?). Without pinning the seam, the test harness and the blamer can drift — the chunking test will pass against a mock that matches the seam it assumed, not the seam the production blamer actually uses. Pin the injection point (e.g., `blame.ts` exports a `runBlame(args): Promise<string>` that is the only spawn caller and is swappable in tests).",
      "severity": "minor"
    }
  ],
  "summary": "Spec v0.5 is mature and all nine mandatory baseline sections are present; the big-ticket items from r4 (multi-marker-per-occurrence, continuation algorithm, ARG_MAX chunking, GIT_CEILING_DIRECTORIES guard, integration schema validation) are landed cleanly. Remaining issues cluster in two buckets. (1) Public-surface ambiguity that will silently fragment conforming implementations: the `--config` file schema is declared as a precedence tier but its keys, merge semantics for list-valued fields, and unknown-key handling are unspecified; Markdown rendering of embedded `\\n` in `text` is called out as a golden-test case but not pinned in the output-shape section that otherwise mandates a single-line list item per finding; the continuation algorithm does not cover the empty-marker-line case that realistic block comments routinely produce. (2) Weak testing: integration invariant 5's `grep -r` probe can falsely flag markers that appear only inside string literals on code-only lines (which the extractor correctly suppresses), flipping CI red on spec-conformant behavior; the `--staged` flag is specified against `git diff --cached --name-only` without `-z`, leaving filenames with whitespace or non-ASCII characters exposed to mis-parsing with no test fixture; the perf-lane 90 s ceiling does not acknowledge hosted-runner variance and has no smoothing. Minor gaps: per-file `git blame` spawn-failure handling is undefined (degrade vs abort), the Sprint-4 \"large-file guard\" has no spec or test, extension dispatch case sensitivity is unstated, extensionless conventional filenames (`Makefile`, `Dockerfile`) fall silently into the generic fallback with its documented asymmetry, and the ARG_MAX test's spawn-injection seam is not pinned in the architecture. No idea-contradiction findings — the non-goals (not a todo-list app, not a CRUD storage tool) are consistently honored throughout.",
  "suggested_next_version": "v0.6",
  "usage": null,
  "effort_used": "max"
}
<!-- samospec:critique end -->
