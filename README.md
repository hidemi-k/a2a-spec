# A2A Specification (`a2a-spec`)

**[🇯🇵 日本語版はこちら / Japanese version](README.ja.md)**

> **The Domain-Specific Protocol & Execution Profile for Autonomous Network Infrastructure Control**

[![Specification](https://img.shields.io/badge/A2A-Specification-0A84FF.svg)](https://github.com/hidemi-k/a2a-spec)
[![License](https://img.shields.io/badge/License-Apache--2.0-blue.svg)](LICENSE)

A2A stands for **Agent-to-Agent**, representing autonomous multi-agent coordination across heterogeneous infrastructure. It also stands for **Autonomy-to-Action**, defining the deterministic execution, safety verification, and governance layer required for enterprise network operations.

---

## 🏛 Ecosystem Architecture

`a2a-spec` defines the core schemas, lifecycles, and interface contracts that govern the entire A2A Ecosystem:

```text
===================================================================
                  A2A Ecosystem Architecture
===================================================================

  a2a-spec                (Public, Apache-2.0)  <-- THIS REPOSITORY
                                                     (defines the contracts below)

  [ Platform Layer ]
  ├── a2a-containment-core  (MIT, not maintained)    <-- Reference prototype
  └── a2a-governance        (Private, license TBD)  <-- Policy & Human-in-the-Loop Safety

  [ Vendor Core Layer ]
  ├── a2a-ceos-core         (MIT, not maintained)    <-- Reference prototype (Arista)
  ├── a2a-junos-core        (Private, license TBD)  <-- Juniper adapter
  ├── a2a-iosxe-core        (Private, in dev)       <-- Cisco IOS-XE adapter
  ├── a2a-iosxr-core        (Private, in dev)       <-- Cisco IOS-XR adapter
  └── a2a-nxos-core         (Private, in dev)       <-- Cisco NX-OS adapter

  [ Community Tooling ]
  └── a2a-console           (Planned, MIT)         <-- Shared multi-vendor UI

  [ Integration Layer ]
  ├── a2a-splunk            (Public, MIT)           <-- Observability & Telemetry
  └── a2a-interconnect      (Public, MIT)           <-- Multi-cloud interconnect
                                                         negotiation (OpenAPI 3.0
                                                         Interconnect / Connection
                                                         Coordinator API)
===================================================================
```

> Note: `maf-ebpf-sase` and `maf-netconf-rag-gui` are **not part of the
> A2A ecosystem**. They predate the current A2A-based direction and were
> built on Microsoft Agent Framework under an earlier project policy.
> They are intentionally excluded from this diagram to avoid implying an
> architectural relationship that doesn't exist.

> **`a2a-spec` is the canonical, actively maintained source of truth for
> the protocol.** `a2a-ceos-core` and `a2a-containment-core` were the
> working prototypes used to derive and validate this specification.
> Development on them has stopped, but both remain **permanently public
> under the MIT License** — they are not being taken down or hidden, only
> frozen as a historical, freely forkable record that the spec was
> extracted from a real, running implementation rather than designed in
> the abstract.
>
> Repositories marked `Private` are under active development. The
> interface contracts for these components are defined publicly in this
> repository regardless of the implementation's status.

---

## 🎯 Vision & Positioning

**Mission Statement**

> A2A aims to unify autonomous decision-making across heterogeneous network
> systems via a governance-driven protocol. It provides the mechanism by
> which multi-agent intent is converted into safe, auditable, and reversible
> infrastructure state changes.

**Positioning Relative to TM Forum's Autonomous Networks Architecture**

TM Forum's AN architecture (IG1251C, "AN Level 4 Target Architecture")
defines a layered structure — Business Operations, Service Operations,
Network Operations, and Network Element (NE) — in which agents
communicate via agent interfaces such as A2A-T (defined in IG1453) and
intent-based APIs. IG1251C also specifies an "Agent/Copilot Governance"
foundational capability at each layer, covering agent deployment,
registration, verification, and monitoring, and a Network Element layer
populated by "Control Agents."

`a2a-spec` is not a competing orchestration layer sitting above or below
A2A-T. Based on IG1251C's own descriptions, it is closer to an execution
and governance **profile for the Network Operations / Network Element
layers**: `a2a-governance`'s role (policy evaluation, audit trail,
full-lifecycle oversight) matches the "Agent/Copilot Governance"
capability IG1251C calls for at the Network layer, and Vendor Core
adapters (e.g. `a2a-junos-core`, `a2a-ceos-core`) correspond to the
"Control Agents" IG1251C describes at the Network Element layer. A2A-T
itself is one of the agent interfaces IG1251C names for carrying task
content between layers — `a2a-spec` does not replace or compete with it.

```text
   [ TM Forum AN Architecture — IG1251C ]
   Business Operations → Service Operations → Network Operations → Network Element (NE)
   (agents communicate via agent interfaces, incl. A2A-T / IG1453)
   Each layer specifies its own "Agent/Copilot Governance" capability
                    │
                    ▼  (Network Operations / Network Element layers)
┌───────────────────────────────────────────────────────────────┐
│  A2A Protocol Specification  (a2a-spec)                        │
│  - a2a-governance  ≈ IG1251C's "Agent/Copilot Governance"       │
│    (deployment, registration, verification, monitoring)         │
│  - Vendor Cores    ≈ IG1251C's "Control Agents" (NE layer)      │
└───────────────────────────────────────────────────────────────┘
```

This reading is based on TM Forum's published IG1251C (v2.0.0) and
IG1453 (v2.1.0) documents — it is a documentary analysis of how
`a2a-spec`'s components map onto named IG1251C function blocks, not a
tested software integration with any A2A-T implementation.

---

## 🧭 Design Principles

Every part of this specification — the lifecycle, the schemas, the
governance model — exists to serve five principles:

1. **Deterministic Execution** — Given the same input state and policy,
   the outcome is predictable and reproducible. Agents do not improvise
   around ambiguity; ambiguous cases fail closed and surface for review.
2. **Governance-Driven Safety** — No execution bypasses the governance
   evaluation step. Policy enforcement is a protocol requirement, not an
   optional per-agent behavior.
3. **Vendor-Neutral Abstraction** — The protocol itself never assumes a
   specific NOS or vendor API surface. Vendor Cores translate; the spec
   does not know or care which CLI/API/NETCONF dialect is underneath.
4. **Reversible State Changes** — Every `act` operation must have a
   corresponding rollback path, and that path must be verifiable in
   `observe`/`report` — not merely assumed to exist.
5. **Auditability & Human-in-the-Loop** — Every governance decision,
   execution, and rollback is recorded in an append-only, tamper-evident
   audit trail, and high-risk actions require human approval before
   commit.

---

## 📖 Terminology

| Term | Definition |
|---|---|
| **Agent Card** | A machine-readable declaration of an agent's identity, skills, and endpoint, used for discovery and registration. |
| **Governance Boundary** | The set of policy constraints, enforced by `a2a-governance`, within which an agent's actions are permitted without additional human approval. |
| **Safety-Reflective Loop** | The `observe` → `act` → `report` cycle (re-entering `observe` if unresolved) that verifies an executed action achieved its intended state before it is considered complete. |
| **Vendor Core** | A NOS-specific adapter (e.g. `a2a-junos-core`) that translates normalized A2A execution contracts into vendor-specific CLI/API/NETCONF calls. |
| **Containment Action** | A network isolation or traffic-rerouting operation issued in response to a detected security event; defined by `containment.json`. |
| **Execution Contract** | The normalized JSON payload format defined in `vendor-core.json` that decouples an agent's intent from any specific vendor implementation. |

---

## 🔄 Agent Lifecycle Model

All A2A-compliant agents follow a 4-phase deterministic lifecycle defined
in this specification:

1. **`init`** — Agent initialization, skill registration via Agent Cards,
   and governance boundary checking.
2. **`observe`** — Multi-vendor telemetry ingestion and state
   pre-verification.
3. **`act`** — Execution of network change policies through normalized
   vendor cores.
4. **`report`** — Post-execution verification, reflection on state
   changes, and audit reporting. If containment/rollback goals are not
   met, the agent re-enters `observe` rather than assuming success.

---

## ⚠️ Failure Modes

A2A-compliant implementations must handle the following failure modes
explicitly rather than treating them as unexpected exceptions:

| Failure Mode | Required Behavior |
|---|---|
| **Vendor Core unresponsive** | Hub times out waiting for a Vendor Core's `act`/`observe` response. Mark the task `failed-unreachable` — do not assume the change was *not* applied (some NETCONF failure modes apply state without confirming). Trigger a state-verification pass before any retry. |
| **Containment Action not achieved** | `observe`/`report` detects the target isolation state was not reached. Trigger an automatic re-attempt or rollback; surface as `error` (per `containment.json`'s `status` enum), not silently retried indefinitely. |
| **Governance denies (`DENY`)** | `a2a-governance` returns `DENY` for a proposed action — the only blocking effect in the model. Log the denial to the audit trail with the policy reason. The agent must not attempt the same action through an alternate path. A non-blocking `REVIEW` effect is not a failure mode: the agent proceeds, and the human checkpoint is the calling agent's own diff/history UI, not a second governance approval queue. |
| **Telemetry inconsistency** | Vendor Cores report conflicting state (e.g. commit succeeds per API response, but a post-commit diff shows no change — this exact scenario occurred during `a2a-junos-core` development). State is treated as **unconfirmed** until a secondary verification method agrees. |

---

## 🔒 Security Model

- **Execution Boundary** — All state-changing operations pass through the
  `act` phase and only through a registered Vendor Core. Ad-hoc device
  access outside this path is not spec-compliant.
- **Governance Approval** — Every state-changing action is evaluated by
  `a2a-governance` against a private policy catalog, returning one of
  three effects: `PERMIT` (proceeds silently), `REVIEW` (proceeds, but
  logged with rationale — the human checkpoint is the calling agent's own
  diff/history UI, not a second approval queue), or `DENY` (the only
  blocking effect; execution stops regardless of any human intent
  expressed elsewhere). Unclassified actions fail safe: read-like
  actions default to `PERMIT`, write-like actions default to `REVIEW`.
- **Audit Trail** — Every governance decision, execution, and human
  override is recorded in an append-only, tamper-evident log.
- **Reversible Actions** — Every `act` operation must define a
  corresponding rollback procedure that is verifiable in `report`.

> **Note on the `REVIEW` assumption.** Treating `REVIEW` as non-blocking
> relies on a precondition: the calling agent already has its own
> human-approval surface (a diff/history UI) where a human is expected to
> approve before the action reaches governance. Agents that lack such a
> surface — e.g. an autonomous negotiation/deploy flow with no human step
> in its own UI — must not treat `REVIEW` as non-blocking by default,
> since doing so would let the action proceed without the human review
> governance is meant to guarantee. The recommended pattern is to
> evaluate such actions under a distinct action namespace (e.g.
> `autonomous_deploy.*` rather than `vendor.write.*`) and treat `REVIEW`
> for that namespace as "transition to awaiting human approval" instead
> of "proceed." This is a caller-side convention on top of the
> `governance.json` contract, not a change to the contract itself.
> `a2a-interconnect` is a working reference implementation of this
> pattern: its autonomous negotiation/deploy flow evaluates deployments
> under `interconnect.autonomous_deploy.*`, and a `REVIEW` effect on that
> namespace transitions the connection to an `AWAITING_HUMAN_APPROVAL`
> state rather than proceeding.

---

## 🧩 Key Schemas (`/schemas`)

| Schema | Purpose |
|---|---|
| `containment.json` | Standardized payload format for network node isolation, traffic rerouting, and containment actions. |
| `governance.json` | Policy enforcement, access control, and Human-in-the-Loop approval requirements. |
| `vendor-core.json` | Normalized interface contract for underlying vendor drivers (Arista, Cisco, Juniper). |

Minimal skeletons for all three are published under [`/schemas`](schemas/).
Field shapes are grounded in what the actual repositories exchange today
(`a2a-junos-core`, `a2a-containment-core`, `a2a-governance`), not an
idealized design — for example, `containment.json` models the real
Low/Medium/High risk-tier response plan rather than a generic action
enum. Note that the four-phase lifecycle above is a protocol-level
description of *when* agents act; it is not a literal `phase` field
inside every payload. Validation rules and worked examples will be added incrementally.

---

## 📐 Design Philosophy & Pattern Compliance

The A2A specification aligns with established Agentic AI design patterns
to ensure operational resilience and enterprise safety:

- **Orchestrator-Worker Pattern** — Platform-layer orchestrators route
  containment/isolation requests to vendor core drivers using normalized
  payloads.
- **Evaluator-Optimizer Pattern (Human-in-the-Loop)** — `a2a-governance`
  evaluates every write-type action and returns `PERMIT` / `REVIEW` /
  `DENY`. Only `DENY` blocks execution; `REVIEW` is intentionally
  non-blocking, since the true human-approval checkpoint is each calling
  agent's own diff/history UI, not a second governance approval queue.
- **Safety-Reflective Loop** — State verification occurs in the
  `observe` / `report` phases, enabling automatic rollback when
  containment goals are not met.
- **Strategy & Facade Abstraction** — Unified JSON/YAML interfaces
  abstract vendor-specific CLI/API differences into a single normalized
  protocol.

---

## 📚 References & Standards Alignment

- **TM Forum IG1251C / IG1453 (A2A-T)** — `a2a-spec`'s components map onto
  named function blocks in TM Forum's published AN architecture: see
  "Positioning Relative to TM Forum's Autonomous Networks Architecture"
  above. This is a documentary analysis of IG1251C v2.0.0 and IG1453
  v2.1.0, not a tested integration with any A2A-T implementation.
- **The Agent2Agent (A2A) Protocol** — implementations in this ecosystem
  (e.g. `a2a-governance`) are built directly on the official
  [a2aproject/A2A](https://github.com/a2aproject/A2A) SDK's messaging
  layer, not just conceptually inspired by it. The base A2A protocol
  originated at Google and, per IG1453's own description, is now
  maintained under the Linux Foundation; IG1453 explicitly extends it
  without modifying the core protocol.
- **Agentic AI Design Patterns** — Built on established multi-agent
  design patterns (orchestration, evaluation, reflection) rather than
  a bespoke, unvalidated architecture.

---

## 📄 License

This specification is released under the Apache 2.0 License.
