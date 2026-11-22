# Phase 0 Research: BrowserAgent Composition

All NEEDS CLARIFICATION items from the plan's Technical Context are resolved here. Sources: the normative architecture plan (arrrrny/zuraffa_browser `handoffs/2026-11-22-pi-zuraffa-browser-agent-architecture-plan.md`), the dependency issues (zuraffa_agent#205, zuraffa_browser_core#2), and issue #1 itself.

## D1 — Package shape and the "not tools-only" decision

- **Decision**: `zuraffa_browser_agent` is a normal Dart library package whose public surface is `BrowserAgent` plus policy/ports/tool-factory libraries. The `_tools` name is rejected.
- **Rationale**: The package's deliverable is a composition (`BrowserAgent`) that hosts embed; tools are one contribution inside it. A tools-only name would invite a second composition to grow in the store app or in `zik_zak`.
- **Alternatives considered**: (a) `zuraffa_browser_agent_tools` — rejected by the architecture plan §1 and the issue; (b) folding the composition into `zuraffa_agent` — rejected, would make the generic package browser-aware; (c) a fifth `BrowserAgent` package — rejected, it is a class in this package.

## D2 — Composition seam: what BrowserAgent wraps and returns

- **Decision**: `BrowserAgent` wraps an injected `zuraffa_agent` runtime (pi-based) and an injected `BrowserRpcClient`. It returns the pi/Zuraffa runtime's own outcomes (conversation, submission, watches, tasks). It owns no mission state machine and no result type of its own.
- **Rationale**: Architecture plan §4/§9.3 fixes the execution shape: host mission request → BrowserAgent → ZuraffaAgent composition → pi durable conversation/submission → browser extension + policy hooks → canonical dispatcher → transport → core → validated result → pi outcome → host validation. Any parallel mission/result model would have to be kept synchronized with pi (plan §5.5 forbids it).
- **Alternatives considered**: a `BrowserMission` state machine — rejected (duplicates pi durable states); a bespoke `BrowserAgentResult` — rejected (hosts would need two outcome vocabularies).

## D3 — Tool generation: consume the neutral catalog, never re-describe capabilities

- **Decision**: The browser core (zuraffa_browser_core#2) owns neutral capability descriptors (stable name, description, input schema, mutability/risk hint, required capability, result contract). This package's tool factory maps each descriptor to a pi/Zuraffa agent tool and layers agent policy on top. Schemas, validation, and operation bodies live once, in the core.
- **Rationale**: Plan §8.3 + core#2 acceptance criteria; DRY without a third tool/result model (plan §5.5). The store MCP surface maps the same catalog; divergent hand-written wrappers would drift.
- **Alternatives considered**: hand-written agent tools per capability — rejected (duplication, drift); generating tools from MCP descriptors — rejected, core must not depend on an agent framework.
- **Consequence**: If a needed descriptor field is missing, the gap is filed against zuraffa_browser_core#2, not patched locally (spec Assumptions).

## D4 — Governance: compose zuraffa_agent#205 hooks, do not build a policy engine

- **Decision**: Allowlists, argument guards, risk classification, result minimization, PII scrubbing, artifact references, and approvals are implemented as pi `beforeTool`/`afterTool`/provider-request hook compositions declared by zuraffa_agent#205. This package supplies browser-specific policy *data* (allowlists, scopes, budgets, redaction rules) and browser-specific hook wiring, not a second engine.
- **Rationale**: zuraffa_agent#205 acceptance criteria already assign these mechanisms to pi primitives; a parallel engine recreates the drift the migration removes (plan §7.4, §13).
- **Alternatives considered**: a `BrowserPolicyEngine` class owning decisions — rejected; browser policy is expressed as hook composition + injected ports.

## D5 — Durable parking and challenge escalation

- **Decision**: CAPTCHA/login/2FA/human decisions escalate through host-provided ports and durably park the relevant tool/task using pi durable primitives (signals, memoized deadlines, parked tools). No autonomous bypass, no credential harvesting, no CAPTCHA solving exists in the package.
- **Rationale**: Plan §9.4 and issue #1 acceptance criterion; durable parking survives process restart (plan §10.3 gates).
- **Alternatives considered**: synchronous "ask the user" prompts — rejected (not durable, not restart-safe); polling loops — rejected (busy-wait is forbidden by fleet rules and wastes budgets).

## D6 — Budgets

- **Decision**: Every mission carries turn, tool-call, wall-clock, page-count, and output-size budgets. Budget state is serialized with the durable mission so it survives restart and disposal; exhaustion produces a typed outcome.
- **Rationale**: Plan §9.1/§10.3 and issue #1; per-page sequential execution on a shared page/WebView is the default concurrency posture, parallelism only across proven isolated contexts.
- **Alternatives considered**: process-local counters — rejected (restart resets them, plan §10.3 explicitly requires restart survival).

## D7 — Redaction and minimization

- **Decision**: Page content is minimized before model egress through host-injected redaction/minimization policies. A host may classify order numbers, addresses, emails, phone numbers, session identifiers, or other sensitive fields; BrowserAgent applies those rules opaquely and never learns the host's domain schema. Cookies, credentials, session payloads, and secure-store values are excluded from transcripts, telemetry, exceptions, and artifacts by construction (allowlist of what may be logged, not denylist). A sanitizer failure fails closed through the dependency-owned typed outcome channel.
- **Rationale**: Plan §9.4, §13 ("Generic secret redaction is insufficient without domain-aware minimization"); issue #1 acceptance criterion.
- **Alternatives considered**: regex-only redaction — rejected; post-hoc log scrubbing — rejected (leak already happened).

## D8 — Transport parity (embedded vs bridge)

- **Decision**: Both `EmbeddedBrowserRpcClient` and `BridgeBrowserRpcClient` run one shared tool contract suite for shared capabilities; each advertises its capability set explicitly; unsupported capabilities fail during mission planning.
- **Rationale**: Core#2 acceptance criteria + plan §8.2/§8.4; embedded must not require sockets/discovery/JSON round trips; bridge preserves the store app's external behavior.
- **Alternatives considered**: separate test suites per transport — rejected (parity would be asserted in prose, not proven).

## D9 — Host ports and composition root

- **Decision**: The host injects credentials/secure storage, egress config, transport (`EmbeddedBrowserRpcClient` or `BridgeBrowserRpcClient`), durable storage/platform environment, audit, telemetry, artifact store, approval presentation, human notification, and domain output schemas/validators. Business logic depends on those contracts only — never `GetIt.instance`, a concrete WebView, a socket, or a provider SDK.
- **Rationale**: Plan §5.4; keeps the package testable over pure-Dart in-memory adapters (SC-001).
- **Alternatives considered**: service locator — explicitly rejected by plan §5.4.

## D10 — Domain ignorance

- **Decision**: `BrowserAgent` knows nothing of orders, SKUs, sellers, or prices. Hosts pass structured-output schemas and finish tools; hosts validate and persist. The SKU ⇄ seller/channel invariant lives in `zik_zak`.
- **Rationale**: Plan §5.1, §9.3, §10.1; issue #1 acceptance criterion.
- **Alternatives considered**: convenience order-aware helpers — rejected (would drag domain types into the package and invert the dependency direction).

## D11 — pi types and dependency declaration

- **Decision**: Where pi defines a type (Storage, Extension, Task, Submission, tool results, messages), the package uses it. If it imports pi types directly it declares `pi` in `pubspec.yaml` rather than relying on transitive visibility (plan §4 compile-time rule).
- **Alternatives considered**: re-exporting pi types through `zuraffa_agent` only — insufficient, hosts and this package both need direct, honest dependency edges.

## D12 — Platform scope

- **Decision**: macOS and iOS hosts only. No Android scaffolding. iOS offscreen WKWebView is documented as an in-app webview, not a background execution guarantee; scheduled host work must respect suspension (plan §13).
- **Alternatives considered**: Android — excluded by product decision (plan §14).

## D13 — Test and verification strategy

- **Decision**: Pure-Dart in-memory adapters for all contract tests; canary corpus for leak tests; architecture fitness tests over imports/exports; restart/crash tests at commit boundaries for durability claims; `dart test -j 2` / `flutter test --concurrency=2` per fleet rule. A green Dart-only suite is necessary but not sufficient for any parity-flavored claim (plan §6.4) — but this package makes no upstream-parity claim; its claims are behavioral (security, durability, catalog fidelity) and are proven by the listed suites.
- **Alternatives considered**: example-driven demonstration only — rejected (not evidence).