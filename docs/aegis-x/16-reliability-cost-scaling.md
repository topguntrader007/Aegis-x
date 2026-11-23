# 16. Reliability, scaling and cost analysis

Status: `[SPEC]` for mechanisms and SLO definitions; SLO numbers are **design targets**. The cost
model is a **parameterized model**; the worked example uses explicitly assumed inputs, not
measurements or quoted prices. Replace inputs with measured values from benchmark runs and
current provider price sheets.

## 16.1 Scale target

- Up to **5,000 in-scope assets per assessment**, **50 concurrent assessments per cell**,
  assessments running **hours to 2 weeks**.
- Rough action volume: ~20–60 actions per asset per full assessment (discovery, probes, reads,
  verification) → 100k–300k actions per large assessment (planning assumption; to be measured).

## 16.2 Reliability mechanisms

| Concern | Mechanism |
|---|---|
| Horizontal scaling | All services stateless behind gRPC load balancing; agent runtime and verification scale on Temporal task-queue backlog (KEDA); Broker scales on pending executions; SEEG scales on connections; cells add capacity in larger steps |
| Durable queues | Temporal task queues (durable, at-least-once activity delivery) for all work; NATS only for notifications (loss tolerated, outbox replays) |
| Worker scheduling | 05 §5.5: per-role and per-execution-class queues; weighted fair dispatcher; per-asset leases |
| Task checkpointing | Workflow state = Temporal history; long assessments use continue-as-new every 2,000 steps carrying a compact state snapshot (budgets, queue, cursors); tool results and observations persisted before workflow advances |
| Agent crashes | Agents are stateless activities; crash → activity retry on another worker; idempotent by `invocation_id` |
| Model outages | Gateway circuit breakers; same-tier fallback (availability only); tasks park in `waiting_model`; assessments keep running non-LLM work (scheduled deterministic checks, verification) |
| Partial failures | Each action independent; failures recorded in EHG; planner sees failures as data; assessment completes with explicit coverage gaps rather than failing wholesale |
| Duplicate task execution | Single-use decision tokens (`jti`), action fingerprints, idempotent activity design, Temporal workflow ID uniqueness |
| Distributed locking | PG leases with fencing tokens for per-asset concurrency (pattern proven in ARTEX's intent claims `[CODE]`); no Redis locks for correctness |
| Database contention | Append-only ledgers instead of counter rows; partitioned event tables; batching of graph upserts (≤ 500 rows/txn); read replicas for dashboards; PgBouncer |
| Backpressure | Bounded queues per tenant; dispatcher stops issuing when Broker/SEEG saturation signals (queue depth, p95 latency) exceed thresholds; API returns 429 on assessment creation when cell is saturated; Model Gateway per-provider token buckets |
| Resource quotas | Per-tenant K8s ResourceQuota in execution namespaces; per-assessment budgets (tokens, cost, sandbox-seconds, requests); per-asset rate limits |
| Graceful cancellation | Cancel signal → workflow stops dispatch, waits up to 30 s for running actions, then Broker kill; final state + partial report produced |
| Recovery after restart | Temporal replays workflows; Broker reconciles running pods vs DB on start (orphans killed, results of completed pods collected if within token validity); SEEG reloads policies from Broker; PDP reloads latest signed bundles |

## 16.3 Service-level objectives (targets)

| SLO | SLI | Target | Window |
|---|---|---|---|
| API availability | Successful (non-5xx) requests / all | 99.9% | 30 d |
| API latency | p95 read endpoints | < 300 ms | 30 d |
| PDP latency | p99 `Decide` | < 10 ms | 30 d |
| PDP availability | Successful decisions / requests | 99.95% | 30 d |
| Kill switch (new actions) | Time from engage to last allowed decision | p99 < 5 s | per event |
| Kill switch (running) | Time from engage to all sandboxes terminated | p99 < 30 s | per event |
| Execution start latency | Decision → sandbox running | p95 < 10 s (pre-warmed pools); p95 < 30 s cold | 30 d |
| Worker reliability | Succeeded / dispatched (excl. policy denials, target-caused failures) | ≥ 99% | 30 d |
| Task durability | Tasks lost after infra failure | 0 | — |
| Recovery time | Node/process failure → assessment progressing | p95 < 2 min | per event |
| Event freshness | Execution event → visible in UI | p95 < 3 s | 30 d |
| Audit integrity | Chain verification job passes | 100% | daily |
| Scope violations | Unauthorized connections | 0 | always (incident if > 0) |

Error budget policy: when an SLO's 30-day budget is exhausted, feature releases for that
component pause in favour of reliability work; scope violation is a Sev-1 incident regardless.

## 16.4 Capacity model (per cell, planning assumptions)

| Component | Sizing driver | Assumption |
|---|---|---|
| Sandboxes | Concurrent actions | 50 assessments × 20 concurrent actions = 1,000 concurrent pods at peak; avg action 15 s, 0.25 vCPU, 256 MiB |
| SEEG | Connections | ~5–10 connections per active action; 10k concurrent connections per cell; 4–8 replicas |
| PDP | Decisions/s | ≤ 200/s per cell peak; trivial for in-memory evaluation; 3 replicas for HA |
| Temporal | Actions/s | Each action ≈ 6–10 workflow events; 200 actions/s → ~2k events/s; size per Temporal guidance |
| PG | Writes/s | Events + observations + graph upserts ≈ 3–5k rows/s peak with batching |
| Agent runtime | Concurrent LLM calls | Bounded by Model Gateway quotas; 200 concurrent calls per cell |

These are starting points for the Milestone 6 load test, not validated numbers.

## 16.5 Cost model

### 16.5.1 Formula

For one assessment:

```
C_total = C_model + C_compute + C_storage + C_telemetry + C_ops_share

C_model     = Σ_roles Σ_calls [ (T_in − T_cached)·P_in(m) + T_cached·P_cached(m) + T_out·P_out(m) ]
C_compute   = Σ_actions (vCPU_s · P_vcpu_s + GiB_s · P_gib_s) + C_seeg + C_broker_share
C_storage   = (evidence_GB · P_obj_GB_month · months_retained) + (PG_GB · P_pg_GB_month · share)
C_telemetry = (spans + log_GB + metric_series) · P_telemetry
C_ops_share = (fixed platform cost per cell per month / assessments per cell per month)

Cost per verified finding = C_total / N_confirmed_true_positive
```

### 16.5.2 Model-call volume drivers

```
calls ≈ N_assets × (r_extract + r_triage) × S-tier
      + N_hypotheses × (r_correlate) × M-tier
      + N_replans × M-tier + 1 initial plan × L-tier + occasional L escalations
      + N_confirmed × (r_remediate) × M-tier
```

The routing policy (11.4) and execution reuse (07 §7.9) are the main levers; the single-turn,
budgeted-context design (11.5) bounds `T_in` per call instead of letting transcripts grow.

### 16.5.3 Worked example (illustrative only)

**All inputs below are assumptions chosen to show the arithmetic.** They are not measured AEGIS-X
values and not quoted provider prices.

| Input | Assumed value |
|---|---|
| Assets | 1,000 |
| S-tier calls | 1,000 × 4 = 4,000 calls × (3k in, 0.4k out) tokens |
| M-tier calls | 1,500 calls × (8k in, 1k out), 50% of input cached |
| L-tier calls | 40 calls × (30k in, 4k out) |
| Prices per 1M tokens (in / cached in / out) | S: $0.20 / $0.05 / $0.80 · M: $3 / $0.30 / $15 · L: $15 / $1.50 / $75 |
| Actions | 40,000 × 15 s × 0.25 vCPU; $0.00001 per vCPU-s, memory included |
| Evidence | 20 GB retained 13 months at $0.023/GB-month |
| Fixed cell cost share | $150 per assessment |
| Confirmed true positives | 60 |

Computation:

- S: in 12M × 0.20 = $2.40; out 1.6M × 0.80 = $1.28 → **$3.68**
- M: in 12M (6M cached): 6M × 3 + 6M × 0.30 = $19.80; out 1.5M × 15 = $22.50 → **$42.30**
- L: in 1.2M × 15 = $18.00; out 0.16M × 75 = $12.00 → **$30.00**
- **C_model ≈ $76**
- C_compute: 40,000 × 15 × 0.25 = 150,000 vCPU-s × $0.00001 = **$1.50** (+ SEEG/Broker share, say $10) → **≈ $11.50**
- C_storage: 20 × 0.023 × 13 ≈ **$6**
- C_telemetry: assume **$5**
- C_ops_share: **$150**
- **C_total ≈ $249; cost per verified finding ≈ $4.15** under these assumptions.

What the example shows (structurally, not numerically): fixed platform cost and M-tier output
tokens dominate; sandbox compute is minor; caching and S-tier routing matter mostly through the M
tier. Compare an "all-L" routing policy under the same assumptions: replacing S+M calls with L
pricing raises C_model several-fold, which is why routing is a benchmark dimension (14.5).

### 16.5.4 Operational overhead (fixed, per cell per month)

Line items to estimate at Milestone 6 with real quotes: managed PG (HA), Temporal (self-hosted
nodes or managed), Kubernetes control planes ×2 (control + execution clusters), node pools
baseline, NAT/egress IPs, KMS/Vault, observability backend, object storage, on-call staffing
share, security operations share, external pentest amortization.

### 16.5.5 Cost controls

Per-assessment budgets enforced pre-call; tenant monthly caps; cost dashboards; stagnation
detector stops low-yield loops; execution reuse; prompt-cache-friendly prompt layout;
self-hosted S-tier models for high-volume extraction when tenant policy and data volume justify
the fixed GPU cost (break-even computed from measured S-tier volume).

## 16.6 Scaling path

1. Single cell, single region (MVP).
2. Multiple cells per region (tenant placement by load; enterprise dedicated cells).
3. Multi-region with data residency: tenants pinned to a home region; no cross-region tenant data.
4. Optimizations triggered by measurements: pre-warmed sandbox pools, SEEG in Rust, graph engine
   (15.2 criteria), self-hosted models.
