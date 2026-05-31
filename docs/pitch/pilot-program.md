# 60-day pilot program

What a paid pilot looks like for your team. Eight weeks from kickoff to score.

## Week 1-2 — Discovery + topology onboard

**You provide:** read-only NetBox export (or equivalent CMDB) for one fabric, one tenant of your choosing.

**We deliver:**

- `fabric.yaml` rendered from your topology — single source of truth for the platform.
- Knowledge Graph seeded with your devices, interfaces, EVIs, VTEPs, BGP-EVPN peers.
- Compliance policy template tailored to your environment (PCI-DSS, SOX, FINRA, HIPAA tags as relevant).
- Read-only MCP adapter for your platform (NX-OS / EOS / SR Linux / IOS-XE as needed).

**You see:** the platform UI showing your real fabric model. No changes proposed yet.

## Week 3-6 — Three incident classes in dry-run

**You pick three:**

- BGP-EVPN session loss (admin-shut, peer down, VTEP unreachable, VNI inactive, MAC inconsistency)
- OSPF underlay neighbor loss (dead timer, interface down)
- BGP-unicast peer loss (admin reset, hold timer)
- Interface fault (link flap, line protocol)
- Security event (failed auth, ACL violation)
- (Or one of your custom triage patterns — we add the rule.)

**We deliver:**

- The 7-pass pipeline running against your fabric in **dry-run mode** (incidents trigger from real syslogs, fixes proposed and validated, but **never applied**).
- Approval cards in the UI for your team to review.
- Each incident's evidence cited to a KG query, policy rule, or DT result.

**You measure:**

- Did the platform's proposed fix match what your engineer would have proposed?
- Did the CRO BLOCK match your change-management policy?
- Did the blast radius dollar number match your tenant economics?
- Was the cited evidence enough to make an approve/reject decision?

## Week 7-8 — Score + decide

**Metrics we score (against your historical baseline):**

| Metric | How it's measured |
|---|---|
| **MTTR delta** | Time from syslog to approval-ready card vs. your current diagnosis-to-fix time |
| **False positive rate** | Of N incidents, how many proposed fixes were ones your engineer would reject |
| **Policy gate hit rate** | How often the CRO BLOCK or MANUAL_REVIEW matched your team's actual judgment |
| **Cited evidence quality** | Engineer survey: "could you make the approval decision from the card alone?" |
| **Operator hours saved** | Time spent reviewing approval cards vs. time spent diagnosing from scratch |

**What you get at week 8:**

- A signed pilot report with the five metrics above + raw data.
- A go/no-go conversation: keep running, expand to production, or stop with no obligation.

## Pricing

| Pilot scope | List | Notes |
|---|---:|---|
| Single fabric, single tenant, 3 incident classes | **$50,000** | Default. Most pilots run this scope. |
| Single fabric, full tenant set, 5 incident classes | **$85,000** | For multi-tenant fabrics with high regulatory complexity. |
| Multi-fabric (2-3), single tenant each | **$120,000** | Stress-tests the topology-as-code single source of truth. |

All pilots include:

- Setup + onboarding (week 1-2)
- Dry-run + tuning (week 3-6)
- Scoring + report (week 7-8)
- 5 engineering hours/week for issue triage + custom rules
- All artifacts (fabric.yaml, policies.json, approval-card schemas) — yours to keep if pilot doesn't continue

What pilots do NOT include:

- Production change-execution authority. The pilot runs in **dry-run mode**. Live application of fixes requires a separate go-live engagement after the pilot scores cleanly.
- Custom vendor adapter development beyond the four shipped (NX-OS, EOS, SR Linux, IOS-XE). Custom vendors are quoted separately.
- 24×7 ops support. Pilot is business-hours (US-East or EU as you prefer).

## What's expected of your team

| Role | Time commitment |
|---|---|
| Network architect | 4 hrs/week — fabric model review, policy tuning |
| Senior network engineer | 2 hrs/week — incident-class selection, approval-card review |
| SecOps / compliance | 2 hrs total — policy template review at week 2 |
| ITSM lead | 1 hr total — MTTR baseline data export at week 1 |

Total team commitment: **~50 person-hours over 8 weeks**.

## Booking a pilot

Email `dulharu.eduard@gmail.com` with:

1. Your fabric size (devices, tenants)
2. Vendor mix
3. Compliance regime (PCI, SOX, FINRA, HIPAA, ITAR, etc.)
4. Two-sentence description of the incident class that hurts most

You'll get a 30-minute discovery call within 48 hours.
