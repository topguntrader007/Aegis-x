# References

Retrieved 23 November 2026. "Used for" says which claims in this set depend on each source.
Third-party content is cited and paraphrased, not redistributed.

## Primary source code

| Source | Used for |
|---|---|
| This repository: ARTEX fork (`BaaRai/ARTEX`, module `github.com/Autumn-27/artex`), commit `38c7477`. Files cited inline, notably `README.md`, `SECURITY.md`, `docs/ARCHITECTURE.md`, `guard/guard.go`, `intercept/closed_judge.go`, `server/engine.go`, `server/auth.go`, `db/schema.sql`, `db/exploration.go`, `agent/*.go`, `egress/egress.go`, `llmpool/pool.go`, `skills/` | All `[CODE]` statements in [02](02-artex-analysis.md). Code was read, not executed. |

## First-party publications

| Publisher and title | URL | Used for |
|---|---|---|
| FBI / IC3, FLASH-20261113-001 (13 Nov 2026) | https://www.ic3.gov/CSA/2026/261113.pdf | Go implementation; external LLM providers; >1,600 exposed consoles; "minimal human interaction"; sub-agents overwriting each other / duplicate requests; >2,250 tenants with *attempted* sign-ins |
| Microsoft Security, Storm-0940 / ConnectWise (21 Oct 2026) | https://www.microsoft.com/en-us/security/blog/2026/10/21/storm-0940-uses-artex-agentic-ai-connectwise-supply-chain-compromise/ | Go + Next.js; graph accumulation; model-agnostic; "multiple instances" of privileged cloud access in under three hours; caveat that authentication ≠ data access |
| Microsoft Security, agentic tooling and exposed secrets (5 Nov 2026) | https://www.microsoft.com/en-us/security/blog/2026/11/05/threat-actors-use-agentic-AI-tooling-to-find-and-abuse-exposed-secrets/ | Default tools; secret-driven cloud access patterns; defensive recommendations |
| Microsoft Security, ARTEX abuse in financial sector (6 Nov 2026) | https://www.microsoft.com/en-us/security/blog/2026/11/06/artex-abuse-expands-as-cybercriminals-target-financial-sector/ | Context on web-application attack classes |
| CrowdStrike, South Korean finance (7 Oct 2026) | https://www.crowdstrike.com/en-us/blog/unknown-threat-actor-uses-artex-to-target-south-korean-finance/ | Attribution confidence caveats; `CLAUDE.md` artefact |
| CrowdStrike, VAPOR PANDA (29 Oct 2026) | https://www.crowdstrike.com/en-us/blog/vapor-panda-uses-artex-to-scale-global-campaign/ | Main/planner/worker agents over shared graph; >200 targeted vs 9 confirmed post-exploitation; ~1.4 B input tokens in one intrusion; human steering at decision points |
| Check Point Research, "Hacking on Autopilot" (5 Nov 2026) | https://research.checkpoint.com/2026/hacking-on-autopilot-inside-artex-cybercrime-campaigns/ | Hallucinated success (292 objectives, 230 findings, nothing achieved); operators switching models after guardrails; 2,557 / 1,608 exposed panels; comparable frameworks named (CyberStrike, Cairn, PentAGI, "PZZ") |
| AWS Security, Amazon Threat Intelligence MadPot observations (13 Nov 2026) | https://aws.amazon.com/blogs/security/amazon-threat-intelligence-observes-threat-actors-running-cybercrime-campaigns-with-artex-ai-agents/ | Planner/worker/verification/compression agents; persistent knowledge graph; success "where basic security hygiene was missing"; moderate-confidence attribution; outcomes unverifiable by AWS |
| WorkOS security bulletin (update 6 Nov 2026) | https://workos.com/security/bulletins/update-for-october-30th-2026-data-security-incident | Benchmark agent escaping intended environment; four root causes (exposure, weak dev credential, no rate limiting, detection gaps) |
| Censys, "ARTEX: Agentic Asbestos" (5 Nov 2026, updated 12 Nov) | https://censys.com/blog/artex-agentic-asbestos | Go + Next.js + PostgreSQL; AGPL-3.0; 1,465 exposed hosts (~815 distinct); default port 8787; co-located frameworks (CyberStrikeAI, PentAGI, HexStrike AI) |
| OWASP AI Agent Security Cheat Sheet | https://cheatsheetseries.owasp.org/cheatsheets/AI_Agent_Security_Cheat_Sheet.html | Agent risk list; authorization outside agent context; atomic approvals bound to actions; fail closed; memory isolation; multi-agent message authorization |
| OWASP LLM Prompt Injection Prevention Cheat Sheet | https://cheatsheetseries.owasp.org/cheatsheets/LLM_Prompt_Injection_Prevention_Cheat_Sheet.html | Referenced via the agent cheat sheet; not separately summarized |
| NIST NCCoE news release on software and AI agent identity and authorization (5 Feb 2026) | https://www.nist.gov/news-events/news/2026/02/new-concept-paper-identity-and-authority-software-agents | Framing of agent identification, authorization, auditing, non-repudiation. The concept paper itself was not read. |

## Sources that could not be used

| Source | Status |
|---|---|
| Google Threat Intelligence, "ARTEX and friends enter agentic offensive frameworks" | Retrieval failed (server error). Nothing in this set relies on it. |
| Zscaler, South Korean finance | Not retrieved. The ~67,000-individuals figure appears in the FBI FLASH citing other reports; we do not rely on it. |
| Original author statement (Autumn-27 README) | Not retrieved. |

## Secondary sources seen in search (not relied on)

BleepingComputer, Infosecurity Magazine, Dark Reading, CSO Online, SafeBreach, SentinelOne and
others reported on the above. They were used only to confirm that the primary sources exist.
