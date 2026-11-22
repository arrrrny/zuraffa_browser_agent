# Contract: Mission Policy and Budget Records (owned by this package)

Serialized forms are JSON-compatible maps persisted through the pi `Storage` adapter. Field names are normative.

## BrowserMissionPolicy

```json
{
  "schemaVersion": 1,
  "allowedTools": ["browser.navigate", "page.read"],
  "domainScope": { "domains": ["example.merchant.test"] },
  "profileScope": ["seller-scoped-profile-id"],
  "pageScope": { "mode": "mission-pages", "sequentialPerPage": true },
  "budgets": {
    "maxTurns": 40,
    "maxToolCalls": 120,
    "maxWallClockSeconds": 900,
    "maxPages": 5,
    "maxOutputBytes": 1048576
  },
  "redactionProfile": { "minimizePageContent": true, "orderPii": true, "failClosed": true },
  "artifactPolicy": { "inlineMaxBytes": 262144 },
  "approvalRequired": { "mutating": true, "scopeCrossing": true }
}
```

Validation (planning time):

- `allowedTools` entries must exist in the neutral catalog; unknown → `PlanningFailed`.
- `budgets.*` must be positive and finite.
- `redactionProfile.failClosed` is always true (no off switch).
- `pageScope.sequentialPerPage` defaults true for restricted missions.

## Challenge/escalation record

```json
{
  "parkedToolCallId": "call-17",
  "kind": "captcha",
  "escalation": { "port": "HumanEscalationPort", "ref": "host-notification-3" },
  "deadline": "2026-11-22T12:00:00Z",
  "signal": "pi-signal-id"
}
```

Resume is only via the host signal; deadline expiry produces a typed timeout outcome, never an auto-bypass.

## ArtifactReference

```json
{
  "store": "AgentArtifactStore",
  "id": "art-9f2",
  "mimeType": "image/png",
  "byteSize": 184320,
  "retention": "policy:agent-artifacts"
}
```

## State transitions

See data-model.md. Budget counters are part of the durable mission record and are re-read after restart; the mission cannot run with reset counters.