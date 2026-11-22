# Contract: Shared Tool Contract Suite (consumed from zuraffa_browser_core#2)

One suite, run against every transport that claims a capability. This package runs the agent-surface leg; the store app runs the MCP-surface leg. Both legs execute the same capability cases.

## Suite shape (per capability)

For each capability in the neutral catalog:

1. **Descriptor fidelity** — the agent tool built from the descriptor exposes the descriptor's name, input schema (byte-identical), risk hint, and result contract.
2. **Happy path** — invoke through the transport with a valid payload; result matches the descriptor's result contract.
3. **Validation rejection** — malformed payload fails with the core's error taxonomy, before the browser operation body runs.
4. **Capability advertisement** — a transport that does not advertise the capability fails at planning through the core/runtime outcome channel with reason `browser.transport_unsupported`, not mid-mission; no local transport-result class is introduced.
5. **Policy interaction** — mutating/risk-flagged capabilities are denied by default under a restricted policy and require allowlist + approval.

## Matrix

| Capability group | Embedded | Bridge | Notes |
|------------------|----------|--------|-------|
| Browser (navigate, state) | required | required | shared cases |
| Page (read, extract) | required | required | shared cases |
| Profiles / sessions | advertised subset | advertised subset | differences explicit + tested |
| Recipes | advertised subset | advertised subset | differences explicit + tested |
| Captures (screenshot/PDF/download) | required | required | artifact path asserted (FR-016) |
| Eval / cookie export / profile deletion / account-mutating | absent by default everywhere | absent by default everywhere | presence requires explicit host opt-in that this package's default policy never grants |

## Ownership

- Suite definition + neutral catalog: zuraffa_browser_core#2 (this package consumes them unchanged).
- This package contributes: the agent-surface adapter leg and the policy-interaction cases.
- Store app contributes: the MCP-surface leg.
- A capability may not ship unless its row is green on every transport that advertises it; an advertised-but-red capability blocks planning for restricted missions.