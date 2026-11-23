# 20. Definition of done for the MVP

Status: `[SPEC]`.

## 20.1 What the MVP is

The MVP is **Milestones 1–3 plus the minimum of Milestone 5 UI** needed to operate them: a
tenant can register and verify assets, get a two-person-approved authorization, run a Mode B
assessment limited to capability ceiling **C3** (plus C2 read-only cloud and code connectors),
see agent-proposed and verified findings with evidence, and export a technical report. Mode A
labs exist for benchmarking. Mode C runs the same read-only checks on a schedule.

Explicitly **not** in the MVP: C4+ capabilities, authenticated app testing, replica
verification, detection validation, ticketing integrations, executive/compliance reports,
multi-region, microVM isolation.

## 20.2 Functional done criteria

- [ ] OIDC SSO login; roles enforced by Cedar with global forbids; MFA step-up for approvals.
- [ ] Assets registered and ownership verified (DNS TXT, HTTP well-known, AWS role assumption, GitHub app install).
- [ ] Authorization record lifecycle with two-person activation; signed bundle compiled and used by PDP and SEEG.
- [ ] Assessment Builder (objectives, authorization, profile, budget) → generated plan → plan review → run.
- [ ] Agents: Planning, Discovery, Vulnerability Analysis, Cloud (AWS), Code, Remediation; Orchestrator; Verification; PDP.
- [ ] Modules: Web/API (C1–C3 subset: discovery, TLS, headers, auth config surface), Cloud (AWS read-only rules + public exposure probe), Application security (secrets, SCA, SAST via one engine).
- [ ] Knowledge graph and execution-history graph with provenance, dedup, versioning, freshness; execution reuse.
- [ ] Findings with verification states; only the verifier confirms (DB-enforced); evidence hashes viewable and re-verifiable in UI.
- [ ] Remediation recommendations with reverify plans; manual owner assignment; reverification run.
- [ ] Views: Executive Dashboard (basic), Asset Explorer, Agent Ops Center, Findings Workspace, Evidence Timeline, Assessment Builder, Governance Console (authorizations, approvals, audit, kill switch), technical report export (Markdown/PDF/JSON). Security Graph may ship as table + simple path view.
- [ ] Kill switch at tenant and assessment level; two-person release.
- [ ] Model Gateway with ≥ 2 providers (one self-hostable), routing tiers, refusal-terminal behaviour, budgets, accounting visible per assessment.

## 20.3 Security done criteria (all must pass)

- [ ] Every test ID listed for M1–M3 in [17](17-implementation-roadmap.md) passes in CI or the lab nightly, with results archived.
- [ ] **Zero scope-policy violations** across: full benchmark runs on WEB-1, CLOUD-1, CODE-1; the INJ-1 corpus; and a red-team scope-escape exercise.
- [ ] Cross-tenant suites (API, DB, events, execution) pass with 100% endpoint coverage.
- [ ] Canary-secret scan finds no canaries in prompts, logs, events, reports or UI.
- [ ] Audit chain verification passes; every decision/approval/execution has an audit record.
- [ ] External penetration test of the platform (web, API, execution plane) with no open High/Critical findings.
- [ ] Threat model (doc 13) reviewed and updated to as-built; residual risks accepted in writing by the accountable executive.
- [ ] No tool in the catalogue above C3 enabled for any MVP tenant; no C9 tool exists.

## 20.4 Quality and benchmark done criteria

- [ ] Benchmark report published for MVP release: AEGIS-X vs conventional scanning baseline vs non-agentic AEGIS baseline, n ≥ 5 runs per config, metrics from [14 §14.4](14-testing-benchmarking.md) with 95% CIs.
- [ ] Any effectiveness statement in product materials cites that report; if AEGIS-X does not beat a baseline on a metric, the materials do not claim it does.
- [ ] Confirmed-finding audit: sample of ≥ 50 confirmed findings independently re-checked by a human; any refuted-on-audit finding triggers a verifier fix before release.
- [ ] Calibration (ECE) reported for model confidence on triage tasks.
- [ ] Exact replay determinism passes on all benchmark recordings.

## 20.5 Reliability and operations done criteria

- [ ] SLOs instrumented with dashboards and burn-rate alerts; kill switch SLOs met in lab drills.
- [ ] Chaos tests: orchestrator worker kill, agent worker kill, PG failover, provider outage, SEEG pod loss — no lost tasks, recovery p95 < 2 min.
- [ ] Load test at 1,000 assets × 10 concurrent assessments (MVP scale; full scale is M6).
- [ ] Backups + one restore drill completed; key epoch bump invalidates tokens after restore.
- [ ] Runbooks for kill switch, provider outage, PG failover, key rotation, tenant offboarding.
- [ ] Cost per verified finding and per assessment visible to tenant admins.

## 20.6 Documentation done criteria

- [ ] This design set updated to as-built, with each component's status label changed from `[SPEC]` to `[IMPLEMENTED]` only where code and tests exist.
- [ ] OpenAPI published; agent and event schemas published; ADRs merged.
- [ ] Customer-facing docs: authorization process, ownership verification, capability classes, egress IPs, data handling, model providers used.
- [ ] Accessibility conformance statement for the shipped views.
