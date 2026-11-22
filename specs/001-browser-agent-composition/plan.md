# Implementation Plan: BrowserAgent Composition over Zuraffa Agent and Browser Core

**Branch**: `001-browser-agent-composition` | **Date**: 2026-11-22 | **Spec**: [spec.md](./spec.md)

**Input**: Feature specification from `/specs/001-browser-agent-composition/spec.md`

## Summary

Create the public `zuraffa_browser_agent` Dart package: a thin `BrowserAgent` composition over `zuraffa_agent` (generic pi-based runtime) and `zuraffa_browser_core`'s transport-neutral `BrowserRpcClient`. The package contributes browser-specific agent tools generated from the core's neutral capability catalog, browser mission policies (default-deny allowlists, scopes, budgets, redaction, artifact handling), durable challenge escalation, and host-integration ports — while owning no agent loop, no browser implementation, and no domain entities.

The technical approach is dependency-inversion-first: every capability (catalog, RPC seam, governance hooks, storage, approvals, telemetry, artifacts) is consumed from a declared contract owned by a dependency repo (`zuraffa_browser_core#2`, `zuraffa_agent#205`); this package only adapts and composes. Verification is pure-Dart in-memory contract testing plus architecture fitness tests over imports and public exports.

## Technical Context

**Language/Version**: Dart 3.x (sound null safety; pure-Dart core with Flutter only where the browser core's platform seam requires it)

**Primary Dependencies**: `zuraffa_agent` (pi-based runtime + governance hooks), `zuraffa_browser_core` (neutral capability catalog + `BrowserRpcClient`), `pi` (declared directly when its types are imported)

**Storage**: None owned. Durable storage, transcript persistence, and task state are pi `Storage` adapters injected by the host; this package only serializes its mission policy records through them.

**Testing**: `dart test -j 2` for pure-Dart suites; `flutter test --concurrency=2` only for platform-coupled tests (macOS/iOS lifecycle). Contract suites run against pure-Dart in-memory fakes; characterization fixtures for the catalog→tool mapping.

**Target Platform**: macOS and iOS hosts (embedded in host applications); library itself is platform-neutral Dart.

**Project Type**: Dart package (library).

**Performance Goals**: Tool resolution and policy evaluation are O(catalog size) per mission start; no per-call overhead beyond pi hook costs. Budgets are checked per tool call with negligible overhead.

**Constraints**: No agent loop / provider client / transcript store / compaction / scheduler / WebView / domain entity may exist in the package (architecture fitness test). Four-package import rules enforced in CI. Test concurrency capped at 2.

**Scale/Scope**: One package, one public class (`BrowserAgent`) plus policy/port/tool-factory libraries; initial catalog is the browser core's shared capability set (browser, page, profiles, sessions, recipes, captures).

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

The repository constitution (`.specify/memory/constitution.md`) is an unfilled template — it carries no ratified principles, so the gate evaluates against the higher-order constraints that do exist: the fleet `AGENTS.md` rules and the normative architecture plan.

| Gate | Source | Status |
|------|--------|--------|
| No hand-written domain/application types in Zuraffa repos — use `zfa` generators | Fleet AGENTS.md (ZFA GENERATION rule) | PASS — planning artifacts only in this task; implementation tasks will use `zfa entity create` / `zfa make` for entities/use cases/providers, never hand-written equivalents |
| Package is `zuraffa_browser_agent`, not tools-only; not a third agent loop | Architecture plan §1, §9.2 | PASS — spec FR-001..FR-003 encode exactly this boundary |
| Dependency direction: only this package imports both `zuraffa_agent` and browser core | Architecture plan §4 | PASS — spec SC-008 / FR-003 make it a CI fitness test |
| Consume declared contracts of core#2 and zuraffa_agent#205; never redefine them | User instruction; plan §8.3, §7.4 | PASS — spec Assumptions make gaps a filing against the owner, not local redefinition |
| Default-deny, budgets, redaction, escalation, parking, transport parity | User instruction; plan §9.4, §10.3 | PASS — FR-009..FR-019, SC-002/003/005 |
| No hand merge; forklift pool owns PRs | Fleet AGENTS.md (pool-pr rule) | PASS — this run commits planning artifacts to master per gh-triage Phase 3; any future code lands via a forklift-reviewed PR |
| Test concurrency ≤ 2 | Fleet AGENTS.md (test-concurrency rule) | PASS — carried into Technical Context |

**Post-design re-check (Phase 1):** No violations introduced. The design adds no new dependency, no parallel runtime, and no domain type.

## Project Structure

### Documentation (this feature)

```text
specs/001-browser-agent-composition/
├── spec.md              # Feature specification
├── plan.md              # This file
├── research.md          # Phase 0 — resolved decisions
├── data-model.md        # Phase 1 — entities, ports, state
├── quickstart.md        # Phase 1 — validation guide
├── contracts/           # Phase 1 — consumed + exposed contracts
│   ├── browser-agent-api.md        # BrowserAgent public API (exposed)
│   ├── mission-policy.md           # MissionPolicy record + budget/approval types (owned)
│   ├── host-ports.md               # Ports this package consumes (defined by hosts/zuraffa_agent)
│   └── tool-contract-suite.md      # Shared embedded/bridge tool contract suite (consumed from core)
└── checklists/
    └── requirements.md  # Spec quality checklist
```

### Source Code (repository root)

```text
# Dart package (created by implementation tasks, not this planning run)
lib/
├── src/
│   ├── browser_agent.dart            # BrowserAgent — thin composition
│   ├── catalog/                      # neutral-catalog → agent-tool factory
│   ├── policy/                       # allowlist, scopes, budgets, redaction, artifact policy
│   ├── escalation/                   # challenge ports + durable parking
│   └── ports/                        # host port interfaces consumed from zuraffa_agent/hosts
├── zuraffa_browser_agent.dart        # public barrel
test/
├── architecture/                     # import/export fitness tests (SC-004, SC-008)
├── catalog/                          # catalog→tool mapping characterization
├── policy/                           # deny-by-default, budgets, redaction canaries
├── escalation/                       # parking, wake, gate re-evaluation
├── contract/                         # shared embedded/bridge tool contract suite runs
└── integration/                      # scripted mission over in-memory adapters (SC-001)
```

## Phases

- **Phase 0** → `research.md` (decisions below, consolidated from the architecture plan and dependency issues)
- **Phase 1** → `data-model.md`, `contracts/`, `quickstart.md`
- **Phase 2** → `tasks.md` (generated later by `/speckit-tasks`, NOT by this plan)