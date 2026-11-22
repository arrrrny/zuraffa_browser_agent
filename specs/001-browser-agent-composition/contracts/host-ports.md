# Contract: Host Ports Consumed (defined by hosts / zuraffa_agent#205)

`BrowserAgent` depends only on these interfaces. Concrete adapters are injected by the host composition root. Pure-Dart in-memory implementations exist for every port so contract tests run without production dependencies (zuraffa_agent#204/#205 contract-test discipline).

| Port | Shape (planning-level) | Owner of the contract | Used for |
|------|------------------------|-----------------------|----------|
| `AgentCredentialProvider` / secure store | get/put/delete secrets by key; never logged | zuraffa_agent#205 | credential injection; exclusion from transcripts/telemetry (FR-014) |
| `AgentArtifactStore` | put(bytes, mimeType) → ArtifactReference; retention policy | zuraffa_agent#205 | oversized results, screenshots, PDFs, downloads (FR-016) |
| `AgentAuditSink` | append(structured event); secrets denied by default | zuraffa_agent#205 | audit trail (FR-021) |
| Telemetry/failure-reporting adapter (pi telemetry contracts) | typed spans/events/metrics; no payload secrets | zuraffa_agent#205 + pi | observability (FR-021) |
| `AgentApprovalPresenter` + human-notification port | present(normalizedArgs, scope) → decision; notify(challenge) | zuraffa_agent#205 | approvals + challenge escalation (FR-012, FR-013) |
| Durable `Storage` + platform environment | pi `Storage` adapter | pi (host injects JSONL/SQLite/memory) | mission policy + budget + park state |
| Egress/HTTP configuration | proxy/direct, allowlists | host | transport construction (browser core consumes) |
| Domain output schema + validator + finish tool | host-owned | host (`zik_zak` for orders) | structured finish validation (FR-017) |

Rules:

- The package never reaches for a concrete implementation (no `GetIt.instance`, no `WebView`, no socket, no provider SDK).
- Ports carry no secret payloads in events or exceptions; redaction/minimization sits between page content and every port.
- A port that fails (e.g., artifact store write error) produces a typed outcome; sanitizer-adjacent failures fail closed.