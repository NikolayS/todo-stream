# Reviewer A — Codex

## summary

The spec is much stronger than earlier versions, but its untrusted-repository posture still overclaims safety. The highest-risk gaps are Git execution hardening, PATH trust, terminal-control handling, invalid-byte handling, and missing global resource ceilings. There is also some v0.1 scope pressure that could delay a secure implementation.

## weak-implementation

- (major) Subprocess hardening still allows repo-local Git config to affect execution. Setting GIT_CONFIG_GLOBAL and GIT_CONFIG_SYSTEM does not disable .git/config, yet the spec claims a malicious repo-local config with pager/alias/core.fsmonitor is harmless. For an untrusted-repo tool, every git invocation should explicitly neutralize dangerous local settings needed by the invoked command, or the claim/test should be narrowed. This is especially important for fsmonitor, external filters, excludes, and any future git subcommands that may consult local config more broadly.
- (major) The output atomicity section contradicts itself. Reporter is said to buffer complete output before writing stdout, but Markdown is later allowed to stream sections, and implementation step 6 says the Reporter buffers the full output. This matters operationally because partial Markdown/JSON behavior drives CI consumer guarantees. Pick one contract per format and make tests match it; preferably keep JSON fully buffered and explicitly define Markdown as either best-effort streaming or fully buffered.
- (major) The invalid-UTF-8 plan is underspecified and likely incompatible with Bun.file(path).text(). The spec says source bytes are treated as Latin-1 byte-preserving and lossy bytes are represented through U+FFFD/hex escaping, but the data flow uses text() before extraction and v0.1 output omits raw. This can corrupt columns, marker matching, and emitted text. Specify byte decoding precisely, use bytes/ArrayBuffer in the extractor, and define exact JSON behavior for invalid byte sequences.

## missing-risk

- (major) The PATH hardening is too weak for the stated untrusted-repository threat model. Inheriting PATH and only removing a trailing '.' component does not protect against empty PATH elements, leading '.', relative directories, or a repo-controlled directory already earlier in PATH. A malicious repo can still influence which git binary is executed in common CI/misconfigured shell environments. Resolve git once from a trusted absolute path or sanitize PATH much more strictly, and test relative/empty PATH components.
- (major) JSON output intentionally preserves ANSI/terminal control sequences in source text. JSON-safe is not ops-safe: CI log viewers, dashboards, and terminals often re-emit decoded JSON strings, so preserving OSC hyperlinks, title changes, cursor movement, and color sequences can create log spoofing or phishing risks. Either strip/control-normalize ANSI for JSON too, add an opt-out raw mode, or clearly classify JSON text as unsafe to render without sanitization and test non-CSI OSC sequences.
- (major) The spec does not define protection against scanning the .git directory or other VCS internals when .gitignore pruning is disabled or unavailable. A default recursive scanner over untrusted repos should exclude .git/**, nested .git files/dirs, and likely common dependency/build directories by default or document why not. Otherwise it may leak commit metadata, scan packed/generated internals, and waste resources.
- (major) There is no explicit maximum number of files, findings, or total output bytes. The spec caps per-file size and blame concurrency, but a repository with millions of small files or comments can still exhaust memory because JSON rendering buffers the full report and text is never truncated. Add global guardrails such as max files scanned, max findings, max output bytes, or documented failure behavior with exit 2.

## unnecessary-scope

- (minor) The v0.1 scope is very large for a first release: Bun static binaries across four platform/arch targets, npm/npx packaging, hosted schema, two pinned real-repo integration clones, a perf lane, config files, staged mode, blame concurrency/timeouts, hostile filename fixtures, Markdown escaping, and many language syntaxes. Several of these are valuable but not required to prove the core linter. Consider cutting v0.1 to JSON-only plus core extraction/blame/staged-or-config, then add Markdown, hosted schema, perf lane, and broad language/platform polish in follow-up versions.

## suggested-next-version

v0.7

<!-- samospec:critique v1 -->
{
  "findings": [
    {
      "category": "weak-implementation",
      "text": "Subprocess hardening still allows repo-local Git config to affect execution. Setting GIT_CONFIG_GLOBAL and GIT_CONFIG_SYSTEM does not disable .git/config, yet the spec claims a malicious repo-local config with pager/alias/core.fsmonitor is harmless. For an untrusted-repo tool, every git invocation should explicitly neutralize dangerous local settings needed by the invoked command, or the claim/test should be narrowed. This is especially important for fsmonitor, external filters, excludes, and any future git subcommands that may consult local config more broadly.",
      "severity": "major"
    },
    {
      "category": "missing-risk",
      "text": "The PATH hardening is too weak for the stated untrusted-repository threat model. Inheriting PATH and only removing a trailing '.' component does not protect against empty PATH elements, leading '.', relative directories, or a repo-controlled directory already earlier in PATH. A malicious repo can still influence which git binary is executed in common CI/misconfigured shell environments. Resolve git once from a trusted absolute path or sanitize PATH much more strictly, and test relative/empty PATH components.",
      "severity": "major"
    },
    {
      "category": "weak-implementation",
      "text": "The output atomicity section contradicts itself. Reporter is said to buffer complete output before writing stdout, but Markdown is later allowed to stream sections, and implementation step 6 says the Reporter buffers the full output. This matters operationally because partial Markdown/JSON behavior drives CI consumer guarantees. Pick one contract per format and make tests match it; preferably keep JSON fully buffered and explicitly define Markdown as either best-effort streaming or fully buffered.",
      "severity": "major"
    },
    {
      "category": "missing-risk",
      "text": "JSON output intentionally preserves ANSI/terminal control sequences in source text. JSON-safe is not ops-safe: CI log viewers, dashboards, and terminals often re-emit decoded JSON strings, so preserving OSC hyperlinks, title changes, cursor movement, and color sequences can create log spoofing or phishing risks. Either strip/control-normalize ANSI for JSON too, add an opt-out raw mode, or clearly classify JSON text as unsafe to render without sanitization and test non-CSI OSC sequences.",
      "severity": "major"
    },
    {
      "category": "weak-implementation",
      "text": "The invalid-UTF-8 plan is underspecified and likely incompatible with Bun.file(path).text(). The spec says source bytes are treated as Latin-1 byte-preserving and lossy bytes are represented through U+FFFD/hex escaping, but the data flow uses text() before extraction and v0.1 output omits raw. This can corrupt columns, marker matching, and emitted text. Specify byte decoding precisely, use bytes/ArrayBuffer in the extractor, and define exact JSON behavior for invalid byte sequences.",
      "severity": "major"
    },
    {
      "category": "missing-risk",
      "text": "The spec does not define protection against scanning the .git directory or other VCS internals when .gitignore pruning is disabled or unavailable. A default recursive scanner over untrusted repos should exclude .git/**, nested .git files/dirs, and likely common dependency/build directories by default or document why not. Otherwise it may leak commit metadata, scan packed/generated internals, and waste resources.",
      "severity": "major"
    },
    {
      "category": "unnecessary-scope",
      "text": "The v0.1 scope is very large for a first release: Bun static binaries across four platform/arch targets, npm/npx packaging, hosted schema, two pinned real-repo integration clones, a perf lane, config files, staged mode, blame concurrency/timeouts, hostile filename fixtures, Markdown escaping, and many language syntaxes. Several of these are valuable but not required to prove the core linter. Consider cutting v0.1 to JSON-only plus core extraction/blame/staged-or-config, then add Markdown, hosted schema, perf lane, and broad language/platform polish in follow-up versions.",
      "severity": "minor"
    },
    {
      "category": "missing-risk",
      "text": "There is no explicit maximum number of files, findings, or total output bytes. The spec caps per-file size and blame concurrency, but a repository with millions of small files or comments can still exhaust memory because JSON rendering buffers the full report and text is never truncated. Add global guardrails such as max files scanned, max findings, max output bytes, or documented failure behavior with exit 2.",
      "severity": "major"
    }
  ],
  "summary": "The spec is much stronger than earlier versions, but its untrusted-repository posture still overclaims safety. The highest-risk gaps are Git execution hardening, PATH trust, terminal-control handling, invalid-byte handling, and missing global resource ceilings. There is also some v0.1 scope pressure that could delay a secure implementation.",
  "suggested_next_version": "v0.7",
  "usage": null,
  "effort_used": "max"
}
<!-- samospec:critique end -->
