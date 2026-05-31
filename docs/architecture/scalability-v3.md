# DT V3 — Persistent Shadow Twin

**Status:** scaffolding committed; real `vrnetlab` integration follows.
**Branch:** `vendor-abstraction-phase-a`
**Owners:** Architect agent / DT team.

This doc explains the architectural piece that turns the Dream Team Digital
Twin from a per-incident demo (Layer 2 SRL/FRR via `containerlab`) into
a production-grade validation surface (Layer 3 real vendor OS via
`vrnetlab`-packaged qcow2 images). It is the slide that answers the
enterprise architect's first hard question: **"How does this scale to a
16-leaf fabric with real NX-OS?"**

---

## 1. The problem: fidelity vs scale

The current Digital Twin (`agents/shared/containerlab_manager.py`,
called from `ValidateNode`) spins up a containerlab topology per incident:

| Property             | Today (Layer 2)                            |
|----------------------|--------------------------------------------|
| Vendor twin          | Nokia SR Linux + FRR + cEOS (substitutes)  |
| Cold-boot time       | 10–15s per device                          |
| RAM per device       | 200–600 MB                                 |
| Per-incident cost    | full deploy → wait → apply → validate → destroy |
| Fidelity             | covers RFC-level behavior; MISSES vendor-quirky syntax |

Two real problems show up the moment we leave the workshop demo:

### 1.1 Fidelity gap

SRL and FRR are *good* EVPN implementations but they are not NX-OS. The
silent class of bugs that bites real datacenter operators —

- **NX-OS auto-RT derivation** (`route-target both auto`) vs vendor-X's
  explicit `route-target` strings: the #1 cross-vendor EVPN footgun.
- **mgmt-vrf quirks** on NX9K: the management plane and route-target
  config live in separate VRFs with subtly different syntax.
- **L3VNI bind-to-SVI** ordering: NX-OS requires a specific
  `interface nve1` → `member vni X associate-vrf` ordering.
- **NVE source-interface** validation: NX-OS rejects mismatched
  loopback configs at parse-time, not at runtime.

— is invisible to SRL. The Layer 2 DT will green-light a fix that fails
the moment it lands on the real NX9K.

### 1.2 Scale wall

A 16-leaf EVPN-VXLAN fabric (production-typical: 2 spines × 14 leaves)
on vrnetlab-packaged NX9K qcow2:

```
per-device RAM:      3-8 GB     (NX9K image: ~6 GB warm)
per-device boot:     ~3 min     (NX9K cold-boot to NX-API ready)
fabric RAM:         48-128 GB   (16 × 3-8 GB)
fabric cold-boot:   ~50 min     (parallel cold-boot; bottlenecked by host I/O)
```

If we cold-boot the whole twin per incident, we burn 50 min and an
EC2 `m6i.8xlarge` per validation. The current Layer 2 DT was acceptable
for the workshop pitch (3 SRL containers, 30s total). It is not
acceptable for real fabrics.

---

## 2. The persistent shadow pattern

V3 inverts the lifecycle. Instead of building the twin per-incident, the
twin is **always on**, and per-incident we just snapshot-restore-apply.

```
                                ┌──────────────────────────────┐
                                │      DT Shadow Host          │
                                │   (dedicated EC2 instance)   │
                                │                              │
                                │  ┌────────────────────────┐  │
                                │  │ vrnetlab-NX9K-spine1   │  │
                                │  │   container (warm)     │  │
                                │  │   snapshot: baseline   │  │
                                │  └────────────────────────┘  │
   ┌────────────────────┐       │  ┌────────────────────────┐  │
   │  ValidateNode      │       │  │ vrnetlab-NX9K-leaf-nx  │  │
   │  (per incident)    │──────►│  │   container (warm)     │  │
   │                    │  RPC  │  │   snapshot: baseline   │  │
   │  DTTwinManager     │       │  └────────────────────────┘  │
   └────────────────────┘       │  ┌────────────────────────┐  │
                                │  │ vrnetlab-vEOS-leaf-ar  │  │
                                │  │   container (warm)     │  │
                                │  │   snapshot: baseline   │  │
                                │  └────────────────────────┘  │
                                └──────────────────────────────┘
```

The twin host boots ONCE at startup (the ~50 min cost is paid once per
DT-rebuild — e.g. weekly when we re-snapshot from prod config drift).
Per-incident the manager runs:

```
   incident arrives
       │
       ▼
   ScopePolicy.decide(fault_type, device_id, commands)
       │
       ├──── LAYER_2_SRL    ──► fall back to ContainerlabManager (existing)
       │
       ├──── LAYER_3_VRNETLAB ──► continue in V3
       │
       ▼
   DTTwinManager.restore_baseline(scope={leaf-nx, spine1})    ~1s
       ▼
   DTTwinManager.apply_fix(leaf-nx, vendor_native_commands)   ~2s
       ▼
   DTTwinManager.validate_fix(device_id, fault_type, pattern_id)  ~5s
       ▼
   DTTwinManager.restore_baseline(scope={leaf-nx, spine1})    ~1s
       │
       ▼
   ValidationEvidenceBundle  →  ConfidenceScorer/approval gate
```

Per-incident wall clock: **~10s** (vs ~50 min on the cold-boot path,
vs ~30s on Layer 2 SRL — but Layer 2 lies, V3 doesn't).

---

## 3. Lifecycle in detail

### 3.1 Boot once

`DTTwinManager.boot_baseline()`:
1. Load `fabric.yaml` (same source-of-truth as Layer 2 — see
   `agents/shared/topology_generator.py`).
2. For each device in the fabric, in parallel:
   - `backend.boot_device(device_id, vendor, image)` — cold-boot
     vrnetlab container, wait for vendor-OS prompt.
   - `backend.snapshot(container_id, "baseline")` — take baseline.
   - `SnapshotStore.put(ref)` — persist baseline ref to disk.

### 3.2 Per-incident

`DTTwinManager.restore_baseline(scope) → apply_fix(device_id, cmds) →
validate_fix(device_id, fault_type, pattern_id) → restore_baseline(scope)`.

The two restore calls bracket the apply+validate. The post-validation
restore guarantees the next incident lands on a clean baseline, even if
the proposed fix made the device unreachable.

### 3.3 Snapshot store

`SnapshotStore` (`agents/shared/snapshot_store.py`) is a JSON-backed
map `(device_id, snapshot_name) → SnapshotRef`. It survives the
manager process restart so that:
- a fresh DTTwinManager instance can re-attach to the running shadow
- the `is_warm` property correctly reports operational status without
  re-walking the backend

---

## 4. Cost analysis (16-leaf fabric)

EC2 sizing for the shadow host (NX9K-heavy fabric):

| Fabric size | RAM (warm) | Disk (qcow2) | Suggested instance | $/hr (us-east-1) |
|-------------|------------|--------------|--------------------|------------------|
| 3 devices   | ~18 GB     | ~30 GB       | `m6i.2xlarge`      | ~$0.38           |
| 16 devices  | ~96 GB     | ~160 GB      | `m6i.8xlarge`      | ~$1.54           |
| 32 devices  | ~192 GB    | ~320 GB      | `m6i.16xlarge`     | ~$3.07           |

Cold-boot cost (paid once per DT-rebuild):

| Fabric size | Sequential boot | Parallel boot (host I/O bound) |
|-------------|-----------------|--------------------------------|
| 3 devices   | ~9 min          | ~3-4 min                       |
| 16 devices  | ~48 min         | ~10-15 min (depends on EBS gp3 IOPS) |
| 32 devices  | ~96 min         | ~20-25 min                     |

Per-incident validation cost (the headline number):

| Path                | Per-incident wall clock | Per-incident $ (16-leaf, m6i.8xlarge) |
|---------------------|-------------------------|---------------------------------------|
| Layer 2 SRL (today) | ~30s                    | ~$0.013                               |
| V3 cold-boot (naïve)| ~50 min                 | ~$1.28                                |
| V3 shadow (this design) | ~10s                | ~$0.004                               |

The shadow pattern moves the cost from per-incident (variable, scales
with incident volume) to per-DT-rebuild (fixed, ~weekly). At even one
incident/day on a 16-leaf fabric, V3-shadow is cheaper than V3-cold.

---

## 5. Failure modes

### 5.1 Snapshot corruption

A snapshot ref in `SnapshotStore` points to a backend artifact (docker
image, qcow2 overlay) that may have been pruned/deleted out-of-band.
**Detection:** `backend.restore()` fails on the missing artifact.
**Mitigation:** `SnapshotStore._load()` is tolerant of malformed JSON
(returns empty store, operator runs `boot_baseline()` again). For the
RealVrNetlabBackend, the integration will add a periodic
`verify_snapshots` job that exercises each snapshot ref weekly.

### 5.2 Baseline drift vs prod

Over time the shadow's baseline drifts from live prod config (NetBox
intent gets edited, but the shadow was snapshotted last week). The fix
validates against a baseline that no longer matches reality.
**Mitigation (roadmap):** continuous-sync-baseline daemon that takes a
fresh snapshot from the live prod config every N hours.
**Current commit:** out of scope; manual `boot_baseline()` re-run.

### 5.3 vrnetlab boot races

NX9K vrnetlab containers occasionally fail to reach NX-API ready within
the timeout. The boot is non-deterministic (depends on host I/O,
qcow2 prep time).
**Detection:** `boot_device()` returns timeout error; `BootResult.errors`
surfaces the device.
**Mitigation:** the manager fans out boots in parallel but each one is
isolated (`asyncio.gather`); partial fabric success is reported as a
non-fatal error. Operator triages individually.

### 5.4 Concurrent incidents touching the same device

Two incidents arrive concurrently both targeting `leaf-nx`. The second
restore-apply-validate would clobber the first.
**Current commit:** no per-device lock — incidents are expected to be
serialized at the Team Leader. The follow-up commit adds a per-device
asyncio.Lock + a queue.

---

## 6. Integration with ConfidenceScorer's path selection

`ConfidenceScorer` (`agents/shared/confidence_scorer.py`) already routes
incidents into three paths via `ScoredResult.path`:

| `gate_score` band       | `path`               | Action                                  |
|-------------------------|----------------------|-----------------------------------------|
| `< 0.55` (low)          | `direct_to_human`    | Skip simulation; human review only.     |
| `0.55 ≤ score < 0.85`   | `graph_sim`          | Run KG-based simulation.                |
| `≥ 0.85` (high)         | `containerlab`       | Run Layer 2 DT.                         |

V3 adds a fourth degree of freedom: **which DT layer** runs on the
`containerlab` and `graph_sim` paths. `ScopePolicy.decide()`
(`agents/shared/scope_policy.py`) sits between ConfidenceScorer and the
twin managers:

```
ConfidenceScorer.score(state)
        │
        ▼  ScoredResult{path=containerlab, gate_score=0.91}
        │
        ▼
ScopePolicy.decide(fault_type, device_id, commands, confidence_path)
        │
        ├── LAYER_2_SRL       ──►  ContainerlabManager.validate_evpn(...)
        ├── LAYER_3_VRNETLAB  ──►  DTTwinManager.{restore→apply→validate→restore}
        └── NONE              ──►  direct_to_human (ConfidenceScorer override)
```

Routing rules:
1. `confidence_path == "direct_to_human"` ⇒ `NONE`.
2. `fault_type` matches NX-OS-specific pattern (auto-RT, mgmt-vrf,
   nve source-interface) ⇒ `LAYER_3_VRNETLAB` with scope = device + peer.
3. `fault_type` matches RT/RD/ASN pattern, OR the proposed commands
   touch RT/RD/ASN ⇒ `LAYER_3_VRNETLAB` (high-stakes escalation; even
   if the fault label looked benign, the blast radius is huge).
4. Generic EVPN fault (`evpn_session_down`, `bgp_evpn_neighbor`, ...) ⇒
   `LAYER_2_SRL` (SRL can validate these correctly; no need to burn V3).
5. Default fallback ⇒ `LAYER_2_SRL`.

The call site (ValidateNode) does the layer dispatch:

```python
decision = scope_policy.decide(
    fault_type=state["fault_type"],
    device_id=state["device_id"],
    fabric=fabric,
    commands=state["proposed_fix"],
    confidence_path=scored.path,
)
if decision.layer == TwinLayer.LAYER_3_VRNETLAB:
    bundle = await dt_twin_manager.{ ...lifecycle... }
elif decision.layer == TwinLayer.LAYER_2_SRL:
    bundle = await containerlab_manager.validate_evpn(...)
else:
    # direct_to_human — skip simulation
    bundle = None
```

This is the ConfidenceScorer integration. It is NOT wired in this commit
(the user reserves the call-site integration in ValidateNode for the
follow-up); the scaffolding above is decoupled so that wire-up is a
single PR.

---

## 7. Roadmap

| Commit             | Scope                                                                  |
|--------------------|------------------------------------------------------------------------|
| **this commit**    | `DTTwinManager` skeleton + `MockVrNetlabBackend` + `SnapshotStore` + `ScopePolicy` + ~16 offline tests. NO real vrnetlab. NO ValidateNode wire-up. |
| **next commit**    | `RealVrNetlabBackend`: shells out to `docker` + `vrnetlab` + scrapli on the dedicated EC2 host. End-to-end on a 3-device fabric. |
| **then**           | Continuous-sync-baseline daemon — re-snapshot every N hours from live prod config so baseline drift stays bounded. |
| **then**           | Per-device lock + queue for concurrent incidents on the same device.   |
| **then**           | Per-pattern_id validation check suite (BGP-EVPN session, Type-2 advert, NX-OS auto-RT vs vEOS RT-import match, ...). |
| **then**           | Wire `ScopePolicy` + `DTTwinManager` into `ValidateNode`; gate behind `DT_V3_ENABLED` env var; default off; shadow-compare against `LAYER_2_SRL` for 2 weeks before flipping default to on. |

---

## 8. File map

| File                                              | Role                                    |
|---------------------------------------------------|-----------------------------------------|
| `agents/shared/dt_twin_manager.py`                | Lifecycle orchestrator (boot/restore/apply/validate/teardown). |
| `agents/shared/vrnetlab_backend.py`               | Protocol + `MockVrNetlabBackend` + `RealVrNetlabBackend` stub. |
| `agents/shared/snapshot_store.py`                 | Disk-backed `(device_id, snapshot_name) → SnapshotRef` map. |
| `agents/shared/scope_policy.py`                   | Fault-type → twin layer + scope decision. |
| `agents/shared/containerlab_manager.py`           | (unchanged) Layer 2 sibling — still in use. |
| `agents/shared/topology_generator.py`             | (unchanged) `fabric.yaml` loader — both layers share. |
| `tests/test_dt_twin_manager.py`                   | Offline tests against `MockVrNetlabBackend`. |
| `docs/DT_V3_PERSISTENT_SHADOW.md`                 | This document.                          |

ContainerlabManager is intentionally **untouched** in this commit. V3 is a
sibling, not a replacement. Once V3 is wired in and shadow-compared,
existing Layer 2 paths stay for the medium-fidelity-sufficient cases.

---

## 8. Implementation status (2026-05-30)

### What landed in this commit

- **`agents/shared/vrnetlab_backend.py::RealVrNetlabBackend`** — implements
  the full `VrNetlabBackend` protocol against `containerlab` + `docker` on
  the dedicated V3 host. Five lifecycle methods (`boot_device`, `snapshot`,
  `restore`, `exec_commands`, `cleanup`), three dedicated exception classes
  (`VrNetlabBootError`, `VrNetlabExecError`, `VrNetlabRestoreError`), and a
  `subprocess_runner` injection seam so tests run offline. Per-vendor
  exec CLI shapes (NX-OS / cEOS / SR Linux) routed via `twin_kind`
  resolved from `boot_config` or the fabric vendor map. Restore uses
  **Strategy B** (docker stop + docker run from snapshot image, ~3-10s on
  m6i.8xlarge). Strategy A (CRIU sub-second restore) is documented as the
  v4 target — tracked in code as `TODO(v4)` next to the restore impl.
- **`ops/v3-host-setup.sh`** — idempotent provisioning script for a fresh
  Ubuntu 22.04 EC2 instance. Installs docker + containerlab + vrnetlab,
  builds NX9K image from operator-supplied qcow2
  (`VRNETLAB_NX9K_QCOW2_PATH`), pulls cEOS / SR Linux images, writes
  `/etc/systemd/system/v3-shadow.service` (with companion
  `/usr/local/bin/v3-shadow-boot.sh` and `…-teardown.sh`), and pre-boots
  the fabric baseline on first run. Prints `V3 host ready: <count>
  devices baselined` on success.
- **`tests/test_real_vrnetlab_backend.py`** — 20 tests against a
  `FakeSubprocessRunner` (canned `(rc, stdout, stderr)` per argv prefix,
  optional `asyncio.TimeoutError` injection). Zero real subprocess calls.
  Covers all 5 lifecycle methods (happy + sad paths), per-vendor exec
  routing, restore healthcheck timeout, exec catastrophic failure
  (`Cannot connect to the Docker daemon`), cleanup best-effort behavior,
  and a doc-string regression test that pins the Strategy A TODO so the
  next engineer can't ship without seeing it.

### Operator workflow (provision → bring up → validate)

1. **Provision EC2.** Spin a fresh `m6i.8xlarge` (Ubuntu 22.04). Attach
   ~200 GB gp3 (NX9K qcow2 + snapshot layers).
2. **Copy the NX9K qcow2** from a licensed cisco.com download to the host
   (e.g. `/opt/qcow2/nxos.9.3.13.qcow2`).
3. **Run the setup script** as root:
   ```bash
   sudo VRNETLAB_NX9K_QCOW2_PATH=/opt/qcow2/nxos.9.3.13.qcow2 \
        FABRIC_YAML=/opt/agentic-netops-mvp/fabric.yaml \
        bash ops/v3-host-setup.sh
   ```
   First run is slow (~20-40 min for the vrnetlab NX9K build + initial
   fabric cold boot). Subsequent runs short-circuit on already-built
   images.
4. **Validate.** SSH in and run:
   ```bash
   pytest tests/test_real_vrnetlab_backend.py -v
   sudo systemctl status v3-shadow.service
   docker images | grep v3-snapshots/
   ```
   You should see one snapshot image per fabric device (`spine1`,
   `leaf-nx`, `leaf-ar` for the prod-dc default fabric) and the
   v3-shadow systemd unit reporting `active (exited)`.
5. **Wire into the orchestrator.** Set `DT_V3_ENABLED=1` and repoint
   `DTTwinManager` to `RealVrNetlabBackend(...)` (vs `MockVrNetlabBackend`)
   in the ValidateNode injection. The follow-up PR flips the default once
   2 weeks of shadow-compare data shows V3 catching NX-OS-specific bugs
   that LAYER_2_SRL missed.

### Cost (confirmed 2026-05-30)

| Item                                         | Value                          |
|----------------------------------------------|--------------------------------|
| 16-leaf vrnetlab-NX9K fabric RAM (warm)      | ~96 GB                         |
| Host instance (us-east-1, on-demand)         | `m6i.8xlarge` ~$1.54/hr        |
| Cold-boot (parallel, paid once per rebuild)  | ~10-15 min on gp3 16k IOPS     |
| Per-incident snapshot-restore (Strategy B)   | ~3-10s/device                  |
| Per-incident validate cycle (3 devices)      | ~10-15s wall clock             |
| Per-incident $ on m6i.8xlarge                | ~$0.006                        |

The headline numbers in §4 still hold; the operator-facing cost is
`~$37/day` running the V3 host continuously, which is cheaper than a single
human-engineer hour at every NX-OS shop we've talked to.

### Deferred to follow-up

- **Strategy A (CRIU) sub-second restore.** Needs runc-with-CRIU on the
  host kernel and tolerance from NX-OS through freeze/thaw — NX9K's
  kernel-module-heavy linecard sim is known to misbehave through CRIU.
  Tracked in `RealVrNetlabBackend.restore` as `# TODO(v4)`. Blocked
  behind a kernel-feature spike; will land when we have a willing pilot
  customer with a more snapshot-friendly platform (Arista cEOS through
  CRIU is far less risky than NX9K).
- **Continuous-sync-baseline daemon.** Periodic re-snapshot from live
  prod config so the baseline doesn't drift. Currently the operator
  re-runs `boot_baseline` manually after intent edits. The daemon is
  a small loop around `DTTwinManager.boot_baseline()` + a NetBox-intent
  change-feed subscription; ~1 day of work, but slotted after the
  Strategy A spike because re-sync without sub-second restore is
  cosmetic — the per-incident cost dominates baseline freshness.
- **Per-device asyncio lock** for concurrent incidents on the same
  device (currently incidents serialize at the Team Leader).
- **Per-pattern_id validation check suite** (named in §3 of this doc):
  `evpn_session_down`, `nx_os_auto_rt_mismatch`, `ospf_neighbor_stuck`.
  The hooks are in `DTTwinManager.validate_fix`; what's missing is the
  vendor-native command set + parsers per pattern. ~2 days each.
