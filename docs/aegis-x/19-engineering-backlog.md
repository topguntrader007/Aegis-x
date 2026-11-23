# 19. Initial engineering backlog

Status: `[SPEC]`. Covers Milestones 1–2 in detail (later milestones are epics only). Estimates in
ideal engineer-days (d). Each story lists its acceptance check; "Req" maps to
[03](03-product-requirements.md) IDs, "Tests" to [13](13-threat-model.md).

## Epic E1 — Repository, CI, dev environment (M1)

| ID | Story | Est | Acceptance | Req/Tests |
|---|---|---|---|---|
| E1-1 | Create monorepo layout per doc 18 with Go workspace, Python workspace, web app | 2d | `make dev` boots compose stack (PG, Temporal, Keycloak, NATS, Redis, MinIO) | NFR-OPS-1 |
| E1-2 | CI: lint (golangci, ruff, biome), type checks (mypy/pyright, tsc), unit tests, coverage report | 3d | Required checks on PRs | — |
| E1-3 | Supply chain: SBOM, vuln scan, secret scan, licence check, cosign signing of images | 3d | Unsigned image cannot be deployed to staging | T-SUPPLY-1 |
| E1-4 | ADR-0001..0010 written and accepted | 3d | Merged in `docs/adr` | — |
| E1-5 | OTel bootstrap package and structured logging conventions (tenant/assessment/action attributes; secret-field denylist) | 2d | Trace spans visible end to end in dev | T-LOG-1 |

## Epic E2 — Identity, tenancy and authorization (M1)

| ID | Story | Est | Acceptance | Req/Tests |
|---|---|---|---|---|
| E2-1 | OIDC login (Auth Code + PKCE), session cookies, refresh rotation | 4d | Login via Keycloak; tokens 10 min | FR-GOV-3, T-AUTH-1..4 |
| E2-2 | Tenants, memberships, roles, role bindings schema + migrations | 3d | Migrations forward-only; seed roles | — |
| E2-3 | `pkg/tenancy`: request → tenant context → `SET LOCAL`; middleware rejects missing tenant | 3d | Query without context fails | T-TENANT-DB-* |
| E2-4 | RLS generator: every tenant table gets ENABLE+FORCE+policy; CI check enumerates tables | 2d | CI fails if a table lacks RLS | T-TENANT-DB-* |
| E2-5 | Cedar integration with global forbid policies; policy unit tests | 4d | Forbid tests pass | T-APPR-1, T-TENANT-API-* |
| E2-6 | Step-up MFA check (`acr`, `auth_time`) helper | 2d | Approval endpoints return 401 step_up_required | T-AUTH-6 |
| E2-7 | Generated cross-tenant API test suite from OpenAPI | 3d | Every endpoint covered; 404 for foreign IDs | T-TENANT-API-* |

## Epic E3 — Assets, ownership, authorization records, bundles (M1)

| ID | Story | Est | Acceptance | Req/Tests |
|---|---|---|---|---|
| E3-1 | Asset registry API + canonicalization (IDNA, CIDR, ARNs) | 4d | Property tests for canonical forms | FR-DISC-1 |
| E3-2 | Ownership proofs: DNS TXT, HTTP well-known, manual attestation with second approver | 5d | Expired/unverified proofs block activation | FR-AUTHZ-4, T-OWN-1..4 |
| E3-3 | Authorization record lifecycle (draft/submit/approve/revoke/expire) with two-person rule | 5d | DB CHECK + Cedar + API tests | FR-AUTHZ-1..3 |
| E3-4 | Bundle compiler: scope items → compiled matchers (CBOR) + Ed25519 signing via KMS interface (dev: local key) | 5d | Deterministic output; signature verifies | FR-AUTHZ-5, T-BUNDLE-1..2 |
| E3-5 | Bundle distribution to PDP with monotonic version check | 2d | Rollback refused | T-BUNDLE-3 |
| E3-6 | Policy simulation endpoint | 2d | Returns PDP decision + reasons for hypothetical action | FR-GOV-4 |

## Epic E4 — PDP (Safety & Governance Controller) (M1)

| ID | Story | Est | Acceptance | Req/Tests |
|---|---|---|---|---|
| E4-1 | Scope matchers: exact host, label-aligned suffix, CIDR (v4/v6 incl. mapped/NAT64), ARN prefix, repo | 5d | Unit + property tests | T-NET-LIT-* (logic) |
| E4-2 | `Decide` implementation per 10 §10.6 with reason codes | 5d | Golden corpus v1 (≥ 200 cases) passes | T-PDP-GOLDEN |
| E4-3 | Fuzz harness with core invariant | 2d | Nightly 10⁷ execs no counterexample | T-PDP-FUZZ |
| E4-4 | Decision tokens: CBOR payload, Ed25519, `jti`, `exp`, action hash, egress list | 3d | Verify lib shared with Broker | T-PDP-3..6 |
| E4-5 | Budget reservation API (append-only ledger) | 3d | Concurrent reservation test: no overspend beyond headroom | T-BUDGET-2 |
| E4-6 | Kill state (tenant/assessment/global) consulted first; broadcast subscriber | 3d | Decisions deny within 1 s of engage in tests | T-KILL-2 |

## Epic E5 — Approvals (M1)

| ID | Story | Est | Acceptance | Req/Tests |
|---|---|---|---|---|
| E5-1 | Approval requests bound to action hash; TTL; votes with step-up; consumption atomic with token issuance | 5d | Reuse/expired/mismatch rejected | FR-GOV-2, T-APPR-1..5 |
| E5-2 | Approval notifications (email/webhook) | 2d | Delivered with deep link | — |

## Epic E6 — Task engine skeleton (M1)

| ID | Story | Est | Acceptance | Req/Tests |
|---|---|---|---|---|
| E6-1 | Temporal deployment in compose/kind; namespace per cell; payload codec with tenant DEK | 4d | Payloads encrypted at rest in Temporal | — |
| E6-2 | AssessmentWorkflow v0: fixed synthetic plan → PDP → (approval) → Broker → record | 6d | Replay test passes; kill/pause/cancel signals | FR-ASMT-3 |
| E6-3 | Execution Broker v0: token verification, jti single-use, gVisor pod for C0 echo tool, cleanup | 6d | T-EXEC-1..3 pass | — |
| E6-4 | Execution events + outbox → NATS → SSE endpoint | 4d | UI receives ordered events with seq | — |

## Epic E7 — Audit (M1)

| ID | Story | Est | Acceptance | Req/Tests |
|---|---|---|---|---|
| E7-1 | Audit append API with per-tenant gapless seq + hash chain | 4d | INSERT-only role; chain verifies | FR-GOV-1, T-AUDIT-1..3 |
| E7-2 | Anchoring job → object-locked bucket (MinIO with object lock in dev) | 3d | Tamper detection test | T-AUDIT-4 |
| E7-3 | Audit coverage check: every PDP decision, approval vote, execution start/finish, authz transition writes audit | 2d | Integration test asserts 1:1 | — |

## Epic E8 — Read-only operator UI (M1)

| ID | Story | Est | Acceptance |
|---|---|---|---|
| E8-1 | App shell, auth, nav, tenant context, kill button (engage only) | 4d | Playwright login journey |
| E8-2 | Authorizations list/detail with version diff | 4d | Displays scope + signatures |
| E8-3 | Assessments list/detail with task states and decision reasons | 4d | Live via SSE |
| E8-4 | Audit log viewer + chain verification UI | 3d | Verification result displayed |

## Epic E9 — Model Gateway (M2)

| ID | Story | Est | Acceptance | Req/Tests |
|---|---|---|---|---|
| E9-1 | Provider interface + adapters (1 commercial, 1 OpenAI-compatible self-hosted) | 6d | Structured output works on both | FR-AI-1 |
| E9-2 | Capability registry (signed config) + tenant model policy (allowlist, residency, data classes) | 4d | Disallowed routes blocked | FR-AI-2, T-MG-1..4 |
| E9-3 | Router + availability fallback; refusal terminal | 4d | T-MG-REFUSAL-1 | FR-AI-4 |
| E9-4 | Redaction + canary scan pre-send | 3d | Canary never sent | T-CRED-1 |
| E9-5 | Accounting + budget reservation/settlement; price table versions | 4d | `model_usage` rows complete | FR-AI-3 |
| E9-6 | Exact cache (tenant-scoped) | 2d | Cache key includes tenant | — |
| E9-7 | Encrypted transcript storage for reproducibility | 3d | Replay can load transcripts | FR-AI-5 |

## Epic E10 — Agent runtime and first agents (M2)

| ID | Story | Est | Acceptance | Req/Tests |
|---|---|---|---|---|
| E10-1 | Python activity worker; schema validation; invocation-id caching | 4d | Retries return cached result | — |
| E10-2 | Context assembler (server-side, Go) with need-to-know filter, trust labels, nonce framing, token budget | 6d | T-CTX-1..4 | — |
| E10-3 | Output validator (refs resolve, no literal targets, size caps) + hallucination counters | 3d | T-AGENT-1..5 | — |
| E10-4 | Planning agent v1 + plan review flow | 6d | Plan screening shown in UI | FR-ASMT-2 |
| E10-5 | Discovery, Cloud, Code agents v1 | 10d | Produce schema-valid outputs on labs | FR-MOD-* |

## Epic E11 — SEEG and first tools (M2)

| ID | Story | Est | Acceptance | Req/Tests |
|---|---|---|---|---|
| E11-1 | SEEG: per-sandbox identity + policy install from token | 5d | No policy → reset | T-TENANT-EXEC-1 |
| E11-2 | SEEG DNS resolver: resolve, CNAME chain checks, special-range filter, pinning | 6d | T-NET-DNS-*, T-NET-REBIND-1, T-NET-CNAME-1 (lab) | — |
| E11-3 | SEEG HTTP(S): CONNECT, redirect evaluation, capture, header stripping | 6d | T-NET-REDIR-1 | — |
| E11-4 | SEEG rate limits (local buckets + Redis coordination, fail-safe) | 3d | T-RATE-1 | — |
| E11-5 | NetworkPolicy + admission policies for exec namespace | 3d | T-SBX-1, T-SBX-3..6 | — |
| E11-6 | Tools: dns-passive, http-probe, tls-scan with manifests, parsers, threat reviews | 8d | Signed and registered | — |
| E11-7 | Cloud readers (AWS first) with STS session policies; secret-scan + SCA tools | 8d | T-CLOUD-RO (lab) | — |

## Epic E12 — Labs and bench harness v1 (M2)

| ID | Story | Est | Acceptance |
|---|---|---|---|
| E12-1 | WEB-1 subset + clean twins as code | 5d | Reproducible provisioning |
| E12-2 | CLOUD-1 (AWS) seeded misconfigs via Terraform | 5d | Seed manifest = answer key |
| E12-3 | CODE-1 known-answer repos with canary-format fake secrets | 3d | Answer key |
| E12-4 | Harness: provision → run → match → metrics with bootstrap CIs → signed report | 8d | First baseline comparison report |
| E12-5 | INJ-1 corpus v1 (≥ 50 payloads across channels) | 4d | T-INJ-1..10 runnable |

## Later epics (M3–M6)

E13 Graph svc (SKG/EHG) · E14 Verification svc + 30 verifiers · E15 Findings + risk scoring ·
E16 Identity readers + effective permissions · E17 Execution reuse · E18 Hardened execution +
credential injection · E19 C4/C5 checks with approvals · E20 Replica verification (C6) · E21
Detection validation (C7) · E22 Full UI (10 views) · E23 Reporting Studio · E24 Remediation +
ticketing · E25 SCIM + integrations · E26 Load/chaos/DR · E27 Adversarial program · E28
Compliance readiness.
