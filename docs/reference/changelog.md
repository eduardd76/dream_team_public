# Changelog — Phases 0 through 3

Reverse chronological. Key commits in `()`; full git log in the private repo.

## Phase 3 (close) — 2026-05-30

**The pitch-ready stack.**

- `validate_evpn()` Phase 3 runtime: deploy → apply → wait → validate → teardown cycle, vendor-aware. (`9957866`)
- 5 EVPN fault-correctness rules added to Pass 2 (`evpn_peer_admin_shut`, `evpn_peer_down`, `evpn_vtep_unreachable`, `evpn_vni_state`, `evpn_mac_inconsistency`). (`9957866`)
- `RealVrNetlabBackend` implementation for V3 persistent shadow. (`9957866`)
- `ops/v3-host-setup.sh` for V3 EC2 provisioner.
- `PITCH_ONE_PAGER.md` + DEMO_RUNBOOK §10 (5/15/30-min pitch scripts).
- 99 unit tests across the new surface.
- Live-verified on EC2 (instance `<INSTANCE_ID>`): all 7 ValidateNode passes produce cited evidence; Pass 2 EVPN rule fired on `evpn_peer_admin_shut`. (`e868473`)

**Hardening:**

- `_CONTAINERLAB_DIR` multi-candidate path resolution for container layout. (`fc96e65`)
- Override bind-mount for `containerlab/` so Pass 3 templates reach `/app/containerlab/templates/`. (`00afc1b`)
- mgmt subnet injection (172.30.0.0/24 default) — eliminates docker network overlap with production stack. (`e868473`)
- Pass 2 wrong-peer rule extracts dotted-quad IP — no more false-failures on `peer address X.X.X.X` vs `neighbor X.X.X.X`. (`e868473`)

## Phase 2 — 2026-05-26 to 2026-05-28

**Topology-as-code + enterprise stack.**

- `fabric.yaml` at repo root — single source of truth. (`Phase 2 wireup`)
- `agents/shared/topology_generator.py` — FabricTopology + ContainerlabGenerator + KGSeedGenerator + NetBoxImportGenerator.
- Change Risk Officer agent + 6 policies + customer-replaceable `policies.json`.
- Blast Radius Forecaster agent + `tenant_catalog.py` + `tenants.json`.
- Post-change Verifier agent + `baseline_store.py` (Redis-backed).
- DT V3 persistent-shadow skeleton (`docs/DT_V3_PERSISTENT_SHADOW.md` + `dt_twin_manager.py` + `vrnetlab_backend.py` + `snapshot_store.py` + `scope_policy.py`).
- `stability/agent_a2a.py` line 392 fix — incident_type was defaulting to `ospf_neighbor_down` on EVPN incidents.

## Phase A + Phase B — 2026-05-19 to 2026-05-26

**Vendor abstraction.**

- `VendorAdapter` ABC + registry + factory (Phase A0).
- `CiscoIOSXEAdapter` + `MockAdapter` (Phase A1).
- Flag-gated reroute of ospf, interface, bgp, routing_table, cdp, stp, aaa, login_policy, security_parser via `_adapter_reroute.py` (Phase A2 + A3).
- `AristaEOSAdapter` (eAPI) — cross-vendor OSPF/BGP/interface normalization (Phase B-1).
- `N9kvAdapter` (NX-API) + `SrlinuxAdapter` (gNMI) + `VyOSAdapter` (Phase B-2).
- Live EVE-NG lab with NX9K + vEOS — cross-vendor EVPN-VXLAN core (Phase B-3).

## EVPN-native push — 2026-05-22 to 2026-05-24

- EVPN triage rules + 4 `evpn_*` fault_class strings added to Team Leader.
- ContextNode dispatcher: tier-aware routing (`_execute_underlay`, `_execute_underlay_bgp`, `_execute_overlay`).
- AnalyzeNode multi-vendor EVPN prompt (NX-OS / EOS / SR Linux recovery commands).
- KG EVPN schema extension: EVI, VTEP, BGPEVPNPeer, MACRoute, REACHABLE_VIA.
- KG live sync: MCP `evpn_parser` → Neo4j EVI/BGPEVPNPeer/MACRoute.
- Verifiability EVPN: pattern_ids + inference + prompt wiring.
- Architect ReAct SYSTEM_PROMPT: EVPN fabric awareness.

## Earlier (Phase 11.4 and before)

See `WORKSHOP_STATUS.md` and `ARCHITECTURE_REVIEW_PHASE4.md` for the foundation: Phase 1 (5-layer architecture), Phase 2.0 (Google ADK migration), Phase 2.1 (ACP+A2A protocol compliance), Phase 11.4 (Architect ReAct UI + SSE panel).
