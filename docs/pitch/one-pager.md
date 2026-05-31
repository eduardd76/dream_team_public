# Agentic NetOps — Risk-Managed Autonomous Network Remediation

*A multi-vendor agentic platform that diagnoses, validates, and gates
every proposed network change against your written policies — before a
human approves it.*

---

## The Problem

Network outages cost Fortune 1000 operators between $300K and $5M per
hour, and 70-80% of them trace back to human change (Uptime Institute,
Gartner, ITIC public surveys). The current state of the art is an
on-call engineer reading a runbook at 02:00 and pasting commands they
have not slept on. MTTR is measured in hours; the gap between detection
and a defensible fix is where revenue, regulatory exposure, and customer
trust leak out. The economics of one prevented Sev-1 fund a year of the
platform.

## What We Built

A 5-layer multi-agent platform that picks up a syslog, classifies it
deterministically, dispatches it to a specialist agent that diagnoses
against live device state plus a Knowledge Graph, then runs the proposed
fix through a 7-pass validation pipeline before it ever reaches a human
approval card.

- **Multi-vendor by design.** First-class Cisco NX-OS, Arista EOS, Nokia
  SR Linux, Cisco IOS-XE; the adapter contract is documented and a new
  NOS is a defined integration. See `agents/shared/vrnetlab_backend.py::_VENDOR_EXEC_SHAPES` (vendor command shapes) + `agents/shared/validate_node.py:152-220` (per-vendor EVPN recovery syntax).
- **Topology-as-code.** One YAML file (`fabric.yaml`) is the source of
  truth. Generators emit the digital twin, the KG seed, the NetBox
  import, and the test fixtures from it.
- **Compliance-aware.** Six default policy classes (business-hours,
  weekend freeze, quarter-close freeze, PCI-DSS, HIPAA, high-risk
  commands). Customer-replaceable JSON data, customer-immutable policy
  code. See `agents/change_risk_officer/policies.json`.
- **Multi-tenant.** Per-VNI tenant mapping with SLA + compliance tags.
  See `agents/blast_radius_forecaster/tenants.json`.
- **Auditable.** Every decision is a chain ID in TimescaleDB; every
  gate cites a rule, a policy, or a KG query. No LLM judgments on
  whether a change is allowed.

## The 7-Pass Validation Pipeline (per incident)

| Pass | What it does | Why your operators trust it |
|---|---|---|
| 1: ConfidenceScorer | Pre-sim gate score from risk signals + LLM confidence. Routes the fix to graph-sim, full DT, or direct-to-human. | Rule-based bands; no LLM coin flip. `agents/shared/confidence_scorer.py` |
| 2: Fault Correctness | Per-fault-type pre/post assertions. Did the fix actually address the fault? | The LLM does not get to grade itself. `agents/shared/validate_node.py::validate_fix_correctness` |
| 3: Graph Simulator (Layer 1 DT) | In-memory SPF, reachability, blast radius against the KG snapshot. | Milliseconds; deterministic; KG-grounded. `agents/shared/graph_simulator.py` |
| 4: Containerlab Twin (Layer 2 DT) | Renders containerlab YAML from `fabric.yaml`, brings up SR Linux + cEOS + FRR, applies the fix, runs the validation suite. | Real network process state, not a model of one. `agents/shared/containerlab_manager.py::validate_evpn` |
| 5: Change Risk Officer | Policy gate. ALLOW / MANUAL_REVIEW / BLOCK with cited policies. | Auditable. Your GRC team owns the JSON; we own the code. `agents/change_risk_officer/policies.py` |
| 6: Blast Radius Forecaster | Translates the proposed fix to dollars: SLA exposure per hour, affected tenants, users, services. | CFO language. KG-driven; convergence windows per RFC. `agents/blast_radius_forecaster/agent.py` |
| 7: Post-Change Verifier baseline | Snapshots pre-change state so the executor can diff post-apply. Verdicts: PASSED / REGRESSION / FIX_DID_NOT_TAKE / INDETERMINATE. | Closes the autonomous loop. `agents/post_change_verifier/agent.py` |

## ROI math (illustrative — calibrate during discovery)

These are the numbers we use to frame the pilot conversation. They are
not your numbers; the pilot measures yours.

- Industry MTTR for human-triaged network change incidents: **2-6 hours**
  (Gartner DC Operations 2024; Uptime Institute Annual Outage Analysis 2024).
- Platform-assisted MTTR target on the pilot fault classes: **under 30
  minutes** (deterministic triage in seconds, specialist diagnosis +
  7-pass validation under 90 seconds, human approval is the wall clock).
- Average Sev-1 outage cost for a Fortune 1000 enterprise: **$300K–$1.4M
  per hour** (ITIC 2024 Hourly Cost of Downtime Survey, conservative band).
- Worked example, conservative midpoint — $500K/hr, 12 Sev-1 incidents
  per year, MTTR delta 90 minutes: **savings ≈ $9M/yr**. Halve every
  assumption and it is still seven figures.
- Indicative 60-day pilot: well under 1% of a single prevented Sev-1.

Conservative is the rule. Every number in the table above is sourced or
the customer can replace it during scoping.

## Why this isn't another AI tool

- **Every decision is cited.** CRO BLOCKs cite a policy ID; BRF dollars
  cite a tenant SLA + a convergence window; triage cites a triage rule.
  The LLM writes the diagnosis text; the gates are code.
- **No LLM judgments on whether a change is allowed.** The decision to
  ALLOW, BLOCK, or MANUAL_REVIEW is made by `BasePolicy.evaluate()`
  subclasses. Regex matches and calendar checks, not opinions.
- **Multi-vendor by design, not bolted on.** EVPN syslogs from NX-OS
  and Arista land in the same dispatcher; the cross-vendor RT-match
  invariant is a unit test (`tests/test_kg_evpn_sync.py`).
- **Topology-as-code.** One YAML drives the DT, the KG seed, the
  NetBox import, the test fixtures. When the design changes, one file
  changes; the drift detector reconciles intent (NetBox) vs actual
  (KG, synced live).
- **Audit trail by default.** TimescaleDB-backed reasoning chains;
  every recommendation has a chain ID; transcripts surface it via
  `:8505`.

## What a 60-day pilot looks like

- **Week 1-2 — Onboarding.** Drop your `fabric.yaml` (or we extract it
  from your NetBox + a config dump). Hook the syslog feed. Stand up the
  KG. Adapt the policy data in `policies.json` to your GRC reality.
- **Week 3-6 — Three fault classes in shadow mode.** You name three
  fault classes that currently cost your team hours. We run the platform
  in shadow against live incidents — no auto-apply, just the gates
  firing and the approval cards rendering. Your team grades them.
- **Week 7-8 — Score.** MTTR delta vs your baseline. False positive
  rate on gates. Operator-hours saved per incident. CRO + BRF citation
  audit-readiness review with your compliance lead.
- **Exit.** Joint write-up. Pilot extension or scoped production
  deployment.

## Get a pilot

Contact: _operator fills in name / email / calendar link_

Workshop / deep-dive references in this repo: `DEMO_RUNBOOK.md §10`
(pitch scripts), `EVPN_NATIVE_ARCHITECTURE.md` (multi-vendor
architecture), `docs/DT_V3_PERSISTENT_SHADOW.md` (the scale answer),
`WORKSHOP_STATUS.md` (live system state).
