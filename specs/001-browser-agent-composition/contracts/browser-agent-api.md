# Contract: BrowserAgent Public API (exposed by this package)

All types are Dart. Signatures are normative for planning; implementation tasks realize them over pi/Zuraffa primitives.

## Constructor / composition

```dart
BrowserAgent({
  required ZuraffaAgentRuntime runtime,        // from zuraffa_agent (pi-based)
  required BrowserRpcClient browser,           // from zuraffa_browser_core (embedded or bridge)
  required BrowserAgentPorts ports,            // host-injected adapters (see host-ports.md)
});
```

- `runtime` and `browser` are interfaces owned by their packages; this package never constructs concrete WebViews, sockets, or provider clients.
- No global locator access anywhere in the package (plan §5.4).

## Mission submission

```dart
Future<PiSubmissionOutcome> runMission({
  required BrowserMissionRequest request,
});
```

`BrowserMissionRequest` (owned here):

| Field | Type | Notes |
|-------|------|-------|
| `policy` | `BrowserMissionPolicy` | See data-model.md; deny-by-default. |
| `prompt` | `String` | Host mission description. |
| `outputSchema` | `StructuredOutputSchema?` | Host-provided; BrowserAgent stays domain-blind (FR-017). |
| `finishTool` | `FinishToolSpec?` | Host-provided finish tool; payload validation is the host's. |
| `idempotencyKey` | `String?` | Required by the host at replay-unsafe boundaries; carried into durable records. |

Returns the pi/Zuraffa submission outcome object unchanged — never a new result type (FR-001).

## Tool surface

```dart
Iterable<AgentTool> resolveTools(BrowserMissionPolicy policy);
```

- Pure function of (neutral catalog, policy): filters by allowlist, risk hints, advertised transport capabilities; unknown names or unsupported capabilities → `PlanningFailed` typed error naming the tool/capability (FR-008).
- Returned tools carry catalog schemas verbatim; argument validation happens once, in the dispatcher (FR-006, FR-007).

## Lifecycle / durability

```dart
Stream<PiWatchEvent> watches();          // pi watches, re-exposed
Future<void> dispose();                  // releases browser transport handles it owns; durable state untouched
```

- Resume after restart: the host re-submits or resumes via pi primitives; `BrowserAgent` reads durable mission policy/state through the injected `Storage` — it owns no scheduler.

## Error taxonomy (typed outcomes, never raw exceptions to hosts)

| Outcome | Raised when |
|---------|-------------|
| `PlanningFailed(reason)` | capability/allowlist/schema resolution failure before any model call |
| `BudgetExhausted(dimension, counters)` | any mission budget tripped (durable) |
| `RedactionFailed(scope)` | sanitizer error — fail closed, no egress |
| `ChallengeParked(kind, parkedToolCallId)` | CAPTCHA/login/2FA/human decision parked durably |
| `ApprovalRequired(normalizedArgs, scope)` | gate trip; re-evaluated after wake/extension change |
| `TransportUnsupported(capability)` | advertised-capability mismatch (normally caught at planning) |
| `PiOutcome(inner)` | wrapped pi/Zuraffa runtime outcomes (finished/aborted/failed) |

## Non-goals visible in the surface

No method on `BrowserAgent` accepts or returns domain types (orders, SKUs, sellers, prices), no method exposes raw cookies/credentials/session payloads, and no method performs arbitrary JS evaluation, cookie export, profile deletion, or account mutation.