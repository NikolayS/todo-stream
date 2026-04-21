# Reviewer A — Codex

## summary

The spec is much tighter than earlier drafts, but the remaining high-risk gaps are operational: hostile filenames, untrusted source text in reports, sensitive data exposure, unbounded resource use, ambiguous staged/root behavior, and CI semantics that do not match the stated user story.

## missing-risk

- (major) The spec does not define robust path handling for adversarial filenames. Git outputs such as `git diff --cached --name-only` and `git check-ignore --stdin -v` are newline/tab-delimited by default, but repositories can contain filenames with newlines, tabs, leading dashes, control characters, or invalid UTF-8. Require NUL-delimited Git modes where available, pass paths only as argv arrays after `--`, define UTF-8 replacement/error behavior, and add fixtures for hostile filenames.
- (major) Markdown and terminal output are an injection surface. File paths, author names, emails, SHAs, and TODO text can contain Markdown metacharacters, HTML, ANSI escape sequences, or CI log control sequences. The spec needs explicit escaping/sanitization rules for Markdown and terminal diagnostics, plus tests proving raw source text cannot create links, tables, headings, hidden content, or terminal control effects.
- (major) The tool emits source comment text plus author emails into CI artifacts and dashboard inputs, but the spec does not treat this as sensitive data. TODO comments often contain internal project names, incident context, credentials accidentally pasted into comments, or employee emails. Add a data-exposure section, redaction options for emails/text, guidance for public CI, and tests that redaction preserves schema shape.
- (major) Error/output atomicity is not specified. If JSON is streamed and a read, blame, or spawn error occurs mid-run, consumers may receive partial JSON on stdout plus an exit 2. Define that diagnostics always go to stderr and either preflight before stdout, buffer until success, or emit a documented partial-output format; add tests for mid-scan failures.
- (minor) Git subprocess trust boundaries are not hardened. Running Git in untrusted repositories can be affected by local config, attributes, environment, PATH selection, and optional locks. Specify spawn without a shell, a controlled environment, `GIT_OPTIONAL_LOCKS=0`, whether textconv/external attributes are disabled, how `git` is located, and how malicious repo fixtures are tested.

## weak-implementation

- (major) Resource exhaustion is under-specified. `Bun.file(path).text()` reads whole files, long block-comment continuations are explicitly untruncated, JSON/Markdown output can grow without bound, and `git blame` has no timeout or concurrency policy. Add max file size, max finding text length or explicit no-truncation risk acceptance, spawn timeouts, concurrency limits, output-size behavior, and failure semantics for too-large inputs.
- (major) The CI gatekeeper story says `--fail-on FIXME` fails PRs that introduce a new FIXME, but the defined behavior fails on any matching finding after filters, including pre-existing debt. Either change the story to whole-repo gating or add an explicit diff/baseline mode; otherwise the main CI use case will be unusable for mature repositories.
- (major) `--staged` semantics are incomplete around the positional root. The spec defines intersection with `--include`, but not whether `todo-stream subdir --staged` scans only staged files under `subdir` or every staged file in the worktree. Pin the worktree discovery root, `git -C` behavior, path relativization, root intersection, submodule handling, and tests for staged files outside the scan root.
- (minor) The Python comment model is misleading. Treating triple-quoted strings as block comments will report `TODO` in ordinary Python string literals such as assigned multiline strings, which conflicts with the broader claim that known-language extraction inspects comment regions and suppresses code-only strings. Either document Python docstring/string false positives explicitly or remove triple-quote handling for v0.1.

## unnecessary-scope

- (minor) The v0.1 plan is over-scoped for a small CLI: two real-repo integration clones, hosted schema infrastructure, static binaries for four platforms, npm `npx` distribution, a perf lane, and a 4 full-time plus 1 part-time team. Cut v0.1 to deterministic fixture coverage, one packaged path, and a local bundled schema unless those release obligations are truly required before first ship.

## suggested-next-version

SPEC v0.6 should add an explicit security/ops section covering path encoding, output escaping, data sensitivity, resource limits, subprocess hardening, stdout/stderr atomicity, and corrected CI gating semantics before expanding release scope.

<!-- samospec:critique v1 -->
{
  "findings": [
    {
      "category": "missing-risk",
      "text": "The spec does not define robust path handling for adversarial filenames. Git outputs such as `git diff --cached --name-only` and `git check-ignore --stdin -v` are newline/tab-delimited by default, but repositories can contain filenames with newlines, tabs, leading dashes, control characters, or invalid UTF-8. Require NUL-delimited Git modes where available, pass paths only as argv arrays after `--`, define UTF-8 replacement/error behavior, and add fixtures for hostile filenames.",
      "severity": "major"
    },
    {
      "category": "missing-risk",
      "text": "Markdown and terminal output are an injection surface. File paths, author names, emails, SHAs, and TODO text can contain Markdown metacharacters, HTML, ANSI escape sequences, or CI log control sequences. The spec needs explicit escaping/sanitization rules for Markdown and terminal diagnostics, plus tests proving raw source text cannot create links, tables, headings, hidden content, or terminal control effects.",
      "severity": "major"
    },
    {
      "category": "missing-risk",
      "text": "The tool emits source comment text plus author emails into CI artifacts and dashboard inputs, but the spec does not treat this as sensitive data. TODO comments often contain internal project names, incident context, credentials accidentally pasted into comments, or employee emails. Add a data-exposure section, redaction options for emails/text, guidance for public CI, and tests that redaction preserves schema shape.",
      "severity": "major"
    },
    {
      "category": "weak-implementation",
      "text": "Resource exhaustion is under-specified. `Bun.file(path).text()` reads whole files, long block-comment continuations are explicitly untruncated, JSON/Markdown output can grow without bound, and `git blame` has no timeout or concurrency policy. Add max file size, max finding text length or explicit no-truncation risk acceptance, spawn timeouts, concurrency limits, output-size behavior, and failure semantics for too-large inputs.",
      "severity": "major"
    },
    {
      "category": "weak-implementation",
      "text": "The CI gatekeeper story says `--fail-on FIXME` fails PRs that introduce a new FIXME, but the defined behavior fails on any matching finding after filters, including pre-existing debt. Either change the story to whole-repo gating or add an explicit diff/baseline mode; otherwise the main CI use case will be unusable for mature repositories.",
      "severity": "major"
    },
    {
      "category": "weak-implementation",
      "text": "`--staged` semantics are incomplete around the positional root. The spec defines intersection with `--include`, but not whether `todo-stream subdir --staged` scans only staged files under `subdir` or every staged file in the worktree. Pin the worktree discovery root, `git -C` behavior, path relativization, root intersection, submodule handling, and tests for staged files outside the scan root.",
      "severity": "major"
    },
    {
      "category": "missing-risk",
      "text": "Error/output atomicity is not specified. If JSON is streamed and a read, blame, or spawn error occurs mid-run, consumers may receive partial JSON on stdout plus an exit 2. Define that diagnostics always go to stderr and either preflight before stdout, buffer until success, or emit a documented partial-output format; add tests for mid-scan failures.",
      "severity": "major"
    },
    {
      "category": "missing-risk",
      "text": "Git subprocess trust boundaries are not hardened. Running Git in untrusted repositories can be affected by local config, attributes, environment, PATH selection, and optional locks. Specify spawn without a shell, a controlled environment, `GIT_OPTIONAL_LOCKS=0`, whether textconv/external attributes are disabled, how `git` is located, and how malicious repo fixtures are tested.",
      "severity": "minor"
    },
    {
      "category": "weak-implementation",
      "text": "The Python comment model is misleading. Treating triple-quoted strings as block comments will report `TODO` in ordinary Python string literals such as assigned multiline strings, which conflicts with the broader claim that known-language extraction inspects comment regions and suppresses code-only strings. Either document Python docstring/string false positives explicitly or remove triple-quote handling for v0.1.",
      "severity": "minor"
    },
    {
      "category": "unnecessary-scope",
      "text": "The v0.1 plan is over-scoped for a small CLI: two real-repo integration clones, hosted schema infrastructure, static binaries for four platforms, npm `npx` distribution, a perf lane, and a 4 full-time plus 1 part-time team. Cut v0.1 to deterministic fixture coverage, one packaged path, and a local bundled schema unless those release obligations are truly required before first ship.",
      "severity": "minor"
    }
  ],
  "summary": "The spec is much tighter than earlier drafts, but the remaining high-risk gaps are operational: hostile filenames, untrusted source text in reports, sensitive data exposure, unbounded resource use, ambiguous staged/root behavior, and CI semantics that do not match the stated user story.",
  "suggested_next_version": "SPEC v0.6 should add an explicit security/ops section covering path encoding, output escaping, data sensitivity, resource limits, subprocess hardening, stdout/stderr atomicity, and corrected CI gating semantics before expanding release scope.",
  "usage": null,
  "effort_used": "max"
}
<!-- samospec:critique end -->
