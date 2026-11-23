# 18. Proposed repository structure

Status: `[SPEC]`. AEGIS-X should live in a **new repository** (clean-room; see
[02 §2.11](02-artex-analysis.md#211-licensing-and-provenance)), not inside this ARTEX fork. This
design set can move with it under `docs/design/`.

```text
aegis-x/
├── README.md
├── SECURITY.md                       # platform vuln disclosure (platform only, never customer targets)
├── LICENSE                           # to be decided (proprietary or permissive); NOT derived from ARTEX
├── Makefile                          # make dev | test | lint | bench | e2e
├── go.work                           # Go workspace for services/*
├── api/
│   ├── openapi/aegis.v1.yaml         # public REST (source of truth; clients generated)
│   ├── proto/aegis/{pdp,broker,graph,audit,gateway}/v1/*.proto
│   ├── schemas/agent/*.json          # AgentInvocation/Result per role, versioned
│   └── schemas/events/*.json         # CloudEvents data schemas
├── services/                         # Go
│   ├── gateway-authn/                # Envoy ext_authz helpers (if needed)
│   ├── identity/                     # tenants, users, roles, SCIM
│   ├── authorization/                # records, ownership proofs, bundle compiler/signer
│   ├── pdp/                          # Safety & Governance Controller (library + server)
│   │   ├── scope/                    # compiled matchers (host, suffix, CIDR, ARN, repo)
│   │   ├── decide.go
│   │   └── testdata/golden/          # ≥500 request→decision cases
│   ├── approval/
│   ├── assessment/
│   ├── orchestrator/                 # Temporal workflows (deterministic) + activities
│   ├── broker/                       # Execution Broker
│   ├── seeg/                         # Scope-Enforcing Egress Gateway
│   ├── verification/                 # verifier registry, CEL predicates
│   ├── graph/                        # SKG + EHG service
│   ├── findings/
│   ├── remediation/
│   ├── reporting/
│   ├── model-gateway/
│   ├── tool-registry/
│   └── audit/
├── pkg/                              # shared Go libs (no service logic)
│   ├── canonical/                    # canonical JSON/CBOR + hashing
│   ├── tenancy/                      # context propagation, PG SET LOCAL helpers
│   ├── cedarx/                       # Cedar loading + global forbids
│   ├── tokens/                       # decision token sign/verify (KMS)
│   ├── untrusted/                    # ModelText/UntrustedText types (no conversion to Evidence)
│   ├── redact/
│   └── otelx/
├── agents/                           # Python (uv/poetry workspace)
│   ├── aegis_agents/
│   │   ├── runtime/                  # Temporal activity worker, schema validation
│   │   ├── context/                  # client for context bundles (assembled server-side)
│   │   └── roles/{planning,discovery,vuln_analysis,validation,identity,cloud,code,def_sim,remediation}/
│   │       ├── prompt.md             # content-addressed, signed at release
│   │       └── role.py
│   ├── evals/                        # calibration, routing experiments
│   └── tests/
├── tools/                            # signed tool catalogue (each dir = one tool)
│   └── tls-scan/{manifest.yaml, Dockerfile, parser/, threat-review.md}
├── verifiers/                        # verifier specs (YAML + CEL) + tests
├── scenarios/                        # detection-validation scenario library (signed)
├── web/                              # Next.js app
│   ├── app/ components/ lib/ tests/e2e/
├── deploy/
│   ├── compose/                      # Mode A laptop
│   ├── helm/aegis/                   # charts per plane
│   ├── terraform/                    # cloud infra (cells, exec clusters, KMS, buckets)
│   └── policies/                     # Kyverno/Gatekeeper admission, NetworkPolicies
├── labs/                             # Mode A benchmark labs as code (WEB-1, NET-1, CLOUD-1, ID-1, CODE-1, INJ-1, DET-1)
├── bench/                            # harness, answer keys, pre-registrations, reports/
├── migrations/                       # forward-only SQL (golang-migrate/atlas)
├── security/
│   ├── threat-model/                 # living version of doc 13
│   ├── tests/                        # T-* security test implementations (non-lab + lab)
│   └── injection-corpus/
├── docs/
│   ├── design/                       # this document set
│   ├── adr/                          # ADR-0001… (see below)
│   └── runbooks/
└── .github/workflows/                # ci.yml, lab-nightly.yml, release.yml, tools-catalogue.yml
```

## Initial ADRs to write in Milestone 1

| ADR | Decision |
|---|---|
| ADR-0001 | Clean-room implementation; no ARTEX code reuse |
| ADR-0002 | Propose/decide/execute/verify separation; models have no authority |
| ADR-0003 | PDP in Go with compiled scope matchers; Cedar for user authorization |
| ADR-0004 | Temporal as workflow engine |
| ADR-0005 | PostgreSQL for graph (no graph DB in MVP) with criteria to revisit |
| ADR-0006 | SEEG as mandatory egress enforcement point |
| ADR-0007 | gVisor runtime for MVP sandboxes |
| ADR-0008 | Refusals are terminal; no model rerouting on refusal |
| ADR-0009 | Hash-chained audit with WORM anchoring |
| ADR-0010 | Python agent runtime behind JSON Schema contracts |

## Ownership (CODEOWNERS sketch)

`services/pdp/`, `services/seeg/`, `services/broker/`, `pkg/tokens/`, `deploy/policies/`,
`tools/`, `verifiers/` require review from the security engineering group in addition to the
owning team. Changes to `pkg/untrusted/` require two security reviewers.
