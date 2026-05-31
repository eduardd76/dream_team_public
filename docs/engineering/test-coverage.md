# End-to-end mock-mode dry-run

Driven against a live `docker compose up` (mock mode), validates the full
5-layer incident pipeline for **both** protocol families the stability
specialist now owns:

| Pipeline | Scope | Built by | Test file |
|---|---|---|---|
| OSPF underlay | router-1, router-2, router-3 (legacy IOS-XE 3-router topology) | CCIE-EI agent | `test_ospf_underlay_e2e.py` |
| EVPN-VXLAN overlay | spine1, leaf-nx, leaf-ar (multi-vendor NX9K + vEOS fabric) | CCIE-DC agent | `test_evpn_overlay_e2e.py` |
| EVPN Phase 3 close | leaf-nx (peer admin-shut) — exercises Pass 2 EVPN fault rules + Pass 4 `validate_evpn()` real evidence | Phase 3 close (2026-05-30) | `test_evpn_phase3_e2e.py` |

Companion `EXPECTED_*.md` docs describe per-layer behavior so a human
operator can read either doc and understand what "PASS" means without
reading the test code.

---

## Quick start

```bash
# 1. Bring the stack up in mock mode (USE_REAL_DEVICES=false)
docker compose up -d

# 2. Wait for health (~30s after up)
curl -sf http://localhost:8001/health
curl -sf http://localhost:8002/acp/health

# 3. Run all dry-run scenarios (7 total: 3 OSPF + 4 EVPN)
tests/dry_run/run_all.sh

# Or one pipeline at a time
tests/dry_run/run_all.sh --ospf-only
tests/dry_run/run_all.sh --evpn-only

# Or one scenario, verbose
/tmp/utestvenv/bin/python tests/dry_run/test_evpn_overlay_e2e.py \
    --scenario evpn_peer_admin_shut --verbose
```

Exit codes (worst-case wins):
- `0` — all scenarios passed
- `1` — at least one assertion failed (test output names which)
- `2` — stack unreachable (one or both scripts couldn't connect)

---

## Scenarios

### OSPF underlay (CCIE-EI, 3 scenarios)
| Scenario | Trigger syslog | Tests |
|---|---|---|
| `ospf_neighbor_loss` | `%OSPF-5-ADJCHG` FULL→DOWN dead-timer | Legacy `_execute_underlay` path + inline OSPF prompt |
| `bgp_unicast_admin_shut` | `%BGP-5-ADJCHANGE` user-reset | Q1 reassignment path → `_execute_underlay_bgp` + `_build_bgp_prompt` |
| `link_flap` | `%LINK-3-UPDOWN` interface down | Transport-tier → stability cascade |

### EVPN-VXLAN overlay (CCIE-DC, 4 scenarios)
| Scenario | Trigger syslog | Vendor | EVPN CASE | Tests |
|---|---|---|---|---|
| `evpn_peer_admin_shut` | `%BGP-5-ADJCHANGE … L2VPN EVPN session admin shut` | NX-OS | CASE 1 | `_execute_overlay` → `_build_evpn_prompt` → `pattern_id="evpn_peer_admin_shut"` → recovery cmd is multi-vendor (NX-OS / EOS / SR Linux) |
| `evpn_vtep_unreachable` | `%NVE-2-VTEP_LOSS` | NX-OS | CASE 3 | NVE source-iface OR underlay /32 fix |
| `evpn_mac_inconsistency` | `MAC_MOVE` with `mobility_seq=3` | agnostic | CASE 5 | Type-2 MAC mobility analysis; KG `REACHABLE_VIA` edge |
| `evpn_vni_state` | `%L2VPN-3-EVI_INACTIVE: EVI 10010 (VNI 10010 RD …)` | NX-OS | CASE 4 | RT mismatch / EVI down |

EOS-flavored scenarios deferred — `triage_rules.py` doesn't yet have
EOS-unique facilities; current rules use vendor-prefixed `%BGP_EVPN-`
and vendor-agnostic `MAC_MOVE` / `HOST_FLAP`.

### EVPN Phase 3 close (1 scenario, extended pipeline)

| Scenario | Trigger syslog | Vendor | Tests |
|---|---|---|---|
| `evpn_peer_admin_shut_phase3` | `%BGP-5-ADJCHANGE … L2VPN EVPN session admin shut` | NX-OS | Same triage path as the EVPN overlay test, but assertions go DEEPER on the approval card: `validation.fault_correctness.passed=True` (new EVPN Pass 2 rules), `validation.fault_correctness.reason in {fix_correct_evpn_peer_admin_shut, fix_correct_evpn_peer_down}`, `validation.containerlab_evidence.passed in {True, False}` (NOT a skip — sister agent A's `ContainerlabManager.validate_evpn` produces real evidence), `validation.containerlab_evidence.evpn_checks` present, CRO verdict in `{ALLOW, MANUAL_REVIEW, BLOCK}`, blast-radius $/hr > 0. Default poll timeout 300s — 7-pass pipeline + DT spin-up takes longer than the OSPF path. |

---

## Per-layer assertions (both pipelines)

| Layer | What's asserted |
|---|---|
| **Team Leader triage** | regex match → correct `agent`, `tier`, `fault_type` (read from `triage_rules.py`); incident consumed off `incident_queue` |
| **Stability ContextNode** | Right dispatcher branch fires (`_execute_overlay` for EVPN / `_execute_underlay_bgp` for BGP-unicast / `_execute_underlay` for OSPF). The right MCP tools are called and the wrong ones are NOT |
| **Stability AnalyzeNode** | Right prompt builder runs (`_build_evpn_prompt` / `_build_bgp_prompt` / inline OSPF). LLM-generated `proposed_fix.pattern_id` ∈ `KNOWN_PATTERN_IDS` (read live from `analyze_node.py`) |
| **Approval card** | Card lands on `approval_queue` with `agent_id="stability"`, `status="needs_approval"`, populated `root_cause` citing data from `evpn_parser`/`ospf_parser` (anti-hallucination), `confidence` ∈ [0,1], `risk_assessment.risk_level` ∈ {Low, Medium, High} |
| **Knowledge Graph** (EVPN only) | After EVPN incident, `leaf-nx` has ≥1 EVI node, ≥2 VTEP nodes (1 local + ≥1 remote), `REACHABLE_VIA` edge present from a MACRoute. Cross-vendor RT match invariant: leaf-nx and leaf-ar both export `65000:10010` |
| **Verifiability** | `proposed_fix.pattern_id` is one of the catalogue entries; `_infer_pattern_id` fallback agrees with the LLM pick (cross-vendor recovery syntax recognition, see `validation_api_client.py`) |
| **Architect** | Asked "What's the OSPF state of router-2?" (OSPF) or "What's the EVPN state of leaf-nx?" (EVPN), returns markdown citing the queried device + uses tools from `ARCHITECT_ALLOW_TOOLS` |

---

## Infrastructure gaps discovered during build

Both agents independently discovered two gaps in the stack's externally
addressable surface. Both are documented here so the operator knows
what's *expected* not-quite-working before running.

### Gap 1 — `/incident/trigger` is enum-limited (HIGH priority)

`mcp-server/main.py:605` accepts only **query params** `device_id` +
`incident_type` from a fixed 4-value set: `ospf_neighbor_down`,
`interface_down`, `interface_errors`, `connectivity_failure`. It does
NOT accept an arbitrary syslog body.

**Consequence:** of the 7 dry-run scenarios, only 2 can use
`/incident/trigger`. The other 5 (1 OSPF link-flap variant + 4 EVPN +
1 BGP-unicast) fall back to direct `LPUSH incident_queue` via
`docker exec netops-redis redis-cli`. This is correct (mirrors the
`syslog-server` contract exactly) but means **the test depends on
`docker exec` access to the host** — the dry-run won't work against
a remote stack unless you have SSH/SSM and forward `docker exec`
through it, or unless you extend the trigger endpoint.

**Suggested fix (post-workshop):** extend `/incident/trigger` to accept
a `syslog` body field, or add a separate `/incident/inject` for
arbitrary syslogs. Tests would then drop the `docker exec` dependency
and become runnable against any reachable stack.

### Gap 2 — Architect port 8006 is not host-mapped

`virtual-architect` listens on `:8006/api/architect/ask` inside its
container but `docker-compose.yml` does not `ports:` map it to the
host. The team-leader proxies *some* architect requests at `:8002/...`
but the read-only ReAct ask endpoint is direct.

**Consequence:** the architect verification step in both dry-run
scripts treats reachability as **best-effort**. If `localhost:8006`
times out, the test logs `SKIPPED (architect endpoint not reachable
from host)` and continues. The OSPF / EVPN approval-card assertions
(steps 1-4) still gate PASS/FAIL.

**Suggested fix:** add `8006:8006` to the `virtual-architect` service
in `docker-compose.yml`, OR add a proxy route on the team-leader to
forward `:8002/api/architect/ask` → `virtual-architect:8006/...`.

---

## Files

```
tests/dry_run/
  README.md                       — this file
  run_all.sh                      — top-level runner (calls both Python scripts)
  test_ospf_underlay_e2e.py       — CCIE-EI: 3 OSPF/BGP-unicast/link scenarios (864 lines)
  test_evpn_overlay_e2e.py        — CCIE-DC: 4 EVPN-VXLAN scenarios (728 lines)
  test_evpn_phase3_e2e.py         — Phase 3 close: 1 EVPN scenario, Pass 2 + Pass 4 deep assertions
  EXPECTED_OSPF.md                — CCIE-EI: per-layer expected behavior (289 lines)
  EXPECTED_EVPN.md                — CCIE-DC: per-layer expected behavior (438 lines)
```

---

## Status

**Built only — has not been executed.** The user has not yet brought
the stack up and run `run_all.sh`. Treat this as a packaged dry-run
ready to ship to the workshop dry-run deadline (~2026-05-31).

When run, expect 2 of 7 scenarios to use `/incident/trigger` directly
and 5 to use the `docker exec` fallback (Gap 1 above). Architect-step
results will be `SKIPPED` until Gap 2 is closed.
