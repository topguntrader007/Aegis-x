# 12. User interface specifications

Status: `[SPEC]` for information architecture, views, components and journeys; `[CONCEPT]`
for final visual design (needs a design pass with real users).

## 12.1 Design principles

1. **Truth status is always visible.** Every finding, node, edge and statement shows whether it
   is *Verified*, *Platform-derived*, *Tool-reported*, *Hypothesis* or *Human-attested*, using
   shape + icon + text, never colour alone.
2. **Authority is visible.** Every action shown in the UI links to the decision that allowed it
   and the approval (if any).
3. **Calm by default, dense on demand.** Executive views summarize; engineering views expose
   every field.
4. **Stop is always one click away** for those allowed to stop (global header control).

## 12.2 Visual system

| Token | Purpose | Light | Dark |
|---|---|---|---|
| `--surface-0/1/2` | Page, panel, raised | neutral 50/0/100 | neutral 950/900/850 |
| `--text-1/2/3` | Primary/secondary/muted | ≥ 7:1 / 4.5:1 / 4.5:1 contrast | same ratios |
| `--sev-critical/high/medium/low/info` | Severity | colour-blind-safe palette, always paired with label | |
| `--truth-verified` | Verified badge: solid fill + check icon | | |
| `--truth-hypothesis` | Hypothesis: dashed outline + "?" icon, no fill | | |
| `--truth-tool` | Tool-reported: outline + wrench icon | | |
| `--danger` | Kill switch, destructive confirmations | | |

Type: Inter/Geist for UI, JetBrains Mono for evidence. 4-px spacing grid; 12-column layout,
breakpoints 640/1024/1440. Motion ≤ 200 ms, respects `prefers-reduced-motion`.

**Truth badges** (component `TruthBadge`):

```
[✔ Verified · 0.9]   [▣ Platform]   [🔧 Tool-reported]   [?  Hypothesis · medium]   [👤 Attested by J. Doe]
 solid                 solid-muted    outline              dashed outline              outline + avatar
```

## 12.3 Navigation

```
Top bar:  [Tenant ▾] [Search ⌘K]                 [Approvals (3)] [Kill ■] [User ▾]
Left nav:
  Overview ─ Executive Dashboard
  Assess   ─ Assessments (Builder, list)  ─ Agent Operations Center
  Inventory─ Asset Explorer ─ Security Graph
  Findings ─ Findings Workspace ─ Evidence Timeline
  Fix      ─ Remediation Center
  Report   ─ Reporting Studio
  Govern   ─ Governance Console (Authorizations, Policies, Roles, Approvals, Audit, Models, Tools)
```

Role-based visibility: Executives see Overview/Report; Owners see Fix; Auditors get read-only
Govern + Evidence. Unauthorized nav items are hidden *and* the API denies.

## 12.4 Views

### 1. Executive Dashboard
```
┌ Posture ─────────────────────────────────────────────────────────────────┐
│ Confirmed critical 4 ▲1   High 22 ▼6   Hypotheses awaiting verify 31     │
│ Coverage: 87% of in-scope assets tested in last 30d   MTTR (high) 9.5d    │
├ Trend (confirmed by severity, 12 wks) ─────┬ Remediation burndown ───────┤
│  stacked area                              │ open vs verified-fixed       │
├ Top risks (risk score, verified only) ─────┴──────────────────────────────┤
│ 1 Public bucket w/ PII tags · aws:prod-data · ✔Verified · Owner: Data    │
│ 2 Stale admin key 412d · id: ci-deployer · ✔Verified · Owner: Platform    │
├ Assessment coverage by business unit (heatmap) ──────────────────────────┤
└───────────────────────────────────────────────────────────────────────────┘
```
Rule: headline numbers count **confirmed** findings only; hypotheses appear as a separate,
clearly labelled count.

### 2. Asset Explorer
Searchable table (faceted: kind, scope state, owner, criticality, last tested, finding count)
with a detail drawer: properties, provenance, relationships list, findings, executions that
touched it, scope state with the matching scope item. Out-of-scope observed assets appear in a
separate tab with an "Add to authorization (request)" action that opens a draft authorization
change (never adds directly).

### 3. Agent Operations Center
```
┌ Assessment: Q4 Web+Cloud  ● running   Budget: $82/$250 ███░░  Tokens 11.2M/40M ┐
│ [Pause] [Cancel]                Time window: open until 18:00 ET                 │
├ Tasks (kanban) ─────────────────────────────────────────────────────────────────┤
│ Proposed 4 │ Awaiting approval 2 │ Running 6 │ Verifying 3 │ Done 141 │ Denied 7 │
├ Selected task ──────────────────────────────────────────────────────────────────┤
│ cloud#58 · Cloud agent v7 · model: tier M (provider-a/medium)                    │
│ Proposal: run public-exposure-probe on res:aws:s3:prod-exports                   │
│ PDP: ALLOW (bundle v12; reasons: scope.included, capability C3 ≤ C4)  [token]   │
│ Reasoning summary (AI, unverified): "Bucket policy allows s3:GetObject to *…"    │
│ Execution: sandbox exec-7f3… 3.2s exit 0 · evidence ev:9a1… [view]               │
├ Live event stream ─────────────────────── Resource use ─────────────────────────┤
│ 14:03:22 task cloud#58 running            sandboxes 6/20 · RPS per asset ≤ 5   │
└──────────────────────────────────────────────────────────────────────────────────┘
```
Reasoning summaries are collapsed by default and labelled "AI reasoning (not evidence)".
Denied tasks show the PDP reason codes in plain language.

### 4. Security Graph
Interactive canvas (WebGL; Sigma.js or Cytoscape.js — choice in Milestone 5 spike) with:
filters by node/edge type and trust class; "verified only" toggle (default on); path finder
(`paths.exposure`), identity blast radius view; out-of-scope nodes rendered greyed with a lock
icon. Hypothetical edges (model-generated) are hidden unless "Show hypotheses" is on, then dashed.
A table view mirrors the canvas for accessibility (screen readers cannot use canvases).

### 5. Findings Workspace
Split view: list (sortable by risk, severity, verification state, age, owner) | detail:
title, TruthBadge, severity + risk factors breakdown, affected assets, evidence list (with
hashes), verifier used and predicate result, reasoning (AI, collapsed), remediation, history.
Bulk actions: assign owner, accept risk (requires expiry + justification), mark false
positive (requires reason; feeds benchmark labels).

### 6. Assessment Builder (stepper)
1. Objectives (templates: external web posture, cloud posture, identity hygiene, code secrets)
2. Authorization (select active record; shows scope summary and ceiling; link to create one)
3. Testing profile (modules, intensity, verifier strictness)
4. Budgets (≤ authorization budgets; estimate panel from cost model)
5. Approvals policy preview ("these step types will need approval")
6. Review → Create → plan generation → plan review (diff-style list of steps with PDP screening)

### 7. Evidence Timeline
Vertical timeline per finding or per asset: decision → execution → raw evidence → parse →
hypothesis → verification → status changes, each with actor, time, hash, and a "verify hash"
action that recomputes the object hash client-side (WebCrypto) and compares with the audit chain.

### 8. Remediation Center
Board and table of remediations: owner, due (SLA by severity), ticket link/status, reverify
status. "Request reverification" runs the stored reverify plan (subject to PDP). Status
`verified_fixed` only after a passing reverification.

### 9. Governance Console
Tabs: Authorizations (lifecycle, versions diff, approvers), Asset ownership proofs, Policies
(approval, model, retention), Roles & bindings, Approvals queue/history, Audit log with chain
verification, Model providers (allowlist, residency), Tool catalogue (versions, review state),
Kill switch (engage / two-person release).

### 10. Reporting Studio
Templates: Executive, Engineering, Compliance (control mappings), Technical appendix.
Sections are generated from data with optional AI prose; AI-written paragraphs are marked and
must be accepted by a human before export. Every report embeds a manifest (hashes of included
findings/evidence, bundle versions) and a "confirmed vs hypotheses" section split.

## 12.5 Component hierarchy (React, Next.js App Router)

```
app/
  (shell)/layout.tsx ─ AppShell
     ├─ TopBar ─ TenantSwitcher, CommandPalette, ApprovalsBell, KillSwitchButton, UserMenu
     ├─ SideNav
     └─ <page>
components/
  truth/ TruthBadge, ConfidenceBand, ProvenanceTooltip
  evidence/ EvidenceList, EvidenceViewer (HTTP exchange, JSON, code excerpt, redaction markers), HashVerifier
  findings/ FindingTable, FindingDetail, RiskFactorBreakdown, VerifierResult
  graph/ GraphCanvas, GraphTableView, PathFinder, FilterPanel
  ops/ TaskBoard, TaskDetail, DecisionCard, EventStream (SSE), BudgetMeter
  governance/ AuthorizationEditor, ScopeItemTable, VersionDiff, ApprovalCard (shows action hash), AuditTable
  builder/ Stepper, ObjectivePicker, ProfileForm, BudgetEstimator, PlanReview
  reports/ TemplatePicker, SectionEditor, AiParagraph (accept/reject), ExportDialog
lib/
  api/ typed client generated from OpenAPI; tenant never set by client
  sse/ reconnecting EventSource with seq gap detection
```

State: TanStack Query for server state; no client-side caching of evidence content beyond the
view lifetime; URLs encode filters for shareable views.

## 12.6 User journeys

**J1 — First enterprise assessment (Maya, approver Priya)**
1. Maya registers `shop.example.com` and AWS account; completes DNS TXT and role-assumption proofs.
2. Maya drafts authorization (Mode B, ceiling C4, weekday window, 5 rps). Submits.
3. Priya receives approval request, reviews scope diff, approves with passkey. Bundle v1 signed.
4. Maya creates "Q4 external posture" assessment; reviews generated plan (63 steps, 4 need
   approval); removes 2; approves plan.
5. Dev (operator) watches Ops Center; approves 1 C4 check after reading the action detail; 1
   approval expires unused.
6. Findings appear as hypotheses, then 9 become Verified, 3 refuted, 2 unverifiable.
7. Maya assigns owners; tickets created; report exported.

**J2 — Emergency stop**
Dev sees unexpected 5xx spikes on a target in the Ops Center → clicks Kill (assessment scope) →
confirmation dialog → within seconds tasks show `cancelled`, sandboxes terminated, event
`kill.engaged` in audit. Resuming requires two people.

**J3 — Remediation verification (Sam)**
Sam fixes a stale access key, clicks "Request reverification" → PDP allows C2 read →
verifier confirms key deleted → status `verified_fixed`; executive trend updates.

**J4 — Auditor review**
Auditor filters audit log for the assessment, runs chain verification (passes), opens a
finding's Evidence Timeline, verifies evidence hashes client-side, exports an audit package.

**J5 — Continuous monitoring drift (Mode C)**
Nightly read-only run detects a new public storage bucket → Cloud agent hypothesis → verifier
confirms via config read + external probe (C3 enabled in record) → notification to owner group.

## 12.7 Accessibility and quality

- WCAG 2.2 AA: keyboard-complete flows (graph has table equivalent), visible focus, ARIA live
  regions for event streams (rate-limited announcements), 4.5:1 contrast minimum.
- Approvals and kill switch dialogs: focus-trapped, explicit text of the action, no
  default-focused destructive button.
- Internationalization-ready (ICU messages); dates in user locale and zone.
- Performance budgets: LCP < 2.5 s on dashboard with 10k findings (server-side aggregation),
  graph canvas interactive at 5k nodes (level-of-detail rendering).
- Security: strict CSP (nonce-based), no `dangerouslySetInnerHTML` for evidence (rendered as
  text with escape of terminal control sequences), Trusted Types enabled.
