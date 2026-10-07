# SIRPI M0 - Sidd-AI API Contracts

| Field | Value |
|---|---|
| Milestone | M0 Discover (`architecture.md` §50) |
| Scope | Answers to the Sidd-AI discovery checklist S1-S12 (`architecture.md` §17.2) |
| Source inspected | `/home/psTech/psTech/sidd-ai`, git HEAD `f0cc6bd` ("feat: add Groq and release 1.0.5") |
| Date | 2026-10-07 |
| Status | M0 findings recorded. No SIRPI code exists yet. |

All Sidd-AI paths below are relative to
`sidd-ai/src/main/java/io/github/prabhusiddarth/sidd_ai/` unless they start with `sidd-ai/`.

Classification used in this document:

- **Verified**: observed directly in Sidd-AI source, `pom.xml`, or Maven Central metadata.
- **SIRPI requirement**: what SIRPI must do as a consequence. Design obligation, not yet implemented.
- **Assumption**: believed true but not verified by inspection or by a build.
- **Unresolved**: open question that must be answered before or during the relevant milestone.

---

## 1. Checklist summary

| # | Question | Answer | Basis |
|---|---|---|---|
| S1 | Coordinates, version, Java baseline | `io.github.prabhusiddarth:sidd-ai:1.0.5`, compiled with `release 17`, depends on `jackson-databind 2.18.6` | Verified |
| S2 | Provider and model enumeration | Providers: registry exists but has no enumeration method. Models: catalog holds only per-provider default entries | Verified (limitation) |
| S3 | Native tool/function calling | Not supported on requests. Any non-empty `toolIds` is rejected | Verified (limitation) |
| S4 | Structured output / JSON mode | OpenAI only (`responseFormat`); other providers reject it | Verified (limitation) |
| S5 | Token usage in responses | Yes, input/output/total from all four response-mapper families | Verified |
| S6 | Typed retryable vs fatal errors | Yes, `ProviderException` with `category`, `retryable()`, `httpStatus()` | Verified |
| S7 | How credentials are supplied | Explicit injection through `ProviderConfiguration` is possible; legacy paths also read env / `.env` | Verified |
| S8 | Prompt/response logging | No prompt or response logging found | Verified by search (see §2.9 for limits) |
| S9 | Streaming and cancellation | Cancellation token wired to HTTP; streaming declared for some providers only; fixed 60 s request timeout | Verified |
| S10 | Thread-safety of client instances | Not determined | Unresolved |
| S11 | Context window / output limits per model | Fields exist on `ModelDescriptor` but are `null` in default catalog entries | Verified (limitation) |
| S12 | Local model / custom endpoint | Ollama supported; OpenAI-compatible and Ollama factories accept a custom endpoint | Verified |

---

## 2. Verified Sidd-AI capabilities

### 2.1 Coordinates and Java compatibility (S1)

- `sidd-ai/pom.xml`: `groupId io.github.prabhusiddarth`, `artifactId sidd-ai`, `version 1.0.5`, `packaging jar`, license Apache-2.0.
- `maven.compiler.release` = `17`; `maven-compiler-plugin` configured with `<release>17</release>`.
- Runtime dependency: `com.fasterxml.jackson.core:jackson-databind:2.18.6`. JUnit 5.10.0 is test-scoped only.
- The build also configures `maven-shade-plugin` producing an attached `all` classifier artifact.
- Maven Central `io/github/prabhusiddarth/sidd-ai/maven-metadata.xml` reports `<latest>1.0.5</latest>` and `<release>1.0.5</release>`.
- No JPMS `module-info.java` was observed in the source listing; `internal.*` packages are ordinary public packages.

### 2.2 Provider registry (S2, providers)

`api/provider/ProviderRegistry.java`:

- Public methods: `register(ProviderFactory)`, `get(String)`, `resolveCanonicalId(String)`, `createChat(String, String, ProviderConfiguration)`, `createProvider(String, ProviderConfiguration)`, static `defaultRegistry()`.
- There is **no method that lists registered providers or ids**.
- The default registry registers eight factories from `internal/provider/BuiltInProviderFactories.java`: `openai`, `gemini`, `anthropic` (alias `claude`), `grok`, `groq`, `nim` (alias `nvidia`), `kimi` (alias `moonshot`), `ollama`.
- Provider keys and aliases are normalized with `toLowerCase(Locale.ROOT).trim()`.
- `defaultRegistry()` returns a copy, so registering into it does not mutate the shared default.

`api/provider/Provider.java`:

- `ProviderDescriptor descriptor()`
- `ModelResponse invoke(ModelRequest request)`
- `default void stream(ModelRequest, Consumer<StreamEvent>)`, which throws `ProviderException(UNSUPPORTED_CAPABILITY)` unless overridden.

Declared provider capabilities (`BuiltInProviderFactories.java`):

| Provider | Declared capabilities |
|---|---|
| `openai` | `STREAMING`, `STRUCTURED_OUTPUT` |
| `gemini` | none |
| `anthropic` | none |
| `grok`, `groq`, `nim`, `kimi`, `ollama` | `STREAMING` |

No provider declares `TOOLS`.

### 2.3 Model registry limitations (S2, models; S11)

- `api/catalog/ModelCatalog.java`: `register(ModelCatalogEntry)`, `resolve(String)`, `list()`, static `defaultCatalog()`.
- `api/catalog/ModelCatalogDefaults.java`: the observed default entries are per-provider placeholders such as `openai:default`, `gemini:default`, `anthropic:default`, `grok:default`, `groq:default`, constructed with `contextWindow = null` and `maxOutputTokens = null`.
- `api/model/ModelDescriptor.java` has `contextWindow` and `maxOutputTokens` as nullable `Integer` fields.
- `api/catalog/ModelRegistry.java` additionally holds alias registration and a per-session active model (`setActiveModel(sessionId, modelRef)`, `getActiveModel(sessionId)`).
- An `api/catalog/ModelDiscovery.java` file exists; its behaviour was not inspected (see U2).

### 2.4 Native tool calling (S3)

- `internal/provider/ModelRequestValidator.java`: if `request.toolIds()` is non-empty, it throws `unsupportedCapability(request, "TOOLS", "Tool schema mapping is not implemented in this phase")`.
- `ModelRequest.toolIds` is a `Set<String>` only; there is no field for tool schemas.
- `internal/provider/openai/OpenAiCompatibleRequestMapper.java` sends each message as `{role, content}` with text-only content (`MessageContentSupport.textOnlyContent`). `TOOL`-role messages are sent as `role: "tool"` with text content and no tool-call id.
- `internal/provider/openai/OpenAiCompatibleResponseMapper.java` does parse `choices[0].message.tool_calls` from responses into `ToolCall`s, but because tools cannot be sent on the request, this path is not usable for a tool protocol.

### 2.5 JSON mode (S4)

- `ModelRequestValidator`: a non-blank `responseFormat` is rejected with `UNSUPPORTED_CAPABILITY` unless the provider declares `STRUCTURED_OUTPUT`. Only `openai` declares it.
- `OpenAiCompatibleRequestMapper` writes `response_format: {type: <responseFormat>}`.
- `OpenAiCompatibleResponseMapper` wraps the content in a `StructuredPart` when `responseFormat` is set. No schema validation was observed.
- `grok`, `groq`, `nim`, `kimi` share the OpenAI-compatible mapper but do **not** declare `STRUCTURED_OUTPUT`, so the validator rejects `responseFormat` for them.

### 2.6 Token usage (S5)

- `api/model/Usage.java`: `inputTokens`, `outputTokens`, `totalTokens`, `cachedInputTokens`, `reasoningTokens`, `audioInputTokens`, `audioOutputTokens`; all validated non-negative.
- Response mappers populate input/output/total:
  - OpenAI-compatible: `usage.prompt_tokens`, `usage.completion_tokens`, `usage.total_tokens`, read with `asInt(0)`.
  - Anthropic: input + output; total computed as their sum.
  - Gemini: input, output, total.
  - Ollama: `prompt_eval_count`, `eval_count`; total computed as their sum.
- Cached, reasoning and audio counts are always set to `0` by these mappers.
- `ModelResponse` defaults `usage` to `Usage.ofTotal(0)` when null.

### 2.7 Typed errors (S6)

- `api/provider/ProviderException` (extends `AiException`) exposes `category()`, `providerId()`, `retryable()`, `httpStatus()`, `providerRequestId()`, `errorCode()`.
- `api/provider/ProviderErrorCategory`: `AUTHENTICATION`, `AUTHORIZATION`, `RATE_LIMIT`, `INVALID_REQUEST`, `MODEL_NOT_FOUND`, `TRANSPORT`, `TIMEOUT`, `PROVIDER_AVAILABILITY`, `MALFORMED_RESPONSE`, `UNSUPPORTED_CAPABILITY`, `CANCELLATION`, `UNKNOWN`.
- Retryability as set by the mappers:
  - `internal/provider/openai/OpenAiCompatibleErrorMapper.java`: auth, authorization, model-not-found, invalid-request, unknown → `false`; rate-limit, provider-unavailable → `true`.
  - `internal/provider/AbstractProviderErrorMapper.java`: timeout (`HttpTimeoutException`) → `TIMEOUT`, `true`; other transport failures → `TRANSPORT`, `true`; malformed response → `false`; unsupported capability → `false`; cancellation → `false`.
- `providerRequestId` is taken from the `x-request-id` response header when present.
- The Anthropic, Gemini and Ollama error mappers were not individually inspected (see A7).

### 2.8 Credentials (S7)

- `api/provider/ProviderConfiguration` is a record `(providerId, credential, endpoint, options)`.
  - `credential` is a `String`.
  - `endpoint`, if present, must be an absolute `http`/`https` URI.
  - `toString()` prints `credential=[REDACTED]` and redacts sensitive option keys via `SensitiveKeyPolicy`.
- `ProviderRegistry.createProvider(providerKey, configuration)` passes the configuration to the factory. This is an explicit injection path that does not require environment variables.
- `internal/provider/openai/OpenAiCompatibleProviderFactory.create(...)` throws `ProviderException(AUTHENTICATION, ..., "missing_credential")` when no credential is supplied; it does not fall back to the environment.
- Other paths in Sidd-AI read the environment or files:
  - `EnvHelper` and the legacy `providers/*Chat(String model)` constructors read `*_API_KEY` env vars or `.env` (per `sidd-ai/API_REFERENCE.md`).
  - `api/config/EnvironmentConfigurationSource`, `api/config/EnvironmentSecretProvider`, `api/config/ConfigurationContext` call `System.getenv()`; `ConfigurationContext.standard(...)` also references `Path.of(".env")`.

### 2.9 Logging (S8)

- A search of `sidd-ai/src/main/java` for `Logger`, `System.out`, `System.err` found only the permissions audit types (`api/permissions/AuditLogger`, `InMemoryAuditLogger`, use in `PermissionManager`). No prompt or response logging was found.
- `ToolCall.toString()` and `ToolResult.toString()` redact ids, arguments and content.
- Limit: the search was pattern-based; `internal/observability/ObserverDispatcher` was not inspected (see A6).

### 2.10 Cancellation and streaming (S9)

- `api/execution/CancellationToken` (interface) and `api/execution/CancellationSource` (supports a parent token and a `Deadline`; `cancel(reason)`, `child()`; token exposes `isCancelled()`, `reason()`, `deadline()`, `onCancel(Runnable)`, `throwIfCancelled()`).
- `ModelRequest` carries a `CancellationToken`.
- `internal/transport/JavaHttpTransport`: uses `sendAsync`; registers `onCancel` to cancel the future (and close the stream for streaming).
- `internal/provider/MappedProvider` checks cancellation before the request and before any retry pause, and maps cancellation to `ProviderException(CANCELLATION)`.
- Request timeout is hard-coded to `Duration.ofSeconds(60)` in the OpenAI-compatible, Anthropic, Gemini and Ollama request mappers. HTTP connect timeout is 10 s (`JavaHttpTransport`).
- Streaming is available only where the descriptor declares `STREAMING` (§2.2); `gemini` and `anthropic` do not.

### 2.11 Ollama and custom endpoints (S12)

- `internal/provider/ollama/OllamaProviderFactory`: uses `configuration.endpoint()` when present, else the default host `http://localhost:11434`; trailing `/` stripped; calls `<host>/api/chat`. No credential is required.
- `OpenAiCompatibleProviderFactory.create(...)` uses `configuration.endpoint()` when present, else the built-in endpoint. This covers `openai`, `grok`, `groq`, `nim`, `kimi`.
- `ProviderDescriptor.endpoint` records each provider's default base URL.
- Custom-endpoint handling in the Anthropic and Gemini factories was not inspected (see A4).

### 2.12 Retry ownership

- `internal/provider/RetryPolicy` is an internal interface; `MappedProvider` honours it.
- Every built-in factory constructs `MappedProvider` with `RetryPolicy.noRetries()` (OpenAI-compatible, Anthropic, Gemini, Ollama factories).
- Result: Sidd-AI performs **no retries** on the `Provider.invoke` path.

### 2.13 Other Sidd-AI subsystems present

The source tree also contains `api/permissions` (`PermissionManager`, `PermissionPolicy`, ...), `api/tools` (`ToolRegistry`, `ToolExecutor`, ...), `api/runtime` (`AiRuntime`, `SessionManager`, ...) and `api/execution` (`ExecutionManager`, ...). Their existence is verified from the file listing; their behaviour was not inspected.

---

## 3. SIRPI requirements derived from the findings

| ID | Requirement | Driven by |
|---|---|---|
| R-M1 | Build with Maven multi-module and consume `io.github.prabhusiddarth:sidd-ai:1.0.5` from Maven Central (`architecture.md` §7.1). | S1 |
| R-M2 | `SiddAiAdapter` (only class importing Sidd-AI) uses `ProviderRegistry.createProvider(...)` with an explicit `ProviderConfiguration`, then `Provider.invoke(...)`. It must not use `AiClient`, `EnvHelper`, Sidd-AI's `router.ModelRouter`, the legacy `*Chat(String model)` constructors, or `ConfigurationContext.standard(...)`, because those read env vars / `.env`. | S7, §31 |
| R-M3 | The adapter must not depend on `io.github.prabhusiddarth.sidd_ai.internal.*`. Enforce with ArchUnit alongside the Sidd-AI confinement rule. | §2.1 (no JPMS) |
| R-M4 | SIRPI must not use Sidd-AI's permissions, tools, runtime, session or execution managers. The Guard, tool runtime, sessions and journal are SIRPI's (ADR-001, ADR-002, ADR-003). | §2.13 |
| R-M5 | SIRPI keeps its own provider list and model registry fed by `~/.sirpi/config.yaml`, because Sidd-AI cannot enumerate providers and its catalog only has `<provider>:default` entries. `ModelGateway.available()` = configured models whose provider key resolves via `ProviderRegistry.resolveCanonicalId(...)`. | S2 |
| R-M6 | `ModelLimits` (context window, max output) come from SIRPI config, with a conservative default when absent. Sidd-AI limits are not used. | S11 |
| R-M7 | Use the text JSON decision envelope (`architecture.md` §17.3) for all providers. Never set `ModelRequest.toolIds`. Tool results go back as SIRPI-rendered text. | S3 |
| R-M8 | `DecisionParser` must be strict and provider-independent, with one repair round. `responseFormat` may be set only for providers that declare `STRUCTURED_OUTPUT` (currently `openai`), as an optimization, never a dependency. | S4 |
| R-M9 | Budgets use Sidd-AI `Usage` input/output/total. A reported total of `0` with non-empty output is treated as "unreported": fall back to the local estimator and mark the budget approximate. | S5 |
| R-M10 | Map `ProviderException.category()` / `retryable()` to SIRPI's sealed `ModelException` hierarchy (`architecture.md` §37). Failover is allowed only for `retryable() == true` and only when no decision from that response was executed (§19). | S6 |
| R-M11 | SIRPI's `ModelRouter` owns retry and failover policy entirely (Sidd-AI does none). | §2.12 |
| R-M12 | Pass a SIRPI-derived cancellation (mapped to Sidd-AI's `CancellationSource`, created inside the adapter only) on every request so Ctrl-C aborts in-flight calls. | S9 |
| R-M13 | P0 is non-streaming. Request timeout is effectively 60 s per call; SIRPI step deadlines must account for that. | S9 |
| R-M14 | Credentials are resolved just-in-time inside the adapter from a `CredentialRef` and placed in `ProviderConfiguration.credential` (a `String`). The residual "cannot zero a String" risk from `architecture.md` §31 applies and is accepted. | S7, §31 |
| R-M15 | Locality: `ollama` with a loopback endpoint → `LOCAL`; other providers → `PUBLIC_CLOUD` unless config declares otherwise for a custom endpoint. Locality is SIRPI config, not inferred from Sidd-AI. | S12, §18 |
| R-M16 | All bytes reach Sidd-AI only through `EgressFilter` in `ModelGateway` (§28.1). No prompt logging was found, so no logger redirection is planned for P0, but this must be re-checked (A6) and re-checked on every Sidd-AI version bump. | S8 |

---

## 4. Assumptions (not verified)

| ID | Assumption | How to verify |
|---|---|---|
| A1 | The local source at `f0cc6bd` matches the published `sidd-ai-1.0.5.jar` on Maven Central. | Compare the published jar's class list / sources jar against the local tree in M1. |
| A2 | Java 17 bytecode from Sidd-AI runs unchanged on SIRPI's Java 21 runtime. | First M1 build plus an adapter test. |
| A3 | `jackson-databind 2.18.6` brought in transitively does not conflict with SIRPI's own Jackson usage. | Pin one Jackson version via BOM in M1; run `mvn dependency:tree`. |
| A4 | Anthropic and Gemini factories, like the OpenAI-compatible factory, refuse to create a provider without an explicit credential and honour `configuration.endpoint()`. | Read `AnthropicProviderFactory` / `GeminiProviderFactory` `create(...)` before writing the adapter. |
| A5 | Anthropic, Gemini and Ollama request mappers, like the OpenAI-compatible mapper, send text-only content per message (so SIRPI's text envelope works for all). | Read the three request mappers before writing the adapter. |
| A6 | No prompt/response logging exists outside the searched patterns (e.g. in `internal/observability`). | Inspect `internal/observability/ObserverDispatcher` and `api/runtime/ExecutionObserver`. |
| A7 | Anthropic, Gemini and Ollama error mappers classify retryability the same way as the OpenAI-compatible mapper. | Read the three error mappers before writing the adapter's exception mapping. |

---

## 5. Unresolved questions

| ID | Question | Blocks |
|---|---|---|
| U1 (S10) | Are `Provider` instances thread-safe? `JavaHttpTransport` uses a static shared `HttpClient` and mappers hold immutable fields, but this was not confirmed. Interim rule: one `Provider` instance per agent. | P1 sub-agent concurrency |
| U2 | What does `api/catalog/ModelDiscovery` do, and can it list a provider's models (e.g. Ollama `/api/tags`)? | `sirpi models` UX; R-M5 stays config-driven either way |
| U3 | Can the 60 s request timeout be overridden through `ProviderConfiguration.options`? Not observed. | Long generations on slow local models |
| U4 | Should SIRPI contribute upstream: native tool schema mapping (S3), structured output for OpenAI-compatible providers, provider enumeration (S2), model limits (S11), configurable timeout? | Optional; not required for P0 |
| U5 | Is the shaded `all` classifier published, and should SIRPI use it to isolate Jackson? | M1 dependency decision |
| U6 | Does Sidd-AI's `Message` / `TextPart` validation (non-empty parts, non-blank text) reject any message shapes SIRPI needs, e.g. an empty tool observation? | Adapter mapping; SIRPI should never send empty text |

These feed Open Question OQ-1 in `architecture.md` §49.

---

## 6. Resulting SIRPI architectural implications

1. **Decision protocol (§17.3):** the JSON text envelope is the only path in P0. Native tool calls are not available from Sidd-AI 1.0.5.
2. **Model registry (§18, §19):** SIRPI-owned and config-driven. Sidd-AI is used as a per-provider transport, looked up by provider key.
3. **Credential isolation (§31):** achievable. The adapter injects credentials explicitly and avoids every Sidd-AI path that reads env vars or `.env`. ArchUnit must also forbid SIRPI code from calling `AiClient`, `EnvHelper`, Sidd-AI's `router.ModelRouter`, and `ConfigurationContext`.
4. **Failover (§19):** fully owned by SIRPI, using Sidd-AI's typed `retryable()` flag.
5. **Context budgeting (§11.2):** `ModelLimits` from config; token estimator calibrated from Sidd-AI usage when present.
6. **Cancellation (§38):** supported end-to-end through Sidd-AI's token; P0 stays non-streaming.
7. **Local-only routing (§19, §20):** supported via the `ollama` provider and custom endpoints; locality classification is SIRPI config.
8. **No reuse of Sidd-AI's agent-like subsystems:** permissions, tools, sessions and runtime in Sidd-AI stay unused, keeping the Guard and state SIRPI-owned.

---

## 7. Environment notes from M0

These are infrastructure observations, not application or security-policy decisions.

| Item | Observation |
|---|---|
| Agent sandbox | Every sandboxed command, file read, search and `apply_patch` write failed at sandbox startup with `bwrap: loopback: Failed RTM_NEWADDR: Operation not permitted` (or reported "aborted"). This is a host/container network-namespace limitation of the agent's sandbox launcher. It is not a SIRPI Guard verdict, not a policy DENY, and has no bearing on SIRPI's design. |
| Workaround used | Read-only commands, and the single write of this document, were run with per-command escalation. Writes were limited to `/home/psTech/psTech/sirpi/docs/api-contracts.md`; reads covered the sibling `sidd-ai` repo, `~/.m2` and Maven Central metadata. No sandbox setting was disabled and no broad approval rule was created. |
| JDK | OpenJDK 21.0.12.1 |
| Maven | Apache Maven 3.8.7 (Gradle not installed) |
| Maven Central | Reachable from escalated commands (HTTP 200). |
| Local `~/.m2` | Does not contain Sidd-AI, Jackson databind, picocli, ArchUnit or jqwik; M1 will need to download them. |

---

## 8. M0 exit criteria

| Criterion (`architecture.md` §50) | Status |
|---|---|
| Read Sidd-AI | Done (sections 2.1-2.13, with file references) |
| Answer S1-S12 with file references | S1-S9, S11, S12 answered from source; S10 recorded as unresolved (U1) with a safe interim rule |
| Write `docs/api-contracts.md` against real APIs | This document |
| Items to close before the adapter is written (M4) | A4, A5, A6, A7, U6 |
