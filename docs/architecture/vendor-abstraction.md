# Vendor-Agnostic, Topology-Adaptable NetOps Architecture

**Project:** agentic-netops-mvp → multi-vendor, multi-topology platform
**Status:** DESIGN (not yet built) · synthesized from a 4-architect review
**Date:** 2026-05-27
**Author:** Architecture working session (Claude + code-architect agents)

---

## 1. Executive summary

The goal is to evolve the current **single-vendor (Cisco IOS-XE), single-topology (3 fixed routers)** demo into a **vendor-agnostic, topology-adaptable platform** that serves different customers — VXLAN EVPN fabrics, SD-WAN, mixed Cisco + Arista — from the same codebase. **Vendor-agnosticism is the product moat.**

The whole design reduces to one principle:

> **Push vendor-specificity to thin edges; keep a vendor-neutral core. Make "add a vendor" and "add a topology/technology" plugin registrations — never edits to the core.**

Two findings make this tractable:

1. **It is ~85–90% reuse.** Almost none of the existing domain logic is vendor-specific — only its *edges* are. The 42 diagnostic trees, 72 scenarios, the specialist reasoning graph, the determinism/memory/guardrail machinery all become the *implementation behind* four thin new interfaces. Cisco-specificity lives in exactly **three places**: the syslog mnemonic table, the CLI parsers, and the fix-command strings.
2. **The architecture is four plugin types around one neutral data model:** `VendorAdapter`, `DomainPack`, `InventorySource`, and the **Neutral Model** that they all speak.

---

## 2. Goals & decisions

| # | Decision | Rationale |
|---|---|---|
| G1 | Vendor-agnostic is the moat | Differentiator; the diagnostic knowledge was never truly vendor-bound |
| G2 | **Normalize at the MCP layer** via structured APIs (Arista eAPI, NX-OS gNMI/OpenConfig) | Normalize once per capability, not per show-command per vendor |
| G3 | **Vendor-abstraction first** (Arista on OSPF/BGP before any EVPN) | De-risk by isolating one hard axis at a time |
| G4 | Capability-based dispatch — core never branches on a vendor string | The structural guarantee that protects the moat |
| G5 | Maximize reuse, minimize change | Wrap/relocate/load-as-is over rewrite |
| G6 | Topology + intent come from a per-customer source of truth (NetBox/Infrahub) | "Different customer" = different tenant, not different code |

---

## 3. Current state — what is locked today

Three, and only three, surfaces are vendor/topology-bound:

| Surface | File | Lock |
|---|---|---|
| **Device access** | `mcp-server/tools/real_device.py` | hardcodes scrapli `IOSXEDriver` + `device_type: cisco_iosxe` |
| **Syslog mnemonics** | `agents/team_leader/workflow/triage_rules.py` (42 rules) + `syslog-server/syslog_server.py` | Cisco-IOS mnemonic regexes; static `map_ip_to_device` IP→name dict |
| **Fix commands** | scenario YAML `fix_patterns[].commands`; agent prompts | IOS-XE CLI strings |
| **Topology seed** | `neo4j-kg/sync.py` `STATIC_DEVICES` | fixed 3-router list |

Everything else — the 42 diagnostic trees (which branch on *tool names*, not vendor strings), the scenario engine, the specialist Context→Recall→Analyze→Validate graph, the digital twin logic, NetBox client (already returns `platform`/`role`/`primary_ip4`), the §9.1 determinism cache, the episodic memory, the input guardrail — is **already vendor-neutral or trivially neutralizable**.

---

## 4. Target architecture

```
 InventorySource(tenant)          ████████ NEUTRAL CORE ████████            VendorAdapter (per platform)
 (wraps netbox_client)      ┌──────────────────────────────────────┐    ┌──────────────────────────────┐
   devices / topology /     │  NEUTRAL MODEL (the spine):            │    │ CiscoIOSXEAdapter             │
   intent (VRF/VNI/SD-WAN)  │    NeutralState · NeutralEvent ·       │    │  (wraps 37 parsers+IOSXEDriver)│
   + capability derivation ─┼──▶ NeutralChange · Capability taxonomy │◀──▶│ CiscoNXOSAdapter (gNMI)        │
   + tenant scoping         │    fault_class = extensible string     │    │ AristaEOSAdapter (pyeapi/JSON) │
                            │                                        │    │  collect()→NeutralState        │
 NetBox / Infrahub ────────▶│   ┌────────── DOMAIN PACKS ──────────┐ │    │  render(NeutralChange)→cmds    │
 (per-tenant source of      │   │ routing · vxlan-evpn · sd-wan ·  │ │    │  syslog_to_event()→NeutralEvent│
  truth; Infrahub adds      │   │ security                         │ │    │  capabilities()                │
  branchable intent)        │   │ (your 42 trees + 72 YAMLs as-is) │ │    └──────────────────────────────┘
                            │   └──────────────────────────────────┘ │      ▲ registry keyed on NetBox `platform`
                            │   Dispatch on CAPABILITY + fault_class  │
                            │   (never on vendor)                     │
                            │   KG · Digital Twin · Memory · Cache:   │
                            │   tenant-scoped, capability-driven      │
                            └──────────────────────────────────────┘
```

**A "different customer" = a different NetBox tenant + enabled domain packs. The core code does not change per customer.**

---

## 5. The Neutral Model (the spine)

The canonical, vendor-neutral vocabulary that all plugins speak. **Phasing:** Phase A ships a thin `NeutralState{raw: dict}` wrapper (the existing 37 parsers already emit the right dicts → *zero behavior change*); typed per-capability schemas are introduced incrementally.

### NeutralState (examples; derived from existing schemas)
```
NeutralInterface   ← InterfaceStatus     (rename status→oper_status; speed→speed_mbps:int; +vrf, +encapsulation)
NeutralOSPFNeighbor← OSPFNeighbor         (+area_id, +process_id; dead_time str→seconds:int)
NeutralBGPSession  ← bgp_parser dict      (per-AFI record incl. l2vpn-evpn; +route_distinguisher)
NeutralVXLANVTEP   ← evpn_parser dict     (evis[]{evi,vni,rd,rt_*,type2/3/5_count}; +local_vtep_ip)
NeutralL2Route     ← (new)                (vni, mac, ip, next_hop, route_type, rd)
NeutralRoute       ← RouteEntry           (+vrf; multi-next-hop list for ECMP; AD from adapter)
NeutralVRF, NeutralSDWANTunnel ← sdwan_parser dict
```
Every schema carries `ext: dict = {}` — the **only** field a DomainPack may write. New entity types (EVPN, SD-WAN) are DomainPack-owned models; the core never imports them. Align field names to OpenConfig/YANG paths where natural.

### NeutralEvent (what syslog/telemetry normalizes to)
```
{ event_id, timestamp, device_id, severity, fault_class (str),
  entity_type, entity_id, description, raw, vendor_mnemonic, source }
```
Per-vendor mnemonics map *into* this inside the adapter. `fault_class` is an **extensible string** (e.g. `ospf_adjacency_change`, `evpn_type2_mac_inconsistency`) — not a closed enum — so a new domain adds fault classes without core changes.

### NeutralChange (vendor-neutral remediation intent)
```
{ change_id, intent (e.g. NO_SHUTDOWN_INTERFACE / RESET_BGP_SESSION / SET_VXLAN_VNI),
  device_id, entity_type, entity_id, parameters, dry_run, approval_id }
```
Adapters render this to per-vendor CLI/NETCONF at execution time — the pack never emits CLI.

### Capability taxonomy (two distinct concepts — keep them apart)
- **DeviceCapability** — dotted strings describing what a device *is/does*: `routing.bgp`, `overlay.vxlan-evpn`, `sdwan.tunnels`, `l2.vlan`, `security.acl`. Derived from NetBox; drives **dispatch**.
- **Collector / Query** — what `adapter.collect()` *fetches*. The adapter advertises which collectors it supports.

---

## 6. The four contracts

### 6.1 VendorAdapter  (`mcp-server/adapters/`)
```python
class VendorAdapter(ABC):
    def connect(self) / disconnect(self)
    def collect(self, capability) -> NeutralState
    def render(self, change: NeutralChange) -> list[str]      # vendor CLI/NETCONF
    def apply(self, commands) -> dict
    def syslog_to_event(self, raw: str) -> NeutralEvent | None
    def capabilities(self) -> frozenset[Capability]
```
Registry keyed on NetBox `platform.name`. `AdapterFactory.for_device(id)` is the single dispatch call. Mock mode → a single `MockAdapter` (preserves `mock_device.apply_config` intent simulation).

### 6.2 DomainPack  (loads existing YAML unchanged)
```python
class DomainPack(Protocol):
    domain_id: str ; fault_classes: frozenset[str]
    required_capabilities: frozenset[str]
    def evidence_plan(fault, event) -> list[CapabilityQuery]   # ← replaces context_node tool calls
    def diagnostic_tree(fault) -> DiagTree                      # ← the existing YAML tree
    def analyze_prompt(fault, context) -> str                  # ← replaces _build_grounded_prompt
    def fix_templates(fault) -> list[NeutralFixTemplate]       # NEUTRAL, no CLI
    def validate_fix_semantics(fault, fix) -> FixCorrectness
```
A `YamlDomainPack` loader reads the existing trees/scenarios. A new domain = drop YAML + register; **zero core code**.

### 6.3 InventorySource  (wraps `netbox_client.py`)
```python
class InventorySource(Protocol):
    def devices(tenant) -> list[DeviceRecord]
    def device(tenant, id) -> DeviceRecord     # incl. derived capabilities
    def topology(tenant) -> TopologyGraph
    def intent(tenant) -> IntentView           # VRFs, VNIs, SD-WAN policies
```
Capabilities derived from `platform`/`role`/`device_type`/`custom_fields`. **Infrahub** is a drop-in alternative that adds branchable intent (propose a change on a branch, validate, merge) — same interface, different backend.

### 6.4 Neutral Model — §5 (the shared data the other three speak).

---

## 7. Capability-based dispatch
```
classify(NeutralEvent) -> (domain, specialist)           # ~20-row table on fault_class, no vendor strings
pack = registry.get(domain)
missing = pack.required_capabilities - device.capabilities
if missing: degrade (narrower pack or investigate-only)
→ route to specialist queue with pack_id
```
Specialists stay 4; the same Context→Recall→Analyze→Validate graph runs with a `DomainPack` injected at dispatch. Determinism, the §9.1 cache, the episodic recall, and the input guardrail are all **preserved unchanged** (the cache key gains `pack_id` + `tenant`).

---

## 8. Multi-tenancy (customer isolation)
| Store | Change |
|---|---|
| Neo4j KG | add `tenant` to node-key + every Cypher `WHERE d.tenant = $tenant` |
| TimescaleDB `incident_outcomes` | add `tenant_id` column + index; recall query filters on it |
| §9.1 answer cache | prepend `tenant_id` to the hash basis |
| Redis queues | `tenant_id` in payload (phase 1) → per-tenant queues (phase 2) |
New customer onboarding = create a NetBox tenant, assign devices (platform/role/custom_fields), set `config_context` intent, add the tenant slug to `NETOPS_TENANTS`. **No Python changes.**

---

## 9. Digital twin generalization
A `TwinBackend` protocol (`deploy/apply_change/collect_evidence/teardown`):
- **FRR backend** — `ios_to_frr` becomes `neutral_to_frr` (input = NeutralFixTemplate). Neutral/OSPF + FRR-EVPN.
- **cEOS backend** — Arista-accurate, containerlab-native.
- **SD-WAN** — no faithful emulator → `requires_live_validation: true`; falls back to Layer-1 graph-sim + KG checks (the existing "unverified" path), surfaced honestly on the approval card.

---

## 10. CHANGES REQUIRED

### 10.1 New files (the four thin layers + adapters)
| File | Purpose |
|---|---|
| `mcp-server/adapters/base.py` | `VendorAdapter` ABC + `Capability` |
| `mcp-server/adapters/registry.py`, `factory.py` | platform→adapter registry; `AdapterFactory.for_device()` |
| `mcp-server/adapters/cisco_iosxe.py` | wraps `RealDevice`/`IOSXEDriver` + the 37 parse statics |
| `mcp-server/adapters/mock_adapter.py` | wraps `MockDevice` |
| `mcp-server/adapters/arista_eos.py` | pyeapi/eAPI (JSON-native), ~120 lines |
| `core/neutral/model.py` | NeutralState / NeutralEvent / NeutralChange / Capability |
| `core/triage/neutral_classifier.py` | fault_class → (domain, specialist) routing table |
| `core/packs/registry.py`, `yaml_pack.py` | DomainPack loader for existing YAML |
| `mcp-server/tools/inventory_source.py` | `InventorySource` facade over netbox_client |
| `mcp-server/tools/capability_engine.py` | derive capabilities from NetBox fields |
| `twin/backends/{base,frr,ceos}.py` | `TwinBackend` protocol + backends |

### 10.2 Files modified (wrap/relocate — minimal diff)
| File | Change | Effort |
|---|---|---|
| `mcp-server/tools/*_parser.py` (×37) | reroute the `USE_REAL_DEVICES` branch through `AdapterFactory` | ~8 lines each |
| `mcp-server/main.py` | move syslog-mnemonic mapping into the Cisco adapter; scope `config_writer` idempotency key by vendor | small |
| `agents/team_leader/workflow/triage_rules.py` | **relocate verbatim** into `adapters/cisco_iosxe` syslog parser; emit `fault_class` | cut-paste |
| `agents/*/workflow/context_node.py` | replace explicit MCP calls with `pack.evidence_plan()` | ~4h each |
| `agents/*/workflow/analyze_node.py` | `_build_grounded_prompt` → `pack.analyze_prompt()` (keep guardrail + cache) | ~3h each |
| `agents/shared/validate_node.py` | `validate_fix_correctness` → `pack.validate_fix_semantics()` | ~3h |
| `agents/shared/graph_simulator.py` | OSPF-only topology → capability topology; `_apply_change` → NeutralChange.action | ~3h |
| `agents/shared/ios_to_frr.py`, `containerlab_manager.py` | refactor into FRR `TwinBackend` (logic intact) | ~5h |
| `agents/shared/llm_client.py` | cache-key basis += `tenant_id` + `pack_id` | small |
| `agents/shared/recall_node.py`, `outcome-tracker/tracker.py` | add `tenant_id` to query + `incident_outcomes` | small |
| `syslog-server/syslog_server.py` | `map_ip_to_device` → NetBox-backed Redis lookup; inject `tenant_id` | small |
| `neo4j-kg/sync.py` | `STATIC_DEVICES` → `seed_from_netbox(tenant)`; tenant on node-key | medium |
| `docker-compose.yml` | add `arista`/adapter deps (pyeapi), `NETOPS_TENANTS` env | small |

### 10.3 Unchanged (reused as-is)
- The **42 diagnostic trees** + **72 scenario YAMLs** (load via `YamlDomainPack`).
- The specialist **Context→Recall→Analyze→Validate** graph (DomainPack injected).
- The §9.1 determinism machinery, episodic memory, input guardrail, denylist, HITL.
- NetBox itself; the MCP HTTP route surface (agents call the same URLs).

---

## 11. Migration plan (each phase independently shippable)

| Phase | Scope | Gate |
|---|---|---|
| **A** | Neutral seam + adapter framework; wrap Cisco; **zero new behavior** | Regression vs the gpt-4o-mini determinism baseline (same answers) |
| **B** | `AristaEOSAdapter` on OSPF/BGP + small dual-vendor lab | **Moat proof:** same incident, two vendors, one neutral diagnosis, vendor-rendered fix |
| **C** | NetBox-driven topology + multi-tenancy (kill hardcodes) | New tenant onboards with zero code |
| **D** | VXLAN EVPN DomainPack on both vendors | Dual-vendor EVPN lab incidents resolved |
| **E** | SD-WAN DomainPack | Proves domain-plugin generality on a different topology |

---

## 12. Risks & mitigations
| Risk | Mitigation |
|---|---|
| Cache-key change (tenant+pack_id) **re-bases determinism** | Re-record the gpt-4o-mini baseline after Phase A (plan for it) |
| Vendor leaks back into the core → moat cracks | `ext`-bag discipline: packs declare `ext` keys in a manifest; CI rejects undeclared reads + any vendor-string branch in core |
| KG tenant-bleed | Audit every `MATCH (d:Device)` for a tenant filter |
| Capability cold-start | Until inventory populated, dispatch on fault_class alone (today's behavior) |
| NetBox/Infrahub unavailable | Redis last-known-good IP map; adapter platform fallback to `cisco_iosxe` |
| EVPN/SD-WAN twin coverage gaps | `requires_live_validation` degrades to Layer-1 + KG checks; card states validation depth honestly |
| `incident_outcomes` live ALTER | `ADD COLUMN ... NULL` is lock-free; recall handles NULL-tenant history then backfill |

---

## 13. Why it stays maintainable
- **Thin vendor edges, neutral core** — vendor-ness confined to adapters.
- **Closed core, open extension** — new tech adds an `ext` namespace + a domain pack; never a core edit.
- **YAML domain packs** — CCIE-grade knowledge editable without code.
- **One registry, one dispatch table, one factory** — adding a vendor = "write an adapter + a syslog map"; adding a topology/technology = "drop in YAML + a capability." Neither touches the core.

---

*End of document.*
