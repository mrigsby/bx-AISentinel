# Changelog

All notable changes to `bx-AISentinel` are documented in this file.

Format follows [Keep a Changelog](https://keepachangelog.com/en/1.1.0/). Versioning follows [Semantic Versioning](https://semver.org/).

## [Unreleased]

## [0.3.1] - 2026-04-19

Real-world host-app fix discovered during the first end-user test of v0.3.0 against a fresh demo install.

### Fixed

- **`includes/token-protocol.md` was being stripped from `box install` packages** because the `box.json` `ignore` field had `"*.md"`, which CommandBox interprets recursively (matches all markdown files at any depth). Result: every host installing the module saw `_loadTokenProtocol()` fall through to its hardcoded fallback string instead of the rich rule-set we ship in the file. The v0.3.0 coaching rewrite (use-vs-discuss split for narrative prose) NEVER reached the LLM in any deployed install. Fixed by dropping `"*.md"` from `ignore` — the README, CHANGELOG, and `includes/token-protocol.md` now ship with every install. Adds ~30 KB to the package; no functional cost.
- README + `SentinelPolicy.bx` JavaDoc examples for `externalDetectors` corrected (carried over from the [Unreleased] entry that was rolled into this release). Examples previously showed `"OnnxNerDetector@bx-AISentinel-ONNX"` — the dashed form, which doesn't actually resolve in WireBox (the parser uses `this.cfmapping`, not `this.name`, and dashes break it). Now use `"OnnxNerDetector@bxAISentinelONNX"`. See `bx-AISentinel-ONNX/CHANGELOG.md` v0.4.1-pre for context.

### Compatibility

Pure file-set fix. Existing v0.3.0 hosts will need to re-`box install` to pull the missing `token-protocol.md` (or the LLM continues to receive the fallback prompt). No code or API changes.

## [0.3.0] - 2026-04-19

Closes a UX gap where the LLM, taught to preserve `⟦SECRET:LABEL:hmac8⟧` placeholders verbatim, would quote them in narrative prose ("the placeholder ⟦SECRET:DATASOURCE_PASSWORD:abc⟧ has been redacted"). After local restoration the user reads "the placeholder sekret-prod-1 has been redacted" — confusing and undermines the protection story. Two coordinated changes address this.

### Added

- **`appendRestorationNotice` policy setting** (default `true`). When enabled and any tokens were minted on the turn, the middleware appends a brief markdown notice to the user-visible content explaining that values were tokenized for the LLM and restored locally before display. Renders cleanly in any UI that uses the standard `markdown()` BIF. Hosts that prefer clean output (no chrome) set the setting to `false`. Mirrored as a typed getter `SentinelPolicy.isRestorationNoticeEnabled()`.
- New integration specs in `AiSentinelMiddlewareSpec.bx` covering: notice appended when tokens minted (default), notice NOT appended when no tokens minted, notice suppressed when `appendRestorationNotice: false`.

### Changed

- **Token-protocol coaching prompt rewritten** ([`includes/token-protocol.md`](includes/token-protocol.md)) to distinguish two contexts the LLM must handle differently:
  - **Generated code / commands / config** — emit the placeholder verbatim so it gets restored to the real value (existing behavior).
  - **Narrative discussion of the redacted value** — refer to it BY ITS TYPE (e.g. "the datasource password field"), never quote or echo the placeholder text. Quoting in prose creates the gaslighting case described above.
  Adds a "quick test" the LLM can run on its own sentences before sending: "if my placeholder is replaced with the actual secret right here, does this sentence still make sense?"
- `_restoreResponseShape()` now returns the restored content with the optional notice concatenated (or with no change when the notice is disabled / no tokens were minted). Behavior change visible to hosts that read assistant `content` strings — `toInclude(...)` assertions remain valid; `toBe(exactString)` assertions on responses with minted tokens will need updating.

### Compatibility

The notice is opt-out via a single setting. Set `appendRestorationNotice: false` in `boxlang.json → modules.bx-aisentinel.settings` to restore exact v0.2.x output. The HMAC token format, restoration semantics, metrics shape, and detector contract are all unchanged.

## [0.2.1] - 2026-04-18

### Fixed

- **SentinelPolicy looked for the wrong config filename.** `_defaultBoxlangJsonPath()` was returning `/boxlang.json` but BoxLang's canonical filename is `.boxlang.json` (the dotfile — that's what `server.json` references as `engineConfigFile` and what BoxLang natively reads). As a result, any configuration written to `.boxlang.json` (including `externalDetectors` / `registeredSecrets` / policy overrides) was silently ignored. Policy now prefers `.boxlang.json` and falls back to `boxlang.json` for projects using the non-dot variant.

## [0.2.0] - 2026-04-18

Adds the external-detector plugin seam so sibling modules can contribute detectors into the pipeline without modifying core code. This is the foundation for [`bx-AISentinel-ONNX`](https://github.com/mrigsby/bx-AISentinel-ONNX) (Tier 1 NER) and any future domain-specific detectors.

### Added

- `IDetector` contract formalized at version `1.0.0`:
  - New `getContractVersion()` method on `IDetector.bx` (default `"1.0.0"`).
  - New `models/detectors/CONTRACT.md` documenting the required surface, Hit struct shape, label / category conventions, and load-failure behavior. External detector authors build against this.
- Two new `SentinelPolicy` settings:
  - `externalDetectors` (array) — WireBox IDs or fully-qualified class paths of sibling-module detectors to resolve into the pipeline. Appended across config layers.
  - `externalDetectorOptions` (struct) — per-detector init args, keyed by resolution ID.
- `AiSentinelMiddleware` extensions:
  - External-detector resolution inside `_composeCore()` with soft-fail semantics (a single broken plugin never takes the sentinel down).
  - Contract-version compatibility gate — major-version mismatch rejects the detector at load time.
  - Duck-type surface check — rejects classes missing `scan` / `getPriority` / `getSourceName`.
  - Public `getLoadReport()` returning `{ compatibleContractVersion, degraded, detectors: [ { name, status, reason? } ] }` for host health endpoints.
- New `onSentinelDetectorLoadFailure` ColdBox interception point. Fires with `{ detectorName, reason, attemptedContractVersion, supportedContractVersion, failureMode }` so host apps can wire their own escalation (Slack, PagerDuty, refuse-to-start, etc.). Declared in `ModuleConfig.bx`.
- `SentinelAuditor.warnDetectorError()` now also called on external-detector load failures with structured context.
- TestBox integration spec `ExternalDetectorsSpec.bx` covering default-empty, happy-path resolution, options pass-through, contract-version mismatch, incomplete-surface rejection, unresolvable-class path, and mixed-outcome load-report ordering.
- Three test fixtures under `tests/specs/integration/fixtures/`: `FixtureExternalDetector` (happy path), `FixtureIncompatibleDetector` (major-version mismatch), `FixtureIncompleteDetector` (missing required methods).
- README "External detectors" section covering wiring, load-report inspection, failure-handling / interceptor pattern, and contract-versioning semantics.

### Compatibility

No breaking changes. Default `externalDetectors = []` means v0.1.x behavior is preserved byte-for-byte when hosts don't opt in.

## [0.1.0] - 2026-04-17

Initial pre-release.

### Added

- `Tokenizer` with Unicode-bracket (`⟦ ⟧`) + truncated-HMAC-SHA256 token format `⟦SECRET:LABEL:hmac8⟧`.
- `TokenManifest` with `char[]`-backed storage and explicit zero-on-release (`zeroAll()`, `remove()`).
- `DetectionEngine` with priority + overlap-based conflict resolution.
- Three Tier 0 detectors:
  - `RegistryDetector` (priority 1) — literal case-sensitive match of user-registered secrets.
  - `RegexDetector` (priority 2) — catalog-driven, with built-in Luhn validator for credit-card patterns.
  - `EntropyDetector` (priority 3) — Shannon entropy + mixed-charset gate for unknown token shapes.
- `PatternCatalog` loading bundled JSON packs:
  - `cloud-keys.json` — AWS access / STS / GCP service account / Azure storage.
  - `vendor-tokens.json` — GitHub (PAT / fine-grained / OAuth / app), GitLab, Stripe, OpenAI, Anthropic, OpenRouter, Slack, Twilio, SendGrid, Square.
  - `generic-credentials.json` — JWT, PEM private keys, Postgres / MySQL / MongoDB URIs, HTTP basic-auth URLs.
  - `pii.json` — US SSN, credit card (Luhn-validated), email, US phone.
  - `boxlang.json` — datasource passwords, CommandBox / ForgeBox tokens, CFML-style password assignments.
- `AiSentinelMiddleware` with all six `bx-ai` hooks wired:
  - `beforeAgentRun` / `afterAgentRun` — manifest lifecycle + audit emission.
  - `beforeLLMCall` / `afterLLMCall` — message redaction / response restoration + `outboundMs` / `inboundMs` timers.
  - `beforeToolCall` / `afterToolCall` — recursive struct-walk redaction of `toolArgs` / restoration of `result` + `toolRedactMs` / `toolRestoreMs` timers.
- `SentinelCore` — redact + restore engine, extracted from the middleware shell for testability.
- `SentinelAuditor` — LogBox wrapper that emits run summaries and unsigned-token warnings (never plaintext).
- `SentinelPolicy` — four-layer config resolver: defaults → `boxlang.json` (`bxSentinel` / `modules.bx-aisentinel`) → project-root `.sentinel.json` → constructor overrides. `registeredSecrets` + `customPatterns` append across layers; all other keys override-replace.
- Runtime `setEnabled(bool)` / `isEnabled()` toggle for instant middleware activation/deactivation.
- Public inspection API: `getLastRunMetrics()`, `getPolicy()`, `registerSecret()`.
- `PatternCatalog.filterByCategories()` to drop disabled categories at the catalog layer.
- TestBox unit specs: `Tokenizer`, `TokenManifest`, `RegistryDetector`, `RegexDetector` (including Luhn), `EntropyDetector`, `DetectionEngine`, `SentinelPolicy`.
- TestBox integration specs: `AiSentinelMiddlewareSpec` (full round-trip, metrics, unsigned-token handling, response-shape coverage), `ToolCallSpec` (recursive walker, deep nesting, round-trip), `PolicyIntegrationSpec` (policy knobs shape middleware behavior).
- Demo application under `../DEMO-APPLICATION/`: ColdBox 8 + CBWire on BoxLang MiniServer, targeting OpenRouter's `openrouter/elephant-alpha` model; includes timing-badge UI, inspector, session-scoped Sentinel dashboard.
- Documentation under `../DOCUMENTATION/`: plan + progress tracker, architecture, detection catalog, token format, threat model, metrics, future-work roadmap.

### Known limitations

- Not streaming-aware. Tokens that straddle two delta chunks may leak.
- No OS-keychain / Vault / 1Password integration — registered secrets live in-process.
- Regex-only PII detection — names / addresses / medical IDs require NER, deferred to Tier 1.
- Test runner has not been executed against a live BoxLang + TestBox runtime in this release cycle; specs will run when the demo app bootstraps TestBox. Unit specs are authored against stable APIs and are expected to pass as-is.
- JVM memory zeroing is best-effort. See `DOCUMENTATION/05-threat-model.md` for the full caveats.
