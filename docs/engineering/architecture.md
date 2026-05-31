# Agentic NetOps — Comprehensive Architecture Analysis & Assessment

**Scope:** every system and subsystem of `agentic-netops-mvp` as it exists now (branch `main`, the reconciled superset), the end-to-end data flow, and an explicit assessment — including the infrastructure source-of-truth question. **§9 (added 2026-05-27)** documents the in-progress **vendor-agnostic evolution** (branch `vendor-abstraction-phase-a`) and the logic/rationale behind every design decision.
**Date:** 2026-05-25 (core §1–8) · **§9 addendum:** 2026-05-27 · **Workshop:** AutoCon5, 2026-06-09.
**Method:** §1–8 synthesized from direct code mapping + two `code-explorer` agents (subsystem internals + data-flow/infra), every load-bearing claim verified against source — **reality, not intent**. §9 documents code on a feature branch with its true status stated explicitly (flags default-OFF, not on `main`, not deployed, **shape-proven but not yet live-proven**).

---

## 1. Executive summary

A 5-layer human-in-the-loop (HITL) agentic system that turns a router **syslog** into a **human-approved, executed fix**, with reasoning recorded for audit. The pipeline is: **deterministic triage → specialist diagnosis (LLM + tools + digital-twin validation) → human approval → guarded execution → audit**.

- **What's strong:** the *control plane* is deterministic by construction (regex triage + answer cache → same incident yields the same routing and the same exact answer, proven 21/21 bit-exact), nothing touches a device without human approval, and the device-tool surface is broad (37 protocol parsers, 42 diagnostic trees).
- **What's weak:** **no infrastructure-as-code** (everything hand-managed), the running box isn't a tracked git checkout, the fine-tuned specialist models underperform gpt-4o-mini (so the demo runs on fallback), and the single-consumer orchestrator caps throughput (~25 incidents / 15 min).
- **What's evolving (the moat):** a **vendor-agnostic** collection seam (Phase A) wraps the Cisco path behind a neutral adapter interface with **zero behavior change** (flag-gated, default OFF, byte-for-byte parity-proven); an **Arista EOS adapter** (Phase B) collects OSPF/BGP/interfaces via eAPI into the *same* neutral shape; and a **DomainPack** layer (Phase D) makes the system topology-aware — a first **EVPN** pack adds cross-vendor EVPN diagnosis (collection, capability-based dispatch, and syslog triage) on both vendors. All on branch `vendor-abstraction-phase-a` — **not merged to `main` or deployed**, **shape/logic-proven offline but not yet validated against live hardware**. Full design + the rationale for every decision in **§9**.

---

## 2. System-level architecture

```
                                   ┌─────────────────────────────────────────────┐
  Cisco devices ──syslog/udp:5140─▶│ L0 syslog-server  (parse → incident_queue)  │
  (mock | real CSR via EVE-NG)     └───────────────────────┬─────────────────────┘
        ▲                                                  │ Redis LPUSH
        │ SSH (real mode, via EVE-NG jump host)            ▼
        │                          ┌──────────────────────────────────────────────┐
        │                          │ L2 TEAM-LEADER  (netops-team-leader :8002)    │
        │                          │  deterministic triage (regex, no LLM)         │
        │   ┌──ACP REST :8002──────│  per-device 3s buffer + 60s dedup + cascade   │
        │   │                      │  A2A delegate ▼            ▲ A2A response      │
        │   │                      └──────────┬─────────────────┴──────────────────┘
        │   │                                 │ Redis a2a_<spec>_inbox
        │   │   ┌─────────────────────────────▼──────────────────────────────────┐
        │   │   │ L3 SPECIALISTS (A2A): stability · troubleshooting · security ·  │
        │   │   │                       virtual-architect                          │
        │   │   │  per agent: Context → Analyze(LLM+cache) → Validate(gate+DT)     │
        │   │   │   ├── KG read (Bolt 7687) ── Neo4j Knowledge Graph               │
        │   │   │   ├── Digital Twin (in-proc): graph_sim + containerlab + ios2frr │
        │   │   │   ├── Scenarios/diagnostic-tree engine + MCP evidence gatherer   │
        │   │   │   ├── CVE scanner (security)                                     │
        │   │   │   └── verifiability (HTTP → verifiability-api → TimescaleDB)     │
        │   │   └───────────────────────────┬──────────────────────────────────────┘
        │   │                               │ MCP / HTTP :8001→8000
        │   │   ┌───────────────────────────▼──────────────────────────────────┐
        │   │   │ L4 MCP SERVER (netops-mcp)  37 protocol parser tools +         │
        │   │   │   config_writer + device_intent + idempotency cache           │
        │   │   └───────────────────────────┬──────────────────────────────────┘
        │   │                               │ SSH/NETCONF (real) | mock_device (mock)
        │   │                               ▼     L5 DEVICES
        │   ▼
   ┌────────────────┐  approval_queue   ┌──────────────┐  approval_responses  ┌──────────────────┐
   │ L1 UI :8501    │◀──────────────────│  team-leader │─────────────────────▶│ change-executor  │
   │ Streamlit      │  (human approve)  │  ACP server  │   (approved fixes)   │ guarded apply →  │
   │ approval cards │──────ACP─────────▶│              │                      │ MCP config_writer│
   └────────────────┘                   └──────────────┘                      └──────────────────┘

  SIDE PLANES:  Redis (bus/queues/cache/locks) · Neo4j+neo4j-sync (KG) · drift-detector
                verifiability-api + TimescaleDB (audit, OUT-OF-COMPOSE) · NetBox (intent SSOT, external)
```

**Protocol map (fixed):** UI↔Team-Leader = **ACP** (REST/HTTP :8002); Team-Leader↔Specialists = **A2A** (Redis inbox queues); Specialists↔MCP = **MCP** (HTTP :8001/8000); MCP↔Devices = **SSH/NETCONF** (real) or in-process mock.

---

## 3. Subsystem catalog (every component, with status)

| # | Subsystem | Key location | Purpose | Status |
|---|-----------|--------------|---------|--------|
| 1 | **Syslog ingest** | `syslog-server/syslog_server.py` (:5140/udp) | Parse device syslog (RFC3164/5424 + EVE-NG relay), detect incident type, `lpush incident_queue` | LIVE |
| 2 | **Team-Leader / orchestrator** | `agents/team_leader/agent_adk.py` + `http_server.py` (:8002) | Deterministic regex triage (`triage_rules.classify_syslog`), per-device 3s buffer + 60s dedup + cascade/victim suppression, A2A delegation, approval routing (ACP) | LIVE |
| 3 | **Stability specialist** | `agents/stability/` (Qwen2.5-7B / fallback) | OSPF/BGP/EIGRP/memory/CPU faults | LIVE (on fallback) |
| 4 | **Troubleshooting specialist** | `agents/troubleshooting/` (CodeLlama-7B / fallback) | Interface/connectivity/BGP-flap/L2/protocol faults | LIVE (on fallback) |
| 5 | **Security specialist** | `agents/security/` (Mistral-7B / fallback) | Auth/ACL/SSH/privilege/port-security + CVE | LIVE (on fallback) |
| 6 | **Virtual architect** | `agents/virtual_architect/` (Phi-4 / fallback) | Read-only ReAct design advisor (UI chat, not syslog path) | LIVE (on fallback) |
| 7 | **Specialist workflow** | `agents/<x>/workflow/{context,analyze,validate}_node.py` | Context (MCP evidence) → Analyze (LLM + answer cache) → Validate (confidence gate + Digital Twin) | LIVE |
| 8 | **MCP server** | `mcp-server/main.py` (:8001→8000) | 37 protocol parser tools + `config_writer` (mutating) + `device_intent` (NetBox) + idempotency cache; mock↔real device dispatch | LIVE |
| 9 | **Device parsers** | `mcp-server/tools/*_parser.py` (37) | Per-protocol `show`-output → structured data (ospf, bgp, isis, evpn, sr, sdwan, macsec, qos, cve, …) | LIVE |
| 10 | **Devices** | `mock_device.py` \| `real_device.py`+`ssh_tunnel.py` | Simulated CSR state machine \| 3× real CSR1000v in EVE-NG via SSH jump host | LIVE (mock default) |
| 11 | **Knowledge Graph** | Neo4j `:7474/7687` + `neo4j-kg/sync.py` + `kg_client.py` | Derived topology graph (Device/Interface/OSPFNeighbor/Route); sole writer = neo4j-sync (30s poll); read by specialists + DT | LIVE |
| 12 | **Digital Twin** | `agents/shared/{graph_simulator,containerlab_manager,ios_to_frr,validate_node}.py` | Pre-approval "what-if": L1 networkx reachability + L2 FRR-container replay; advisory gate | LIVE (L1; L2 self-skips off-box) |
| 13 | **Scenarios / diagnostic-tree engine** | `agents/scenarios/` (42 trees + ~70 scenarios, `loader/catalogue.py`, `schema/`) | Machine-readable failure-pattern KB; drives evidence-gathering 3-stage MCP tool walks + fix templating | LIVE |
| 14 | **MCP evidence gatherer** | `agents/shared/mcp_evidence_gatherer.py` | Resolves the matched scenario's diagnostic tree → deterministic stage-1/2/3 MCP tool calls → structured evidence for the LLM | LIVE |
| 15 | **CVE scanning** | `mcp-server/tools/cve_scanner.py` + `agents/shared/cve_scanner_client.py` | Hybrid-search CVE lookup (vexpertai scanner API, mock fallback); feeds `state["cve_scan"]` into the security prompt | LIVE (security path) |
| 16 | **Change-executor** | `change-executor/executor.py` | HITL execution bridge: consumes `approval_responses` → guarded apply via MCP `config_writer`; idempotency + anti-storm + maintenance-window gates; writes `applied_history` | LIVE |
| 17 | **Determinism / answer cache (§9.1)** | `agents/shared/llm_client.py` | `DETERMINISM_STRICT=1` → temp0 + seed + Redis `answer:{incident_sig}`; same INC → same exact answer | LIVE (enabled on box) |
| 18 | **Verifiability / observability** | `agents/shared/verifiable_client.py` → `verifiability-api` → TimescaleDB | Records reasoning chains (observation→data→hypothesis→analysis→decision) per incident; best-effort/non-blocking | LIVE (out-of-compose) |
| 19 | **Drift detector** | `drift-detector/` | NetBox intent vs Neo4j actual → `(:Drift)` nodes (60s) | LIVE |
| 20 | **LLM client** | `agents/shared/llm_client.py` | Multi-provider (HF dedicated → local vLLM → OpenAI fallback), per-agent endpoints/models, the determinism wrapper | LIVE |
| 21 | **Message bus / stores** | Redis `:6379` | A2A inboxes, approval/exec queues, dedup/storm locks, idempotency + answer caches | LIVE |

---

## 4. End-to-end data flow (incident lifecycle)

```
 device syslog
    │ UDP :5140
    ▼
 syslog-server ──lpush──▶ incident_queue
                              │ brpop
                              ▼
 team-leader: noise/recovery filter ─▶ syslog_buffer:{dev} (3s) ─▶ _process_device_buffer
     │  pick min-priority · dedup incident_active:{dev} (60s) · cascade victim-suppress (→ dead_letter_queue)
     │  classify_syslog() [deterministic regex]   (LLM fallback only if no rule matches)
     ▼ A2A TASK_DELEGATION (lpush)
 a2a_<specialist>_specialist_inbox
     │ brpop
     ▼
 SPECIALIST workflow:
   ContextNode ── MCP HTTP ──▶ mcp-server tools (scenario diagnostic-tree walk; +CVE for security)
   AnalyzeNode ── answer cache GET answer:{sig} ─ hit→return ─ miss→LLM(temp0,seed)→SET answer:{sig}
   ValidateNode ── confidence gate → DigitalTwin (graph_sim L1 / containerlab L2) → blast radius
   (reasoning steps streamed → verifiability-api → TimescaleDB)
     │ A2A TASK_RESPONSE (lpush)
     ▼
 a2a_team_leader_inbox ──▶ team-leader assembles approval card ──lpush──▶ approval_queue
     │                                                                      │ read
     ▼                                                                      ▼
 (HITL) UI :8501 renders card ── operator Approve/Reject ──ACP POST :8002──▶ team-leader
     │ lpush approval_responses                                            (removes from approval_queue)
     ▼ brpop
 change-executor: preflight gates (idempotency applied:{id} 24h · anti-storm storm:{dev} · maint-window)
     │ MCP HTTP → config_writer (dry_run | real)
     ▼ SSH (real mode) → device
 applied_history (Redis, cap 500) ──▶ UI "applied" badge
```

**Authoritative Redis key map**

| Key | Type | Producer → Consumer |
|---|---|---|
| `incident_queue` | list | syslog-server → team-leader |
| `syslog_buffer:{dev}` / `incident_active:{dev}` | list / str(60s) | team-leader (buffer + dedup lock) |
| `dead_letter_queue` | list | team-leader (suppressed victims) |
| `a2a_{stability,troubleshooting,security,virtual_architect}_specialist_inbox` | list | team-leader → specialist |
| `a2a_team_leader_inbox` | list | specialist → team-leader |
| `approval_queue` | list | team-leader → UI |
| `approval_responses` | list | team-leader ACP → change-executor |
| `applied:{change_id}` / `storm:{dev}` | str(24h) / str(30s) | change-executor gates |
| `applied_history` | list (cap 500) | change-executor → UI |
| `answer:{incident_sig}` | str | llm_client answer cache (determinism) |

---

## 5. Subsystem-level diagrams

**Digital Twin (in-process inside each specialist's ValidateNode — 4-pass gate)**
```
proposed fix ─▶ [1] ConfidenceScorer ──gate band──▶ <0.55 human · 0.55–0.85 L1 · ≥0.85 L2
             ─▶ [2] fault-correctness regex (does the fix match the fault?)
             ─▶ [3] graph_simulator (networkx): build graph from KG OSPF neighbors →
                    apply change (shutdown/no-router removes node) → nx.is_connected? + risk score
             ─▶ [4] containerlab_manager: ios_to_frr.translate() → deploy 3 FRR containers →
                    apply via vtysh → 12s converge → check neighbors/routes/ping → teardown
   NOTE: advisory only (UI does not block Approve on it); L2 self-skips when containerlab absent.
```

**Knowledge Graph (derived, eventually-consistent)**
```
MCP device state ──poll 30s──▶ neo4j-sync (ONLY writer) ──MERGE──▶ Neo4j
   nodes: Device · Interface · OSPFNeighbor · Route     rels: HAS_INTERFACE · OSPF_NEIGHBOR · HAS_ROUTE
   readers (Bolt 7687, kg_client.py): specialists (get_device_context) · graph_simulator
                                      (get_all_ospf_neighbors, get_blast_radius) · drift-detector
   mock mode: seeded 3-router triangle once, then static (heartbeat only).
```

**Scenarios / diagnostic-tree engine**
```
42 _diagnostic_trees/*.yaml + ~70 scenario YAMLs ──validated by──▶ schema/scenario_schema.py (Pydantic)
   loader/catalogue.py (singleton): syslog regex table + alias index + tree index
   matched pattern_id ─▶ mcp_evidence_gatherer ─▶ tree: stage1 baseline tools → stage2 branch rules
        (sandboxed `when` exprs via simpleeval) → stage3 NetBox intent cross-check → structured evidence
```

**Verifiability / audit plane (best-effort, non-blocking)**
```
each specialist ──HTTP──▶ verifiability-api :8000 ──▶ TimescaleDB (ccie_verifiability)
   /reasoning/start → /step (observation,data_collection,hypothesis,analysis,decision) → /finalize
   tables: agent_reasoning_chains · reasoning_steps · data_provenance · audit_events
   failure = silent no-op (never fails the primary path). UI :8505 shows transcripts.
```

---

## 6. Infrastructure & the SSOT question

### Is there an SSOT for infra? — **No. There is no infrastructure-as-code.**

A full search found **zero** Terraform/`*.tf`, CloudFormation, CDK, Ansible, Pulumi, or Serverless files. The only declarative artifact is `docker-compose.yml` — and that covers the **application layer only** (no EC2, networking, IAM, or the GPU fleet), and it **drifts** (TimescaleDB, verifiability-api, NetBox are referenced but not defined in it). The "infra" is therefore **hand-managed**.

**Compute inventory (manually provisioned):**

| Host | Role | Managed via |
|---|---|---|
| `AI_Dream_Team` (i-0117…ce60, eu-central-1, <APP_HOST_IP>) | the whole Docker stack | **AWS SSM only** (SSH keys rotated/stale); tarball/rsync deploy, **not a git checkout** |
| 4× GPU instances (g5.xlarge: GPU-LLM-Server / GPU-Stability / GPU-Specialists / GPU-Troubleshooting) | self-hosted fine-tune vLLM endpoints (10.42.20.x) | manual; nightly EventBridge stop/start; serve `/model` (name-mismatch → 404→fallback) |
| `eveng-prod` (<JUMP_HOST_IP>) | EVE-NG with 3× CSR1000v + the DT runner (:8007) | manual; SSH jump host |

**Out-of-compose, manually-run:** TimescaleDB (`netops-timescale`), `verifiability-api` (:8000), the `:8505` transcripts UI, NetBox. A clean `docker compose up` does **not** reproduce these.

**Management reality:** EC2s created by hand (console/CLI); day-to-day ops by `aws ssm send-command`; app deploys via `deploy.sh` (assumes the host exists); a `netops-boot-reconcile` systemd unit + nightly EventBridge schedule keep the stack alive across the 22:00/08:00 cost cycle. **No state file, no drift protection, no reproducible rebuild.**

### Code SSOT (separate from infra) — fragile, now reconciled
- **Git** `eduardd76/dream_team_original@main` is the code SSOT (this session merged the box-superset + team-leader P3 + answer cache into it).
- The **running box is not a git checkout** (tarball deploy) — it was bidirectionally diverged from git and was reconciled by hand this session. Future code changes still need an SSM file-push, not `git pull`.

---

## 7. Assessment

**Strengths**
- **Deterministic control plane.** Triage is pure ordered-regex (bit-exact); with the §9.1 answer cache, the same incident yields the same exact answer — **proven 21/21 bit-identical** across routing, root-cause, fix, and confidence.
- **Safety-first.** Nothing reaches a device without human approval; `change-executor` adds idempotency, anti-storm, and maintenance-window gates; destructive commands are hard-blocked pre-human.
- **Breadth.** 37 protocol parsers, 42 diagnostic trees, CVE enrichment, a 2-layer digital twin, a knowledge graph, and a full audit plane — a genuinely rich NetOps surface.
- **Graceful degradation.** Every heavy dependency (vLLM, containerlab, Timescale, KG) fails soft.

**Risks / gaps (ranked)**
1. **No infra SSOT** — the environment can't be reproducibly rebuilt; recovery is manual. *Highest long-term risk.*
2. **Box ≠ git** — the demo host is a tarball deploy, drift-prone; reconciliation is manual.
3. **Fine-tuned models underperform gpt-4o-mini** (security worst: composite 0.49, hallucination 0.28) → demo runs on **fallback**; retrain in progress.
4. **Single-consumer throughput ceiling** — team-leader blocks per-incident; ~25 incidents / 15 min regardless of cache. Bites at concurrency, not device count.
5. **Digital Twin is advisory + degrades silently** — looks like a safety gate but the UI doesn't enforce it; L1 passes vacuously without KG topology, L2 self-skips off-box.
6. **Graceful degradation hides missing capability** — green doesn't prove the twin/audit actually ran.

**Demo-readiness (2026-06-09):** the deterministic pipeline + HITL + audit are solid and the demo runs reliably on gpt-4o-mini fallback. The honest talking points are the deterministic routing/answers (now provable) and the breadth; the things to *not* over-claim are the fine-tune superiority and the DT enforcement.

---

## 8. Recommendations

- **P0 (long-term):** introduce an infra SSOT — even a minimal Terraform/Ansible that captures the EC2s, the GPU fleet, security groups, EventBridge, and the out-of-compose containers — so the environment is reproducible.
- **P1:** make the box a tracked git checkout (preserving `.env`/keys/volumes) so deploys are `git pull`; fold TimescaleDB + verifiability-api into compose.
- **P1:** fix the vLLM `--served-model-name` (or point specialists straight at gpt-4o-mini) to drop the 404→fallback hop.
- **P2:** decouple team-leader incident consumption from processing (async tasks) to lift the throughput ceiling.
- **P2:** decide whether the Digital Twin should *enforce* (gate Approve on `dt_passed`) or stay advisory — and pre-seed the KG so L1 actually simulates.
- **Ongoing:** retrain the specialists (separate session); re-run `determinism_baseline.py` against the committed gpt-4o-mini reference for a like-for-like comparison.
- **Vendor-agnostic merge gate (P1):** keep `vendor-abstraction-phase-a` off `main` until the Arista live-validation checklist (`VENDOR_LAB_BRINGUP_RUNBOOK.md` §4) passes against a real device; merge is then a clean fast-forward. Rationale in §9.3 (D10).

---

## 9. Vendor-agnostic architecture (the moat) — design, decisions & rationale

*Added 2026-05-27 (refreshed same day to fold in Phase D). Documents work on branch `vendor-abstraction-phase-a` (commits `1c45d29`→`41e71b3`). Two axes: **Phase A/B = the vendor (adapter) side** — how you talk to one box; **Phase D = the topology (DomainPack) side** — how a fault domain is diagnosed across vendors.*

> **Status discipline (read first).** Everything in §9 is on a **feature branch**, every adapter flag defaults **OFF**, nothing here is on `main` or deployed to the box, and the Arista path (+ the EVPN eAPI shapes / mnemonics from Phase D) is **shape/logic-proven, not live-proven** (§9.6). The §1–8 assessment of `main` — and the demo that runs from it — is **unchanged** by this section. This is staged capability with the demo's downside fully protected.

### 9.1 Why vendor-agnostic at all — the strategic rationale

The system in §1–8 is **Cisco-only by construction**: every parser speaks IOS-XE CLI, every triage rule keys on a Cisco mnemonic, every connection is scrapli→IOS-XE. That is a *product* ceiling, not just a technical one — real NetOps customers run mixed fleets. The ability to drive **Cisco *and* Arista** (and later others) from a single reasoning core is the durable differentiator: the "moat."

**Guiding principle — thin vendor edges, fat neutral core.** Vendor-specificity should live in exactly three places: (1) syslog mnemonics, (2) CLI/RPC parsers, (3) fix-command strings. Everything above them — triage, specialists, digital twin, knowledge graph, audit, determinism — stays vendor-blind. Onboarding a vendor should be an *additive plugin*, never a fork.

### 9.2 Target shape (recap)

A **neutral core + four plugin types**:

| Plugin type | Responsibility | Phase |
|---|---|---|
| **VendorAdapter** | `connect` / `collect(capability)→NeutralState` / `render(NeutralChange)→cmds` / `syslog_to_event` / `capabilities()` | A (seam) + B (Arista) |
| **Neutral Model** | `NeutralState` / `NeutralEvent` / `NeutralChange` + capability taxonomy | A |
| **DomainPack** | fault taxonomy + neutral evidence plan + fix templates (wraps the existing 42 trees / 72 scenarios) | **D (D0–D4: interface + EVPN pack)** |
| **InventorySource** | NetBox-backed devices/topology/intent **per tenant** | C (deferred) |

Phase A builds the VendorAdapter seam + the thin Neutral Model and wraps the existing Cisco path behind it; Phase B adds the Arista adapter. **Phase D builds the DomainPack plugin — the *topology* side: the catch-all default pack (proven identical to the catalogue) plus the first specialised pack (EVPN), selected by device capability.** InventorySource + multi-tenancy (Phase C) remain deliberately later (and carry the determinism-baseline re-record risk, so they're sequenced last).

### 9.3 Decision records — the logic behind each choice

Each decision is stated with its **rationale (why)** and the **trade-off (what we accepted)**. This is the core of "explain the logic for all these decisions."

| # | Decision | Rationale (why) | Trade-off accepted |
|---|----------|-----------------|--------------------|
| **D1** | **Normalize at the MCP (collection) layer** — not in the specialists, not "at the LLM". | MCP is the single choke point where *all* device data enters the reasoning plane (specialists pull evidence via MCP tools). Normalize here once and everything upstream is vendor-blind for free. Normalizing in 4 specialists would duplicate vendor logic; letting the LLM eat raw vendor output would destroy determinism and bloat prompts. | The MCP layer owns the neutral-shape contract and the per-vendor parsing complexity concentrates here. |
| **D2** | **Capability-based dispatch** — the core never branches on a vendor string. | `if vendor == "cisco"` scattered through code is exactly how "vendor-agnostic" systems rot back into vendor-coupled ones. A device advertises dotted capabilities (`routing.ospf`, `overlay.vxlan-evpn`); the core asks "can it do X?", never "is it Cisco?". Adding a vendor never touches dispatch. | Needs a capability taxonomy and accurate per-adapter declarations; one layer of indirection. |
| **D3** | **Thin `NeutralState{raw: dict}` in Phase A** — defer full typed per-protocol neutral schemas. | The 37 parsers already emit well-shaped dicts. Wrapping that dict *unchanged* lets Phase A wrap every parser with provably zero behavior change — fast — and defers the large typed-schema work until a second vendor reveals what's *truly* common. YAGNI: don't design the rich schema before the data tells you its shape. | `raw` is loosely typed in Phase A; the richer typed neutral model is scheduled debt, not an oversight. |
| **D4** | **Strangler reroute behind default-OFF flags + shadow mode.** | The system is demo-critical ~2 weeks out; a big-bang refactor risks the demo. Reroute one parser at a time behind `MCP_USE_ADAPTER` (OFF ⇒ byte-identical legacy path) with `MCP_ADAPTER_SHADOW` (run both, log mismatches, **return legacy**), so parity is validated on live traffic at zero risk before the new path is trusted. | Temporary dual-path code + flag bookkeeping + a shared reroute helper. |
| **D5** | **Golden byte-for-byte parity** is the acceptance test. | "Looks the same" is insufficient when determinism depends on *exact bytes* (§9.4). The bar is `adapter.collect().raw == legacy_parser_output` exactly — turning "trust me" into a mechanical proof. | Needs a pydantic-capable test venv (homebrew py3.11 lacks pydantic, so parity tests would *silently skip*); solved with a dedicated parity venv. A "passed" count is meaningless until you check what skipped. |
| **D6** | **Arista via eAPI (structured JSON)** — not CLI screen-scraping. | Arista exposes eAPI returning native JSON; parsing JSON is robust and deterministic, whereas regex-scraping `show` text is the exact brittleness we're trying to escape. JSON-native collection also future-proofs toward gNMI/OpenConfig. | A `pyeapi` dependency (imported **lazily** so `import adapters` stays offline-safe); requires eAPI enabled on the device. |
| **D7** | **Arista is diagnose/collect-only** for the workshop (`render`/`apply` raise `NotImplementedError`). | The demo decision is "diagnose + recommend." Config-push to a second vendor is higher-risk and outside the demo's value story. Build the read path well; defer the write path until there's a reason. | No automated remediation on Arista yet — HITL "recommend" covers the demo. |
| **D8** | **Two synchronous env maps:** dispatch (`MCP_DEVICE_PLATFORMS`) + addressing (`ARISTA_EAPI_HOSTS`). | The collection path runs **synchronously inside the server's event loop**; the async NetBox resolver can't run there (loop-bound locks ⇒ it silently fell back to `cisco_iosxe` for *every* device — multi-vendor never engaged). A static, I/O-free env map is the only loop-safe resolution *and* makes the demo deterministic (no live-NetBox in the hot path). Splitting "which adapter" from "which host" keeps each map tiny; adding a device = two JSON entries, zero code. | Env-driven rather than NetBox-driven config (NetBox stays the eventual SSOT, wired in a later async-aware phase). |
| **D9** | **Value-based collector rollout** — reroute the ~10 actively-used collectors; defer the ~27 long-tail parsers. | Not all 37 parsers are on the hot reasoning path. Rerouting the 10 the specialists actually call delivers the multi-vendor capability that matters now; tail parsers get rerouted per-protocol when that protocol actually goes multi-vendor. Effort follows value. | The long tail stays Cisco-only until needed — a deliberate, documented backlog. |
| **D10** | **Hold the merge to `main`** until live validation passes. | Shape-proven ≠ live-proven. `main` is the demo/deploy source of truth *and* the safe mock fallback. Merging is a statement of confidence; doing it before Arista shapes are confirmed against a real device would put unproven code in the demo lineage for no benefit (flags are OFF anyway). | The branch lives longer; the eventual merge is a clean fast-forward (main hasn't diverged). |

**Phase D (topology side) decisions:**

| # | Decision | Rationale (why) | Trade-off accepted |
|---|----------|-----------------|--------------------|
| **D11** | **Topology is data, not code** — packs are selected by capability/domain, never by branching on a topology name. | Same anti-pattern guard as D2: `if topology == "spine-leaf"` is how a topology-agnostic system rots. A device advertises `overlay.vxlan-evpn`; the resolver picks the EVPN pack. Topology emerges from *which devices have which capabilities*, not a hardcoded shape — so a new fabric is a pack + a capability, not a fork. | Needs a capability→pack resolver and per-pack claim declarations. |
| **D12** | **DomainPack wraps the catalogue; `DefaultDomainPack` is the catch-all, proven identical** — specialised packs are added behind capability selection. | Exact mirror of the A1→A2 adapter strangler: wrap today's behavior behind an interface, prove byte-identity (**diagnosis parity** — the pack delegates to the catalogue across every accessor), *then* add the specialised pack. No consumer is rewired in the foundation unit, so it's zero behavior change. | A temporary delegation layer; consumer reroute onto the pack seam is a later unit. |
| **D13** | **EVPN forces the first typed neutral schema** (`EVPNParserResponse`), the debt deferred in D3. | A second *domain* (not just a second vendor) is the right forcing function for a typed neutral shape. Grounding it in the EXISTING `evpn_parser` output means today's Cisco parser validates against it unchanged — the shape-proven link the cross-vendor EVPN collectors build on. | One typed schema now exists alongside the thin `raw` dicts; the rest stay thin until similarly justified. |
| **D14** | **The agents-layer pack returns collector SLUG STRINGS, never imports the MCP `Collector` enum.** | The DomainPack lives in the reasoning (agents) layer; the `Collector` enum lives in the MCP layer, reached over HTTP. A cross-layer import would couple them — the exact coupling the HTTP boundary prevents. The neutral evidence plan is expressed in the string slugs both sides already agree on. | The string↔enum agreement is a convention; a *test-only* `Collector` import guards against drift. |
| **D15** | **EVPN syslog triage lives at the adapter vendor-edge (`syslog_to_event`), not the team-leader.** | Vendor mnemonic→neutral-fault mapping is one of the three thin vendor edges (D-principle). Putting it in the adapter keeps it additive: only EVPN lines change, OSPF/plain stay `unclassified`, and the legacy 42-rule live triage is untouched (no demo risk). | The neutral event path isn't yet wired into live triage — a separate later unit. |

### 9.4 Why this preserves the determinism guarantee

The §7 strength — *same incident → same exact answer, proven 21/21 bit-exact* — depends entirely on the **bytes fed to the LLM**, and the MCP collection output *is* those bytes (via the evidence gatherer). Therefore the adapter seam is determinism-safe **iff** the adapter reproduces the legacy parser output byte-for-byte. That is precisely what golden parity (D5) and shadow mode (D4) prove.

- **Cisco adapter:** byte-identical to the legacy parser (proven) ⇒ flipping `MCP_USE_ADAPTER` **cannot change Cisco reasoning**.
- **Arista adapter:** normalizes *into the same shape* ⇒ once its eAPI shapes are live-validated (§9.6), Arista reasoning is deterministic on the same footing.
- **Caveat:** if a future cache-key gains a tenant/vendor dimension, the gpt-4o-mini determinism baseline must be **re-recorded** (a known follow-up, not a silent risk).

### 9.5 Build status (branch `vendor-abstraction-phase-a`)

| Unit | Commit | What it delivered | Gate |
|---|---|---|---|
| A0 | `fbdffd2` | Foundation: neutral model + `VendorAdapter` ABC + registry + factory | QA-fixed an asyncio loop/Lock hazard |
| A1 | `95598a8` | `CiscoIOSXEAdapter` + `MockAdapter` (wrap existing device + parsers) | golden parity byte-for-byte |
| A2 | `741ab63` | Flag-gated reroute of ospf+interface (default OFF; shadow mode) | route byte-identity 10/10 |
| A3.0 | `c5644eb` | Shared reroute helper; reroute bgp + routing_table | parity/shadow 69 passed |
| A3.1 | `3da3da1` / `d38c7cf` / `ca37b41` | Rerouted the 10 active collectors (cdp, stp, aaa, login_policy, security, bgp_summary) | parity 100 passed |
| **B-1** | `1c45d29` | **AristaEOSAdapter** via eAPI — OSPF/BGP/interfaces → the Cisco-shaped neutral dict (collect-only) | 33 shape-parity tests |
| **B-2** | `4b948b9` | Loop-safe **platform map** (`MCP_DEVICE_PLATFORMS`) — fixes the silent-cisco fallback in-loop | offline 320 / parity 141 |
| **B-3** | `1df602a` | Per-device **eAPI host map** (`ARISTA_EAPI_HOSTS`) — multi-device lab addressable | offline 329 / parity 150 |
| docs | `fc0cf4a` | `VENDOR_LAB_BRINGUP_RUNBOOK.md` — turnkey live bring-up + validation | — |
| **D-0** | `cab4e4e` | `DomainPack` interface + `DefaultDomainPack` (wraps the catalogue) | **diagnosis parity** (17 tests) |
| **D-1** | `a452887` | `EVPNParserResponse` — the first typed neutral schema | 8 tests (Cisco shape-proven + Arista-shaped validates) |
| **D-2** | `7260131` | `Collector.EVPN` on Cisco + Mock + Arista → the neutral EVPN shape | parity venv 185 |
| **D-3** | `2e1d692` | `EVPNDomainPack` + capability-based resolver selection | parity venv 197 |
| **D-4** | `41e71b3` | EVPN syslog triage in `syslog_to_event` (both vendors → neutral fault) | parity 214 / py3.11 345 |

Net: **both axes are code-complete and proven offline.** Vendor side: dispatch + addressing + normalization (a lab device = two JSON env entries). Topology side: the EVPN fault domain is diagnosable end-to-end — *capability → pack selection → neutral evidence plan → neutral collectors → both adapters → neutral shape*, plus *EVPN syslog → neutral fault_class*.

### 9.6 What's NOT done — the live-validation gate

**Shape-proven ≠ live-proven.** The eAPI JSON *fixtures* encode best-knowledge of EOS output, not *observed* output. The shape-parity tests prove the normalizers are internally consistent and produce the right neutral shape; they do **not** prove the eAPI paths/casing/types are real. Until validated against a live Arista device, the moat is "proven in principle."

The gate is **`VENDOR_LAB_BRINGUP_RUNBOOK.md` §4**: per-collector, run the raw eAPI command on a real device and confirm each mapping (`peerState` casing, the OSPF `instList` path, `interfaceAddress` list-vs-dict, the real interface-drops key, `asn`/`upDownTime` types). Each divergence ⇒ fix the normalizer **and** correct the fixture to the observed shape, then re-run the gate. The merge to `main` (D10) is held until this passes.

**Phase D adds two more best-knowledge items to that same gate** (the most uncertain yet, since they're a second domain *and* a second vendor):
- **EVPN eAPI shapes (D-2):** `show bgp evpn summary` peer shape + `show vxlan vni` (the `_iter_eos_vxlan_vnis` variants); per-type route counts + MAC-inconsistency are stubbed at 0 pending the `show bgp evpn` route-table walk.
- **EVPN syslog mnemonics (D-4):** the `_CISCO_FAULT_TABLE` / `_EOS_FAULT_TABLE` patterns are plausible, not device-observed — confirm against real EVPN incident logs and adjust the regexes.

### 9.7 How this updates the §7 risk picture

- **Reduces:** vendor lock-in / the single-vendor product ceiling (Phase A/B) **and** the topology-coupling ceiling (Phase D) — a new vendor is an adapter, a new fault domain/topology is a DomainPack, both additive.
- **Adds (managed):** the Arista eAPI shapes (B-1/D-2) and the EVPN mnemonics (D-4) are unverified against hardware — mitigated by flags-OFF, the §9.6 validation gate, and the safe mock fallback on `main`.
- **Net:** the demo's *downside* is protected (`main` untouched, flags OFF, fast-forward merge later) while the *upside* — credible dual-vendor Cisco+Arista with an EVPN-aware diagnosis path — is staged and ready to validate the moment the lab is reachable. The remaining Phase D unit (D-5, twin overlay reachability) is an optional stretch; Phase C (InventorySource/tenancy) is the larger follow-on and is sequenced last because its cache-key change forces a determinism re-baseline.
