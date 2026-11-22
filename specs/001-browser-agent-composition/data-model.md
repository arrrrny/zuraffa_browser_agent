# Phase 1 Data Model: BrowserAgent Composition

Entities are grouped by ownership. "Owned here" means this package defines and serializes it; "consumed" means the shape is declared by a dependency and only referenced.

## Owned by this package

### BrowserMissionPolicy

The durable policy record for one mission. Serialized through the pi `Storage` adapter injected by the host.

| Field | Type | Notes |
|-------|------|-------|
| `allowedTools` | `Set<String>` | Default empty — deny-by-default (FR-009). Names come from the neutral catalog; unknown names fail planning. |
| `domainScope` | `UriPattern` (host-supplied matcher contract) | Normalized registrable domain(s) the mission may navigate. |
| `profileScope` | `Set<ProfileId>` | Browser profiles the mission may touch. |
| `pageScope` | `PageScopeRule` | Which pages/tabs are addressable; per-page sequential execution (FR-010). |
| `budgets` | `MissionBudgets` | Turn / tool-call / wall-clock / page-count / output-size (FR-011). Durable; checked after restart. |
| `redactionProfile` | `RedactionProfile` | Minimization + order-PII redaction rules; fail-closed on error (FR-015). |
| `artifactPolicy` | `ArtifactPolicy` | Inline-size threshold; larger results become artifact references (FR-016). |
| `approvalRequired` | `ApprovalRequirement` | Which classes (mutating, scope-crossing) require gates; bound to normalized args + scope (FR-013). |
| `schemaVersion` | `int` | Forward-compatible deserialization. |

Validation: `allowedTools ⊆ catalog tool names`; `budgets` all positive; `redactionProfile` always active (no "off"); scopes non-empty for restricted missions.

### MissionBudgets

| Field | Type | Default |
|-------|------|---------|
| `maxTurns` | `int` | finite (host-set) |
| `maxToolCalls` | `int` | finite |
| `maxWallClock` | `Duration` | finite |
| `maxPages` | `int` | finite |
| `maxOutputBytes` | `int` | finite |

Exhaustion → `BudgetExhausted` typed outcome naming the exhausted dimension and the durable counter values.

### RedactionProfile

- `minimizePageContent`: structural minimization before model egress (strip non-essential DOM/UI text; keep the extraction-relevant content).
- `piiRules`: order-PII matchers (order ids, addresses, emails, phones, session ids) applied in addition to credential patterns.
- `failClosed`: always true; a rule engine error produces `RedactionFailed`, never passthrough.

### ArtifactPolicy

- `inlineMaxBytes`: results above this become an `ArtifactReference` stored through the host's `AgentArtifactStore` port.
- `kinds`: screenshots, PDFs, downloads, oversized page results.

### Challenge/escalation record (pi-shaped)

| Field | Type | Notes |
|-------|------|-------|
| `parkedToolCallId` | `String` | The tool call that raised the challenge. |
| `kind` | `ChallengeKind` | `captcha` / `login` / `twoFactor` / `humanDecision`. |
| `hostPort` | `HumanEscalationPort` | Invoked with normalized context; never with secrets. |
| `signal` | pi signal/deadline | Durable park; resume only via host signal (FR-012, FR-018). |

### ArtifactReference

| Field | Type |
|-------|------|
| `store` | `AgentArtifactStore` (host port) |
| `id` | `String` |
| `mimeType` | `String` |
| `byteSize` | `int` |
| `retention` | policy from zuraffa_agent#205 adapters |

## Consumed (declared by dependencies — never redefined here)

| Type | Owner | Consumed for |
|------|-------|--------------|
| Neutral capability descriptor (name, description, input schema, mutability/risk hint, required capability, result contract) | zuraffa_browser_core#2 | Tool factory input (FR-006). |
| `BrowserRpcClient` + `EmbeddedBrowserRpcClient` / `BridgeBrowserRpcClient` + capability discovery | zuraffa_browser_core#2 | The transport seam (FR-008, FR-019). |
| pi `Storage`, `Extension`, `Task`, `Submission`, tool results, messages, signals, parked tools, approval gates | pi (via zuraffa_agent) | Everything durable/executional (FR-003, FR-018). |
| `beforeTool` / `afterTool` / provider-request hook contracts, approval ports, artifact store, audit/telemetry adapters, `AgentCredentialProvider`, human-notification port | zuraffa_agent#205 | Governance composition (FR-020, FR-021). |

If a needed shape is absent from a dependency's declared contract, the gap is filed against that repo — this package does not define a local variant (spec Assumptions).

## State transitions (mission lifecycle, all via pi primitives)

```text
planned (capability check ok)
  → running ⇄ parked (challenge/approval; durable)
  → finished | aborted | failed(typed) | budgetExhausted(typed)
```

- `planned → running`: allowlist + capability + scope checks pass.
- `running → parked`: challenge or approval gate; budget counters frozen.
- `parked → running`: host signal resumes; gates re-evaluated against current args/scope (extension-change rule).
- Any → `failed(typed)`: sanitizer failure, transport error surfaced by core, schema-invalid finish payload (zero persistence).
- Restart at any point: durable state restores the same node; no duplicate persistence, no replay-unsafe duplicate effects (host idempotency keys required at those boundaries).