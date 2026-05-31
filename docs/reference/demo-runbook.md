# Workshop Demo Runbook — Agentic NetOps (WS 2026-06-09)

Click-by-click script for rehearsal + the live demo. Default posture: **mock devices**
(deterministic, no lab dependency). Real-device lab is the optional stretch (§7).

**Endpoints** (AI_Dream_Team, `<APP_HOST_IP>`):
- Main UI (chat + approvals): `http://<APP_HOST_IP>:8501`
- Transcripts & HITL tab (verifiability): `http://<APP_HOST_IP>:8505`
- MCP health: `http://<APP_HOST_IP>:8001/health` · Team-Leader ACP: `:8002/acp/health`

---

## 1. Morning pre-flight (do this ~30 min before)
Instances auto-start 08:00 Europe/Berlin; the boot-reconcile brings the stack up + heals Redis-DNS.
Run the preflight (via SSM on `<INSTANCE_ID>`):

```bash
# stack health
curl -s -o /dev/null -w "MCP %{http_code}\n" http://localhost:8001/health
curl -s -o /dev/null -w "UI  %{http_code}\n" http://localhost:8501
sudo docker ps --format '{{.Names}} {{.Status}}' | grep -E "mcp|team-leader|stability|security|troubleshoot|virtual|ui|redis|timescale"
# mock mode confirmed
sudo docker inspect netops-mcp --format '{{range .Config.Env}}{{println .}}{{end}}' | grep USE_REAL_DEVICES   # expect false
# Redis reachable from MCP (the recurring DNS gotcha — reconcile self-heals, verify anyway)
sudo docker exec netops-mcp python -c "import os,redis;print(redis.from_url(os.getenv('REDIS_URL','redis://redis:6379')).ping())"
```
Expect: MCP 200, UI 200, all containers `Up`, `USE_REAL_DEVICES=false`, redis ping `True`.

**Flush stale state so the demo starts clean:**
```bash
for q in approval_queue approved_history rejected_history dead_letter_queue; do
  sudo docker exec netops-redis redis-cli DEL $q; done
sudo docker exec netops-redis redis-cli --scan --pattern 'incident_active:*' | xargs -r sudo docker exec netops-redis redis-cli DEL
```

**Warm the models** (cold start is 30–60s — do one throwaway call so the live demo is snappy):
trigger one incident (below), let it complete, then flush again.

**Smoke (must be green):** the 4-incident smoke from `TEST_PLAN.md` §8 — one per specialist + VA-NO-CLI.

---

## 2. The core loop — incident → triage → specialist → human approval
1. **Trigger** an incident (or let a router syslog arrive). Manual trigger:
   ```bash
   curl -s -X POST http://localhost:8001/incident/trigger -H "X-MCP-API-Key: <MCP_API_KEY>"
   ```
   (returns `incident_type: ospf_neighbor_down` for router-1)
2. **Narrate the flow** (it completes in ~60–90s): the **Team Leader** does *deterministic* regex triage
   (`ospf_neighbor_loss_dead_timer → stability`), delegates over A2A; the **Stability specialist**
   analyzes (root cause + evidence + proposed fix + confidence).
3. **Show the approval card** in the UI (`:8501`): root_cause, evidence list, proposed_fix commands,
   risk level, confidence. Emphasize: **nothing is applied without human approval.**
4. **Approve** (or reject) in the UI → card moves to history; the change-executor would apply it.
   *Talking point:* destructive commands (`write erase`, `reload`) are **hard-blocked** before they ever
   reach a human (confidence-scorer gate).

### 2a. Multi-vendor EVPN-VXLAN variant (added 2026-05-30)
Same pipeline, different protocol family — shows the **3-way protocol-family dispatcher** (OSPF / BGP-unicast / EVPN-VXLAN) the stability specialist now runs.

1. **Trigger** an EVPN fault. Pick one (all route to `stability` with `tier="overlay"`):
   ```bash
   # NVE / VTEP unreachable (NX-OS-style)
   curl -s -X POST http://localhost:8001/incident/trigger \
     -H "X-MCP-API-Key: <MCP_API_KEY>" \
     -d '{"device_id":"leaf-nx","syslog":"%NVE-2-VTEP_LOSS: Lost reachability to remote VTEP 10.0.1.13"}'

   # BGP-EVPN session admin-shut
   curl -s -X POST http://localhost:8001/incident/trigger \
     -H "X-MCP-API-Key: <MCP_API_KEY>" \
     -d '{"device_id":"leaf-nx","syslog":"%BGP-5-ADJCHANGE: neighbor 10.0.0.1 Down L2VPN EVPN session admin shut"}'

   # MAC inconsistency (vendor-agnostic)
   curl -s -X POST http://localhost:8001/incident/trigger \
     -H "X-MCP-API-Key: <MCP_API_KEY>" \
     -d '{"device_id":"leaf-nx","syslog":"MAC_MOVE: host 0050.5600.0001 moved between VTEPs 10.0.1.12 and 10.0.1.13"}'
   ```
2. **Narrate the dispatch:** triage emits `agent=stability, tier=overlay, fault_type=evpn_*`; ContextNode's dispatcher routes to `_execute_overlay()` (calls `evpn_parser` + `bgp_summary`, NOT `ospf_parser`); AnalyzeNode renders the **multi-vendor EVPN prompt** with NX-OS / EOS / SR Linux fix syntax.
3. **Show the approval card:** root cause cites the actual peer address / EVI / VTEP IP from `evpn_parser`; `pattern_id` is one of `evpn_peer_admin_shut`, `evpn_peer_down`, `evpn_vtep_unreachable`, `evpn_vni_state`, `evpn_mac_inconsistency`.
4. **Optional architect follow-up:** in the Architect chat, ask "What's the EVPN state of leaf-nx?" — the va-v2 ReAct loop calls `evpn_parser` and explains the state. SYSTEM_PROMPT (updated `08f58c6`) now knows the fabric: spine1 / leaf-nx / leaf-ar / NX9K / vEOS.

*Talking point:* same pipeline, **different protocol world** — the stability specialist used to be OSPF-with-EVPN-bolted-on; after the EVPN-native push it's a routing-control-plane specialist that dispatches on protocol family. See `EVPN_NATIVE_ARCHITECTURE.md` for the architecture.

### 2b. Enterprise validation stack on the approval card (added 2026-05-30)
The card now carries the **enterprise validation chain** — what enterprise architects look for to fund pilots. Every approve/reject decision lands with:

1. **⚖️ Change Risk Officer (CRO)** — `ALLOW / MANUAL_REVIEW / BLOCK` verdict
   with **cited policies** (e.g. `business_hours_only`, `pci_dss_freeze`).
   Auditable: the operator's compliance officer can defend this to an
   external auditor. Policies are customer-replaceable via
   `agents/change_risk_officer/policies.json`.
2. **💥 Blast Radius Forecaster (BRF)** — business-impact translation.
   "VNI 10010 hosts tenant1-payment-processing; $340K/hour SLA exposure;
   4,200 users at risk; PCI-DSS + SOX compliance tags." Speaks CFO,
   not "8 devices."
3. **🔬 DT Validation** — Layer 1 (graph_sim) or Layer 2 (containerlab live).
   Layer 3 (`DTTwinManager` persistent shadow with real vrnetlab images)
   is the scalability story for V3 — slide `DT_V3_PERSISTENT_SHADOW.md`.
4. **🔁 Post-change Verifier (PCV)** — baseline captured pre-approval;
   diff runs post-apply. Verdicts: `PASSED`, `REGRESSION_DETECTED`,
   `FIX_DID_NOT_TAKE`, `INDETERMINATE_PROBABLY_OK`. Closes the
   autonomous-remediation loop.

*Talking point:* "Every fix the platform proposes carries four
enterprise-grade gates before it touches your fabric — risk policy,
blast radius, DT simulation, and post-change diff. None of them are
LLM judgments; each is rule-based and citable. That's what makes this
sellable into a SOX/PCI environment."

## 3. Determinism talking point
Re-trigger the **same** incident → **same agent, same routing, every time** (deterministic regex triage —
not an LLM coin-flip). Reference `TEST_PLAN.md` determinism contract: routing is bit-exact by construction.

## 4. Virtual Architect — ReAct design chat
Health verified: `GET /api/architect/health` → ok, 10 read-only tools. Ask via the UI chat (or
`POST /api/architect/ask`). Watch the streamed ReAct events: `thought → tool_call → observation →
final_answer`. Talking point: the architect is **read-only** — it advises, never emits CLI.

**⚠️ Demo ONLY a conceptual design question — NOT a device-state / "fix" question (on the fallback model).**
Verified live behavior on the current **gpt-4o-mini fallback**:
- **USE THIS (verified clean):** *"Compare HSRP, VRRP, and GLBP for first-hop redundancy at a campus
  distribution layer and recommend one with the design rationale. Do not give CLI."* → streams `open →
  thought → final_answer` with a full comparison, **0 CLI rejections**. Reliable on stage.
- **DON'T demo device-state prompts** ("is OSPF healthy on router-1…", even with "no CLI"): the model
  *correctly* calls `ospf_parser` and gets the data, but then keeps emitting CLI in its answer; the
  read-only guard rejects it and re-asks until the 8-turn budget is exhausted → *"couldn't reach a
  conclusion."* The ReAct tool-loop starts but the answer never lands. (Two prompts verified to loop.)

**Talking point either way:** the read-only safety guard is *working* — it will not let the architect
emit configuration, period. As of 2026-05-30 the VA runs **Qwen2.5-7B-Instruct + va-v2 LoRA** (self-hosted
vLLM on AWS, `10.42.20.202:8005`), gate-cleared per the gated-finetuning firewall. The retired Phi-4 v1
adapter was rejected by the same gate (worse than gpt-4o-mini) and is no longer in the path.

## 5. Cascade / dedup (the differentiator)
Fire 3 incidents on the **same device** within 60s → the team-leader buffers, picks the **root cause**
(LINK over OSPF/BGP) and **suppresses the victims** → **one** approval card, not three. (3 incidents on
3 *different* devices → 3 cards.)

## 6. Transcripts & verifiability tab (`:8505`)
Show the audit trail / transcripts (TimescaleDB-backed — restored). Every recommendation has a logged
reasoning chain id.

## 6a. Compliance audit trail (`:8501` History tab → CSV)

**Target audience:** CISO, Compliance Lead, Audit Manager.
**Regulation anchors:** DORA (EU) 2022/2554 + NIS2 (EU) 2022/2555.
**Pre-condition:** 2–3 incidents approved in §2/§3 so `agent_reasoning_chains` has content.

> **Full 90-second walkthrough script → `docs/pitch_assets/COMPLIANCE_DEMO_RUNBOOK.md`**
> (includes regulation cue card, objection handling, and timing guide).

**Step 1 — Hook (~20s).**
> "Every recommendation lands as a timestamped row in TimescaleDB — chain ID, named
> approver, tamper-evident HMAC. What auditors want is a mapping into their regulation.
> Watch."

**Step 2 — Pull (~20s).** Click "Download compliance CSV" in the History tab footer.
Open the file. Resize `dora_controls` and `nis2_controls` columns.

**Step 3 — Walk one row (~30s).** Pick the EVPN VTEP-loss row. Read across:
- `started_at` → Art.10 §1 (detection time)
- `severity=high` → Art.10 §2 / Art.17 §1 (classification)
- `root_cause` → Art.17 §3 (RCA)
- `proposed_commands` → Art.9 §4 (change management)
- `approver` + `decision_rationale` → Art.5 §2 / NIS2 Art.21(2)(i) (named approver)
- `outcome=resolved` → Art.17 §3 / NIS2 Art.21(2)(f) (lessons learned)

> "One row in your auditor's spreadsheet. One defensible change."

**Step 4 — Close (~20s).**
> "Today this is a CSV. Item 6 (next sprint) derives the mapping automatically from
> the catalogue scenario. The audit trail is a by-product of the platform, not a
> separate compliance system."

**Fallback:** if the button errors, open `docs/pitch_assets/compliance_sample.csv`
(pre-generated 7-row golden CSV, pre-loaded in a Numbers tab). Narrate Step 3
identically. The narrative survives a DB outage.

**Files:**
- `compliance/control_map.py` — DORA/NIS2 mapping (20-row table from the plan §2)
- `compliance/export_compliance_csv.py` — TimescaleDB → CSV exporter
- `docs/pitch_assets/compliance_sample.csv` — stage fallback

---

## 7. Stretch: real-device lab (only if rehearsed + stable)
1. Start the EVE-NG router VMs (eveng-prod) — they don't auto-start after the nightly cycle.
2. `sed -i 's/USE_REAL_DEVICES=false/USE_REAL_DEVICES=true/' .env && docker compose up -d --no-deps --force-recreate mcp-server`
3. Verify host→router reachability (the self-healing `netops-mgmt-ip.timer` puts the host on `192.168.255.1`).
4. Known real-device gaps: `aaa_checker` 500, `connectivity_tester` intermittent — avoid those tools on stage.
**Recommendation: demo in mock; only switch to real if the dry-run is clean.**

## 8. If something breaks mid-demo
- **MCP tool errors / Redis `-3`:** `sudo /usr/local/bin/netops-boot-reconcile.sh` (heals DNS + restarts deps).
- **A model is slow/erroring:** it auto-falls-back to gpt-4o-mini — keep going, it's expected.
- **A panel is stuck:** refresh the UI; queues persist in Redis.
- **Worst case:** re-run the morning pre-flight; the stack is reproducible.

## 9. Go/no-go per component (set during rehearsal)
| Piece | Demo on | Fallback |
|---|---|---|
| Triage/routing | deterministic engine | — (always works) |
| Stability / Troubleshooting | fine-tune if gate passes | gpt-4o-mini |
| Security | **gpt-4o-mini** (Mistral hallucination gate not yet met) | — |
| Architect | **Qwen-7B + va-v2 LoRA** (gate-cleared 2026-05-30) | gpt-4o-mini |
| **EVPN-VXLAN multi-vendor incident** | **mock** (3-leaf fabric: spine1 NX9K + leaf-nx NX9K + leaf-ar vEOS); fault classes verified end-to-end via §2a | OSPF incident in §2 (always works) |
| Devices | **mock** | real-lab stretch |

---

## 10. Pitch Demo Scripts

The runbook above is rehearsal/operator-facing. This section is **pitch-facing** —
three scripts (5/15/30 min) that turn the live system into a paid-pilot conversation.
Audience changes per script; the arc does not:

> Hook → trigger an EVPN incident on prod-dc → walk the 7-pass chain → CRO blocks on
> a cited policy → BRF quotes the dollar exposure → close with a pilot ask.

The same Saturday-on-Saturday demo of `%NVE-2-VTEP_LOSS` against `leaf-nx` is the spine
of all three. The differences are depth, time spent in the architecture, and which
files we click into.

**Read this once before every pitch:**
- Every claim is defensible against a file path in the repo. Architects will ask.
- "AI" appears in the pitch zero times unless they say it first. We say *deterministic
  triage*, *rule-based gate*, *cited policy*. The LLM only writes the diagnosis text;
  every gate is code.
- The CRO `BLOCK` on `weekend_freeze` is the strongest single moment in the demo.
  Build the timing around it. Do not rush past it.
- If anything breaks, the recovery is to **switch to a pre-recorded approval card
  screenshot** kept in `docs/pitch_assets/` and narrate the rest. The pitch survives
  a stack glitch; it does not survive a "let me debug this" detour.

---

### 10.1 The 5-minute pitch (executive: VP-Infrastructure / CISO)

**Goal:** book a 45-minute pilot scoping call.

**Pre-flight (do silently, before they walk in):**
- Browser tabs open: UI `:8501` Approvals, UI `:8501` Architect, `:8505` Transcripts.
- One throwaway incident already fired so models are warm.
- Date check: confirm it really is Saturday (or any frozen weekday in
  `policies.json::business_hours_only.allowed_weekdays`). If it isn't, manually set
  `params.frozen_weekdays` in a scratch copy of `policies.json` *before* the pitch —
  the CRO BLOCK is the money moment and it MUST fire.

#### Step 1 — The hook (~30s)

What the user does: open with a closed laptop.
What the audience sees: nothing yet.
What the user says (verbatim):

> "Most production outages come from change. Most change is human. Your operators are
> tired, they're on a 02:00 page, and they paste the right command on the wrong device.
> We built a platform that gates every proposed change against your policies, your
> blast radius, and a digital twin — before a human even sees the approval. Let me
> show you."

#### Step 2 — Trigger the incident (~30s)

What the user does: open laptop, run from a terminal already showing the curl prompt:

```bash
curl -s -X POST http://<APP_HOST_IP>:8001/incident/trigger \
  -H "X-MCP-API-Key: <MCP_API_KEY>" \
  -d '{"device_id":"leaf-nx","syslog":"%NVE-2-VTEP_LOSS: Lost reachability to remote VTEP 10.0.1.13"}'
```

What the audience sees: a one-line `{"status":"accepted"}` reply.
What the user says:

> "That's a real EVPN-VXLAN syslog from an NX9K leaf. Cross-vendor fabric — NX9K
> spine, NX9K leaf, Arista vEOS leaf. VNI 10010 is the tenant-payment-processing
> overlay. Watch what the platform does in the next sixty seconds."

#### Step 3 — Walk the chain (~2 min)

What the user does: switch to the UI `:8501`, Approvals tab. A new card lands within
~60s. Click it open. Walk these in order, pointing at each block of the card:

| What you point at | What you say (one sentence each) |
|---|---|
| Top of card: `agent=stability, tier=overlay, fault_type=evpn_vtep_unreachable` | "Triage is deterministic — regex rules, no LLM. Same input, same routing, every time." |
| Diagnosis block | "The specialist queried the live EVPN state of the leaf, the BGP-EVPN session, and the KG baseline, then proposed a fix in NX-OS syntax." |
| Pass 1 — ConfidenceScorer | "Pre-sim gate score from risk signals plus the LLM's own confidence." |
| Pass 2 — Fault correctness | "Rule-based check that the fix actually addresses the fault type. The LLM does not get to grade itself." |
| Pass 3 — Graph Simulator | "Layer 1 digital twin. In-memory reachability, blast radius, SPF simulation against the KG." |
| Pass 4 — Containerlab twin | "Layer 2 digital twin. The fix is applied to a real Nokia SR Linux + cEOS container fabric. Validated before a human sees it." |

#### Step 4 — The CRO money shot (~1 min)

What the user does: scroll to the `⚖️ Change Risk Officer` panel. Make sure the
audience is looking at it. Slow down.

What the audience sees: a red `BLOCK` verdict with cited policies. Specifically
`policy_id: weekend_freeze` (because today is Saturday).

What the user says (slow, verbatim):

> "The platform just blocked its own fix. Why? Because today is Saturday and your
> weekend change-freeze policy says no automated changes on weekends. That policy
> isn't an LLM opinion — it's a row in a JSON file your GRC team owns. See
> `agents/change_risk_officer/policies.json`. The platform team owns the policy
> engine. Your auditor owns the data. This is the citation an external auditor
> needs to sign off your SOX or PCI control."

Pause. Let it land. This is the demo.

#### Step 5 — The BRF money shot (~45s)

What the user does: scroll to the `💥 Blast Radius Forecaster` panel.

What the audience sees: a dollar figure (~$340K/hr SLA exposure), a tenant count
(~4,200 users), compliance tags (PCI-DSS, SOX).

What the user says:

> "And here's what that fix would have cost if we'd gotten it wrong. VNI 10010 hosts
> tenant-payment-processing — $340K/hour SLA exposure, 4,200 users, PCI-DSS plus SOX
> tagged. The KG knows because the topology is in code: see `fabric.yaml`. The
> approval card now speaks CFO, not router."

#### Step 6 — Close (~30s)

What the user does: close the card. Look at the audience. Do not click anything else.

What the user says:

> "Sixty seconds, end to end. Every decision cited. The fix never touched the fabric.
> That's what changes the economics of your NOC. Can we book forty-five minutes next
> week to scope a 60-day pilot?"

#### What if X breaks during the 5-min demo

**(a) Incident never lands in the UI within 90s.**
Recovery: switch to the second already-open tab `:8501` showing a previously fired
EVPN approval card (kept open from the warm-up run). Say *"I already had one fired
earlier — same shape, here it is"* and walk steps 3-5 against that card. The card
state is in Redis and persists across UI refreshes; the previous one is still there
unless you cleared `approval_queue`.

**(b) The CRO panel does not BLOCK (we miscalculated the weekday).**
Recovery: the card still shows `pci_dss_freeze` or `hipaa_no_auto` for the
tenant1-payment-processing tenant tag. Pivot the script: *"Today the freeze policy
that fires is PCI-DSS — quarter-end audit window — and you can see it cited by
`policy_id: pci_dss_freeze`. Same shape: rule-based, cited, auditable."* If neither
fires, fall back to the high-risk-command BLOCK by re-triggering the incident with a
syslog that pulls `clear ip bgp *` into the proposed fix (see §2a variants), which
trips `high_risk_commands` deterministically.

**(c) LLM endpoint is cold and the specialist takes >90s to return.**
Recovery: while waiting, narrate Pass 1-7 against the architecture in
`EVPN_NATIVE_ARCHITECTURE.md §4` shown on a second screen. The chain still arrives;
the silence is fatal, not the latency. Have one slide queued with the 7-pass diagram
ready to put up.

#### What NOT to say in the 5-min

- Do **not** say "AI-powered". Say "rule-based gate" and "deterministic triage".
- Do **not** say "fully autonomous". Say "human-in-the-loop with auditable gates".
- Do **not** quote MTTR numbers as if they are the customer's — the ROI math in
  `PITCH_ONE_PAGER.md` is illustrative and labeled as such. Their MTTR is whatever
  the pilot measures.
- Do **not** claim the platform "predicts" outages. It validates proposed fixes
  against a twin. That is enough.

---

### 10.2 The 15-minute pitch (Director of Network Engineering)

**Goal:** secure a technical discovery call with their team + a target use case.

**Pre-flight:** same as §10.1 plus open these files in the editor on a second screen:
- `agents/change_risk_officer/policies.json`
- `fabric.yaml`
- `agents/team_leader/workflow/triage_rules.py`
- `tests/test_kg_evpn_sync.py` (specifically the cross-vendor RT-match test)
- `EVPN_NATIVE_ARCHITECTURE.md` open at §4 and §6

#### Step 1 — The hook (~1 min)

What the user says (verbatim):

> "Your team owns thousands of devices across multiple vendors. Every change is one
> typo away from a four-hour MTTR call. The state of the art is a ticketing tool
> that asks an engineer to read a runbook at 03:00. We built a platform that does
> the diagnosis, validates the fix on a digital twin, gates it against your written
> policies, and tells you in dollars what you're risking — before your engineer
> even opens the ticket. Let me show you how it works."

#### Step 2 — Trigger and walk (~3 min)

Run the same `%NVE-2-VTEP_LOSS` trigger as §10.1. Open the approval card. For each
ValidateNode pass, open the **actual log line** in the terminal (you should have
`docker logs -f netops-stability-agent` streaming on a second tab):

```
[ValidateNode] Pass 1 ConfidenceScorer score=0.87 path=containerlab
[ValidateNode] Pass 2 Fault correctness: passed (evpn_vtep_unreachable rule matched)
[ValidateNode] Pass 3 GraphSimulator: 2 devices in blast radius, no SPF break
[ValidateNode] Pass 4 ContainerlabManager.validate_evpn: topology rendered
[ValidateNode] Pass 5 CRO verdict=BLOCK policies=[weekend_freeze]
[ValidateNode] Pass 6 BlastRadius $340K/hr, 4200 users, tags=[pci_dss, sox]
[ValidateNode] Pass 7 PCV baseline captured: ref=<uuid>
```

What the user says, pointing at the log stream:

> "These aren't slides — those are the log lines coming off the container right now.
> Seven passes, each one writing structured evidence into the card. Every line you
> see is a unit-tested function in `agents/shared/validate_node.py`."

#### Step 3 — Architecture walk (~3 min)

Switch to `EVPN_NATIVE_ARCHITECTURE.md §4` (the dispatcher diagram). Walk it.

What the user says:

> "Three-way dispatch on protocol family. EVPN syslog hits `_execute_overlay`. BGP
> unicast hits `_execute_underlay_bgp`. OSPF hits the legacy `_execute_underlay`.
> Each branch fetches the right data shape, builds a vendor-specific prompt, and
> returns a fault classification from a closed set. That dispatcher is twelve lines
> of code. The reason it works is because triage emits a `tier` — `overlay`,
> `underlay`, `transport`, `platform`, `security`, `service` — and the dispatcher
> reads it. Adding IS-IS or MPLS-TE as a new family is a four-step recipe; it does
> not touch the OSPF path."

Then switch to `EVPN_NATIVE_ARCHITECTURE.md §6` (the KG schema). Walk the
`EVI / VTEP / BGPEVPNPeer / MACRoute` graph.

What the user says:

> "The KG isn't a database of tables with foreign-key-as-string IPs. It's the actual
> topology. MAC mobility — has this MAC moved across VTEPs — is a two-hop traversal.
> Blast radius — which MACs lose connectivity if this VTEP fails — is a single
> match. These are the queries that take your team an hour in a spreadsheet today."

#### Step 4 — The silent-bug demo (~2 min)

Open `tests/test_kg_evpn_sync.py` and find the cross-vendor RT-match invariant test
(search for `test_rt_extraction` in `tests/test_kg_evpn_sync.py`).

What the user says:

> "This test pins the RT-match invariant between the NX9K leaf and the Arista vEOS
> leaf. Both should advertise `65000:10010` for VNI 10010. If they ever diverge — if
> someone fat-fingers an RT on one side — this test fails, the drift detector pages,
> and the platform refuses to propose any fix until you fix the intent. This is the
> kind of silent bug that takes a real datacenter team four hours to isolate at
> 02:00. We catch it in the YAML schema at commit time. See `EVPN_NATIVE_ARCHITECTURE.md §6.2`
> for the cross-vendor RT-consistency query that powers it."

#### Step 5 — CRO + policies.json (~2 min)

Switch back to the approval card. Show the BLOCK. Then open
`agents/change_risk_officer/policies.json` in the editor.

What the user says:

> "The policy that just blocked your fix is in this file. Six policies ship out of
> the box: `business_hours_only`, `weekend_freeze`, `quarter_close_freeze`,
> `pci_dss_freeze`, `hipaa_no_auto`, `high_risk_commands`. Your team owns this file.
> Your auditor reviews this file. The policy code is in `policies.py` — that's
> ours. The data is yours. In a real deployment this is fed from ServiceNow GRC or
> Drata. None of the gates are LLM judgments — every BLOCK is a regex match or a
> calendar check."

#### Step 6 — BRF + ROI bridge (~1 min)

Scroll to the BRF panel. Walk the dollar figure.

What the user says:

> "The dollar figure isn't fabricated — it's the tenant SLA from your CMDB times
> the worst-case convergence window for this fault class. For `evpn_vtep_unreachable`,
> the convergence window is 90s — see `agents/blast_radius_forecaster/agent.py`.
> When your CFO asks 'what's the autonomous platform actually saving us' you can
> answer it in dollars, per incident, with cited evidence."

#### Step 7 — Close (~3 min Q&A buffer)

What the user says:

> "What I'd like to do next: book ninety minutes with your senior engineers and your
> compliance lead. We bring our team. You bring one fault class you currently chase
> in spreadsheets. We come back in two weeks with a pilot plan that says: drop your
> topology in, hook to NetBox, validate three fault classes for sixty days, measure
> MTTR delta and false-positive rate. What would the right use case be for your
> team?"

#### What if X breaks during the 15-min demo

**(a) Containerlab Pass 4 returns "topology renderable, deploy timed out".**
Recovery: this is a known docker-in-docker race on cold start. Say: *"Pass 4 has two
modes — render and deploy. Render is what runs in the workshop instance; deploy is
what runs in production. The fix and the assertions are identical." Show
`agents/shared/containerlab_manager.py::validate_evpn` and point at the render path.
The audience sees the function, the architecture is intact.

**(b) MCP returns empty data on `evpn_parser` (real-device mode flipped on by mistake).**
Recovery: `sudo docker exec netops-mcp env | grep USE_REAL_DEVICES` from the
streaming terminal — confirm it's `true`, flip it: `sed -i
's/USE_REAL_DEVICES=true/USE_REAL_DEVICES=false/' /home/ubuntu/agentic-netops-mvp/.env
&& sudo docker compose up -d --no-deps --force-recreate mcp-server`. Recovery
takes ~20s. Talking point during the wait: *"This is the mock/real toggle — same
adapter contract, different backend; the agents don't know the difference."*

**(c) The LLM emits a malformed pattern_id and Pass 2 fails the fix.**
Recovery: this is actually a *good* demo moment. Say: *"Pass 2 just rejected its own
specialist's fix because it didn't match the expected pattern_id taxonomy in
`agents/stability/workflow/analyze_node.py::KNOWN_PATTERN_IDS`. This is the
self-defense layer — we don't trust the LLM's structured output. We grade it
against a closed set."* Re-trigger and move on.

#### What NOT to say in the 15-min

- Do **not** promise multi-vendor parity across **every** NOS feature. Say:
  "first-class for the vendors in the adapter registry — Cisco IOS-XE, NX-OS,
  Arista EOS, SR Linux, Mock. Adding a new NOS is a defined contract: see
  `agents/shared/vendor_adapters/`."
- Do **not** call the digital twin a "production network simulator". Call it a
  "high-fidelity validation surface for proposed fixes". The difference matters
  to engineers.
- Do **not** quote 100% routing accuracy as "the platform is right 100% of the
  time". The 100% is **routing**, not diagnosis. Diagnosis quality is in
  `WORKSHOP_STATUS.md §4` and is mixed — be honest about it.

---

### 10.3 The 30-minute deep-dive (Chief Architect / SRE Director)

**Goal:** discovery call with their architecture team + one named use case + access
to their staging NetBox.

**Pre-flight:** same as §10.2 plus:
- `docs/DT_V3_PERSISTENT_SHADOW.md` open on a second screen
- `verifiability-api` transcripts UI `:8505` warm
- The 5-layer architecture diagram (the ASCII one in `WORKSHOP_STATUS.md §2`) open
  in markdown preview, or printed
- `agents/shared/scope_policy.py` open in the editor

#### Step 1 — The hook (~1 min)

Identical to §10.2 Step 1.

#### Step 2 — The 5-layer architecture (~4 min)

Put up `WORKSHOP_STATUS.md §2` diagram. Walk top to bottom, naming each layer + its
protocol contract.

What the user says:

> "Five layers, three protocol contracts: ACP between UI and Team Leader, A2A
> between Team Leader and specialists, MCP between specialists and devices.
> Specialists never SSH directly. UI never writes Redis directly. These contracts
> are enforced by the code organization, not by hope. See
> `CLAUDE.md::Critical Architecture Rules`. Replacing a layer — say swapping
> Streamlit for a custom React UI — does not touch the agents. Replacing the LLM
> behind a specialist does not touch the MCP tools."

#### Step 3 — Trigger + walk the chain (~4 min)

Same trigger as §10.1 / §10.2. This time, while the chain runs, open the streaming
`docker logs -f netops-team-leader netops-stability-agent` view on the second
screen and walk the A2A handoffs:

```
[TeamLeader] classify_syslog: agent=stability tier=overlay fault_type=evpn_vtep_unreachable
[TeamLeader] A2A delegate -> a2a_stability_specialist_inbox
[Stability] ContextNode dispatcher: tier=overlay -> _execute_overlay
[Stability] MCP /tools/evpn_parser device=leaf-nx
[Stability] AnalyzeNode dispatcher: _build_evpn_prompt (multi-vendor)
[Stability] LLM call: <endpoint> tokens=..., elapsed=...s
[Stability] ValidateNode: Pass 1-7 ...
[Stability] A2A respond -> a2a_team_leader_inbox
[TeamLeader] approval_queue LPUSH
```

What the user says:

> "Every queue you see is in Redis. Every Redis key is auditable. Every recommendation
> has a chain ID stored in TimescaleDB — see the `:8505` transcripts tab. If you want
> to ask 'what evidence was on the card the night of the outage three months ago',
> the chain is there."

#### Step 4 — Topology-as-code (~3 min)

Open `fabric.yaml`. Walk the structure: devices, vendor map, VNIs, RTs, peers.

What the user says:

> "One YAML file is the source of truth for the topology. From it,
> `agents/shared/topology_generator.py` emits four things: a Containerlab YAML for
> the digital twin, a Neo4j seed Cypher for the KG, a NetBox import CSV for the
> intent layer, and the test fixtures the dispatcher and adapter tests run against.
> When you add a leaf, you edit one file and re-run the generators. The drift
> detector then compares NetBox intent — this file — against KG actual — synced
> live from your fabric. If your operator pushed a change that isn't in NetBox, the
> drift detector pages."

#### Step 5 — The 7-pass deep walk (~5 min)

Walk each pass with the code path open. This is the deepest section — be technical.

| Pass | File / function | What you say |
|---|---|---|
| 1 | `agents/shared/confidence_scorer.py::ConfidenceScorer.score` | "Three-band gate. Below 0.55, skip simulation, go direct to human. 0.55-0.85, graph sim. Above 0.85, full containerlab. The bands are tunable per fault class." |
| 2 | `agents/shared/validate_node.py::validate_fix_correctness` | "Per fault-type pre/post assertions. The LLM can lie about its own fix; this is the rule-based grader. Currently five EVPN rules, ten OSPF rules, expanding." |
| 3 | `agents/shared/graph_simulator.py` | "Layer 1 DT. In-memory SPF, reachability, blast radius from the KG snapshot. Runs in milliseconds." |
| 4 | `agents/shared/containerlab_manager.py::validate_evpn` | "Layer 2 DT. Renders containerlab YAML from `fabric.yaml`, brings up SR Linux + cEOS + FRR, applies the fix, runs the validation suite. ~30s wall clock." |
| 5 | `agents/change_risk_officer/policies.py` | "Six policy classes; data in JSON. The verdict carries a `cited_policies` list — show me the law I broke. This is what your auditor reads." |
| 6 | `agents/blast_radius_forecaster/agent.py` | "Translates fix to dollars. KG traversal: which VNIs are affected, which tenants hang off those VNIs, what's the SLA per tenant, what's the worst-case convergence window. The math is in code; the numbers are in your CMDB." |
| 7 | `agents/post_change_verifier/agent.py::capture_baseline` | "Snapshots pre-change state so the change-executor can diff after apply. Verdicts: PASSED, REGRESSION_DETECTED, FIX_DID_NOT_TAKE, INDETERMINATE_PROBABLY_OK. The autonomous loop closes on this pass." |

#### Step 6 — The scale answer (~5 min)

Every chief architect asks: how does this scale to my 16-leaf fabric with real NX-OS?

Open `docs/DT_V3_PERSISTENT_SHADOW.md`. Walk §1, §2, §4.

What the user says:

> "The current digital twin is Layer 2 — SR Linux as a high-fidelity NX-OS twin. It
> covers RFC-level behavior, but it misses the NX-OS-specific syntax footguns:
> auto-RT derivation, mgmt-vrf quirks, L3VNI bind-to-SVI ordering. V3 is the
> persistent shadow — vrnetlab-packaged real NX9K qcow2, always warm, snapshot-based
> per-incident lifecycle. Per-incident wall clock is 10 seconds; the 50-minute
> cold-boot cost is paid once per DT rebuild — weekly. For a 16-leaf fabric,
> per-incident validation cost is around 4 cents — see §4 table. V3 is scaffolded
> today; real vrnetlab integration is the next commit. The wiring point is
> `agents/shared/scope_policy.py` — a fault-type-to-twin-layer router. Layer 2 stays
> for the easy cases; Layer 3 fires for NX-OS-specific patterns. That's how this
> scales."

#### Step 7 — Verifiability + audit trail (~3 min)

Open `:8505` transcripts. Find a recent chain. Walk it.

What the user says:

> "Every reasoning chain is persisted to TimescaleDB. Every decision is keyed by a
> chain ID. The `pattern_id` we infer from the LLM diagnosis is checked against a
> closed set — see `agents/stability/workflow/analyze_node.py::KNOWN_PATTERN_IDS`.
> When the LLM picks `evpn_peer_admin_shut` we know which playbook fired, which
> commands ran, what the BRF said, what the CRO cited. Three months from now your
> SRE postmortem opens this tab, types the incident ID, and gets the full chain."

#### Step 8 — Close (~5 min Q&A)

What the user says:

> "Two asks. First, can we put ninety minutes on the calendar with your architecture
> team for a discovery? We come prepared with one named use case from your
> environment. Second, would you give us read-only access to your staging NetBox so
> our 60-day pilot starts at week three instead of week six? In return, you get a
> joint design document for the integration, no commitment, no obligation. The
> pilot scope and the pricing follow that call."

#### What if X breaks during the 30-min demo

**(a) Dry-run timeout on Pass 4 (containerlab deploy >60s).**
Recovery: switch to V3 talk track immediately. Say *"This is exactly why V3 exists.
Pass 4 is on the cold-boot path today; the persistent shadow eliminates this latency.
Let me show you the V3 architecture."* Open `docs/DT_V3_PERSISTENT_SHADOW.md §2`.
The break becomes the segue.

**(b) `verifiability-api` is crash-looping (known degraded — see WORKSHOP_STATUS.md §6).**
Recovery: skip the `:8505` walkthrough in Step 7. Show the TimescaleDB schema
directly from a psql terminal instead: `\d agent_reasoning_chains`. The data is
there; the rendering is broken. Talking point: *"The transcripts UI is the lookup
view; the chain data is in TimescaleDB — production deployments wire this into
Datadog or Grafana."*

**(c) An architect asks "how do you handle non-routing fault classes — multicast,
QoS, SDN policy?" and the demo system has none of those.**
Recovery: open `agents/team_leader/workflow/triage_rules.py` and walk the 70 rules.
Say *"Multicast and QoS aren't in scope today — the workshop fabric is EVPN-VXLAN
unicast. Adding a new tier and a new dispatcher branch is the four-step recipe in
`EVPN_NATIVE_ARCHITECTURE.md §4.1`. The architecture supports it; the data
collectors and prompts are the work. We can scope this for the pilot."*

#### What NOT to say in the 30-min

- Do **not** claim V3 is in production. Say "V3 is scaffolded; vrnetlab integration is
  the next commit". Reference `docs/DT_V3_PERSISTENT_SHADOW.md §8` and §3 commit map.
- Do **not** say "we replace your NOC". Say "we make the human approval step the
  fastest defensible step, not the slowest". This is a job-keeper, not a job-killer.
- Do **not** call the KG "the source of truth". The KG is the *actual* — `fabric.yaml`
  and NetBox are *intent*. The drift detector reconciles them. Architects will
  notice if you conflate these.
- Do **not** promise sub-second LLM inference. Be honest: 16-22s/specialist call
  today, AWQ quantization on the roadmap (see `WORKSHOP_STATUS.md §6` P2). The
  network engineer doesn't care; the platform engineer does.
- Do **not** quote a price. Say "the pilot scope and pricing follow the discovery
  call". Pricing in the deck is for the CFO conversation, not the architecture
  conversation.

