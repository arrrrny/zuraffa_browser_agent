# Contract: BrowserAgent Public Composition

This is a composition contract, not a declaration of substitute types for unfinished dependencies. Final Dart type names and signatures for the runtime facade, submissions, watches, tasks, tools, and outcomes are bound from `zuraffa_agent#204/#205`, pi#12, and `zuraffa_browser_core#2` when those dependencies land. This package must not invent `Pi*` aliases or local lookalike records for them.

## Composition inputs

`BrowserAgent` receives, through explicit constructor/factory injection:

- the runtime facade published by `zuraffa_agent`;
- the `BrowserRpcClient` published by browser core;
- browser mission policy and tool/extension factories owned here;
- host/dependency-owned approval, artifact, audit, telemetry, notification, credential, storage, and redaction ports.

The package never constructs concrete WebViews, sockets, provider clients, global service locators, or a second runtime.

## Mission request

`BrowserMissionRequest` is input configuration owned by this package, not a durable mission/result state machine:

| Field | Contract |
| --- | --- |
| `policy` | `BrowserMissionPolicy`; default-deny tools and explicit scopes/budgets. |
| `prompt` | Host mission description. |
| `outputSchema` | Optional host-provided structured-output schema; interpreted by the host, not BrowserAgent. |
| `finishTool` | Optional host-provided finish-tool specification and validator. |
| `idempotencyKey` | Optional host key required at replay-unsafe persistence/effect boundaries. |

No request field contains an order, SKU, seller, price, or other product-domain type.

## Submission and outcome identity

`runMission` delegates submission to the injected runtime and returns the exact future/outcome object that facade publishes. Its return type is therefore dependency-owned and is finalized only after the wrapper contract is available.

Rules:

- No `BrowserAgentResult`, `BrowserMissionResult`, `PiOutcome`, `PiSubmissionOutcome`, wrapper enum, or “inner outcome” record is introduced.
- The host can resume, watch, inspect, and abort through the same runtime conversation/submission/task identities used for non-browser missions.
- `watches()` and related observation methods return the runtime's own stream/event types unchanged; no `PiWatchEvent` substitute is declared.
- `dispose()` releases only resources owned by this composition and does not erase durable runtime state.

## Tool resolution

Tool resolution is a pure planning operation over the core neutral catalog, advertised transport capabilities, and mission policy:

- it yields the agent-tool type owned by pi/Zuraffa;
- catalog schemas are carried unchanged;
- browser argument validation and operation bodies stay in core;
- unknown, unallowed, or unsupported capabilities fail before a model/provider call;
- no local wrapper duplicates a tool result or capability descriptor.

## Failure and parked-work representation

Failures, budget exhaustion, redaction refusal, approval requirements, unsupported capabilities, and challenge waits use the pi/Zuraffa task/submission/tool-outcome mechanisms. BrowserAgent contributes browser-namespaced reason codes and sanitized metadata through those mechanisms; it does not define a parallel result union or exception taxonomy.

| Reason | Required semantics |
| --- | --- |
| `browser.planning_failed` | Capability/allowlist/schema resolution failed before model execution; includes only sanitized reason metadata. |
| `browser.budget_exhausted` | Names the exhausted dimension and durable counter values. |
| `browser.redaction_failed` | Fails closed before model egress; never carries unredacted content. |
| `browser.challenge_parked` | References the dependency-owned parked tool/task and host escalation; no autonomous wake or bypass. |
| `browser.approval_required` | References normalized arguments and scope using the governance approval primitive; gates re-run after wake/extension change. |
| `browser.transport_unsupported` | Uses the core capability error during planning rather than failing after model execution. |

These are reason identifiers inside dependency-owned outcomes, not classes declared by this package.

## Domain-specific privacy without domain knowledge

Redaction/minimization rules are host-injected policy values. A host may supply rules for order identifiers, addresses, or other sensitive fields; BrowserAgent applies them opaquely and never learns an order schema or imports the host's domain package. Artifact references returned by the Zuraffa artifact port are passed through unchanged rather than redefined locally.

## Non-goals visible in the surface

No method accepts or returns product-domain entities, exposes raw cookies/credentials/session payloads, or enables arbitrary evaluation, cookie export, profile deletion, or account mutation by default.
