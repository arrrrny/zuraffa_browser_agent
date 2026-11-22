# Feature Specification: BrowserAgent Composition over Zuraffa Agent and Browser Core

**Feature Branch**: `001-browser-agent-composition`

**Created**: 2026-11-22

**Status**: Draft

**Input**: User description: "Create the public `zuraffa_browser_agent` package containing browser tools, durable policies, extensions, and the `BrowserAgent` composition over `zuraffa_agent` and `zuraffa_browser_core` (arrrrny/zuraffa_browser_agent#1, normative architecture: https://github.com/arrrrny/zuraffa_browser/blob/master/handoffs/2026-11-22-pi-zuraffa-browser-agent-architecture-plan.md)."

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Thin BrowserAgent composition (Priority: P1)

An embedded host application (for example the `zik_zak` order-extraction flow) registers `BrowserAgent` with its composition root, hands it a host mission request with a structured-output schema and a finish tool, and receives the pi/Zuraffa runtime outcome. The agent drives browser capabilities through the canonical dispatcher without the host ever importing browser internals or pi types.

**Why this priority**: This is the package's reason to exist — a reusable, secure browser-agent composition that hosts can embed without forking an agent loop. Everything else is a policy or transport refinement of this path.

**Independent Test**: Compose `BrowserAgent` over a pure-Dart in-memory `zuraffa_agent` runtime and a capability-advertising `BrowserRpcClient` fake; run one scripted mission that navigates, reads, and finishes; assert the returned value is the pi/Zuraffa submission outcome and that no parallel mission state machine exists in the package.

**Acceptance Scenarios**:

1. **Given** a host mission request with a structured-output schema and finish tool, **When** `BrowserAgent` runs it, **Then** the call returns pi/Zuraffa runtime outcomes (conversation/submission/watches/tasks) rather than a new mission result type.
2. **Given** a browser capability the transport does not advertise, **When** mission planning resolves tools, **Then** the capability is absent from the plan and the failure is raised during planning, not after a model call.
3. **Given** a host that imports `pi` types directly in its own code, **When** it consumes `zuraffa_browser_agent`, **Then** the package declares and re-exports only the pi types it genuinely uses, and the host composes without importing `zuraffa_browser_core` internals.

---

### User Story 2 - Catalog-driven browser tool surface (Priority: P1)

The browser core publishes neutral capability descriptors (stable name, description, input schema, mutability/risk hint, required capability, result contract). `BrowserAgent` maps that catalog into pi/Zuraffa agent tools plus agent-specific policy, so there is exactly one schema source and one dispatcher for both the agent surface and the store app's MCP surface.

**Why this priority**: Without catalog-driven generation, this package would fork into hand-written tool wrappers that drift from the store MCP surface — the exact duplication the architecture forbids.

**Independent Test**: Feed a fixture neutral catalog with two capabilities into the tool factory and assert the generated agent tools carry the catalog's schemas verbatim, apply agent policy hints, and dispatch through the canonical dispatcher; assert no schema or browser operation body is redefined in this package.

**Acceptance Scenarios**:

1. **Given** a neutral capability descriptor, **When** the agent tool factory runs, **Then** the produced pi/Zuraffa tool reuses the descriptor's name, input schema, risk hint, and result contract without duplication.
2. **Given** a descriptor flagged as mutating or high-risk, **When** a restricted mission's allowlist is computed, **Then** the tool is denied by default and requires an explicit allowlist entry plus approval gate.
3. **Given** two transports (embedded and bridge) implementing the same capability, **When** the same tool contract suite runs, **Then** both pass for shared capabilities and each advertises its own unsupported set.

---

### User Story 3 - Security defaults: deny-by-default, budgets, redaction, escalation (Priority: P1)

Every restricted mission runs with default-deny tool policy, explicit allowlists, domain/profile/page scopes, per-page sequential execution, and turn/tool/time/page/output budgets. CAPTCHA, login, 2FA, and human decisions escalate to host ports and durably park the work. Cookies, credentials, session payloads, and unauthorized page PII never reach transcripts, telemetry, exceptions, or artifacts.

**Why this priority**: This package may process authenticated pages and sensitive host data; a single leak or autonomous-bypass path makes the whole composition unsafe to embed. Security defaults are release-blocking, not hardening.

**Independent Test**: Drive a scripted restricted mission through pure-Dart fakes and assert: an unlisted tool is denied; a budget (turn, tool-call, wall-clock, page-count, output-size) trips and produces a typed outcome that survives restart; a parked challenge never auto-bypasses; and a canary cookie/session/PII value never appears in transcript, telemetry, exception, or artifact outputs.

**Acceptance Scenarios**:

1. **Given** a mission with no allowlist, **When** any browser tool is requested, **Then** the tool is denied and no browser execution occurs.
2. **Given** a mission that exhausts its turn/tool/time/page/output budget, **When** the budget is checked after restart and disposal, **Then** the budget state is durable and the outcome is typed (not a crash or silent continuation).
3. **Given** a CAPTCHA, login, or 2FA wall, **When** the challenge fires, **Then** the relevant tool/task is durably parked, the host escalation port is invoked, and no autonomous bypass or credential harvesting is attempted.
4. **Given** a page containing host-classified PII and a canary session cookie, **When** results flow to model egress, transcript, telemetry, exception, or artifact, **Then** minimization and redaction remove them, and a failed sanitizer fails closed with a typed outcome instead of leaking unredacted content.
5. **Given** a mutating action or a scope crossing (domain/profile/page), **When** the action is about to run, **Then** an approval gate bound to normalized tool arguments and scope is required, and the gate re-runs after wake or extension change.

---

### Edge Cases

- **Transport capability gap**: an embedded transport lacks a capability the bridge has — the tool is absent from the plan (US1 scenario 2), never a runtime surprise.
- **Bridge transport downgrade**: the bridge app is an older version advertising fewer capabilities; mission planning fails typed before any model call.
- **Extension change while parked**: a parked approval resumes after an extension or policy change — the gate is re-evaluated against current arguments and scope.
- **Replay-unsafe browser effects**: durable replay cannot make merchant side effects exactly-once; missions that cross replay-unsafe boundaries produce typed outcomes requiring host-side idempotency keys, and restart tests prove no duplicate persistence or duplicate effects.
- **Sanitizer failure**: redaction/minimization failure never falls through to the model — typed fail-closed outcome.
- **Oversized results**: screenshots, PDFs, downloads, and large page results are converted to artifact references by policy rather than inlined into the transcript.
- **iOS suspension**: an offscreen WKWebView is not a background execution guarantee; scheduled host work degrades to typed outcomes rather than promising suspended execution.
- **Scope drift**: a mission attempts to cross domain/profile scope mid-run — denied and approval-gated, not silently clamped.
- **Concurrent missions on one page**: actions on a shared page/WebView serialize; parallelism is permitted only across proven isolated contexts.
- **Null/empty capability catalog**: mission planning produces an empty tool set and the mission fails typed rather than running with implicit defaults.

## Requirements *(mandatory)*

### Functional Requirements

#### Composition and boundaries

- **FR-001**: `BrowserAgent` MUST be a thin composition over `zuraffa_agent` and `BrowserRpcClient`, returning pi/Zuraffa runtime outcomes (conversations, submissions, watches, tasks) rather than introducing a mission state machine, session model, or result type.
- **FR-002**: The package MUST NOT contain an agent loop, provider client, transcript store, compaction, durable scheduler, WebView implementation, or order/invoice/seller/SKU/price entity.
- **FR-003**: The package MUST declare `pi` as a direct dependency when it imports pi types, and MUST NOT reimplement pi-owned algorithms (storage, tasks, submissions, extension, telemetry contracts).
- **FR-004**: `BrowserAgent` MUST NOT make any host application import `zuraffa_browser_core` internals; hosts compose against `BrowserAgent` and the `BrowserRpcClient` seam only.
- **FR-005**: Host applications own the final composition root: credentials, secure storage, egress config, durable storage, audit/telemetry/artifact/notification adapters, and domain schemas — the package consumes those injected contracts, never a global service locator or a concrete WebView/socket/provider SDK.

#### Catalog-driven tools

- **FR-006**: Browser extension/tool factories MUST be generated from the browser core's neutral capability catalog (name, description, input schema, mutability/risk hint, required capability, result contract); the package MUST NOT duplicate JSON Schemas, argument validation, or browser operation bodies.
- **FR-007**: One canonical capability definition MUST drive both the agent tool surface here and the store app's MCP metadata mapping in the core; argument validation, permission checks, and result truncation MUST exist once.
- **FR-008**: Every `BrowserRpcClient` transport MUST advertise an explicit capability set, and mission planning MUST fail typed for unsupported capabilities before any model call.

#### Security defaults

- **FR-009**: Restricted missions MUST deny tools by default and receive explicit allowlists; arbitrary evaluation, cookie export, profile deletion, and account-mutating tools MUST NOT be available by default.
- **FR-010**: Restricted missions MUST enforce domain/profile/page scopes and per-page sequential execution on a shared page/WebView; parallelism is permitted only across proven isolated contexts.
- **FR-011**: Every mission MUST carry scoped budgets — turn, tool-call, wall-clock, page-count, and output-size — that are durable across restart and disposal and produce typed outcomes when exhausted.
- **FR-012**: CAPTCHA, login, 2FA, and required human decisions MUST escalate to host ports and durably park the relevant tool/task; no autonomous bypass or credential harvesting may exist anywhere in the package.
- **FR-013**: Approval gates MUST bind to normalized tool arguments and scope, MUST be required for mutating actions and scope crossings, and MUST re-run after wake or extension change.
- **FR-014**: Cookies, credentials, session payloads, and secure-store values MUST be kept out of transcripts, telemetry, exceptions, and artifacts by default.
- **FR-015**: Page content MUST be minimized before model egress using host-injected redaction/minimization policies, which may classify host-classified PII or other host-specific sensitive data without BrowserAgent learning that domain; a failed sanitizer MUST fail closed through the dependency-owned typed outcome channel and unredacted content MUST never fall through to the model.
- **FR-016**: Browser artifacts (screenshots, PDFs, downloads, large page results) MUST be handled through artifact references and oversized-result policy rather than being inlined into transcripts.

#### Host integration and parity

- **FR-017**: Host-provided structured-output schemas and finish tools MUST be supported so hosts can validate orders without making `BrowserAgent` aware of orders, SKUs, sellers, or prices; invalid or schema-failing payloads MUST cause zero persistence and a typed outcome.
- **FR-018**: Browser-specific durable tasks, approval gates, mission policies, result projections, and artifact handling MUST compose pi/Zuraffa primitives (signals, memoized deadlines, tool-result hooks, artifact stores) rather than adding a parallel engine.
- **FR-019**: Embedded (`EmbeddedBrowserRpcClient`) and bridge (`BridgeBrowserRpcClient`) transports MUST pass the same tool contract suite for shared capabilities, with differences explicitly advertised and tested.
- **FR-020**: Tool allowlists, argument guards, and risk classification MUST use the governance hooks and primitives declared by `zuraffa_agent` (pi `beforeTool`/`afterTool`); this package MUST NOT create a second policy engine.
- **FR-021**: Audit, telemetry, and usage observation MUST compose the adapters declared by `zuraffa_agent`, without receiving secret payloads by default.

### Key Entities *(include if feature involves data)*

- **BrowserAgent**: the package's public composition. Wraps a `zuraffa_agent` runtime and a `BrowserRpcClient`; owns browser policy, tool factories, and mission defaults. Holds no domain entities.
- **Neutral capability descriptor** (consumed, owned by browser core): stable name, description, input schema, mutability/risk hint, required capability, result contract. Read-only input to the tool factory.
- **Browser mission policy** (owned here): allowlist, domain/profile/page scopes, per-page serialization rule, budgets (turn/tool/time/page/output), redaction/minimization rules, artifact policy. Serialized durably.
- **Challenge/escalation record** (owned here, pi-shaped): the parked tool/task plus the host escalation port invocation; durably parked until a host signal resumes or aborts it.
- **Artifact reference** (owned here as policy, stored by host adapters): pointer plus metadata for screenshots, PDFs, downloads, and oversized results.
- **Host ports consumed**: credential/secure-storage, egress/HTTP, durable storage/platform environment, audit, telemetry, artifact store, approval presentation, human notification, domain output schemas/validators.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: A host can complete one end-to-end scripted browser mission (navigate → read → structured finish) against pure-Dart in-memory adapters with zero production browser or network dependencies, and the mission returns the pi/Zuraffa submission outcome unchanged.
- **SC-002**: 100% of restricted-mission denials, budget trips, and challenge parks are observable as typed outcomes in tests — zero crashes, zero silent continuations, zero autonomous bypasses across the whole test matrix.
- **SC-003**: A canary corpus (cookies, credentials, session payloads, host-classified PII) injected into fixtures appears in zero transcripts, telemetry events, exception messages, or artifacts after a full mission + restart + sanitizer-failure test run.
- **SC-004**: The package contains no agent loop, provider client, transcript store, compaction, durable scheduler, WebView code, or domain entity — verified by an architecture fitness test over imports and public exports.
- **SC-005**: The same tool contract suite passes against both `EmbeddedBrowserRpcClient` and `BridgeBrowserRpcClient` for every shared capability, and every unsupported capability is advertised and fails during planning (zero post-model-call capability failures).
- **SC-006**: A single neutral catalog fixture generates the full agent tool surface with byte-identical input schemas to the catalog, and no schema or browser operation body is duplicated in this package.
- **SC-007**: Durable budget and approval state survives process restart and tab/WebView disposal in tests, and gate re-evaluation after extension change denies or approves per current policy — proven by restart tests, not asserted in prose.
- **SC-008**: The four-package compile-time import rules hold in CI (pi imports nothing Zuraffa; browser core imports no agent code; this package is the only browser package importing `zuraffa_agent`), and the package publishes with no `publish_to: none` and no signing assets.

## Assumptions

- **Dependency contracts are consumed, not defined here.** The neutral catalog and `BrowserRpcClient` are declared by `arrrrny/zuraffa_browser_core#2`; governance hooks, approval ports, artifact/audit adapters are declared by `arrrrny/zuraffa_agent#205`. This spec consumes those declared contracts; if a needed shape is absent, it is a gap filed against the owning repo, not redefined here. Both `Depends on:` declarations on the issue are normative and preserved.
- `zuraffa_agent` is the generic wrapper over pi; `BrowserAgent` never re-implements its runtime, and pi types are used wherever pi already defines a type (no parallel Mission/Session/Task/ChatMessage models).
- Browser platform scope is macOS and iOS only; Android is excluded by product decision.
- The store app (`zuraffa_browser`) keeps its MCP host, bridge wiring, and GUI; this package supplies the agent surface over the same canonical dispatcher.
- Hosts are `zik_zak` (first consumer) and future embedded apps; domain models (orders, SKUs, sellers, prices) stay in hosts.
- Domain-specific redaction matchers, minimization policies, output schemas, and validators are host-injected values. BrowserAgent applies them opaquely and never learns an order or other product-domain schema.
- Restricted mission defaults are the norm; any permissive mode is an explicit host opt-in that still cannot enable arbitrary evaluation, cookie export, profile deletion, or account-mutating tools.
- Durable replay cannot make arbitrary merchant side effects exactly-once; replay-unsafe boundaries require host idempotency keys.
- `pub.zuzu.dev` is the controlled prerelease channel; pub.dev publication is a separate release decision.
- Planning artifacts are produced in this disposable `/tmp` clone and committed to `master`; implementation (if ever) happens on a feature branch through the forklift PR pipeline, never by hand.