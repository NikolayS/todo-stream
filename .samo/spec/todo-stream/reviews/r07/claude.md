# Reviewer B — Claude

## summary

Spec v0.7 is rigorous and addresses the r6 security and determinism feedback thoroughly. Remaining blockers are primarily in the extractor's comment-region semantics: a direct contradiction between the scope example ('single TODO finding' when a marker appears inside a string within a comment) and the stated algorithm (no string-literal tokenization, global regex); unspecified text-capture behavior when multiple markers share a line; and a latent false-positive for `# TODO` inside Python triple-quoted multi-line strings. A cluster of smaller ambiguities covers `--max-findings` stdout/exit-code interactions, empty-marker-line continuation edge cases, block-comment state detection, and missing concrete values (pin SHAs, perf baseline, schema host). All nine mandatory baseline sections are present.

## contradiction

- (major) Scope section claims: 'A TODO inside a string within a comment (e.g. `// TODO: see "FIXME" in docs`) remains a single TODO finding.' This directly contradicts the extractor algorithm. The extraction algorithm explicitly states the regex runs globally over the comment region without tokenizing string literals within comments. Since FIXME is a default marker, the global regex WILL also match FIXME inside the quoted string, producing two Findings — not one. This is also contradicted by the unit-test bullet 'Single line with two markers → two Findings, distinct column, ordered ascending.' Either the Scope example must be corrected (to say 'produces TODO + FIXME findings, since string-within-comment is not tokenized'), or the algorithm must perform string-literal suppression inside comment regions (with a new fixture to pin it). As stated, the spec promises behavior it cannot deliver.

## ambiguity

- (major) Multi-marker same-line text capture is unspecified. The dynamic regex `...[:\s-]?\s*(.*)` greedily captures to end-of-line, so for `// TODO: a FIXME: b`, the TODO Finding's `text` field will be `'a FIXME: b'` and FIXME's will be `'b'` — the earlier marker swallows the later one's descriptive text. The renderer-goldens bullet 'Two findings on same (file, line) from two markers — distinct column' pins columns but is silent on text content. Pin whether `text` for a marker on a multi-marker line is (a) up to the next marker occurrence, (b) to end-of-line (current behavior), or (c) something else. Add a golden fixture to lock it.
- (minor) `--max-findings` cap stdout behavior is unpinned. Spec says 'a single stderr warning is emitted … The process exits 2' but — unlike `--max-output-bytes`, which explicitly states 'writes nothing to stdout' — `--max-findings` does not say whether the truncated report is still written to stdout or suppressed. Tests bullet 'fixture triggering --max-findings → exit 2, stderr warning, no stdout write' implies no-write, but the implementation-details section doesn't match. Pin explicitly in one place.
- (minor) Exit-code precedence between `--max-findings` (exit 2) and `--fail-on` (exit 1) is undefined. If a repo has FIXMEs that would trigger `--fail-on FIXME` but the `--max-findings` cap is also hit, the spec does not state which exit code wins. Also: if the cap drops FIXMEs before `--fail-on` evaluation, a fail-on condition might be hidden. Add a rule (e.g., 'cap violations exit 2 regardless of fail-on state; fail-on is never evaluated when cap is tripped') and a test fixture.
- (minor) Empty-marker-line special case is underspecified for multiple leading empty continuations. The rule says 'If the marker-line's text is empty after stripping … the concatenation omits the leading \n and begins with the first continuation segment.' But for `/* TODO:\n *\n * body */` (empty marker line + empty first continuation + non-empty body), is the result `'body'` (leading empties collapsed), `'\nbody'` (only the marker-line's leading \n dropped), or `'\n\nbody'` (only the very first \n dropped, subsequent preserved)? Pin one interpretation and add a fixture; worked example in spec covers only the single-leading case.
- (minor) `TODO_STREAM_NOW` env-var determinism seam has no validation spec. Is a malformed value ignored (fallback to `new Date()`), rejected at startup (exit 2), or passed through verbatim into the output (risking malformed `generated_at`)? Pin one rule; tests that rely on this seam can then assume the validation contract.
- (minor) Column indexing implementation is underspecified. The regex runs on bytes-as-Latin-1 and produces a byte offset, but `column` is defined in UTF-8 code points. No fixture enumerated for multi-byte-comment column indexing (e.g., a comment containing `// — TODO` where `—` is a 3-byte UTF-8 sequence). Add one, and state in the algorithm that the byte-offset-to-code-point conversion uses a fatal-safe UTF-8 decode of the line prefix.
- (minor) `git check-ignore --stdin -z -v` delegation is mentioned but the invocation pattern (one call fed all candidate paths via stdin? batched? per-path?) is not specified. For a 100k-file tree this matters for both performance and ARG_MAX-equivalent stdin-buffer behavior. Pin the pattern (stdin streaming is implied by `--stdin`; say so explicitly and describe how results are correlated back to candidate paths).

## weak-testing

- (major) The Python `#` comment dispatch has a latent false-positive mode not covered by tests or Scope. Inside a Python triple-quoted string that spans multiple physical lines, any line beginning with `# TODO` will be detected as a comment (since the extractor processes line-by-line against the language's CommentSyntax, not a multi-line string-state machine), producing a spurious Finding even though the spec explicitly says triple-quoted strings are NOT comments. The only Python fixture enumerated is `"""TODO"""` (triple-quote on one line, no `#`), which does not exercise this case. Add a fixture with a multi-line triple-quoted string containing `# TODO:` on an interior line and pin the expected behavior (either: documented limitation accepted, or: suppress).
- (minor) Block-comment state detection (identifying which lines belong to an open `/* … */` span) is presupposed but never specified as an algorithm. How are these cases handled: (a) `/*` opened and never closed before EOF; (b) `/*` appearing inside a string literal on the same line as real code (the reducer doesn't tokenize strings — would it open a spurious block comment?); (c) nested block comments in Rust (`/* a /* b */ c */` — Rust supports nesting, C does not); (d) block comment opened on line with code preceding it (`int x=5; /* TODO */`). Add unit-test fixtures for each and pin the intended reducer semantics.
- (minor) Line-to-comment-region reduction is not formalized for mixed code+comment lines or multi-line block comments. The Extractor section says 'first reduces the line to its comment region using the language's CommentSyntax' without stating the algorithm. Readers cannot independently verify behavior on cases like `int x = 5; // TODO: fix` or `/* start */ int x; // TODO`. Given the spec's strong emphasis on byte-level determinism, this reducer deserves its own algorithm block plus fixtures.
- (minor) Perf lane threshold (90 s, best-of-3, 4 vCPU/16 GB) has no measured baseline. No sprint deliverable establishes what the tool's actual postgres-subtree runtime is, so the 90 s ceiling is arbitrary — it might pass trivially or fail on day one. Add a sprint 1 or sprint 4 deliverable: 'publish initial perf run, set ceiling at 1.5× measured p95 or 90 s, whichever is larger.' Otherwise this gate provides no meaningful signal.

## missing-requirement

- (minor) Integration tests reference `PG_PIN_SHA` and `DBLAB_PIN_SHA` as placeholders but no concrete commit SHAs appear anywhere in the spec or in the curated allow-list. Since the allow-list is pinned-at-SHA and invariants depend on content reachable at that SHA, the placeholders block integration-harness implementation. Either pin the SHAs in the spec, or add an explicit sprint 1 deliverable to select and commit them before sprint 4 gates on them.
- (minor) JSON schema URL `https://todo-stream.dev/schema/v1.json` is declared as a stable public surface but no sprint deliverable specifies domain registration, hosting, or content-negotiation. Release engineer's sprint 4 line 'publish hosted $schema' assumes the domain is already controlled. Add a sprint 1 prerequisite (acquire domain + host static schema doc) or reference an existing-owned host.

## suggested-next-version

v0.8

<!-- samospec:critique v1 -->
{
  "findings": [
    {
      "category": "contradiction",
      "text": "Scope section claims: 'A TODO inside a string within a comment (e.g. `// TODO: see \"FIXME\" in docs`) remains a single TODO finding.' This directly contradicts the extractor algorithm. The extraction algorithm explicitly states the regex runs globally over the comment region without tokenizing string literals within comments. Since FIXME is a default marker, the global regex WILL also match FIXME inside the quoted string, producing two Findings — not one. This is also contradicted by the unit-test bullet 'Single line with two markers → two Findings, distinct column, ordered ascending.' Either the Scope example must be corrected (to say 'produces TODO + FIXME findings, since string-within-comment is not tokenized'), or the algorithm must perform string-literal suppression inside comment regions (with a new fixture to pin it). As stated, the spec promises behavior it cannot deliver.",
      "severity": "major"
    },
    {
      "category": "ambiguity",
      "text": "Multi-marker same-line text capture is unspecified. The dynamic regex `...[:\\s-]?\\s*(.*)` greedily captures to end-of-line, so for `// TODO: a FIXME: b`, the TODO Finding's `text` field will be `'a FIXME: b'` and FIXME's will be `'b'` — the earlier marker swallows the later one's descriptive text. The renderer-goldens bullet 'Two findings on same (file, line) from two markers — distinct column' pins columns but is silent on text content. Pin whether `text` for a marker on a multi-marker line is (a) up to the next marker occurrence, (b) to end-of-line (current behavior), or (c) something else. Add a golden fixture to lock it.",
      "severity": "major"
    },
    {
      "category": "weak-testing",
      "text": "The Python `#` comment dispatch has a latent false-positive mode not covered by tests or Scope. Inside a Python triple-quoted string that spans multiple physical lines, any line beginning with `# TODO` will be detected as a comment (since the extractor processes line-by-line against the language's CommentSyntax, not a multi-line string-state machine), producing a spurious Finding even though the spec explicitly says triple-quoted strings are NOT comments. The only Python fixture enumerated is `\"\"\"TODO\"\"\"` (triple-quote on one line, no `#`), which does not exercise this case. Add a fixture with a multi-line triple-quoted string containing `# TODO:` on an interior line and pin the expected behavior (either: documented limitation accepted, or: suppress).",
      "severity": "major"
    },
    {
      "category": "ambiguity",
      "text": "`--max-findings` cap stdout behavior is unpinned. Spec says 'a single stderr warning is emitted … The process exits 2' but — unlike `--max-output-bytes`, which explicitly states 'writes nothing to stdout' — `--max-findings` does not say whether the truncated report is still written to stdout or suppressed. Tests bullet 'fixture triggering --max-findings → exit 2, stderr warning, no stdout write' implies no-write, but the implementation-details section doesn't match. Pin explicitly in one place.",
      "severity": "minor"
    },
    {
      "category": "ambiguity",
      "text": "Exit-code precedence between `--max-findings` (exit 2) and `--fail-on` (exit 1) is undefined. If a repo has FIXMEs that would trigger `--fail-on FIXME` but the `--max-findings` cap is also hit, the spec does not state which exit code wins. Also: if the cap drops FIXMEs before `--fail-on` evaluation, a fail-on condition might be hidden. Add a rule (e.g., 'cap violations exit 2 regardless of fail-on state; fail-on is never evaluated when cap is tripped') and a test fixture.",
      "severity": "minor"
    },
    {
      "category": "ambiguity",
      "text": "Empty-marker-line special case is underspecified for multiple leading empty continuations. The rule says 'If the marker-line's text is empty after stripping … the concatenation omits the leading \\n and begins with the first continuation segment.' But for `/* TODO:\\n *\\n * body */` (empty marker line + empty first continuation + non-empty body), is the result `'body'` (leading empties collapsed), `'\\nbody'` (only the marker-line's leading \\n dropped), or `'\\n\\nbody'` (only the very first \\n dropped, subsequent preserved)? Pin one interpretation and add a fixture; worked example in spec covers only the single-leading case.",
      "severity": "minor"
    },
    {
      "category": "weak-testing",
      "text": "Block-comment state detection (identifying which lines belong to an open `/* … */` span) is presupposed but never specified as an algorithm. How are these cases handled: (a) `/*` opened and never closed before EOF; (b) `/*` appearing inside a string literal on the same line as real code (the reducer doesn't tokenize strings — would it open a spurious block comment?); (c) nested block comments in Rust (`/* a /* b */ c */` — Rust supports nesting, C does not); (d) block comment opened on line with code preceding it (`int x=5; /* TODO */`). Add unit-test fixtures for each and pin the intended reducer semantics.",
      "severity": "minor"
    },
    {
      "category": "weak-testing",
      "text": "Line-to-comment-region reduction is not formalized for mixed code+comment lines or multi-line block comments. The Extractor section says 'first reduces the line to its comment region using the language's CommentSyntax' without stating the algorithm. Readers cannot independently verify behavior on cases like `int x = 5; // TODO: fix` or `/* start */ int x; // TODO`. Given the spec's strong emphasis on byte-level determinism, this reducer deserves its own algorithm block plus fixtures.",
      "severity": "minor"
    },
    {
      "category": "weak-testing",
      "text": "Perf lane threshold (90 s, best-of-3, 4 vCPU/16 GB) has no measured baseline. No sprint deliverable establishes what the tool's actual postgres-subtree runtime is, so the 90 s ceiling is arbitrary — it might pass trivially or fail on day one. Add a sprint 1 or sprint 4 deliverable: 'publish initial perf run, set ceiling at 1.5× measured p95 or 90 s, whichever is larger.' Otherwise this gate provides no meaningful signal.",
      "severity": "minor"
    },
    {
      "category": "missing-requirement",
      "text": "Integration tests reference `PG_PIN_SHA` and `DBLAB_PIN_SHA` as placeholders but no concrete commit SHAs appear anywhere in the spec or in the curated allow-list. Since the allow-list is pinned-at-SHA and invariants depend on content reachable at that SHA, the placeholders block integration-harness implementation. Either pin the SHAs in the spec, or add an explicit sprint 1 deliverable to select and commit them before sprint 4 gates on them.",
      "severity": "minor"
    },
    {
      "category": "ambiguity",
      "text": "`TODO_STREAM_NOW` env-var determinism seam has no validation spec. Is a malformed value ignored (fallback to `new Date()`), rejected at startup (exit 2), or passed through verbatim into the output (risking malformed `generated_at`)? Pin one rule; tests that rely on this seam can then assume the validation contract.",
      "severity": "minor"
    },
    {
      "category": "ambiguity",
      "text": "Column indexing implementation is underspecified. The regex runs on bytes-as-Latin-1 and produces a byte offset, but `column` is defined in UTF-8 code points. No fixture enumerated for multi-byte-comment column indexing (e.g., a comment containing `// — TODO` where `—` is a 3-byte UTF-8 sequence). Add one, and state in the algorithm that the byte-offset-to-code-point conversion uses a fatal-safe UTF-8 decode of the line prefix.",
      "severity": "minor"
    },
    {
      "category": "ambiguity",
      "text": "`git check-ignore --stdin -z -v` delegation is mentioned but the invocation pattern (one call fed all candidate paths via stdin? batched? per-path?) is not specified. For a 100k-file tree this matters for both performance and ARG_MAX-equivalent stdin-buffer behavior. Pin the pattern (stdin streaming is implied by `--stdin`; say so explicitly and describe how results are correlated back to candidate paths).",
      "severity": "minor"
    },
    {
      "category": "missing-requirement",
      "text": "JSON schema URL `https://todo-stream.dev/schema/v1.json` is declared as a stable public surface but no sprint deliverable specifies domain registration, hosting, or content-negotiation. Release engineer's sprint 4 line 'publish hosted $schema' assumes the domain is already controlled. Add a sprint 1 prerequisite (acquire domain + host static schema doc) or reference an existing-owned host.",
      "severity": "minor"
    }
  ],
  "summary": "Spec v0.7 is rigorous and addresses the r6 security and determinism feedback thoroughly. Remaining blockers are primarily in the extractor's comment-region semantics: a direct contradiction between the scope example ('single TODO finding' when a marker appears inside a string within a comment) and the stated algorithm (no string-literal tokenization, global regex); unspecified text-capture behavior when multiple markers share a line; and a latent false-positive for `# TODO` inside Python triple-quoted multi-line strings. A cluster of smaller ambiguities covers `--max-findings` stdout/exit-code interactions, empty-marker-line continuation edge cases, block-comment state detection, and missing concrete values (pin SHAs, perf baseline, schema host). All nine mandatory baseline sections are present.",
  "suggested_next_version": "v0.8",
  "usage": null,
  "effort_used": "max"
}
<!-- samospec:critique end -->
