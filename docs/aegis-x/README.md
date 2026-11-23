# AEGIS-X design document set

AEGIS-X is a proposed **authorized, defensively oriented, multi-agent security assessment
platform**. This directory is its implementation-ready design. It is a design, not a build:
nothing in this directory is running code, and nothing here changes the behaviour of the
ARTEX code elsewhere in this repository.

> **Why this lives in an ARTEX fork.** This repository is a fork of the public English-language
> ARTEX fork (`BaaRai/ARTEX`, upstream `Autumn-27/ARTEX`, AGPL-3.0). That made it possible to
> ground the ARTEX analysis in **inspected source code** rather than press reports alone.
> AEGIS-X is designed as a **clean-room, new codebase**: it reuses ideas, not ARTEX code, so
> that AGPL obligations, ARTEX's offensive defaults, and its unreviewed dependency tree do not
> carry over. See [18-repository-structure.md](18-repository-structure.md) for the proposed new
> repository and [02-artex-analysis.md §2.11](02-artex-analysis.md#211-licensing-and-provenance) for
> the licensing note.

## Reading order (the 20 required deliverables, in order)

| # | Document | What it answers |
|---|---|---|
| 1 | [01-executive-vision.md](01-executive-vision.md) | Why AEGIS-X exists, what it is and is not, the bets it makes |
| 2 | [02-artex-analysis.md](02-artex-analysis.md) | ARTEX architecture from code + primary sources, limitations, comparison matrix |
| 3 | [03-product-requirements.md](03-product-requirements.md) | Personas, deployment modes A/B/C, functional and non-functional requirements |
| 4 | [04-system-architecture.md](04-system-architecture.md) | Services, planes, communication model, the core execution path |
| 5 | [05-multi-agent-design.md](05-multi-agent-design.md) | The 12 roles, task lifecycle, scheduling, conflict resolution, orchestration pseudocode |
| 6 | [06-data-flow-trust-boundaries.md](06-data-flow-trust-boundaries.md) | Data-flow and trust-boundary diagrams, what crosses each boundary and how |
| 7 | [07-knowledge-graph.md](07-knowledge-graph.md) | Security knowledge graph + execution-history graph schema, provenance, dedup, versioning |
| 8 | [08-database-design.md](08-database-design.md) | PostgreSQL DDL, row-level security, partitioning, retention |
| 9 | [09-api-specification.md](09-api-specification.md) | REST/gRPC endpoints, examples, authz checks, events, agent contracts, state machines, idempotency |
| 10 | [10-secure-execution.md](10-secure-execution.md) | Sandboxed tool execution, scope-enforcing egress, signed manifests, policy enforcement pseudocode |
| 11 | [11-ai-model-orchestration.md](11-ai-model-orchestration.md) | Model gateway, routing, memory, RAG, caching, confidence, reproducibility |
| 12 | [12-ui-specifications.md](12-ui-specifications.md) | 10 views, wireframes, navigation, component hierarchy, user journeys |
| 13 | [13-threat-model.md](13-threat-model.md) | STRIDE + agent-specific threats, controls, and a concrete test per safeguard |
| 14 | [14-testing-benchmarking.md](14-testing-benchmarking.md) | Evaluation harness, metrics definitions, baselines, statistical method |
| 15 | [15-infrastructure-deployment.md](15-infrastructure-deployment.md) | Technology choices with tradeoffs, Kubernetes topology, per-mode deployment |
| 16 | [16-reliability-cost-scaling.md](16-reliability-cost-scaling.md) | Reliability mechanisms, SLOs, capacity model, cost model |
| 17 | [17-implementation-roadmap.md](17-implementation-roadmap.md) | Milestones 1–6 with deliverables, effort, acceptance criteria, security gates |
| 18 | [18-repository-structure.md](18-repository-structure.md) | Proposed monorepo layout |
| 19 | [19-engineering-backlog.md](19-engineering-backlog.md) | Initial epics and stories for Milestones 1–2 |
| 20 | [20-mvp-definition-of-done.md](20-mvp-definition-of-done.md) | MVP exit criteria |
| — | [references.md](references.md) | Sources, with what each one was used for |

## Labels used throughout

Every non-trivial claim in these documents carries one of these labels, as the brief requires.

**Provenance labels (for statements about ARTEX and the outside world)**

| Label | Meaning |
|---|---|
| `[CODE]` | Verified by reading ARTEX source in this repository (file path given). Reflects this fork at commit `38c7477`; the withdrawn upstream may differ. |
| `[DOC]` | Stated in a primary or first-party source (listed in [references.md](references.md)). |
| `[PRESS]` | Reported only by secondary press; not independently confirmed here. |
| `[GENERAL]` | Widely known public information about other products and projects (from the authors' background knowledge), not re-verified for this document. Verify before relying on it. |
| `[INFERRED]` | Our inference from code or sources; plausible but not stated. |
| `[UNKNOWN]` | We looked and could not establish it. |

**Status labels (for AEGIS-X components)**

| Label | Meaning |
|---|---|
| `[SPEC]` | Specified to implementation level in this document set; nothing is built yet. |
| `[CONCEPT]` | Direction is set, details need a design spike before implementation. |
| `[RESEARCH]` | Open problem; we do not yet know a reliable approach. Must not be promised to users. |

Nothing in AEGIS-X is `[IMPLEMENTED]` today. No benchmark numbers appear anywhere in this set;
where a number appears it is a **target**, a **budget**, or a **cost-model input**, and is marked
as such.

## Non-negotiable invariants (referenced everywhere as INV-n)

1. **INV-1 Authorization first.** No assessment activity without a signed, active authorization
   record (scope + rules of engagement + approver identities + validity window).
2. **INV-2 Scope outside the model.** Scope is compiled into a deterministic policy bundle and
   enforced by the Policy Decision Point (PDP) and the Scope-Enforcing Egress Gateway (SEEG).
   No prompt, model output, tool output, environment variable, DNS answer, or task field can
   widen it.
3. **INV-3 No unmediated action.** Every tool invocation is an `ActionRequest` evaluated by the
   PDP before dispatch and enforced again at the network layer during execution.
4. **INV-4 Humans approve risk.** Actions above the assessment's risk ceiling require a human
   approval bound to the exact action hash. A model can never be an approver.
5. **INV-5 Read-only by default.** Modes B and C start with a read-only capability set; anything
   that changes target state is off unless the authorization record enables it.
6. **INV-6 Evidence over claims.** A finding becomes `confirmed` only when the Verification
   service reproduces or checks it against stored, hash-addressed evidence. Model text is never
   evidence.
7. **INV-7 Immutable, tenant-scoped audit.** Every decision is appended to a hash-chained audit
   log per tenant.
8. **INV-8 External content is hostile.** Tool output, web content, repository content, and
   scanner output are data, never instructions, and are typed as such end to end.
9. **INV-9 Secrets never reach models.** Credentials are short-lived, brokered at execution
   time, and redacted from everything a model or a log can see.
10. **INV-10 Claims need tests.** Every security control listed in
    [13-threat-model.md](13-threat-model.md) has at least one automated test that must pass in CI.
