# Test Coverage TODOs

Captured from a test-coverage audit on 2026-04-17. These are gaps the existing
specs do not reach. Ranked by real behavioral risk if the gap stays open.
Addressed items move into spec files under [`specs/`](specs/).

## 1. JSON load robustness (PatternCatalog, SentinelPolicy) — HIGH

A corrupted `includes/patterns/*.json` or a malformed `boxlang.json` is
silently dropped. Host apps get no audit signal that redaction has been
degraded.

- Malformed JSON in a bundled pattern file → `jsonDeserialize` throws →
  current behavior unverified.
- `boxlang.json` exists but is syntactically broken → caught silently inside
  [`SentinelPolicy._readBoxlangJson`](../models/policy/SentinelPolicy.bx),
  defaults apply with no warning.
- `.sentinel.json` pointing at a non-existent or unreadable path.

**What to test once decided:** Either (a) auditor emits a structured warning
for each silent-skip, OR (b) the method propagates a named exception.
Product decision needed before writing specs — the tests should enforce
whichever we pick.

## 2. `_restoreResponseShape` provider variance — HIGH

[`AiSentinelMiddleware._restoreResponseShape`](../models/AiSentinelMiddleware.bx)
covers three response shapes: a plain string, a struct with `.content`, and a
struct with `.choices[].message.content`. A non-standard shape means tokens
leak through unrestored to the caller.

Cases with no coverage today:

- `context.result` is `null` / `javaCast("null","")`.
- `choices` is an empty array.
- `.content` exists but is a struct or array (not a simple value).
- `choices[].message.content` is an array (multimodal message parts).
- Response is a top-level array of choice objects (some providers).
- Deeply nested tool-call payloads inside `.choices[].message`.

## 3. Manifest unbounded growth (conversation scope) — HIGH

[`PolicyIntegrationSpec`](specs/integration/PolicyIntegrationSpec.bx) verifies
that `manifestScope = "conversation"` preserves the sessionKey across runs,
but no spec checks whether the manifest itself grows without bound.

- 100+ sequential runs under conversation scope → does
  [`TokenManifest`](../models/TokenManifest.bx) get unbounded-large?
- Large manifest restoration performance floor (does it stay sub-10ms at
  1k entries?).
- Mid-conversation scope switch (`"run"` → `"conversation"` or vice versa):
  does the prior manifest get zeroed correctly?
- Concurrent tool calls touching the same manifest — the struct is not
  explicitly thread-safe.

## 4. Policy precedence layer interactions — MEDIUM-HIGH

`SentinelPolicySpec` exercises each layer (defaults, `boxlang.json`,
`.sentinel.json`, constructor) in isolation, but multi-layer merging has
edge cases the specs skip:

- `registeredSecrets` across all four layers — append or replace at each
  boundary?
- Layer 2 sets `categoriesEnabled: []` (empty) — does the empty array wipe
  layer 1 defaults, or does the merge treat empty-array as "no opinion"?
- `customPatterns` from multiple layers — union or override?
- `.sentinel.json` referencing a pattern catalog that fails to load.

## 5. Detector regex failures (silent crashes) — MEDIUM

[`RegexDetector.scan()`](../models/detectors/RegexDetector.bx) wraps
`pattern.regex` in `reFind` with no try/catch. One poisoned pattern (invalid
syntax, catastrophic backtracking on a large payload) takes the entire
middleware down mid-request.

- Pattern with unclosed bracket class (`[A-`) → currently propagates.
- Catastrophic-backtracking regex against a multi-KB payload.
- Consider wrapping each pattern in a try/catch that reports via
  [`SentinelAuditor.warnDetectorError`](../models/SentinelAuditor.bx) — then
  add the spec that verifies the poisoned pattern is skipped instead of
  crashing.

## Ranking summary

| # | Gap | Risk if unaddressed |
|---|-----|---------------------|
| 1 | JSON load robustness | Silent config degradation — weaker detection with no signal |
| 2 | `_restoreResponseShape` variance | Plaintext tokens leak to caller on non-standard provider shapes |
| 3 | Manifest unbounded growth | Memory leak across long conversations |
| 4 | Policy layer merging | Multi-team / multi-env configs apply unexpected rule sets |
| 5 | Detector regex crashes | A bad bundled pattern kills the entire middleware |

## When to come back to this list

Don't pre-emptively burn cycles on these unless one of the following
triggers fires:

- A related bug surfaces in the field (a provider response the middleware
  doesn't handle, a customer ships a broken `.sentinel.json`, etc.).
- Policy or middleware logic is being materially changed — tighten the net
  before the rewrite lands.
- Preparing for a v1.0 release — walk the list and close everything.
