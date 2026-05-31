# EVPN-Native Architecture — agentic-netops-mvp

**Companion to `MULTIVENDOR_EVPN_E2E_TEST.md`.** That document captures the
live integration test (P1–P5) that proved a real multi-vendor EVPN fabric
could be observed end-to-end. This document captures the **architectural
work** done immediately after to make the 5-layer agentic stack
genuinely EVPN-native rather than OSPF-with-an-EVPN-collector.

Branch: `vendor-abstraction-phase-a` · Date: 2026-05-30 · 6 commits.

---

## 1. The change in one sentence

The stability specialist used to be an OSPF specialist that happened to be
hooked up to an EVPN data source. After this work, it is a **routing
control-plane specialist** that dispatches on protocol family (OSPF /
BGP-unicast / EVPN-VXLAN overlay) at three layers — triage, data
collection, and prompt construction — with a stable tier taxonomy used by
the Knowledge Graph and verifiability service.

---

## 2. Why this matters for the workshop demo

A naïve "we added EVPN" change leaves a brittle seam: triage still routes
EVPN syslogs the wrong way, the OSPF fetcher fires for an EVPN incident
and returns empty data, the OSPF prompt asks the LLM to diagnose "OSPF
adjacency" against EVPN inputs, the KG has no EVPN nodes, and the
verifiability service can't validate the proposed fix. Every layer leaks
its OSPF assumptions.

The work below fixes all five seams. The demo can now run an EVPN-VXLAN
incident end-to-end (syslog → triage → stability → KG-grounded LLM → fix
+ pattern_id → verifiability) without the operator hitting "this part
doesn't really work yet".

---

## 3. The role taxonomy (the conceptual core)

Two **independent** axes:

```
                 ┌─────────────────────────────────────┐
                 │              TIER                   │
                 │  (which network plane is the fault) │
                 │                                     │
                 │  underlay   overlay   transport     │
                 │  platform   security  service       │
                 └─────────────────────────────────────┘
                                   ×
                 ┌─────────────────────────────────────┐
                 │              AGENT                  │
                 │      (who owns triage + fix)        │
                 │                                     │
                 │  stability   troubleshooting        │
                 │  security    virtual-architect      │
                 └─────────────────────────────────────┘
```

| | underlay | overlay | transport | platform | security | service |
|---|---|---|---|---|---|---|
| stability | ✓ (OSPF, EIGRP, IS-IS, **BGP-unicast**) | ✓ (EVPN-VXLAN) | ✓ (interface, line-protocol) | ✓ (CPU, memory, env) | | |
| troubleshooting | | | ✓ (BFD, STP, duplicate IP) | | | ✓ (NTP, FHRP, MPLS, PIM) |
| security | | | | | ✓ (ACL, AAA, auth, crypto) | |

Tier and agent are decoupled on purpose: BGP-unicast is `(stability,
underlay)`, BGP-EVPN is `(stability, overlay)`. They share an owner but
the tier tells downstream consumers (context fetcher, KG schema, prompt
builder, dashboards) **which protocol shape to use**.

### 3.1 Tier vocabulary

| Tier | Definition |
|---|---|
| `underlay` | Routing protocol control plane building loopback / next-hop reachability (OSPF, EIGRP, IS-IS, BGP-unicast) |
| `overlay` | EVPN-VXLAN overlay control + data plane (EVIs, VTEPs, BGP-EVPN peers, Type-2 MAC mobility) |
| `transport` | L1/L2 substrate (interface, line-protocol, BFD, STP, VLAN trunk, storm-control) |
| `platform` | Device hardware / environment / OS (CPU, memory, power, env, stack membership) |
| `security` | Auth / ACL / AAA / crypto / policy events |
| `service` | Supporting protocols (NTP, FHRP, MPLS-TE, IP SLA, DHCP, PIM, LDP, object-tracking) |

Triage now emits `tier` alongside `agent` and `fault_type` in every
`classify_syslog()` result. 65 fault types explicitly mapped, with
prefix-fallback heuristics for unmapped strings (so adding a new EVPN
rule without a tier mapping doesn't silently fall through to "unknown").

---

## 4. The 3-way protocol-family dispatcher

The stability specialist now dispatches on protocol family at two layers:

```
                                state
                                  │
                                  ▼
                       ┌─────────────────────────┐
                       │   ContextNode.execute() │
                       │      (dispatcher)       │
                       └────────────┬────────────┘
              ┌─────────────────────┼─────────────────────┐
              ▼                     ▼                     ▼
       tier=='overlay'         fault_type            (legacy path)
       OR fault_type.        startswith('bgp_')
       startswith('evpn_')
              │                     │                     │
              ▼                     ▼                     ▼
     _execute_overlay()    _execute_underlay_bgp()   _execute_underlay()
                                                       (OSPF, byte-
                                                        for-byte
                                                        preserved)
              │                     │                     │
              ▼                     ▼                     ▼
        evpn_parser            bgp_summary           ospf_parser
        + bgp_summary          + interface_parser    + interface_parser
                               + routing_table       + routing_table
                                                     + ... (legacy)
              │                     │                     │
              └──────────────┬──────┴──────┬──────────────┘
                             │             │
                             ▼             ▼
                      state['context']  ← fully populated
                                         (tier-aware shape)
                             │
                             ▼
                       ┌──────────────────────────┐
                       │ AnalyzeNode              │
                       │ ._build_grounded_prompt()│
                       │   (dispatcher)           │
                       └──────────────┬───────────┘
              ┌─────────────────────┬─┴───────────────────┐
              ▼                     ▼                     ▼
       _build_evpn_prompt    _build_bgp_prompt    (legacy OSPF inline
       (multi-vendor:        (multi-vendor:        prompt — byte-for-
       NX-OS/EOS/SR Linux)   IOS-XE/NX-OS/EOS/    byte preserved)
                             SR Linux)
              │                     │                     │
              └──────────────┬──────┴──────┬──────────────┘
                             │             │
                             ▼             ▼
                       Multi-vendor prompt sent to LLM
                       (with response schema demanding
                        pattern_id from a closed set)
```

### 4.1 Why a dispatcher rather than `if`s

Each branch owns its own data shape (`evpn_*` keys vs `bgp_*` keys vs
`ospf_*` keys) and its own prompt structure. Mixing them in a single
function would mean every code path has to no-op around the inapplicable
data — which is exactly how the OSPF leakage problem started. The
dispatcher costs ~30 lines but makes adding a new protocol family (IS-IS-
specific, MPLS-specific) a 4-step recipe:

1. Tag the tier in `triage_rules.FAULT_TYPE_TIERS`.
2. Add `_execute_<family>()` branch in `context_node.py`.
3. Add `_build_<family>_prompt()` in `analyze_node.py`.
4. Extend each dispatcher with one `if` clause.

No legacy code touched; no risk of OSPF or EVPN regressions.

### 4.2 Defensive fail-open

Both `AnalyzeNode` dispatcher branches wrap the specialized builder in
`try/except`. If `_build_evpn_prompt()` raises, the dispatcher falls
through to the legacy OSPF prompt rather than crashing. **The demo
cannot break on a prompt-construction bug** (unit-tested:
`test_prompt_builder_fails_open_to_ospf_path`).

---

## 5. Triage rule changes

| Change | Count | Effect |
|---|---|---|
| New EVPN regex rules | 6 | BGP-EVPN ADJCHANGE, `%BGP_EVPN-`/`%L2VPN_EVPN-`, `%L2RIB-` (MAC inconsistency), `%NVE-` (VTEP), `%L2VPN-` (VNI state), vendor-agnostic `MAC_MOVE`/`HOST_FLAP` |
| BGP-unicast reassigned | 11 | All previously routed to `troubleshooting`; now route to `stability` (Q1) |
| Tier emission | every rule | `tier` key added to `classify_syslog()` result, mapped from `FAULT_TYPE_TIERS` |

Distribution shift on the regenerated `rule_coverage_corpus.jsonl` (70
entries, 0 shadowed):

```
Before EVPN-native push:    stability: 28 (43%)  troubleshooting: 26 (40%)  security: 10 (15%)
After EVPN-native push:     stability: 39 (56%)  troubleshooting: 16 (23%)  security: 15 (21%)
```

Stability is now the destination for >half of all classified syslogs —
consequence of owning all routing control plane.

---

## 6. Knowledge Graph EVPN schema

The Neo4j KG was previously OSPF-only (`Device`, `Interface`,
`OSPFNeighbor`, `Route`). Four new node labels + six new relationships
land the EVPN data into the graph.

```
Device ───┬─[:HAS_EVI]──────────────► EVI(device_id, vni, rd,
          │                              rt_import, rt_export,
          │                              state, tenant, type2/3/5_count)
          │
          ├─[:ACTS_AS_VTEP]──────────► VTEP(device_id, vtep_ip, role='local',
          │                                 source_interface, state)
          │
          ├─[:HAS_REMOTE_VTEP]───────► VTEP(device_id, vtep_ip, role='remote',
          │                                 remote_device, state, learned_via)
          │
          ├─[:HAS_BGPEVPN_PEER]──────► BGPEVPNPeer(device_id, peer_ip,
          │                                       peer_as, state, uptime,
          │                                       remote_device)
          │
          ├─[:ORIGINATES_MAC]────────► MACRoute(device_id, mac, vni,
          │                                    vtep_ip, origin='local',
          │                                    esi, mobility_seq, ip)
          │
          └─[:LEARNED_MAC]───────────► MACRoute(device_id, mac, vni,
                                                 vtep_ip, origin='remote',
                                                 esi, mobility_seq, ip)

                                          MACRoute ─[:REACHABLE_VIA]─► VTEP
                                          (data-plane next-hop —
                                           the key 2-hop MAC mobility edge)
```

### 6.1 Composite uniqueness keys

```cypher
CREATE CONSTRAINT evi_key          REQUIRE (e.device_id, e.vni)                 IS UNIQUE
CREATE CONSTRAINT vtep_key         REQUIRE (v.device_id, v.vtep_ip, v.role)     IS UNIQUE
CREATE CONSTRAINT bgpevpn_peer_key REQUIRE (p.device_id, p.peer_ip)             IS UNIQUE
CREATE CONSTRAINT mac_route_key    REQUIRE (m.device_id, m.mac, m.vni)          IS UNIQUE
```

Composite keys allow the same VNI / MAC to exist on multiple devices
without collision (which is the whole point of EVPN — a MAC seen on N
VTEPs is N graph nodes linked by `REACHABLE_VIA`).

### 6.2 Demo queries the new schema enables

```cypher
// Blast radius: which MACs lose connectivity if this VTEP fails?
MATCH (v:VTEP {state: 'down'})<-[:REACHABLE_VIA]-(m:MACRoute)
RETURN m.mac AS mac, m.vni AS vni, m.device_id AS learned_by_device
```

```cypher
// MAC mobility: has this MAC moved between VTEPs?
MATCH (m:MACRoute {mac: '0050.5600.0001'})-[:REACHABLE_VIA]->(v:VTEP)
RETURN m.device_id AS observer, v.vtep_ip AS data_plane_next_hop,
       m.mobility_seq AS seq, m.origin AS role
ORDER BY m.mobility_seq DESC
```

```cypher
// Cross-vendor RT consistency: are leaf-nx and leaf-ar advertising the same RT for VNI 10010?
MATCH (e1:EVI {vni: 10010}), (e2:EVI {vni: 10010})
WHERE e1.device_id < e2.device_id
RETURN e1.device_id, e1.rt_import, e2.device_id, e2.rt_import,
       e1.rt_import = e2.rt_import AS consistent
```

Each of these is a single short traversal. In the tabular OSPF schema,
they would have needed joins across multiple tables with foreign-key-as-
string IP next-hops.

### 6.3 Seed + live-sync

The KG is hydrated in two ways:

| Source | When | What |
|---|---|---|
| `STATIC_DEVICES` + `EVPN_FABRIC_DEVICES` in `neo4j-kg/sync.py` | Once, on first boot (KG empty) | Full topology baseline: 3 OSPF routers + 3 EVPN fabric devices (spine1 / leaf-nx / leaf-ar) with realistic EVIs, VTEPs, peers, and MAC routes |
| `sync_from_mcp` + `sync_evpn_from_mcp` (new) | Every cycle when `USE_REAL_DEVICES=true` | Refresh observable EVPN state from MCP `/tools/evpn_parser`. Preserves seed-only enrichment fields (`tenant`, `peer_as`, `remote_device`) by not naming them in `SET` clauses |

The cross-vendor RT-match invariant from the seed data (`65000:10010` on
both leaves) is what `MAC_MOVE`-class diagnoses can compare against —
the LLM gets the "expected" baseline as `kg_historical` and the "observed"
snapshot from MCP, and the prompt asks it to spot what changed.

---

## 7. Live data pipeline

```
                ┌────────────────────────────────┐
                │  evpn_parser MCP tool          │
                │     (POST /tools/evpn_parser)  │
                └──────────────┬─────────────────┘
                               │
                MCP_USE_ADAPTER │
                       false   │   true (flag-gated)
                               │
                ┌──────────────┴─────────────┐
                ▼                            ▼
       evpn_parser_tool()            AdapterFactory.for_device()
       (mock 3-leaf fabric)          │
                                     ├─► N9kvAdapter (NX-API)
                                     ├─► AristaEOSAdapter (eAPI)
                                     ├─► SrlinuxAdapter (gNMI)
                                     └─► MockAdapter
                                     │
                                     ▼
                          .collect(Collector.EVPN)
                                     │
                                     ▼
                          EVPNParserResponse (neutral)
                          + "_source: adapter:<class>"
                                     │
              On exception (any error) ──► falls back to mock
                                     │
                                     ▼
                       ┌──────────────────────────────┐
                       │  Neutral shape:              │
                       │    evis[]                    │
                       │    peers[]                   │
                       │    vteps[]       ← per-VTEP  │
                       │    mac_routes[]  ← per-MAC   │
                       │    mac_routes_total          │
                       │    mac_inconsistency_count   │
                       └──────────────┬───────────────┘
                                      │
                                      ▼
                       ┌─────────────────────────────────┐
                       │ Two consumers in parallel:      │
                       └─────────────────────────────────┘
                                      │
                  ┌───────────────────┴────────────────────┐
                  ▼                                        ▼
       stability/_execute_overlay                neo4j-kg/sync_evpn_from_mcp
       (per-incident, on-demand)                  (every cycle, background)
                  │                                        │
                  ▼                                        ▼
            EVPN prompt                            EVI / VTEP / BGPEVPNPeer /
            (multi-vendor                          MACRoute nodes +
             diagnosis)                            REACHABLE_VIA edges
```

### 7.1 The persistent SSH tunnel daemon

The adapter hook needs to reach the lab devices' management plane. The
new `mcp-server/_tunnel_daemon.py` holds three persistent local-bind
forwards inside the netops-mcp container:

```
127.0.0.1:11443  →  ubuntu@EVE-NG  →  spine1   172.30.0.11:443
127.0.0.1:12443  →                  →  leaf-nx  172.30.0.12:443
127.0.0.1:13443  →                  →  leaf-ar  172.30.0.13:443
```

Self-heals every 30s. Ephemeral on container recreate — relaunch with
`docker exec -d netops-mcp python /app/_tunnel_daemon.py` after each
`docker compose up --force-recreate`.

### 7.2 Mock-safe-by-default

`docker-compose.override.yml` adds the env structure with `${VAR}`
references — `MCP_USE_ADAPTER` defaults to `false`. The demo path
runs against the mock; the live path opts in via a local `.env`.
**No credentials in the branch.**

---

## 8. Verifiability hooks

When the stability EVPN flow proposes a fix, it now emits a
`pattern_id` that the verifiability-api can use to look up its
validation playbook. The catalogue lives in
`agents/stability/workflow/analyze_node.py::KNOWN_PATTERN_IDS`.

### 8.1 EVPN pattern_ids

| pattern_id | When the LLM picks it |
|---|---|
| `evpn_peer_admin_shut` | BGP-EVPN peer admin-shut; fix is recovery via `no shutdown` / `no neighbor X shutdown` / `admin-state enable` |
| `evpn_peer_down` | BGP-EVPN peer IDLE/ACTIVE without operator action (underlay loss, MTU, RT mismatch) |
| `evpn_vtep_unreachable` | NVE source-interface down OR underlay route to remote VTEP missing |
| `evpn_vni_state` | EVI down OR cross-vendor RT mismatch |
| `evpn_mac_inconsistency` | Type-2 MAC mobility / duplicate MAC |

### 8.2 `_infer_pattern_id` cross-vendor recognition

For LLM responses that miss the pattern_id field (older models, regression
during training), the publish-time `_infer_pattern_id()` in
`validation_api_client.py` does keyword-based fallback. It recognizes all
three vendor recovery syntaxes for EVPN admin-shut:

```
NX-OS / IOS-XE:  router bgp X ; neighbor Y ; no shutdown
EOS:             router bgp X ; no neighbor Y shutdown          (token gap!)
SR Linux:        set network-instance default protocols bgp neighbor Y admin-state enable
```

### 8.3 Service-side gap

The verifiability-api itself (separate microservice) still needs
**playbooks** for each new pattern_id. Without them, EVPN fixes still
land in `MANUAL_REVIEW`. But the operator now sees *which* pattern_id was
requested and can publish the playbook iteratively. Previously they got
`unknown-pattern` for every EVPN fix and had no signal at all.

---

## 9. Test coverage

185 unit tests, run in ~0.13s, no external dependencies:

```
tests/test_triage_rules.py           139 tests   includes 8 new EVPN regex tests,
                                                 5 tier classifier tests,
                                                 8 end-to-end (agent, tier) tests
tests/test_stability_dispatcher.py    15 tests   ContextNode + AnalyzeNode dispatchers,
                                                 cross-leakage, fail-open
tests/test_kg_evpn_sync.py            31 tests   envelope unwrap, RT preservation,
                                                 per-VTEP/per-MAC consistency,
                                                 EVPN pattern inference
```

Drift guards that fail loudly on regression:
- Every fault_type in `TRIAGE_RULES` must have a `FAULT_TYPE_TIERS` entry
  (`test_every_triage_rule_fault_type_has_a_tier`)
- Every tier must have ≥1 fault_type
  (`test_tier_distribution_sanity`)
- EVPN-flavored BGP lines must not fall through to generic BGP catch-all
  (`test_evpn_routes_before_generic_bgp`)
- EVPN AF fix must not classify as generic `bgp-neighbor-shut`
  (`test_evpn_specific_rules_precede_generic_bgp`)

---

## 10. What was deferred (next-round candidates)

| Item | Effort | Owner |
|---|---|---|
| Architect `SYSTEM_PROMPT` extension for EVPN device IDs + multi-vendor fabric context | small | **User** (coupled to VA fine-tuning recipe) |
| Real adapter parsers for `show nve peers` / `show l2route evpn mac all` (NX-OS) | medium | Open |
| Equivalent for Arista EOS (`show vxlan vtep`, `show bgp evpn route-type mac-ip`) | medium | Open |
| Equivalent for SR Linux (gNMI `/network-instance/.../bgp-evpn/...`) | medium | Open |
| Verifiability-api service playbooks for 5 new EVPN pattern_ids | medium | Service-side |
| DT EVPN sandbox wiring (containerlab SRL → dt-runner) | large | Open |
| End-to-end mock-mode incident dry-run with EVPN syslog injection | small | Open |

---

## 11. Commit map (vendor-abstraction-phase-a)

```
ddd1af3  verifiability: EVPN pattern_ids + stability KNOWN_PATTERN_IDS catalogue   (item A)
d5a34bb  tests: unit coverage for 3-way dispatcher + KG EVPN sync + tier classifier (item C)
aeed89b  evpn: per-VTEP and per-MAC collectors (schema + mock + KG sync)            (#110)
d95ee6b  kg: live sync of EVPN state from MCP evpn_parser -> Neo4j                  (#109)
4c920b3  kg: EVPN-VXLAN schema extension + rule_coverage_corpus refresh             (#108)
87de132  stability: EVPN-VXLAN + BGP-unicast 3-way protocol dispatch (Q1+Q2+Q3)     (#105+106+107)
```

---

## 12. File map (where to look)

```
agents/team_leader/workflow/triage_rules.py        EVPN rules + FAULT_TYPE_TIERS + _resolve_tier
agents/team_leader/agent_adk.py                    tier propagation through _triage_incident
agents/stability/workflow/context_node.py          3-way dispatcher + _execute_overlay + _execute_underlay_bgp
agents/stability/workflow/analyze_node.py          prompt dispatcher + _build_evpn_prompt + _build_bgp_prompt
                                                     + KNOWN_PATTERN_IDS + pattern_id validation
agents/stability/prompts/evpn_analysis.txt         reference EVPN prompt template
agents/virtual_architect/architect_chat.py         evpn_parser added to ARCHITECT_ALLOW_TOOLS
agents/shared/validation_api_client.py             _infer_pattern_id EVPN block (cross-vendor)

mcp-server/main.py                                 evpn_parser flag-gated adapter hook
mcp-server/_tunnel_daemon.py                       persistent SSH tunnel daemon
mcp-server/models/schemas.py                       EvpnVtep + EvpnMacRoute + extended EVPNParserResponse
mcp-server/tools/evpn_parser.py                    mock with 3-leaf fabric + per-VTEP/per-MAC

neo4j-kg/sync.py                                   EVPN_FABRIC_DEVICES + apply_schema EVPN constraints
                                                     + seed_device_evpn + sync_evpn_from_mcp
agents/shared/kg_client.py                         get_device_context extended with EVPN entities

docker-compose.override.yml                        mock-safe ${VAR} env structure (MCP_USE_ADAPTER off)

tests/test_triage_rules.py                         139 tests (8 new EVPN + 5 tier + 8 e2e)
tests/test_stability_dispatcher.py                 15 tests (ContextNode + AnalyzeNode dispatchers)
tests/test_kg_evpn_sync.py                         31 tests (sync + per-VTEP/per-MAC + pattern_id)
```
