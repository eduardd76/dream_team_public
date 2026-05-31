# ROI math

!!! warning "Illustrative — adjust to your environment"
    These numbers are conservative defaults. Replace them with **your** outage cost, **your** MTTR, and **your** incident count. The math is structured so you can defend every line to a CFO.

## The model

```
Annual outage savings = (Current MTTR − Platform MTTR) × Outage cost/hr × Outages/year
```

## Default inputs

| Variable | Default | Source for your environment |
|---|---:|---|
| Current MTTR | 4 hr | Your incident-management system (ServiceNow, PagerDuty, Jira) |
| Platform MTTR (after pilot) | 4 min | Pilot-measured; see [pilot program](pilot-program.md) week 7-8 |
| Outage cost/hour | $50,000 | Industry: $40K-$300K/hr for Fortune 1000 (Gartner 2024); your CFO has the actual number |
| Incidents/year | 10 | Pull from your ITSM; common range 5-50 for mid/large fabrics |

## The math at default inputs

| Line | Value |
|---|---:|
| MTTR delta | 3 hr 56 min ≈ **3.93 hr** |
| Avoided cost per incident | 3.93 × $50K = **$196,500** |
| Annual savings | $196,500 × 10 = **$1,965,000** |
| 60-day pilot cost | **$50,000** (single fabric, one tenant) |
| Net Year-1 ROI | **38×** |

## Sensitivity

| Scenario | MTTR delta | Outage cost/hr | Incidents/yr | Annual savings |
|---|---:|---:|---:|---:|
| Conservative | 2 hr | $25K | 5 | **$250K** |
| Default | 3.93 hr | $50K | 10 | **$1.96M** |
| Aggressive | 3.93 hr | $150K | 25 | **$14.7M** |

## What we don't claim

- We don't claim 0% false positive rate. The pilot's week 7-8 scoring includes false-positive rate as a primary metric.
- We don't claim 0 operator hours. We claim **fewer hours per incident** because the diagnosis + validation work happens before a human reads the card.
- We don't claim the platform replaces senior network engineers. It changes their workload from "diagnose at 3am" to "review the cited evidence + approve."

## What's NOT in this ROI number

Risk that's hard to price but real:

- **Audit defensibility.** SOX / PCI auditors increasingly ask "show me the policy that gated this change." This platform produces that artifact.
- **Operator retention.** Senior engineers leave when the on-call burden is high. Reducing 3am pages has retention impact.
- **Cross-team confidence.** A change that BLOCKs at the CRO is a change that doesn't reach the CAB meeting, doesn't require a 2-hour debate, doesn't get rolled back at 6pm Friday.

## Counter-arguments your CFO might raise

| "But..." | Response |
|---|---|
| "We already have a CAB / change board." | The platform produces the CAB-grade decision *in 90 seconds*, with cited evidence. It feeds your CAB; it doesn't replace human governance. |
| "What if the digital twin is wrong?" | Pass 4 is **one of seven gates**. A DT mistake doesn't auto-approve. The human still sees the card. |
| "$50K outage/hr seems high." | Pull the number from your last 3 P1 outages. Multiply downtime hours × revenue/hr. Many CFOs find the real number is higher than they expect. |
| "What if the LLM hallucinates?" | The LLM never decides "is this safe?". Rule-based gates do. The LLM proposes; rules dispose. Every approval-card line is rule-cited. |

See [pilot program](pilot-program.md) for how week 7-8 scoring quantifies these for your specific fabric.
