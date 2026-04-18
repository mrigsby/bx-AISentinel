# bx-AISentinel

> A BoxLang AI middleware that tokenizes secrets, credentials, and PII in outbound LLM prompts and restores them on the inbound response. Drop-in, provider-agnostic, with per-call timing metrics.

**Status:** v0.2.0 · pre-release · [Changelog](CHANGELOG.md)

## What it does

- **Before** every `bx-ai` LLM call, scans messages + tool-call args for sensitive content (regex catalog, Shannon entropy, user-registered secrets) and replaces hits with reversible, HMAC-signed tokens of the form `⟦SECRET:LABEL:hmac8⟧`.
- **After** the LLM responds, scans the response for its own tokens, verifies signatures, and restores the plaintext before the caller sees it.
- **Records** category counts + wall-clock timings via LogBox. Never logs plaintext.

## Why it matters

Developers routinely paste sensitive material into AI agents without meaning to: stack traces with DB passwords, `.env` snippets, customer records, datasource blocks, API keys in log excerpts. Each of those ends up on a cloud provider's disk, in their logs, and potentially in their future training data. For HIPAA / PCI / SOC 2 / GDPR shops, that is a blocker to AI adoption.

bx-AISentinel reduces this exposure to one line of code per agent.

## Install

```sh
box install bx-aisentinel
```

BoxLang 1.12+ and [`bx-ai`](https://ai.ortusbooks.com/) are required.

## Quick start

```javascript
var sentinel = new bxAiSentinel.models.AiSentinelMiddleware();

var memory = aiMemory(
    memory         : "window",
    conversationId : "my-chat",
    config         : { "maxMessages": 10 }
);

var agent = aiAgent(
    name         : "assistant",
    instructions : "You are a helpful assistant.",
    memory       : memory,
    middleware   : sentinel
);

var reply = agent.run(
    "My AWS key is AKIAIOSFODNN7EXAMPLE. What is the cli command to rotate it?"
);
// The LLM saw only: "…AWS key is ⟦SECRET:AWS_ACCESS_KEY:a1b2c3d4⟧…"
// The returned `reply` has the plaintext restored.

var metrics = sentinel.getLastRunMetrics();
// → { outboundMs, inboundMs, toolRedactMs, toolRestoreMs, tokensMinted, categories, llmRoundtripMs }
```

Provider / apiKey / default model live in [`boxlang.json`](https://boxlang.io) under `modules.bxai.settings`; a single config change swaps the backend, no code edit.

## How it works

```text
┌──────────────────────────────────────────────────────────────────────┐
│ Caller: agent.run( "my AWS key is AKIA... what is the cli command    │
│                     to run to rotate it?" )                          │
└──────────────────────────────────────────────────────────────────────┘
                               │
                               ▼
          beforeAgentRun (AiSentinelMiddleware)
             • prime manifest for this run
                               │
                               ▼
          beforeLLMCall (AiSentinelMiddleware)
             • scan every message.content
             • mint ⟦SECRET:LABEL:hmac8⟧ per hit
             • store plaintext (char[]) in TokenManifest
             • replace in place
                               │
                               ▼
          (LLM sees tokenized text only, e.g. "⟦SECRET:AWS_ACCESS_KEY:a1b2c3d4⟧")
                               │
                               ▼
          afterLLMCall (AiSentinelMiddleware)
             • parse response for token shapes
             • verify HMAC, look up manifest
             • restore plaintext, or pass through unchanged + warn
                               │
                               ▼
          beforeToolCall :  repeat redaction on context.toolArgs
          afterToolCall  :  repeat restoration on context.result
                               │
                               ▼
          afterAgentRun (AiSentinelMiddleware)
             • zero manifest (char[] overwrite)
             • emit audit log (counts + timings, never plaintext)
                               │
                               ▼
          Plaintext response → caller
```

```mermaid
sequenceDiagram
    participant Caller
    participant Agent as bx-ai Agent
    participant MW as AiSentinelMiddleware
    participant Mfst as TokenManifest
    participant Eng as DetectionEngine
    participant LLM as LLM Provider

    Caller->>Agent: agent.run("…AWS key is AKIA…")
    Agent->>MW: beforeAgentRun(context)
    MW->>Mfst: new (per-run sessionKey)
    Agent->>MW: beforeLLMCall(context)
    MW->>Eng: scan(message.content)
    Eng-->>MW: hits[]
    MW->>Mfst: store(token ↔ char[] value)
    Note over MW,Agent: messages[].content mutated in place
    Agent->>LLM: POST request (tokens only)
    LLM-->>Agent: response (may echo tokens)
    Agent->>MW: afterLLMCall(context)
    MW->>Mfst: verify + lookup per token
    MW-->>Agent: response plaintext restored
    Agent->>MW: afterAgentRun(context)
    MW->>Mfst: zeroAll() + emit audit log
    Agent-->>Caller: "run: aws iam delete-access-key --access-key-id AKIA…"
```

## Provider examples

The middleware is provider-agnostic. All `bx-ai`-supported providers work the same way.

### OpenRouter (one key, many backend models)

```javascript
var model = aiModel(
    provider : "openrouter",
    model    : "openrouter/elephant-alpha",      // or any slug from https://openrouter.ai/models
    apiKey   : getSystemSetting( "OPENROUTER_API_KEY" )
);

var agent = aiAgent(
    model      : model,
    middleware : new bxAiSentinel.models.AiSentinelMiddleware()
);
```

### OpenAI

```javascript
var model = aiModel(
    provider : "openai",
    model    : "gpt-4o-mini",
    apiKey   : getSystemSetting( "OPENAI_API_KEY" )
);

var agent = aiAgent(
    model      : model,
    middleware : new bxAiSentinel.models.AiSentinelMiddleware()
);
```

### Anthropic

```javascript
var model = aiModel(
    provider : "anthropic",
    model    : "claude-3-5-sonnet-latest",
    apiKey   : getSystemSetting( "ANTHROPIC_API_KEY" )
);

var agent = aiAgent(
    model      : model,
    middleware : new bxAiSentinel.models.AiSentinelMiddleware()
);
```

### Ollama (local)

```javascript
var model = aiModel(
    provider : "ollama",
    model    : "llama3.2",
    baseUrl  : "http://localhost:11434"
);

var agent = aiAgent(
    model      : model,
    middleware : new bxAiSentinel.models.AiSentinelMiddleware()
);
```

## Configuration

Four layers, highest wins:

1. Constructor overrides: `new AiSentinelMiddleware( settings: { … } )`
2. Project-root `.sentinel.json`
3. `boxlang.json` → `bxSentinel` (or `modules.bx-aisentinel`)
4. Module defaults

| Setting | Default | Purpose |
| --- | --- | --- |
| `manifestScope` | `"run"` | `"run"` = fresh manifest each `agent.run()`. `"conversation"` = persist across turns. |
| `metricsEnabled` | `true` | Disable to skip timing capture + audit writes for max throughput. |
| `auditEnabled` | `true` | Emit LogBox records (counts + timings only, never plaintext). |
| `auditLogFile` | `"bx-aisentinel"` | LogBox channel name. |
| `entropyThreshold` | `4.5` | Shannon entropy bits/char for `EntropyDetector` to flag a run. |
| `entropyMinLength` | `20` | Minimum run length before entropy is evaluated. |
| `requireMixedCharset` | `true` | Entropy detector demands ≥2 char classes (letter / digit / symbol / case). |
| `enableRegistryDetector` | `true` | Toggle literal-match user-registered secrets. |
| `enableRegexDetector` | `true` | Toggle catalog-driven regex detection. |
| `enableEntropyDetector` | `true` | Toggle entropy fallback. |
| `categoriesEnabled` | see below | Which pattern categories to apply. |
| `registeredSecrets` | `[]` | Array of `{ label, value, category }` (appended across config layers). |
| `customPatterns` | `[]` | Array of `{ label, regex, category, confidence, validator }` (appended across layers). |
| `externalDetectors` | `[]` | WireBox IDs (or fully-qualified class paths) of sibling-module detectors. See "External detectors" below (appended across layers). |
| `externalDetectorOptions` | `{}` | Per-detector init args, keyed by the detector's resolution ID. |
| `enabled` | `true` | Runtime master switch. When `false`, every hook short-circuits. |

Default `categoriesEnabled`: `cloud-keys`, `vendor-tokens`, `generic`, `pii`, `boxlang`, `entropy`, `registry`.

## External detectors

bx-AISentinel exposes a plugin seam so sibling modules can contribute additional detectors to the pipeline without modifying the core. This is how Tier 1 NER detection ([`bx-AISentinel-ONNX`](https://github.com/mrigsby/bx-AISentinel-ONNX)) is delivered, and how you would ship custom detectors for proprietary entity formats, regulatory-specific patterns, or domain-specific heuristics.

An external detector is any class that implements the [`IDetector@1.0.0` contract](models/detectors/CONTRACT.md): a `scan()` method returning an array of `Hit` structs plus the usual `getPriority()`, `getSourceName()`, and `getContractVersion()` introspection methods. The cleanest implementation path is to extend [`models/detectors/IDetector.bx`](models/detectors/IDetector.bx); duck-typed classes with the same surface are also accepted.

### Wiring

```javascript
var sentinel = new AiSentinelMiddleware( settings: {
    "externalDetectors" : [
        "OnnxNerDetector@bx-AISentinel-ONNX"            // ColdBox-convention WireBox ID
        // or:
        // "my.app.detectors.CustomDetector"            // fully-qualified class path
    ],
    "externalDetectorOptions" : {
        "OnnxNerDetector@bx-AISentinel-ONNX" : {
            "modelPath" : "~/.bx-aisentinel/models/gliner-pii-v1/",
            "assetMode" : "manual"
        }
    }
} );
```

Resolution order: IDs containing a `.` go through `createObject( "component", ... )`; everything else goes through `application.wirebox` (the ColdBox convention). Host apps without ColdBox should prefer class paths.

### Load report

```javascript
var report = sentinel.getLoadReport();
// → {
//       compatibleContractVersion : "1.0.0",
//       degraded                  : false,
//       detectors                 : [
//           { name: "OnnxNerDetector@bx-AISentinel-ONNX", status: "loaded" }
//       ]
//   }
```

`degraded: true` means at least one declared external detector failed to load. Surface this from a health endpoint so ops can see when a plugin silently fell out of the pipeline.

### Failure handling

bx-AISentinel never hard-fails on a single broken plugin — the sentinel stays up on its built-in detectors. On any external-detector resolution error (missing class, contract-version mismatch, constructor throw, incomplete surface), it:

1. Logs a structured entry via `SentinelAuditor.warnDetectorError( detectorName, reason )`.
2. Marks the detector `status: "degraded"` in `getLoadReport()` and sets `degraded: true`.
3. Fires the `onSentinelDetectorLoadFailure` interception point.

Host apps decide their own escalation. Subscribe an interceptor in any ColdBox `Interceptor.cfc`:

```javascript
function onSentinelDetectorLoadFailure( event, interceptData ) {
    // interceptData = { detectorName, reason, attemptedContractVersion, supportedContractVersion, failureMode }
    // Send to Slack, page oncall, refuse to start — whatever fits your risk posture.
    logBox.getLogger( this ).error( "Sentinel detector failed: " & interceptData.detectorName );
}
```

### Contract versioning

The supported contract version is `1.0.0`. A detector declaring a major-version mismatch (e.g. `2.0.0`) is rejected at load time with `failureMode: "contract-mismatch"`. Minor / patch versions within the same major are accepted. Breaking changes to the contract require a major bump on both sides.

## Public API

```javascript
var sentinel = new AiSentinelMiddleware( settings: { /* ... */ } );

// Runtime mutation
sentinel.registerSecret( label: "DB_PASSWORD", value: "sekret-prod-1" );
sentinel.setEnabled( false );    // instant disable, every hook no-ops
sentinel.setEnabled( true );

// Inspection
var metrics = sentinel.getLastRunMetrics();
// → { outboundMs, inboundMs, toolRedactMs, toolRestoreMs, tokensMinted, categories, llmRoundtripMs }

var policy = sentinel.getPolicy();
policy.getSourceReport();
// → { defaultsApplied, boxlangJsonLoaded, sentinelJsonLoaded, overridesApplied }

var load = sentinel.getLoadReport();
// → { compatibleContractVersion, degraded, detectors: [ { name, status, reason? } ] }

sentinel.isEnabled();
```

## What this does NOT protect against

Be honest about scope. bx-AISentinel **reduces** risk; it does not eliminate it.

- **Malicious local code.** Anything running in the same JVM with access to `variables` scope can read the manifest or session key. If your app is compromised, Sentinel is compromised.
- **Compromised `bx-ai` or its dependencies.** The middleware chain runs in-process; another malicious middleware can see plaintext before Sentinel redacts.
- **Side-channel inference.** A long conversation may reveal structural information about redacted values through the LLM's responses.
- **Deliberate user exfiltration.** A user who wants to leak data can disable Sentinel, use a different agent, or paste into a different tool.
- **False negatives.** Any secret format not in the catalog, below the entropy threshold, and not explicitly registered is transmitted plaintext. **The catalog is a floor, not a ceiling.**
- **Out-of-band channels.** Values transmitted via other paths (direct HTTP, unrelated SDKs, CLI tools) never flow through Sentinel.
- **Response-generated plaintext.** If the LLM guesses a secret correctly from context, Sentinel cannot redact something it never saw.
- **JVM memory residue.** Plaintext is held as `char[]` and explicitly zeroed in `afterAgentRun`, but the JVM may retain copies in internal buffers (string pool, IO buffers), GC may move/copy without our knowledge, and heap pages may swap to disk. **This is best-effort, not a guarantee.**

bx-AISentinel avoids absolute security claims. Prefer language like "reduces the risk of," "helps prevent," or "mitigates exposure to." Avoid absolutes like "secrets never leave your machine," "100% secure," or "guaranteed PII protection."

## Performance

Tier 0 (regex + entropy + registry) target: **sub-10 ms on prompts up to 8 KB on a developer laptop**. Measured as `outboundMs + inboundMs` per `agent.run()`.

Runtime performance is dominated by regex scanning. If you disable `enableEntropyDetector` you cut the per-turn cost by roughly half on small prompts; leave it on for safety against unknown token formats.

## Development

```sh
git clone https://github.com/mrigsby/bx-AISentinel.git
cd bx-AISentinel/tests

# Install TestBox into the isolated test harness
box install

# Start the harness server (port 8082). On first run this also installs
# bx-ai, bx-compat-cfml, bx-esapi, and bx-markdown into a local
# BoxLang engine under tests/.engine/
box server start

# Runner URL: http://localhost:8082/runner.bxm
# or run headlessly:
box testbox run
```

Coverage: 12 TestBox spec files covering detectors, tokenizer, manifest, detection engine, policy, auditor, pattern catalog, and three integration specs that drive the full hook sequence.

## Demo application

The companion [**bx-AISentinel-demo**](https://github.com/mrigsby/bx-AISentinel-demo) repository is the end-to-end acceptance test for this module. It's a ColdBox 8 + CBWire chat app on BoxLang Server that drives a real OpenRouter call through bx-AISentinel and surfaces the middleware's behavior in the UI.

### Features

- **Reactive chat**: CBWire-powered chat page hitting a real `aiAgent()` against OpenRouter (one API key in `.env`; the default `openrouter/elephant-alpha` model is free).
- **Sentinel master toggle**: flip the middleware off mid-session and send the same prompt to compare raw vs tokenized payloads.
- **Token-coaching toggle**: switch "Explain tokens to LLM" to inject (or suppress) the protocol system message that teaches the model to keep `⟦SECRET:…⟧` placeholders byte-exact.
- **Seeded sample prompts**: one-click buttons that paste fake-but-realistic fixtures (AWS keys, GitHub tokens, SendGrid tokens, SSNs, customer emails, Postgres URIs, full `.env` blocks) so you can see detection fire immediately.
- **Raw request / response inspector**: side-by-side view of what the LLM actually saw vs what the caller sees after restoration.
- **Per-reply timing badge**: outbound, inbound, tool redact, tool restore, and LLM round-trip, color-coded green / amber / red for quick overhead visibility.
- **Session-scoped dashboard** at `/dashboard`: running totals for tokens minted, category breakdown, overhead ratio, and a recent-runs ring.
- **Developer affordances**: sidebar link that opens the TestBox runner for the demo's own spec suite, plus a GitHub link to this module's repo.

Setup + usage instructions live in the [demo repo README](https://github.com/mrigsby/bx-AISentinel-demo#readme).

## License

Apache 2.0 (see [`LICENSE`](LICENSE)).

Regex patterns in [`includes/patterns/`](includes/patterns/) are derived from:

- [gitleaks](https://github.com/gitleaks/gitleaks) (MIT)
- [detect-secrets](https://github.com/Yelp/detect-secrets) (Apache-2.0)

Each bundled JSON file preserves source attribution in its `_meta` block.
