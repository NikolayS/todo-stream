# Reviewer A — Codex

## summary

v0.7 is much stronger, but the remaining high-risk issues are mostly about claims that are hard to implement safely: Git config/environment neutralization, byte-vs-string escaping, unescaped diagnostics, and CI semantics based on forgeable author dates. The spec should tighten those before implementation starts.

## weak-implementation

- (major) The subprocess hardening still relies on Git config wildcard clearing that Git does not actually provide in the way described. `-c filter.*.process=`, `alias.*=`, and `url.*.insteadOf=` set literal wildcard-named keys; they do not clear all repo-local `filter.<name>`, `alias.<name>`, or `url.<base>.insteadOf` entries. This means the spec may claim repo-local `.git/config` neutralization while leaving dangerous config active. v0.8 should replace this with behavior verified against real Git semantics or narrow the hardening claim.
- (major) ANSI stripping is specified after UTF-8 decoding, but the grammar is byte-oriented. 8-bit CSI/OSC bytes like `0x9b` and `0x9d` are invalid standalone UTF-8 and decode to U+FFFD before the stripper can recognize them, so the promised 8-bit ANSI removal test cannot pass as written. Strip terminal-control sequences on raw bytes before UTF-8 decoding, or redefine the renderer pipeline precisely enough to handle C1 controls.
- (major) The `--max-findings` behavior is internally inconsistent. Resource limits say capped findings are retained/emitted and the process exits 2, while the tests plan expects exit 2 with no stdout. Output atomicity also implies consumers should not receive a partial/truncated report. v0.8 should choose one contract, preferably stdout empty on cap violation, and update data flow, tests, and README guidance accordingly.
- (minor) The spec promises support for filenames with non-ASCII bytes, but the implementation stack is JavaScript/Bun path strings, which generally cannot faithfully represent arbitrary invalid-UTF-8 POSIX filenames. The fixture only covers a valid 4-byte UTF-8 codepoint. Either narrow the claim to valid UTF-8 filenames or specify byte-preserving filesystem APIs and tests for invalid byte sequences.
- (minor) `git check-ignore --stdin -z -v` has a structured verbose output and returns exit code 1 when no paths match. The spec says delegated pruning uses it but does not pin parsing or normal non-match exit handling. This is likely to produce either incorrect ignores or false internal errors. v0.8 should define the exact parser and exit-code interpretation for check-ignore.
- (minor) The extractor claims known languages will not report `TODO` inside strings, while also rejecting string-literal tokenization. That is not generally achievable for lines like `const s = "// TODO";` because comment delimiters inside strings look like comments to a regex reducer. Either add lightweight string/comment lexing for supported languages or document this as a false-positive caveat and adjust tests.

## missing-risk

- (major) `HARDENED_GIT_ENV` does not explicitly define an environment allowlist or clear inherited Git control variables such as `GIT_DIR`, `GIT_WORK_TREE`, `GIT_INDEX_FILE`, `GIT_CONFIG_COUNT`, `GIT_CONFIG_KEY_*`, `GIT_CONFIG_VALUE_*`, `GIT_CONFIG_PARAMETERS`, `GIT_OBJECT_DIRECTORY`, and `GIT_ALTERNATE_OBJECT_DIRECTORIES`. Inherited variables can redirect repository context or inject config despite the `-c` flags. v0.8 should specify a minimal env passed to Git and a denylist/allowlist test for hostile inherited `GIT_*` variables.
- (major) stderr diagnostics are not escaped even though they include hostile paths and possibly Git-derived reasons. A filename containing newlines, tabs, ESC, OSC, or carriage returns can spoof CI logs through warnings such as large-file skips or blame failures. The spec hardens JSON/Markdown but leaves the operational log channel exposed. Add stderr-safe rendering rules: printable ASCII escaping, newline/control escaping, and ANSI/OSC stripping for every diagnostic field.
- (major) Using blame author-date for `--since` makes the CI workaround bypassable and operationally misleading: commit authors can backdate author timestamps so new TODOs fall before the baseline. The spec should explicitly state that `--since` is not a security or PR-regression gate, or use a different mode for CI. The deferred diff/baseline mode should be the recommended CI gate, not author-date filtering.
- (major) There is no cap on files/directories walked, path count, total bytes read below the per-file cap, or wall-clock time. A repository with millions of tiny files and no findings can still exhaust CI time, memory, or stderr/log budget, especially because stable global ordering may require collecting paths before processing. Add `--max-files`, traversal timeout, bounded ignore-check batches, and clear behavior when those limits are hit.
- (minor) PATH sanitization rejects repo-local `git`, but it still trusts any absolute PATH component outside the scan root, including user-writable or world-writable directories such as `/tmp/...`. In CI, PATH can be influenced before the tool runs. For an untrusted-repo threat model, require a trusted system directory allowlist, reject writable directories, or let users provide an explicit trusted Git path.

## unnecessary-scope

- (minor) The v0.1 scope combines a regex parser for many languages, hostile filename support, Git hardening against malicious repos, Markdown/JSON terminal-safety, staged-file semantics, compiled multi-platform release, pinned real-repo integrations, and performance lanes. That is a large security-sensitive release surface for a first version. The next spec should either split v0.1 into a narrower local-trust scanner or make the untrusted-repo hardening the central acceptance gate and defer language/platform breadth.

## suggested-next-version

v0.8

<!-- samospec:critique v1 -->
{
  "findings": [
    {
      "category": "weak-implementation",
      "text": "The subprocess hardening still relies on Git config wildcard clearing that Git does not actually provide in the way described. `-c filter.*.process=`, `alias.*=`, and `url.*.insteadOf=` set literal wildcard-named keys; they do not clear all repo-local `filter.<name>`, `alias.<name>`, or `url.<base>.insteadOf` entries. This means the spec may claim repo-local `.git/config` neutralization while leaving dangerous config active. v0.8 should replace this with behavior verified against real Git semantics or narrow the hardening claim.",
      "severity": "major"
    },
    {
      "category": "missing-risk",
      "text": "`HARDENED_GIT_ENV` does not explicitly define an environment allowlist or clear inherited Git control variables such as `GIT_DIR`, `GIT_WORK_TREE`, `GIT_INDEX_FILE`, `GIT_CONFIG_COUNT`, `GIT_CONFIG_KEY_*`, `GIT_CONFIG_VALUE_*`, `GIT_CONFIG_PARAMETERS`, `GIT_OBJECT_DIRECTORY`, and `GIT_ALTERNATE_OBJECT_DIRECTORIES`. Inherited variables can redirect repository context or inject config despite the `-c` flags. v0.8 should specify a minimal env passed to Git and a denylist/allowlist test for hostile inherited `GIT_*` variables.",
      "severity": "major"
    },
    {
      "category": "weak-implementation",
      "text": "ANSI stripping is specified after UTF-8 decoding, but the grammar is byte-oriented. 8-bit CSI/OSC bytes like `0x9b` and `0x9d` are invalid standalone UTF-8 and decode to U+FFFD before the stripper can recognize them, so the promised 8-bit ANSI removal test cannot pass as written. Strip terminal-control sequences on raw bytes before UTF-8 decoding, or redefine the renderer pipeline precisely enough to handle C1 controls.",
      "severity": "major"
    },
    {
      "category": "missing-risk",
      "text": "stderr diagnostics are not escaped even though they include hostile paths and possibly Git-derived reasons. A filename containing newlines, tabs, ESC, OSC, or carriage returns can spoof CI logs through warnings such as large-file skips or blame failures. The spec hardens JSON/Markdown but leaves the operational log channel exposed. Add stderr-safe rendering rules: printable ASCII escaping, newline/control escaping, and ANSI/OSC stripping for every diagnostic field.",
      "severity": "major"
    },
    {
      "category": "missing-risk",
      "text": "Using blame author-date for `--since` makes the CI workaround bypassable and operationally misleading: commit authors can backdate author timestamps so new TODOs fall before the baseline. The spec should explicitly state that `--since` is not a security or PR-regression gate, or use a different mode for CI. The deferred diff/baseline mode should be the recommended CI gate, not author-date filtering.",
      "severity": "major"
    },
    {
      "category": "weak-implementation",
      "text": "The `--max-findings` behavior is internally inconsistent. Resource limits say capped findings are retained/emitted and the process exits 2, while the tests plan expects exit 2 with no stdout. Output atomicity also implies consumers should not receive a partial/truncated report. v0.8 should choose one contract, preferably stdout empty on cap violation, and update data flow, tests, and README guidance accordingly.",
      "severity": "major"
    },
    {
      "category": "missing-risk",
      "text": "There is no cap on files/directories walked, path count, total bytes read below the per-file cap, or wall-clock time. A repository with millions of tiny files and no findings can still exhaust CI time, memory, or stderr/log budget, especially because stable global ordering may require collecting paths before processing. Add `--max-files`, traversal timeout, bounded ignore-check batches, and clear behavior when those limits are hit.",
      "severity": "major"
    },
    {
      "category": "weak-implementation",
      "text": "The spec promises support for filenames with non-ASCII bytes, but the implementation stack is JavaScript/Bun path strings, which generally cannot faithfully represent arbitrary invalid-UTF-8 POSIX filenames. The fixture only covers a valid 4-byte UTF-8 codepoint. Either narrow the claim to valid UTF-8 filenames or specify byte-preserving filesystem APIs and tests for invalid byte sequences.",
      "severity": "minor"
    },
    {
      "category": "weak-implementation",
      "text": "`git check-ignore --stdin -z -v` has a structured verbose output and returns exit code 1 when no paths match. The spec says delegated pruning uses it but does not pin parsing or normal non-match exit handling. This is likely to produce either incorrect ignores or false internal errors. v0.8 should define the exact parser and exit-code interpretation for check-ignore.",
      "severity": "minor"
    },
    {
      "category": "weak-implementation",
      "text": "The extractor claims known languages will not report `TODO` inside strings, while also rejecting string-literal tokenization. That is not generally achievable for lines like `const s = \"// TODO\";` because comment delimiters inside strings look like comments to a regex reducer. Either add lightweight string/comment lexing for supported languages or document this as a false-positive caveat and adjust tests.",
      "severity": "minor"
    },
    {
      "category": "missing-risk",
      "text": "PATH sanitization rejects repo-local `git`, but it still trusts any absolute PATH component outside the scan root, including user-writable or world-writable directories such as `/tmp/...`. In CI, PATH can be influenced before the tool runs. For an untrusted-repo threat model, require a trusted system directory allowlist, reject writable directories, or let users provide an explicit trusted Git path.",
      "severity": "minor"
    },
    {
      "category": "unnecessary-scope",
      "text": "The v0.1 scope combines a regex parser for many languages, hostile filename support, Git hardening against malicious repos, Markdown/JSON terminal-safety, staged-file semantics, compiled multi-platform release, pinned real-repo integrations, and performance lanes. That is a large security-sensitive release surface for a first version. The next spec should either split v0.1 into a narrower local-trust scanner or make the untrusted-repo hardening the central acceptance gate and defer language/platform breadth.",
      "severity": "minor"
    }
  ],
  "summary": "v0.7 is much stronger, but the remaining high-risk issues are mostly about claims that are hard to implement safely: Git config/environment neutralization, byte-vs-string escaping, unescaped diagnostics, and CI semantics based on forgeable author dates. The spec should tighten those before implementation starts.",
  "suggested_next_version": "v0.8",
  "usage": null,
  "effort_used": "max"
}
<!-- samospec:critique end -->
