# Reviewer B — Claude

## summary

Spec v0.6 is mature: all nine mandatory sections are present, the non-goals section strictly honors the idea's 'NOT a todo-list / NOT CRUD' disclaimers, and contradictions from earlier rounds (per-line vs per-occurrence, author-date source, `--staged` scope, config discovery) are resolved. Remaining issues cluster around (a) ambiguities at operational edges — ANSI-escape grammar, CRLF handling, `--author` boundary-straddling substrings, cross-file output ordering, git-missing-inside-worktree, single-file `--staged` root — and (b) weak-testing for interactions the spec already promises: `--fail-on` × `--since` filter interaction, hardened env on non-blame git calls, author-tz boundary on `--since`, Markdown atomicity parity with JSON. No major contradictions detected; no mandatory section missing. Recommend one more tightening round before freezing v0.1.0.

## ambiguity

- (major) The `--author` matcher is specified as a case-insensitive substring match against the concatenation `author + " <" + email + ">"`. This creates a surprising class of matches that span the name/email boundary — e.g. `--author "e <j"` matches a commit by `Jane <jane@example.com>` because the substring straddles the injected `" <"` separator. Users reasonably expect `--author` to match name OR email as independent fields, not a spliced artifact. Either tighten the spec to match against `author` and `email` as two independent fields (OR'd), or explicitly document and test that boundary-straddling substrings are a supported matching mode. No test exercises a boundary-straddling substring today, so whichever behavior you land on is unverified.
- (major) The ANSI-escape stripping rule in Output Escaping says `\x1b[...m` and similar CSI/OSC sequences are stripped, but leaves the exact grammar open (`"and similar"`). The Markdown rendering must be deterministic for golden-file tests, so pin the exact set: e.g. 7-bit CSI (`\x1b\[[\x30-\x3f]*[\x20-\x2f]*[\x40-\x7e]`), OSC terminated by BEL (`\x07`) or ST (`\x1b\\`), and whether 8-bit CSI (`\x9b`) and DEC private sequences are also stripped. Without a pinned grammar, two implementations could produce diverging Markdown for the same fixture. The ANSI probe test asserts a single `\x1b[31mRED\x1b[0m` case but does not exercise OSC, DEC, or 8-bit CSI.
- (major) Line-ending handling is unspecified. The extractor's block-comment continuation algorithm joins segments with `\n`, but does not say what happens when source bytes contain CRLF. Are `\r` bytes preserved in the captured `text`, stripped before joining, or left in place and then escaped by the JSON renderer? This affects both JSON `text` field bytes and Markdown rendering (CRLF inside `text` would split the bullet continuation awkwardly). Pin a rule (recommend: normalize to LF before the continuation algorithm and before marker-line text trimming) and add a fixture with CRLF line endings.
- (major) Cross-file ordering of findings in both JSON and Markdown output is unspecified. Within a file, findings sort by `(line, column)` ascending (pinned), but the walker's path yield order — driven by `Bun.Glob` + `git check-ignore` — is not declared stable. Downstream consumers (story 5's 'automated dashboard' comparing successive runs, golden-file tests) require deterministic across-file ordering. Pin the output order (e.g. lexicographic by POSIX-relative `path`) and add a test asserting the same fixture tree produces byte-identical output across runs and across filesystems.
- (major) `.gitignore` pruning behavior when `git` is absent is internally contradictory. The spec says (a) if git is missing, the run degrades to `--no-blame` + single stderr warning, and (b) `.gitignore` pruning is delegated to `git check-ignore` inside a worktree, else skipped. But detecting 'inside a worktree' itself requires `git rev-parse --is-inside-work-tree`. When git is missing, (b)'s branch is undefined: is the walker in 'no pruning' mode (equivalent to `--no-gitignore`)? Pin this explicitly and add a test case: PATH=/usr/bin:/bin with no `git` binary → walker treats the tree as not-in-worktree and applies no `.gitignore` pruning; findings all have `blame: null`.
- (minor) `--redact-emails` formula ('local-part → first char + `***`') leaks the full local-part for single-character usernames: `a@b.com` → `a***@b.com` reveals the entire original local-part. The test case even pins this: `a@b.com → a***@b.com`. For privacy-sensitive workflows (the stated motivation for the flag), this is a regression vs. a minimum redaction (e.g. replace entire local-part with `***` when len ≤ 2, or always use fixed `***@domain` without preserving the first char). Either document this leakage explicitly as an accepted trade-off or change the formula.
- (minor) Behavior when the positional scan root does not exist (typo, wrong cwd) is not specified. Exit code? Stderr message? Empty-findings exit 0? Given exit 2 is reserved for 'usage or internal error,' a non-existent root should almost certainly exit 2 with a readable message, but the spec never says. Add to the CLI surface and test matrix.
- (minor) `--staged` with a single-file positional root is underspecified. The spec says worktree root is discovered via `git -C <resolved-root> rev-parse --show-toplevel`, but `git -C <file>` is invalid. Either resolve to the file's parent before anchoring (and document), or reject `--staged` + single-file root with a usage error. Add a test case.
- (minor) Shebang lines in `.sh`/`.bash`/`.zsh` files begin with `#!` which matches the `#` line-comment syntax. A shebang like `#!/usr/bin/env bash # TODO: switch to bun` would yield a Finding on line 1 column N. This is not wrong per se, but is likely unintended for user story 3 (Dana orienting in `postgres/postgres`, which has shebangs everywhere). Either explicitly document that shebang lines are treated as comments (and thus scanned), or suppress them. Add a fixture either way.
- (minor) Binary-file detection ('NUL byte in first 8 KiB') will also skip UTF-16/UTF-32 source files (which contain NUL bytes in ASCII-range codepoints). For v0.1 the language table has no UTF-16 formats, so this is unlikely to matter, but the spec should either document the UTF-16 skip as an accepted consequence or add a narrow exception. Minor.
- (minor) The `runBlame` spawn seam receives `args` that begin with the git subcommand ('blame'), and the production implementation prepends 'git': `Bun.spawn(["git", ...args])`. This convention is only pinned for blame; other git subprocess calls (`check-ignore`, `rev-parse`, `diff --cached`) are described without specifying their spawn-seam naming or test-swap discipline. If only `runBlame` is swappable, then the per-file-spawn-failure and hardened-env guarantees for non-blame calls are untestable in isolation. Introduce a single `runGit` seam or name each seam explicitly, and pin their test-swap contracts.

## weak-testing

- (major) The subprocess-hardening env assertions are only exercised through the `runBlame` seam (`blame_env.test.ts`), which covers `git blame` only. The spec claims `HARDENED_GIT_ENV` applies to ALL git invocations (`check-ignore`, `rev-parse --show-toplevel`, `rev-parse --is-inside-work-tree`, `diff --cached --name-only -z`), but none of those paths have a spawn-seam with env-capture assertions. The malicious-config-repo end-to-end test is a weak proxy: if the hardened env is missing on, say, `check-ignore`, the failure mode is subtle (gitconfig-driven alias/pager side-effects) and may not register as a test failure. Introduce a common spawn seam (e.g. `runGit(args)`) and assert hardened env on every call site, or add per-subcommand env-capture tests.
- (major) `--fail-on` + filter interaction has a spec rule ('`--fail-on` is evaluated AFTER all filters; a repo with FIXMEs that are all filtered out by `--since` exits 0') but no test pins it. The CLI test table has `--since 2099-01-01 → empty findings` and `--fail-on FIXME with no FIXMEs → exit 0` separately, but no test combines them: fixture with real FIXMEs + `--since <future-date>` + `--fail-on FIXME` → assert exit 0 AND empty findings. Add this as a dedicated test case; it's a load-bearing promise to user story 1.
- (minor) `--since` comparison semantics are pinned to commit author-date in UTC, but no test exercises the timezone-boundary edge case: a commit authored at `2024-01-01T00:30:00+0900` (= `2023-12-31T15:30:00Z`) with `--since 2024-01-01` must be DROPPED because its UTC date is 2023-12-31. Add a synthetic-fixture commit with a non-UTC author-tz exactly on the boundary to prevent silent regression if implementation accidentally uses committer-time or local-time.
- (minor) Output-atomicity tests are described for JSON only ('stdout is empty OR a complete valid JSON; never partial'). Markdown atomicity is spec'd at a weaker granularity ('a file section is never split across stdout writes'), but there is no test that mid-run failure after some file sections have been flushed leaves a parseable-prefix Markdown document. Either specify + test the Markdown atomicity contract at the same rigor as JSON, or document that Markdown output may be truncated and README-gate on exit code for both formats (currently only JSON is gated).

## suggested-next-version

v0.7

<!-- samospec:critique v1 -->
{
  "findings": [
    {
      "category": "ambiguity",
      "text": "The `--author` matcher is specified as a case-insensitive substring match against the concatenation `author + \" <\" + email + \">\"`. This creates a surprising class of matches that span the name/email boundary — e.g. `--author \"e <j\"` matches a commit by `Jane <jane@example.com>` because the substring straddles the injected `\" <\"` separator. Users reasonably expect `--author` to match name OR email as independent fields, not a spliced artifact. Either tighten the spec to match against `author` and `email` as two independent fields (OR'd), or explicitly document and test that boundary-straddling substrings are a supported matching mode. No test exercises a boundary-straddling substring today, so whichever behavior you land on is unverified.",
      "severity": "major"
    },
    {
      "category": "ambiguity",
      "text": "The ANSI-escape stripping rule in Output Escaping says `\\x1b[...m` and similar CSI/OSC sequences are stripped, but leaves the exact grammar open (`\"and similar\"`). The Markdown rendering must be deterministic for golden-file tests, so pin the exact set: e.g. 7-bit CSI (`\\x1b\\[[\\x30-\\x3f]*[\\x20-\\x2f]*[\\x40-\\x7e]`), OSC terminated by BEL (`\\x07`) or ST (`\\x1b\\\\`), and whether 8-bit CSI (`\\x9b`) and DEC private sequences are also stripped. Without a pinned grammar, two implementations could produce diverging Markdown for the same fixture. The ANSI probe test asserts a single `\\x1b[31mRED\\x1b[0m` case but does not exercise OSC, DEC, or 8-bit CSI.",
      "severity": "major"
    },
    {
      "category": "weak-testing",
      "text": "The subprocess-hardening env assertions are only exercised through the `runBlame` seam (`blame_env.test.ts`), which covers `git blame` only. The spec claims `HARDENED_GIT_ENV` applies to ALL git invocations (`check-ignore`, `rev-parse --show-toplevel`, `rev-parse --is-inside-work-tree`, `diff --cached --name-only -z`), but none of those paths have a spawn-seam with env-capture assertions. The malicious-config-repo end-to-end test is a weak proxy: if the hardened env is missing on, say, `check-ignore`, the failure mode is subtle (gitconfig-driven alias/pager side-effects) and may not register as a test failure. Introduce a common spawn seam (e.g. `runGit(args)`) and assert hardened env on every call site, or add per-subcommand env-capture tests.",
      "severity": "major"
    },
    {
      "category": "ambiguity",
      "text": "Line-ending handling is unspecified. The extractor's block-comment continuation algorithm joins segments with `\\n`, but does not say what happens when source bytes contain CRLF. Are `\\r` bytes preserved in the captured `text`, stripped before joining, or left in place and then escaped by the JSON renderer? This affects both JSON `text` field bytes and Markdown rendering (CRLF inside `text` would split the bullet continuation awkwardly). Pin a rule (recommend: normalize to LF before the continuation algorithm and before marker-line text trimming) and add a fixture with CRLF line endings.",
      "severity": "major"
    },
    {
      "category": "ambiguity",
      "text": "Cross-file ordering of findings in both JSON and Markdown output is unspecified. Within a file, findings sort by `(line, column)` ascending (pinned), but the walker's path yield order — driven by `Bun.Glob` + `git check-ignore` — is not declared stable. Downstream consumers (story 5's 'automated dashboard' comparing successive runs, golden-file tests) require deterministic across-file ordering. Pin the output order (e.g. lexicographic by POSIX-relative `path`) and add a test asserting the same fixture tree produces byte-identical output across runs and across filesystems.",
      "severity": "major"
    },
    {
      "category": "ambiguity",
      "text": "`.gitignore` pruning behavior when `git` is absent is internally contradictory. The spec says (a) if git is missing, the run degrades to `--no-blame` + single stderr warning, and (b) `.gitignore` pruning is delegated to `git check-ignore` inside a worktree, else skipped. But detecting 'inside a worktree' itself requires `git rev-parse --is-inside-work-tree`. When git is missing, (b)'s branch is undefined: is the walker in 'no pruning' mode (equivalent to `--no-gitignore`)? Pin this explicitly and add a test case: PATH=/usr/bin:/bin with no `git` binary → walker treats the tree as not-in-worktree and applies no `.gitignore` pruning; findings all have `blame: null`.",
      "severity": "major"
    },
    {
      "category": "weak-testing",
      "text": "`--fail-on` + filter interaction has a spec rule ('`--fail-on` is evaluated AFTER all filters; a repo with FIXMEs that are all filtered out by `--since` exits 0') but no test pins it. The CLI test table has `--since 2099-01-01 → empty findings` and `--fail-on FIXME with no FIXMEs → exit 0` separately, but no test combines them: fixture with real FIXMEs + `--since <future-date>` + `--fail-on FIXME` → assert exit 0 AND empty findings. Add this as a dedicated test case; it's a load-bearing promise to user story 1.",
      "severity": "major"
    },
    {
      "category": "ambiguity",
      "text": "`--redact-emails` formula ('local-part → first char + `***`') leaks the full local-part for single-character usernames: `a@b.com` → `a***@b.com` reveals the entire original local-part. The test case even pins this: `a@b.com → a***@b.com`. For privacy-sensitive workflows (the stated motivation for the flag), this is a regression vs. a minimum redaction (e.g. replace entire local-part with `***` when len ≤ 2, or always use fixed `***@domain` without preserving the first char). Either document this leakage explicitly as an accepted trade-off or change the formula.",
      "severity": "minor"
    },
    {
      "category": "ambiguity",
      "text": "Behavior when the positional scan root does not exist (typo, wrong cwd) is not specified. Exit code? Stderr message? Empty-findings exit 0? Given exit 2 is reserved for 'usage or internal error,' a non-existent root should almost certainly exit 2 with a readable message, but the spec never says. Add to the CLI surface and test matrix.",
      "severity": "minor"
    },
    {
      "category": "ambiguity",
      "text": "`--staged` with a single-file positional root is underspecified. The spec says worktree root is discovered via `git -C <resolved-root> rev-parse --show-toplevel`, but `git -C <file>` is invalid. Either resolve to the file's parent before anchoring (and document), or reject `--staged` + single-file root with a usage error. Add a test case.",
      "severity": "minor"
    },
    {
      "category": "ambiguity",
      "text": "Shebang lines in `.sh`/`.bash`/`.zsh` files begin with `#!` which matches the `#` line-comment syntax. A shebang like `#!/usr/bin/env bash # TODO: switch to bun` would yield a Finding on line 1 column N. This is not wrong per se, but is likely unintended for user story 3 (Dana orienting in `postgres/postgres`, which has shebangs everywhere). Either explicitly document that shebang lines are treated as comments (and thus scanned), or suppress them. Add a fixture either way.",
      "severity": "minor"
    },
    {
      "category": "weak-testing",
      "text": "`--since` comparison semantics are pinned to commit author-date in UTC, but no test exercises the timezone-boundary edge case: a commit authored at `2024-01-01T00:30:00+0900` (= `2023-12-31T15:30:00Z`) with `--since 2024-01-01` must be DROPPED because its UTC date is 2023-12-31. Add a synthetic-fixture commit with a non-UTC author-tz exactly on the boundary to prevent silent regression if implementation accidentally uses committer-time or local-time.",
      "severity": "minor"
    },
    {
      "category": "ambiguity",
      "text": "Binary-file detection ('NUL byte in first 8 KiB') will also skip UTF-16/UTF-32 source files (which contain NUL bytes in ASCII-range codepoints). For v0.1 the language table has no UTF-16 formats, so this is unlikely to matter, but the spec should either document the UTF-16 skip as an accepted consequence or add a narrow exception. Minor.",
      "severity": "minor"
    },
    {
      "category": "weak-testing",
      "text": "Output-atomicity tests are described for JSON only ('stdout is empty OR a complete valid JSON; never partial'). Markdown atomicity is spec'd at a weaker granularity ('a file section is never split across stdout writes'), but there is no test that mid-run failure after some file sections have been flushed leaves a parseable-prefix Markdown document. Either specify + test the Markdown atomicity contract at the same rigor as JSON, or document that Markdown output may be truncated and README-gate on exit code for both formats (currently only JSON is gated).",
      "severity": "minor"
    },
    {
      "category": "ambiguity",
      "text": "The `runBlame` spawn seam receives `args` that begin with the git subcommand ('blame'), and the production implementation prepends 'git': `Bun.spawn([\"git\", ...args])`. This convention is only pinned for blame; other git subprocess calls (`check-ignore`, `rev-parse`, `diff --cached`) are described without specifying their spawn-seam naming or test-swap discipline. If only `runBlame` is swappable, then the per-file-spawn-failure and hardened-env guarantees for non-blame calls are untestable in isolation. Introduce a single `runGit` seam or name each seam explicitly, and pin their test-swap contracts.",
      "severity": "minor"
    }
  ],
  "summary": "Spec v0.6 is mature: all nine mandatory sections are present, the non-goals section strictly honors the idea's 'NOT a todo-list / NOT CRUD' disclaimers, and contradictions from earlier rounds (per-line vs per-occurrence, author-date source, `--staged` scope, config discovery) are resolved. Remaining issues cluster around (a) ambiguities at operational edges — ANSI-escape grammar, CRLF handling, `--author` boundary-straddling substrings, cross-file output ordering, git-missing-inside-worktree, single-file `--staged` root — and (b) weak-testing for interactions the spec already promises: `--fail-on` × `--since` filter interaction, hardened env on non-blame git calls, author-tz boundary on `--since`, Markdown atomicity parity with JSON. No major contradictions detected; no mandatory section missing. Recommend one more tightening round before freezing v0.1.0.",
  "suggested_next_version": "v0.7",
  "usage": null,
  "effort_used": "max"
}
<!-- samospec:critique end -->
