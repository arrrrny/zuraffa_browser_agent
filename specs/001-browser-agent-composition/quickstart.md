# Quickstart: Validating the BrowserAgent Composition

A runnable validation guide for reviewers and implementers. Every scenario uses pure-Dart in-memory adapters; no production browser, socket, or network is required. Commands assume the implementation tasks have landed the package skeleton.

## Prerequisites

- Dart 3.x SDK (`dart --version`).
- Dependencies resolved: `dart pub get` (the package depends on `zuraffa_agent`, `zuraffa_browser_core`, and — where pi types are imported — `pi`, declared directly).
- Test concurrency capped at 2 (`dart test -j 2`) per fleet rule.

## Scenario 1 — Scripted mission returns a pi/Zuraffa outcome (SC-001)

1. Build an in-memory `ZuraffaAgentRuntime` fake and an in-memory `BrowserRpcClient` fake that advertises `browser.navigate`, `page.read`, and a host finish tool.
2. Compose `BrowserAgent`, submit a mission whose `outputSchema` is a trivial host schema.
3. Expected: `runMission` completes; the returned object is the pi/Zuraffa submission outcome; no type named `*MissionResult` or `*BrowserAgentResult` exists in the package's public API (checked by the architecture fitness test).

## Scenario 2 — Deny-by-default (FR-009)

1. Submit a mission whose policy has an empty `allowedTools`.
2. Expected: `PlanningFailed` naming the requested tool; zero browser operations executed (the fake records zero calls).

## Scenario 3 — Budget exhaustion survives restart (FR-011, SC-007)

1. Submit a mission with `maxToolCalls: 2`; let it consume the budget mid-mission.
2. Crash/restart the runtime (fresh `BrowserAgent` over the same durable `Storage` fake).
3. Expected: `BudgetExhausted('toolCalls', …)` typed outcome with durable counter values; the mission cannot continue with reset counters.

## Scenario 4 — Challenge parks durably, never auto-bypasses (FR-012)

1. Script the fake transport to surface a CAPTCHA/login challenge mid-mission.
2. Expected: `ChallengeParked(kind, parkedToolCallId)`; the host escalation port is invoked exactly once; the parked tool/task survives restart; resume happens only on the host signal.

## Scenario 5 — Canary leak test (FR-014, FR-015, SC-003)

1. Seed fixtures with a canary cookie, a canary credential, a canary session payload, and canary order PII in page content.
2. Run a full mission plus restart plus a forced sanitizer failure.
3. Expected: canaries appear in zero transcripts, telemetry events, exception messages, or artifact bytes; the sanitizer failure produces `RedactionFailed` and no model egress occurred.

## Scenario 6 — Catalog fidelity and transport parity (FR-006, FR-019, SC-005/SC-006)

1. Load the core's neutral catalog fixture; call `resolveTools` with a permissive test policy.
2. Expected: each generated tool's input schema is byte-identical to its descriptor; no schema literal exists in this package.
3. Run the shared tool contract suite against both the in-memory embedded fake and the bridge fake: all shared-capability rows green; unsupported capabilities fail at planning.

## Scenario 7 — Structured finish and domain ignorance (FR-017)

1. Submit a mission with a host finish tool that rejects invalid payloads.
2. Expected: invalid payload → typed outcome, zero persistence (the host validator records zero writes); `BrowserAgent` never names an order/SKU/seller/price type.

## Scenario 8 — Architecture fitness (SC-004, SC-008)

1. Run the architecture tests over imports and public exports.
2. Expected: no import of a WebView, socket, provider SDK, or domain package; no `Mission`/`Session`/`Task` parallel model; pi imported directly only where used; the four-package import rules hold.

## Definition of done for planning

All eight scenarios have a corresponding test file path in the implementation task list (generated later by `/speckit-tasks`), and this guide's commands are wired into the package CI with `--concurrency=2` / `-j 2`.