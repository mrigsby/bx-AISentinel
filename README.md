# bx-AISentinel

> A BoxLang AI middleware that tokenizes secrets, credentials, and PII in outbound LLM prompts and restores them on the inbound response. Drop-in, provider-agnostic, with per-call timing metrics.

**Status:** v0.1.0 · pre-release · [Changelog](CHANGELOG.md)

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
import bxModules.bxaisentinel.models.AiSentinelMiddleware;

var sentinel = new AiSentinelMiddleware();

var agent = aiAgent(
    name         : "assistant",
    instructions : "You are a helpful assistant.",
    memory       : aiMemory( "window", { maxMessages: 10 } )
).withMiddleware( sentinel );

var reply = agent.run(
    "My AWS key is AKIAIOSFODNN7EXAMPLE — what is the cli command to rotate it?"
);
// The LLM saw only: "…AWS key is ⟦SECRET:AWS_ACCESS_KEY:a1b2c3d4⟧…"
// The returned `reply` has the plaintext restored.

var metrics = sentinel.getLastRunMetrics();
// → { outboundMs, inboundMs, toolRedactMs, toolRestoreMs, tokensMinted, categories, llmRoundtripMs }
```

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
          (LLM sees tokenized text only — e.g. "⟦SECRET:AWS_ACCESS_KEY:a1b2c3d4⟧")
                               │
                               ▼
          afterLLMCall (AiSentinelMiddleware)
             • parse response for token shapes
             • verify HMAC, look up manifest
             • restore plaintext — or pass through unchanged + warn
                               │
                               ▼
          beforeToolCall  — repeat redaction on context.toolArgs
          afterToolCall   — repeat restoration on context.result
                               │
                               ▼
          afterAgentRun (AiSentinelMiddleware)
             • zero manifest (char[] overwrite)
             • emit audit log (counts + timings — never plaintext)
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

See [`DOCUMENTATION/02-architecture.md`](../DOCUMENTATION/02-architecture.md) for hook-by-hook sequence, [`DOCUMENTATION/04-token-format.md`](../DOCUMENTATION/04-token-format.md) for the token wire format.

## Provider examples

The middleware is provider-agnostic. All `bx-ai`-supported providers work the same way.

### OpenRouter (one key, many backend models)

```javascript
var model = aiModel(
    provider : "openrouter",
    model    : "openrouter/elephant-alpha",      // or any slug from https://openrouter.ai/models
    apiKey   : getSystemSetting( "OPENROUTER_API_KEY" )
);

var agent = aiAgent( model: model ).withMiddleware( new AiSentinelMiddleware() );
```

### OpenAI

```javascript
var model = aiModel(
    provider : "openai",
    model    : "gpt-4o-mini",
    apiKey   : getSystemSetting( "OPENAI_API_KEY" )
);

var agent = aiAgent( model: model ).withMiddleware( new AiSentinelMiddleware() );
```

### Anthropic

```javascript
var model = aiModel(
    provider : "anthropic",
    model    : "claude-3-5-sonnet-latest",
    apiKey   : getSystemSetting( "ANTHROPIC_API_KEY" )
);

var agent = aiAgent( model: model ).withMiddleware( new AiSentinelMiddleware() );
```

### Ollama (local)

```javascript
var model = aiModel(
    provider : "ollama",
    model    : "llama3.2",
    baseUrl  : "http://localhost:11434"
);

var agent = aiAgent( model: model ).withMiddleware( new AiSentinelMiddleware() );
```

## Configuration

Four layers, highest wins:

1. Constructor overrides — `new AiSentinelMiddleware( settings: { … } )`
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
| `registeredSecrets` | `[]` | Array of `{ label, value, category }` — appended across config layers. |
| `customPatterns` | `[]` | Array of `{ label, regex, category, confidence, validator }` — appended across layers. |
| `enabled` | `true` | Runtime master switch. When `false`, every hook short-circuits. |

Default `categoriesEnabled`: `cloud-keys`, `vendor-tokens`, `generic`, `pii`, `boxlang`, `entropy`, `registry`.

## Public API

```javascript
var sentinel = new AiSentinelMiddleware( settings: { /* ... */ } );

// Runtime mutation
sentinel.registerSecret( label: "DB_PASSWORD", value: "sekret-prod-1" );
sentinel.setEnabled( false );    // instant disable — every hook no-ops
sentinel.setEnabled( true );

// Inspection
var metrics = sentinel.getLastRunMetrics();
// → { outboundMs, inboundMs, toolRedactMs, toolRestoreMs, tokensMinted, categories, llmRoundtripMs }

var policy = sentinel.getPolicy();
policy.getSourceReport();
// → { defaultsApplied, boxlangJsonLoaded, sentinelJsonLoaded, overridesApplied }

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
- **JVM memory residue.** Plaintext is held as `char[]` and explicitly zeroed in `afterAgentRun`, but the JVM may retain copies in internal buffers (string pool, IO buffers), GC may move/copy without our knowledge, and heap pages may swap to disk. **This is best-effort, not a guarantee.** See [`DOCUMENTATION/05-threat-model.md`](../DOCUMENTATION/05-threat-model.md) for the full treatment and operational hardening recommendations.

Prefer framing like "reduces the risk of," "helps prevent," "mitigates exposure to." Avoid "secrets never leave your machine," "100% secure," "guaranteed PII protection."

## Performance

Tier 0 (regex + entropy + registry) target: **sub-10 ms on prompts up to 8 KB on a developer laptop**. Measured as `outboundMs + inboundMs` per `agent.run()`. See [`DOCUMENTATION/06-metrics.md`](../DOCUMENTATION/06-metrics.md) for measurement boundaries and test coverage.

Runtime performance is dominated by regex scanning. If you disable `enableEntropyDetector` you cut the per-turn cost by roughly half on small prompts; leave it on for safety against unknown token formats.

## Development

```sh
# Clone this repo, then from the module root:
cd bx-AISentinelMiddleware/

# Unit + integration tests
boxlang run tests/runner.bxm

# Lint / type check
# (none yet — TestBox is the primary signal)
```

The demo application in [`../DEMO-APPLICATION/`](../DEMO-APPLICATION/) is the end-to-end acceptance test: ColdBox + CBWire against a real OpenRouter call.

## Documentation

- [`DOCUMENTATION/00-plan-and-progress.md`](../DOCUMENTATION/00-plan-and-progress.md) — canonical plan + live progress
- [`DOCUMENTATION/02-architecture.md`](../DOCUMENTATION/02-architecture.md) — hook-by-hook sequence
- [`DOCUMENTATION/03-detection-catalog.md`](../DOCUMENTATION/03-detection-catalog.md) — full pattern list
- [`DOCUMENTATION/04-token-format.md`](../DOCUMENTATION/04-token-format.md) — wire format + HMAC design
- [`DOCUMENTATION/05-threat-model.md`](../DOCUMENTATION/05-threat-model.md) — what it defends against, what it doesn't
- [`DOCUMENTATION/06-metrics.md`](../DOCUMENTATION/06-metrics.md) — timing boundaries + audit log schema
- [`DOCUMENTATION/07-future-work.md`](../DOCUMENTATION/07-future-work.md) — Tier 1 NER, local-LLM arbitration, OS keychain, streaming

## License

Apache 2.0 — see [`LICENSE`](LICENSE).

Regex patterns in [`includes/patterns/`](includes/patterns/) are derived from:

- [gitleaks](https://github.com/gitleaks/gitleaks) (MIT)
- [detect-secrets](https://github.com/Yelp/detect-secrets) (Apache-2.0)

Each bundled JSON file preserves source attribution in its `_meta` block.
