# Reviewer B — Claude

## summary

v0.2 covers all nine baseline sections and correctly honours the 'not a todo-list / not a CRUD tool' disclaimers. The main weaknesses are internal consistency (spec/product version, JSON schema shape, marker regex vs. config) and ambiguity around flag semantics (`--staged`, `--since`/`--author` with missing blame). Testing is strong for pure modules but thin on CLI-flag coverage, and the integration 'pinned TODO' assertion is still brittle.

## contradiction

- (major) Version is inconsistent across the spec. Header says 'SPEC v0.2'; the 'Scope & non-goals (v0.1)' section is labelled v0.1; the JSON schema example pins `"version": "0.1.0"`; and the embedded changelog only contains a v0.1 entry dated 2026-04-21 with no v0.2 entry describing what changed in this revision. Pick one convention (e.g. spec version vs. product version) and make the changelog reflect the current header.
- (major) The JSON schema is self-inconsistent. Architecture → Renderer says the json renderer 'emits a versioned schema `{ $schema, tool, version, generated_at, findings: [...] }`', but the concrete example under Implementation details → JSON schema omits `$schema` and adds an undocumented `root` field. Downstream consumers promised a 'stable, documented schema' cannot rely on either shape.
- (major) The extractor regex hardcodes the marker list — `/(^|[^A-Za-z0-9_])(TODO|FIXME|HACK|XXX)\b[:\s-]?\s*(.*)/` — yet the surrounding text says 'markers from config' and the CLI exposes `--markers` to change them. Either the regex must be constructed from `config.markers`, or `--markers` is a no-op. Spec should state that the pattern is built dynamically.

## ambiguity

- (major) `--staged` is defined two different ways. The CLI surface says it 'limit[s] to files in `git diff --cached --name-only`' (file-level), but User story 4 says it prints 'TODOs touched by the diff' (line-level — only markers inside staged hunks). These produce very different outputs on a file where only an unrelated line is staged. Pin one semantics.
- (major) Interaction between `--since` / `--author` and missing blame is undefined. When `--no-blame` is passed, or the tree is not a git worktree, the spec says blame fields are `null`. It does not state whether `--since 2024-01-01` drops all findings (null date < threshold), keeps all (treated as unknown), or errors out. Same question for `--author`. Specify the precedence explicitly.
- (minor) Block-comment handling is under-specified when a single block contains multiple markers (e.g. a `/* ... */` block with both `TODO:` and `FIXME:` on different lines). The spec only says 'continuation lines are attached to the marker on the opening line', which would silently drop the second marker. Define whether each marker in a block yields its own Finding.
- (minor) `.gitignore` support is described as 'a small parser' with no specified subset (negation `!`, `**`, directory-only `/`, nested `.gitignore` precedence, `.git/info/exclude`, global gitignore). Walker tests mention 'nested ignores' but not these rules. Either enumerate the supported subset or delegate to `git check-ignore` (which conflicts with 'zero runtime deps' only nominally, since git is already assumed).
- (minor) The performance gate 'runtime < 60 s on CI hardware' does not define CI hardware (GitHub-hosted runner class, self-hosted, vCPU/RAM). Without a concrete reference, this can't be enforced or diagnosed when it regresses. Pin the runner (e.g. 'GitHub-hosted `ubuntu-latest`, 4 vCPU') or drop the hard number.

## weak-testing

- (major) The postgres-ai/database-lab integration assertion — 'spot-check: at least one known long-standing TODO is present (pin by file path + marker, not by exact text which may rot)' — is still brittle: file paths themselves rot, and any legitimate TODO cleanup will fail nightly CI with no signal about regression vs. intended change. Prefer a structural assertion (e.g. `findings.length >= N` and schema validity) plus a separate, pinned synthetic fixture for 'known long-standing TODO' coverage.
- (major) The tests plan covers extractor, walker, renderers, and blame parser, but not the CLI flags that v0.1 ships: `--since`, `--author`, `--fail-on`, `--staged`, `--config`, `--no-gitignore`, `--no-blame`, and exit-code semantics (0 / 1 / 2). These are the gate-keeping behaviours Priya and Sam depend on in the user stories. Add explicit unit or black-box CLI tests per flag, including negative cases (e.g. `--fail-on FIXME` with no FIXMEs → exit 0).
- (minor) The 'strings-that-look-like-comments (negative case)' fixture implies the extractor distinguishes string literals from comments, but the Finding-extraction algorithm only describes regex-stripping to the comment region using `CommentSyntax`. No mention of string-literal tokenisation. Either the test will pass trivially (regex already avoids it) or the implementation will need a real tokenizer that isn't specified. Spell out the expected behaviour.

## missing-requirement

- (minor) The embedded changelog is present but contains only a v0.1 entry even though the spec header is v0.2. Add a v0.2 entry describing what changed between the v0.1 draft and this revision (scope tightening, new user stories, renderer schema, etc.).

## suggested-next-version

0.3

<!-- samospec:critique v1 -->
{
  "findings": [
    {
      "category": "contradiction",
      "text": "Version is inconsistent across the spec. Header says 'SPEC v0.2'; the 'Scope & non-goals (v0.1)' section is labelled v0.1; the JSON schema example pins `\"version\": \"0.1.0\"`; and the embedded changelog only contains a v0.1 entry dated 2026-04-21 with no v0.2 entry describing what changed in this revision. Pick one convention (e.g. spec version vs. product version) and make the changelog reflect the current header.",
      "severity": "major"
    },
    {
      "category": "contradiction",
      "text": "The JSON schema is self-inconsistent. Architecture → Renderer says the json renderer 'emits a versioned schema `{ $schema, tool, version, generated_at, findings: [...] }`', but the concrete example under Implementation details → JSON schema omits `$schema` and adds an undocumented `root` field. Downstream consumers promised a 'stable, documented schema' cannot rely on either shape.",
      "severity": "major"
    },
    {
      "category": "contradiction",
      "text": "The extractor regex hardcodes the marker list — `/(^|[^A-Za-z0-9_])(TODO|FIXME|HACK|XXX)\\b[:\\s-]?\\s*(.*)/` — yet the surrounding text says 'markers from config' and the CLI exposes `--markers` to change them. Either the regex must be constructed from `config.markers`, or `--markers` is a no-op. Spec should state that the pattern is built dynamically.",
      "severity": "major"
    },
    {
      "category": "ambiguity",
      "text": "`--staged` is defined two different ways. The CLI surface says it 'limit[s] to files in `git diff --cached --name-only`' (file-level), but User story 4 says it prints 'TODOs touched by the diff' (line-level — only markers inside staged hunks). These produce very different outputs on a file where only an unrelated line is staged. Pin one semantics.",
      "severity": "major"
    },
    {
      "category": "ambiguity",
      "text": "Interaction between `--since` / `--author` and missing blame is undefined. When `--no-blame` is passed, or the tree is not a git worktree, the spec says blame fields are `null`. It does not state whether `--since 2024-01-01` drops all findings (null date < threshold), keeps all (treated as unknown), or errors out. Same question for `--author`. Specify the precedence explicitly.",
      "severity": "major"
    },
    {
      "category": "weak-testing",
      "text": "The postgres-ai/database-lab integration assertion — 'spot-check: at least one known long-standing TODO is present (pin by file path + marker, not by exact text which may rot)' — is still brittle: file paths themselves rot, and any legitimate TODO cleanup will fail nightly CI with no signal about regression vs. intended change. Prefer a structural assertion (e.g. `findings.length >= N` and schema validity) plus a separate, pinned synthetic fixture for 'known long-standing TODO' coverage.",
      "severity": "major"
    },
    {
      "category": "weak-testing",
      "text": "The tests plan covers extractor, walker, renderers, and blame parser, but not the CLI flags that v0.1 ships: `--since`, `--author`, `--fail-on`, `--staged`, `--config`, `--no-gitignore`, `--no-blame`, and exit-code semantics (0 / 1 / 2). These are the gate-keeping behaviours Priya and Sam depend on in the user stories. Add explicit unit or black-box CLI tests per flag, including negative cases (e.g. `--fail-on FIXME` with no FIXMEs → exit 0).",
      "severity": "major"
    },
    {
      "category": "weak-testing",
      "text": "The 'strings-that-look-like-comments (negative case)' fixture implies the extractor distinguishes string literals from comments, but the Finding-extraction algorithm only describes regex-stripping to the comment region using `CommentSyntax`. No mention of string-literal tokenisation. Either the test will pass trivially (regex already avoids it) or the implementation will need a real tokenizer that isn't specified. Spell out the expected behaviour.",
      "severity": "minor"
    },
    {
      "category": "ambiguity",
      "text": "Block-comment handling is under-specified when a single block contains multiple markers (e.g. a `/* ... */` block with both `TODO:` and `FIXME:` on different lines). The spec only says 'continuation lines are attached to the marker on the opening line', which would silently drop the second marker. Define whether each marker in a block yields its own Finding.",
      "severity": "minor"
    },
    {
      "category": "ambiguity",
      "text": "`.gitignore` support is described as 'a small parser' with no specified subset (negation `!`, `**`, directory-only `/`, nested `.gitignore` precedence, `.git/info/exclude`, global gitignore). Walker tests mention 'nested ignores' but not these rules. Either enumerate the supported subset or delegate to `git check-ignore` (which conflicts with 'zero runtime deps' only nominally, since git is already assumed).",
      "severity": "minor"
    },
    {
      "category": "ambiguity",
      "text": "The performance gate 'runtime < 60 s on CI hardware' does not define CI hardware (GitHub-hosted runner class, self-hosted, vCPU/RAM). Without a concrete reference, this can't be enforced or diagnosed when it regresses. Pin the runner (e.g. 'GitHub-hosted `ubuntu-latest`, 4 vCPU') or drop the hard number.",
      "severity": "minor"
    },
    {
      "category": "missing-requirement",
      "text": "The embedded changelog is present but contains only a v0.1 entry even though the spec header is v0.2. Add a v0.2 entry describing what changed between the v0.1 draft and this revision (scope tightening, new user stories, renderer schema, etc.).",
      "severity": "minor"
    }
  ],
  "summary": "v0.2 covers all nine baseline sections and correctly honours the 'not a todo-list / not a CRUD tool' disclaimers. The main weaknesses are internal consistency (spec/product version, JSON schema shape, marker regex vs. config) and ambiguity around flag semantics (`--staged`, `--since`/`--author` with missing blame). Testing is strong for pure modules but thin on CLI-flag coverage, and the integration 'pinned TODO' assertion is still brittle.",
  "suggested_next_version": "0.3",
  "usage": null,
  "effort_used": "max"
}
<!-- samospec:critique end -->
