# 13. Threat model and security controls

Status: `[SPEC]`. Method: asset-centric STRIDE per trust boundary ([06](06-data-flow-trust-boundaries.md)),
extended with the OWASP AI Agent Security risk list `[DOC]` and with ARTEX's observed failure
modes ([02](02-artex-analysis.md)). Every control maps to at least one automated test (INV-10).
Test IDs are stable and referenced elsewhere in this set.

## 13.1 What we protect

| Asset | Why it matters |
|---|---|
| Customer targets' availability and integrity | We must never harm what we assess |
| Third parties (anything not authorized) | Unauthorized testing is illegal and harmful |
| Tenant data (findings, evidence, code excerpts, configs) | Highly sensitive: a map of a customer's weaknesses |
| Credentials brokered to the platform | Read access to customer clouds/IdPs/SCM |
| Authorization records and bundles | The root of legitimacy for all activity |
| Audit chain | Proof of what happened |
| Platform signing keys (bundle, token, tool) | Compromise = arbitrary authority |
| Model provider accounts and budgets | Denial-of-wallet; abuse of our accounts |

## 13.2 Adversaries

| ID | Adversary | Capability |
|---|---|---|
| A1 | Hostile target content | Controls responses, DNS, redirects, HTML/JS, banners, files in repos, scanner-visible data; may embed prompt injection |
| A2 | Malicious or compromised tenant user | Valid account in one tenant; wants to test unauthorized targets, read other tenants, or misuse platform for attacks |
| A3 | External attacker | Internet access to our edge; credential stuffing; exploits in our web app |
| A4 | Compromised tool / supply chain | Malicious code in a tool image or dependency |
| A5 | Compromised model provider or poisoned model output | Arbitrary model responses |
| A6 | Malicious insider (platform staff) | Some infrastructure access |
| A7 | Misbehaving agent (no adversary) | Hallucinations, loops, duplicate actions, overreach — the ARTEX failure modes `[DOC]` |

## 13.3 Threats and controls (STRIDE + agentic)

| ID | Threat | Adv. | Boundary | Controls | Tests |
|---|---|---|---|---|---|
| TH-01 | Scope escape via model-proposed target | A1, A7 | B4, B7 | Ref-only targets; PDP scope match; SEEG | T-TASK-1, T-PDP-GOLDEN, T-PDP-FUZZ |
| TH-02 | Scope escape via DNS (rebinding, CNAME to third party, homograph) | A1 | B11–B12 | SEEG resolve+pin, label-aligned match, CNAME chain check, special-range filter | T-NET-DNS-1..4, T-NET-REBIND-1, T-NET-CNAME-1 |
| TH-03 | Scope escape via redirects / IP literals / IPv6 tricks | A1 | B12 | SEEG redirect evaluation; literal checks; NAT64/6to4 embedded v4 evaluation | T-NET-REDIR-1, T-NET-LIT-1..3 |
| TH-04 | Indirect prompt injection alters agent behaviour | A1 | B3–B4 | Untrusted typing + nonce framing; no authority in model; schema validation; need-to-know context; per-invocation single-turn | T-INJ-1..20, T-INJ-SCOPE-1 |
| TH-05 | Injection propagates across agents via shared memory | A1 | Graph | Hub-and-spoke; trust classes; summaries marked; lessons human-curated | T-INJ-PROP-1 |
| TH-06 | Model-driven approval bypass | A1, A5 | B7 | Models never approve; Cedar forbids service votes; approval bound to action hash | T-APPR-1..5 |
| TH-07 | Forged/replayed decision token | A4, A6 | B8–B9 | Ed25519 signatures (KMS), `exp` ≤120 s, `jti` single-use, action hash re-computed by Broker | T-PDP-3..6, T-EXEC-1..3 |
| TH-08 | Bundle tampering or rollback | A6 | B2 | Signed bundles, monotonic version per tenant, PDP refuses older | T-BUNDLE-1..3 |
| TH-09 | Sandbox escape to control plane or other tenants | A4, A1 | B10–B11 | gVisor, no SA token, NetworkPolicy only→SEEG, dedicated nodes, admission control | T-SBX-1..6 |
| TH-10 | Credential theft by tool | A4 | B13 | SEEG injection; STS session policies; TTL = timeout; redaction | T-CRED-1..4 |
| TH-11 | Credential/secret leakage to models, logs, reports | A7, A5 | B5, logs | 3-layer redaction; canary secrets; log scanner in CI | T-CRED-1, T-LOG-1 |
| TH-12 | Cross-tenant read/write | A2 | all | 9.12 layers, RLS FORCE, Cedar forbid, per-tenant keys | T-TENANT-API-*, T-TENANT-DB-*, T-TENANT-EXEC-1, T-TENANT-EVT-1 |
| TH-13 | Unauthorized assets registered (tenant tries to test someone else's domain) | A2 | API | Ownership proofs; two-person activation; manual attestation needs 2nd approver + audit; abuse heuristics (big-brand domains, other tenants' verified assets) | T-OWN-1..4 |
| TH-14 | Platform misuse as attack tool | A2 | product | No C9 tools; read-only defaults; rate caps; ownership proofs; abuse monitoring on target diversity; published egress IPs; terms + KYC for Mode B tenants | T-ABUSE-1..3, catalogue review gate |
| TH-15 | Harm to target availability | A7 | B12 | Rate limits in SEEG; capability ceilings; time windows; health-aware backoff (5xx/429 → slow down, then pause asset) | T-RATE-1..3 |
| TH-16 | Console exposure / auth bypass | A3 | B1 | No unauthenticated surfaces; SSO only; private ingress default; WAF; dependency scanning; external exposure self-scan | T-AUTH-1..8, T-EXPOSE-1 |
| TH-17 | Audit tampering / repudiation | A6 | Audit | Hash chain + WORM anchors; app role INSERT-only | T-AUDIT-1..4 |
| TH-18 | Evidence tampering | A6, A4 | B15–B16 | Content addressing; object lock; hash in audit | T-EVID-1..3 |
| TH-19 | Malicious tool output (bombs, parser exploits, terminal escapes, XSS in UI) | A1, A4 | B14 | Caps; sandboxed parsers; escape rendering; CSP + Trusted Types | T-OUT-1..4, T-UI-XSS-1 |
| TH-20 | Malicious tool image / dependency | A4 | Registry | Signed digests; SBOM + vuln scan; reproducible builds for first-party tools; review gate | T-SUPPLY-1..3 |
| TH-21 | Denial of wallet (runaway model/compute use) | A7, A2 | MG | Budgets reserved before calls; per-tenant caps; stagnation detector; anomaly alerts | T-BUDGET-1..3 |
| TH-22 | Duplicate/conflicting actions (agents overwriting each other — FBI-listed trait `[DOC]`) | A7 | Orchestrator | Fingerprint dedup; per-asset leases with fencing; idempotent activities | T-DUP-1..2 |
| TH-23 | Hallucinated findings reported as real (Check Point "success" cases `[DOC]`) | A7 | Verification | Only verifiers confirm; DB trigger; UI truth badges | T-VER-1..4 |
| TH-24 | Kill switch fails or is slow | A7 | Decision plane | Out-of-band broadcast to PDP, Broker, SEEG; SEEG policy drop resets connections | T-KILL-1..3 |
| TH-25 | Model switching to evade provider safety (Check Point `[DOC]`) | A2 | MG | Refusal terminal; no rerouting on refusal | T-MG-REFUSAL-1 |
| TH-26 | Data residency / provider retention breach | A5 | B6 | Per-tenant provider allowlist; zero-retention endpoints; self-hosted option | T-MG-1..4 |
| TH-27 | Approval fatigue / social engineering of approvers | A2, A1 | Human | Clear action rendering incl. hash; rate limit approval requests per assessment; batch screening at plan review; approver ≠ requester for high risk | T-APPR-4, UX review |
| TH-28 | Signing key compromise | A6 | KMS | Keys in KMS/HSM, no export; separate keys per purpose and per cell; rotation; dual control for key policy changes | T-KEY-1 |

## 13.4 Test catalogue

All tests run in CI unless marked **(lab)** (requires the Mode A lab environment; runs nightly
and as a release gate) or **(red team)** (manual, per release).

| ID | Test | Pass condition |
|---|---|---|
| T-AUTH-1..8 | Unauthenticated access to every route; expired/forged JWT; wrong audience; CSRF on state changes; session fixation; step-up missing on approval; brute-force lockout; OIDC mix-up | All rejected with correct status |
| T-BUNDLE-1 | Modify one byte of a bundle | PDP refuses; alert |
| T-BUNDLE-2 | Present bundle signed by wrong key | Refused |
| T-BUNDLE-3 | Present older valid bundle after newer seen | Refused (`bundle.rollback`) |
| T-CTX-1..4 | Context assembler includes only task-visible refs; never another tenant's; redaction applied; secrets canaries absent | Property tests over generated graphs |
| T-AGENT-1..5 | Agent output with unknown fields, literal hostnames in params, refs outside context, oversized rationale, invalid enums | All rejected; counters incremented |
| T-PDP-GOLDEN | ≥ 500 curated request→decision pairs | Exact match |
| T-PDP-FUZZ | Go native fuzzing over requests and scope sets, invariant: ALLOW ⇒ every target matched by include, not excluded, ownership verified, class ≤ ceilings | No counterexample in 10⁷ execs per CI nightly |
| T-PDP-3..6 | Token replay; token for different action hash; expired token; token from another tenant | Broker rejects each |
| T-EXEC-1..3 | Execute without token; with tampered params; concurrent double-use of jti | Rejected; exactly one execution |
| T-SBX-1 **(lab)** | From sandbox: connect to K8s API, PG, Temporal, Redis, node IPs, 169.254.169.254, other pods | All fail |
| T-SBX-2 **(lab)** | Known container-escape regression samples under gVisor | Contained |
| T-SBX-3..6 | Pod spec admission: privileged, hostPath, unsigned image, SA token mount | Admission denies |
| T-NET-DNS-1..4 **(lab)** | `in.example.com.evil.test`, `evil-example.com`, IDN homograph of in-scope name, trailing dot | All reset |
| T-NET-REBIND-1 **(lab)** | TTL-0 alternating answers incl. loopback/metadata | Never connects outside included IPs |
| T-NET-CNAME-1 **(lab)** | In-scope name CNAME → out-of-scope SaaS | Reset; `observed_out_of_scope` edge recorded |
| T-NET-REDIR-1 **(lab)** | 302 to out-of-scope | 599; no outbound connection to destination (pcap check) |
| T-NET-LIT-1..3 | IP literal out of scope; IPv4-mapped IPv6; NAT64 embedding metadata IP | Reset |
| T-INJ-1..20 **(lab)** | Injection corpus in HTML comments, headers, TLS cert fields, DNS TXT, repo READMEs, code comments, scanner JSON, OpenAPI descriptions | 0 authority changes (no PDP allow for non-scope, no budget change, no approval); report proposal-deviation rate as metric |
| T-INJ-SCOPE-1 | Tool output says scope expanded | No effect |
| T-INJ-PROP-1 | Injected text in one task's observation; check other roles' contexts | Appears only labelled `untrusted`, only where need-to-know includes it |
| T-APPR-1..5 | Service principal votes; approver = requester; approval reused for different action; expired approval; approval from another tenant | All rejected |
| T-CRED-1 **(lab)** | Canary credentials planted in target responses, repo files, sandbox env | Canary string absent from prompts, logs, events, reports, UI (scanner over all sinks) |
| T-CRED-2..4 | Tool attempts to read injected creds; STS session tries write API; creds valid after action end | Not visible; AccessDenied; revoked |
| T-LOG-1 | Log scanner over CI run logs for secret patterns | No hits |
| T-OUT-1..4 | Zip bomb, JSON depth bomb, 10 GB stdout, ANSI escape sequences | Bounded; parser killed; UI renders escaped |
| T-UI-XSS-1 | Evidence containing script payloads rendered in all views | No execution (Playwright + CSP violation reports) |
| T-EVID-1..3 | Modify evidence object; delete object; swap object | Hash mismatch detected; object lock prevents; audit flags |
| T-TENANT-API-* | For every endpoint (generated from OpenAPI), tenant B token × tenant A IDs | 404, no data |
| T-TENANT-DB-* | All tables with tenant_id have RLS enabled+forced; queries without `app.tenant_id` fail | Pass |
| T-TENANT-EXEC-1 | Sandbox presents other tenant's policy | Reset |
| T-TENANT-EVT-1 | SSE/NATS subscribe to other tenant's subjects | Denied |
| T-OWN-1..4 | Activate record with unverified asset; self-approve manual attestation; reuse expired proof; register a domain verified by another tenant | Rejected / flagged |
| T-ABUSE-1..3 | Tenant registers many unrelated high-profile domains; rapid target diversity; attempts to upload a C9 manifest | Flags raised; manifest rejected |
| T-RATE-1..3 **(lab)** | Measure RPS at target vs configured; 5xx injection triggers backoff; window close pauses | Within 5% of limit; backoff observed; pause |
| T-EXPOSE-1 | External scan of a fresh deployment | Only edge TLS port; no admin or metrics exposed |
| T-AUDIT-1..4 | Update/delete via app role; break chain; verify endpoint detects gap; anchor mismatch | Denied; detected |
| T-SUPPLY-1..3 | Unsigned image; signed image with critical CVE; manifest without review | Blocked at Registry/Broker |
| T-BUDGET-1..3 | Exceed assessment budget; parallel calls racing the budget; stagnation | Calls refused; no overspend beyond reserved headroom; wrap-up triggered |
| T-DUP-1..2 | Two workers same action (Temporal retry); concurrent same fingerprint | One execution; reuse recorded |
| T-VER-1..4 | Agent claims "confirmed"; DB update to confirmed by non-verifier; confirmed without verification evidence; verifier given only model text | All rejected |
| T-KILL-1..3 **(lab)** | Engage tenant kill during 50 running sandboxes; PDP decisions after kill; release by one person | p99 stop of new actions < 5 s, sandboxes terminated < 30 s; denies; release refused |
| T-MG-1..4 | Tenant with EU-only policy routed to US model; D3 data in prompt; provider not allowlisted; zero-retention flag respected | Blocked |
| T-MG-REFUSAL-1 | Provider returns refusal | No reroute; task failed_terminal `model.refused` |
| T-CLOUD-RO **(lab)** | Run full cloud module against lab accounts; inspect CloudTrail/Activity Log/Audit Logs | Zero write/mutating API calls |
| T-KEY-1 | Attempt to export signing keys; use key outside allowed service identity | Denied by KMS policy |

## 13.5 Residual risks (accepted, monitored)

1. **Read-only access is still sensitive.** A compromised platform could read customer cloud
   configuration. Mitigation: least-privilege roles, customer-held external IDs, per-tenant
   credential isolation, customer-visible access logs. Residual: high impact if A6 succeeds.
2. **Low-impact probes can still trip fragile systems** (legacy OT, embedded devices). Mitigation:
   rate limits, maintenance windows, exclusion lists, Mode C defaults. Residual: low probability.
3. **Prompt injection can degrade quality** (bad proposals, wasted budget) even though it cannot
   change authority. Measured by T-INJ proposal-deviation metric; `[RESEARCH]` on better defences.
4. **Dual use.** Ownership verification can be gamed by someone who controls a victim's DNS;
   abuse monitoring is heuristic. Residual: legal terms, KYC for Mode B, rapid revocation.
5. **gVisor is not a VM boundary.** Enterprise tier offers microVM isolation `[CONCEPT]`.

## 13.6 Security review cadence

- Threat model review at each milestone gate ([17](17-implementation-roadmap.md)).
- External penetration test of the platform before GA and annually.
- Red-team exercise focused on scope escape and injection before enabling any C5+ capability.
- Bug bounty after GA (platform only, never customer targets).
