# Changelog

All notable changes to `bx-AISentinel` are documented in this file.

Format follows [Keep a Changelog](https://keepachangelog.com/en/1.1.0/). Versioning follows [Semantic Versioning](https://semver.org/).

## [Unreleased]

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
