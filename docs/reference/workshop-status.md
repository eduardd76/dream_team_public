# Workshop Readiness Assessment — Agentic NetOps

_First assessment · 2026-05-23 · workshop target: 2026-06-16 (~3.5 weeks out)_

> **One-line verdict:** The system is **functionally complete and demo-able end-to-end**,
> now running **100% on self-hosted fine-tuned models** (no Hugging Face, no OpenAI for
> domain reasoning). The headline risks are **operational reliability** (the stack comes
> up degraded after the nightly auto-stop) and **inference latency** (specialists are slow).
> Routing/orchestration is solid; one model (security) needs quality work.

---

## 1. What it is
A 5-layer agentic Network Operations Center: syslog incidents are triaged and delegated
to specialist AI agents that diagnose using read-only network tools and a fine-tuned LLM,
then surface a human-approval card. A separate "Virtual Architect" answers design questions
via a live ReAct (reason + tool-call) trace.

## 2. Workflow & data flow

```
                          ┌─────────────────────────── Streamlit UI (:8501) ───────────────────────────┐
                          │  AI Chat · 🏗 Architect (ReAct) · ✅ Approvals · 📜 History                  │
                          └───────────────┬───────────────────────────────────┬───────────────────────┘
                                          │ ACP (REST :8002)                  │ SSE (ReAct stream)
                                          ▼                                   ▼
   syslog ──▶ Redis `incident_queue` ──▶ TEAM LEADER ───────────────┐   VIRTUAL ARCHITECT (Qwen-7B + va-v2 LoRA)
   (router)                              (deterministic triage)      │   read-only · catalogue + ReAct
                                          │ classify_syslog()        │   tools: ospf/bgp/intf/route/…
                                          │ (regex rules)            │
                                          ▼ A2A (Redis inbox)        │
              ┌────────────────────┬──────┴───────────┬─────────────┘
              ▼                    ▼                   ▼
        STABILITY            TROUBLESHOOTING        SECURITY            ← specialist agents
        (qwen2.5-7b)         (codellama-7b)         (mistral-7b)
              │                    │                   │
              ▼ MCP (HTTP :8001/8000, read-only tools, idempotency_key)
        ┌─────────────────────────────────────────────┐
        │ MCP SERVER → mock devices / EVE-NG routers   │
        └─────────────────────────────────────────────┘
              │ structured diagnosis {root_cause, evidence, proposed_fix, risk, confidence}
              ▼ A2A
        Redis `a2a_team_leader_inbox` ──▶ TEAM LEADER ──▶ Redis `approval_queue` ──▶ UI (human approve/reject)
                                                          (+ approved/rejected_history)
        (audit/verifiability → TimescaleDB `agent_reasoning_chains`)   ← currently DEGRADED
```

**Step-by-step:**
1. A syslog lands on `incident_queue` (router → syslog server, or direct).
2. **Team Leader** triages it **deterministically** (regex rule table — *not* an LLM) and delegates over an A2A Redis inbox to the right specialist. Per-device 3 s buffer + 60 s dedup lock.
3. The **specialist** collects device state via **read-only MCP tools**, runs its **fine-tuned LLM**, and returns a structured diagnosis + proposed fix.
4. Team Leader forwards it to `approval_queue` as a **human-in-the-loop** card (`needs_approval`).
5. The **UI** shows approvals/history; the **Architect** tab streams a live ReAct trace from the Qwen-7B + va-v2 LoRA architect.

## 3. Where every model runs (all self-hosted on AWS)
| Agent | Model | Host / endpoint | Quantization |
|---|---|---|---|
| stability | Qwen2.5-7B (fine-tuned) | GPU `10.42.20.168:8004` | vLLM 0.6.4, bnb 4-bit |
| troubleshooting | CodeLlama-7B (fine-tuned) | GPU `10.42.20.204:8003` | vLLM 0.6.4, bnb 4-bit |
| security | Mistral-7B (fine-tuned) | GPU `10.42.20.193:8002` | vLLM 0.6.4, bnb 4-bit |
| virtual_architect | **Qwen2.5-7B-Instruct + va-v2 LoRA** (gate-cleared 2026-05-30; replaced Phi-4 v1 — rejected by gated-finetuning firewall) | GPU `10.42.20.202:8005` | **vLLM 0.6.4, bf16** |
| team-leader | (deterministic rules; Mistral-7B fine-tune available as backup) | GPU `10.42.20.136:8001` | bnb (backup only) |

**No Hugging Face endpoints. No OpenAI for domain reasoning** (OpenAI remains only as an automatic safety fallback if a local endpoint is unreachable).

## 4. How well it works (measured)

**Orchestration / routing — STRONG**
- E2E synthetic test (18 incidents): **routing 100% correct** for every incident that completed; valid structured approval cards produced.
- Triage rule coverage: **97.1%** of known patterns classified correctly (offline suite).

**Model quality — MIXED** (rubric vs gpt-4o-mini baseline, composite 0–1)
| Agent | Fine-tune | Baseline | Note |
|---|---|---|---|
| stability | **0.846** | 0.906 | near-tie; emits clean JSON — the best specialist |
| troubleshooting | 0.757 | 0.978 | correct content, weaker structured-output adherence |
| security | **0.486** | 0.998 | **weakest** — hallucinates entities (0.276); needs work |
| architect (phi4, RETIRED) | 0.669 | 1.0 | historical eval — rejected by the gated-finetuning firewall; superseded 2026-05-30 by Qwen-7B + va-v2 LoRA (gate-cleared) |
- Caveat: the gap is largely **output-format**, not reasoning — measured with a generic prompt; production agents wrap models with their own parsers. The genuine model concern is **security hallucination**.

**Latency — WEAK**
- Specialists: **16–22 s per LLM call** (bitsandbytes 4-bit is unoptimized).
- E2E: incidents process **sequentially**, ~90–100 s each → an 18-incident burst exceeds a 10-min window (11/18 completed in the test; the rest were queued, not failures).

## 5. Current live state (this moment)
- **Overnight-off (expected):** the `autocon5` schedule stops all 6 instances at 22:00 Berlin and starts them at 08:00. Right now (post-22:00) AI_Dream_Team + specialists are stopped; the system serves during **08:00–22:00 Berlin** only.
- **Reliability caveat:** on auto-start the stack currently comes up **degraded** — GPU boxes can boot `impaired` (need a reboot), NVMe device names swap (orphaning model weights), and redis/verifiability don't always restart. This needs hands-on recovery today; see `REMEDIATION_PLAN.md` P1.
- **7-pass ValidateNode pipeline live on EC2 (verified 2026-05-30):** triage classifies the EVPN trigger to `stability, tier=overlay`; ContextNode dispatcher routes to `_execute_overlay`; AnalyzeNode renders the multi-vendor EVPN prompt; ValidateNode runs Pass 1 (ConfidenceScorer) → Pass 2 (fault correctness, 5 EVPN rules) → Pass 3 (GraphSimulator / Layer 1 DT) → Pass 4 (ContainerlabManager.validate_evpn / Layer 2 DT, render path) → Pass 5 (Change Risk Officer; `BLOCK` cited to `policy_id: weekend_freeze` on Saturday) → Pass 6 (Blast Radius Forecaster; ~$340K/hr SLA exposure across PCI-DSS + SOX tenants) → Pass 7 (PCV baseline captured). The approval card lands in Redis with full cited evidence; CRO/BRF/PCV panels render in the Streamlit UI.
- **4 new enterprise agents wired into ValidateNode:** Change Risk Officer, Blast Radius Forecaster, Post-change Verifier, plus the V3 DT skeleton. See `agents/change_risk_officer/`, `agents/blast_radius_forecaster/`, `agents/post_change_verifier/`, `agents/shared/validate_node.py`.
- **DT V3 persistent-shadow architecture documented + scaffolded:** `docs/DT_V3_PERSISTENT_SHADOW.md` + `agents/shared/dt_twin_manager.py` + `agents/shared/vrnetlab_backend.py` + `agents/shared/snapshot_store.py` + `agents/shared/scope_policy.py`. Real vrnetlab integration is the next commit; tests pass against `MockVrNetlabBackend`.
- **KG seeded from `fabric.yaml`:** the multi-vendor EVPN fabric (spine1 NX9K + leaf-nx NX9K + leaf-ar vEOS, VNI 10010, cross-vendor RT `65000:10010`) is in Neo4j; the cross-vendor RT-match invariant is unit-tested.
- **Test coverage:** ~300 unit tests pass offline in ~0.5s; 185 cover EVPN dispatcher + KG sync + tier classifier per `EVPN_NATIVE_ARCHITECTURE.md §9`; additional suites cover CRO policy evaluation, BRF traversal, PCV diff classification, and ScopePolicy routing.

## 6. Workshop-readiness scorecard
| Dimension | Status | Notes |
|---|---|---|
| Self-hosted models (no HF/OpenAI) | 🟢 GREEN | achieved for all 5 agents |
| End-to-end chain correctness | 🟢 GREEN | 100% routing; full chain verified |
| Architect ReAct demo | 🟢 GREEN | Qwen-7B + va-v2 LoRA live (gate-cleared 2026-05-30), real architectural-design output |
| **Multi-vendor EVPN-VXLAN coverage** | 🟢 GREEN | **EVPN-native push 2026-05-30** — triage + dispatcher + KG schema + prompt + verifiability all EVPN-aware. 185/185 unit tests green. See `EVPN_NATIVE_ARCHITECTURE.md` |
| Triage coverage | 🟢 GREEN | 70-rule deterministic coverage (was 64, +6 EVPN); 0 shadowed; LDP fix landed; tier emission per rule |
| **7-pass ValidateNode pipeline (incl. enterprise stack)** | 🟢 GREEN | **All 7 passes fire on EC2, verified 2026-05-30.** CRO BLOCK cites `weekend_freeze`; BRF emits ~$340K/hr SLA across PCI/SOX tenants; PCV baseline captured; cards render in UI. See `agents/shared/validate_node.py` + `DEMO_RUNBOOK.md §2b`. |
| **DT V3 persistent shadow** | 🟢 GREEN (design) / 🟡 (impl) | Architecture + skeleton + tests landed (`docs/DT_V3_PERSISTENT_SHADOW.md`). `RealVrNetlabBackend` integration is the next commit; pitch deck has the scale answer ready. |
| **Unit-test suite (~300 tests)** | 🟢 GREEN | 185 EVPN-native + 16 V3 DT + CRO/BRF/PCV/ScopePolicy suites. Pass offline in ~0.5s, no external deps. |
| Model quality | 🟡 YELLOW | stability good; **security weak (hallucination)** |
| Inference latency | 🟡 YELLOW | 16–22 s/call; fine for single demos, weak under load |
| **Operational reliability** | 🔴 RED | **stack not reboot-safe; degrades on nightly auto-start** |
| Audit/verifiability chain | 🔴 RED | `verifiability-api` crash-looping (exit 3); EVPN pattern_ids now emitted but service-side playbooks pending |
| **Pitch deliverables** | 🟢 GREEN | `DEMO_RUNBOOK.md §10` (3 pitch scripts), `PITCH_ONE_PAGER.md` (Fortune 1000 sell sheet) — both committed 2026-05-30. |

## 7. Recommended demo path (what reliably shows well)
1. **Architect ReAct** (UI Architect tab): "What OSPF neighbors does router-2 have?" → live mode→thought→tool_call→observation→final_answer trace from self-hosted Qwen-7B + va-v2 LoRA. *(Strongest, most visual.)*
2. **Single incident triage→approval**: trigger one OSPF/BGP/ACL incident → watch route → specialist diagnosis → approval card. *(Use one at a time — latency makes bursts slow.)*
3. **EVPN-VXLAN multi-vendor incident** (NEW 2026-05-30): trigger `%NVE-2-VTEP_LOSS` or `%BGP-5-ADJCHANGE ... L2VPN EVPN` for `leaf-nx` → watch the 3-way dispatcher pick the overlay branch → stability EVPN prompt with NX-OS / EOS / SR Linux fix syntax → architect can be asked about `leaf-nx`/`leaf-ar` (SYSTEM_PROMPT now learns the fabric). *Showcases the multi-vendor claim.*
4. **"All on our own models"** narrative: show the 4 GPU endpoints + the deterministic triage — no external API dependency.

## 8. Must-fix before the workshop (priority order)
1. **P1 reliability** — make the 08:00 auto-start come up green untouched (weights on persistent root + UUID mounts, restart policies, impaired-boot guard, post-boot reconcile). *Without this, every demo morning starts with manual recovery.*
2. **P5 verifiability-api** — restore the audit chain (or hide it from the demo).
3. **P2 latency (AWQ)** — makes live demos snappier and multi-incident demos viable.
4. **P4 security model** — fix hallucination, or steer the demo away from the security agent.
5. **P3 triage** — fix LDP bug + add the unmapped-mnemonic rules / LLM fallback.

_See `REMEDIATION_PLAN.md` for the full wave-based plan and agent assignments._
