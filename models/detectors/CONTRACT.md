# IDetector Contract — `1.0.0`

This document is the source of truth for the detector interface that sibling
modules (e.g. [`bx-AISentinel-ONNX`](https://github.com/mrigsby/bx-AISentinel-ONNX))
must implement to plug into `bx-AISentinel` via the `externalDetectors` policy
setting.

Any breaking change to this contract requires a major version bump and a
matching version gate in `AiSentinelMiddleware._composeCore()`.

## Required methods

An external detector MUST expose the following methods. `IDetector.bx` provides
usable defaults; extending it is the recommended implementation path, but
duck-typed classes that expose the same surface are also accepted.

### `scan( required string text ) → array<Hit>`

Scan the provided text and return an array of zero or more `Hit` structs
(shape below). MUST NOT mutate the input. MUST NOT throw on empty or huge
input — return an empty array for zero hits, and handle large payloads by
chunking internally if needed.

### `getPriority() → numeric`

Returns the detector's priority. `AiSentinelMiddleware`'s `DetectionEngine`
resolves overlapping hits by priority — lower wins.

Reserved slots:

- `1` — registry (literal user-supplied values)
- `2` — regex (catalog-driven pattern matching)
- `3` — entropy (Shannon-entropy fallback)

External detectors SHOULD claim a non-reserved numeric. NER-based detectors
typically slot at `1.5` (between registry and regex) so a user's explicit
registered values still win, but contextual entity recognition beats regex
when ranges overlap.

### `getSourceName() → string`

A short, stable identifier used in audit logs and hit provenance
(e.g. `"OnnxNerDetector"`). Human-readable; case-sensitive equality is used in
some reports, so don't change it across versions.

### `getContractVersion() → string`

Semantic version string declaring which IDetector contract version the
detector implements. `AiSentinelMiddleware` refuses to load an external
detector whose major version does not match its own supported contract
version.

Current contract: `"1.0.0"`.

## Hit struct shape

Every entry returned from `scan()` MUST be a struct with exactly these keys:

```text
{
    start      : numeric   // 1-based inclusive start offset in the scanned text
    end        : numeric   // 1-based inclusive end offset
    value      : string    // the matched plaintext substring
    label      : string    // UPPER_SNAKE category tag (e.g. PII_PERSON_NAME)
    category   : string    // grouping string (e.g. "pii-ner", "cloud-keys")
    priority   : numeric   // detector's priority, copied onto the hit
    confidence : numeric   // 0.0 – 1.0 (detectors may default to 1.0)
    source     : string    // detector's `getSourceName()`, copied onto the hit
}
```

Use `IDetector.makeHit(...)` to build conforming structs without having to
remember the shape.

## Label and category conventions

- Labels are `UPPER_SNAKE_CASE` and describe what the redacted value *is*
  (e.g. `PII_PERSON_NAME`, `PII_ADDRESS`, `MEDICAL_RECORD_NUMBER`). They
  surface in the minted token (`⟦SECRET:LABEL:hmac8⟧`) that the LLM sees, so
  the label doubles as a type hint to the model.
- Categories are lowercase kebab-case and group related labels for the
  `categoriesEnabled` policy gate (e.g. `"pii"`, `"pii-ner"`, `"cloud-keys"`).

## Load-failure behavior

If an external detector fails to resolve (missing WireBox mapping, contract
version mismatch, constructor throws), `AiSentinelMiddleware` will:

1. Log the failure via `SentinelAuditor.warnDetectorError(...)` with
   structured context.
2. Fire the `onSentinelDetectorLoadFailure` interception point with
   `{ detectorName, reason, attemptedContractVersion, supportedContractVersion, failureMode }`.
3. Mark the detector as `"degraded"` in `sentinel.getLoadReport()`.
4. Continue operating with the remaining detectors — the sentinel does not
   hard-fail on a single broken plugin.

Host applications that want stricter handling (e.g. refuse to start if any
detector fails) subscribe an interceptor and escalate from there.

## Minimal example

```javascript
// In a sibling module's models/
class extends="bxAiSentinel.models.detectors.IDetector" {

    function init() {
        variables.priority        = 2.5;
        variables.sourceName      = "MyCustomDetector";
        variables.contractVersion = "1.0.0";
        return this;
    }

    array function scan( required string text ) {
        var hits = [];
        // ... detection logic ...
        hits.append( makeHit(
            start      : 10,
            end        : 22,
            value      : "some-match",
            label      : "CUSTOM_TOKEN",
            category   : "custom",
            confidence : 0.95
        ) );
        return hits;
    }

}
```

## Version history

- **1.0.0** — Initial contract. Methods: `scan`, `getPriority`, `getSourceName`,
  `getContractVersion`. Hit struct shape as above.
