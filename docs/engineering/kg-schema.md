# Digital Twin & Knowledge Graph — How They Work, and How Far They Scale

**Audience:** plain-language explanation, but precise. Every mechanism box cites the exact code.
**Date:** 2026-05-25 · **Method:** three agents (two code-explorers + one reviewer) mapped the code in parallel; every load-bearing claim was then verified directly against the source.

---

## The 30-second version

- **The Knowledge Graph (KG)** is a *live diagram of the network* kept in a graph database (Neo4j). Routers, ports, OSPF neighbors and routes are stored as dots, and the connections between them as lines you can walk. It's the system's "memory of what the network looks like."
- **The Digital Twin (DT)** is a *what-if sandbox*. Before a human approves a fix, the system tries that fix on a model of the network to see whether it would break anything. There are **two models**: a fast rough sketch (graph math) and a slow realistic one (actual mini-routers in containers).
- **Scale today:** this is an excellent **demo-scale prototype** — built for ~3 routers, one incident at a time, one user. It is not yet a production system, and two of its limits can be hit *at the workshop* without adding a single device.

---

## Part A — How the Digital Twin works

### The idea, in simple terms
When a specialist agent proposes a fix (e.g. "run `no shutdown` on Gi0/0"), we don't want to blindly trust the AI. So the DT acts like a **flight simulator for network changes**: it rehearses the change on a copy of the network and reports whether things stay healthy. Only then does the change reach a human for approval.

### The gate has 4 steps (and picks which simulator to use)
Every fix goes through `ValidateNode`, which runs up to four passes. A **confidence score** decides how much rehearsal is warranted:

| Step | What happens | In plain terms |
|------|--------------|----------------|
| 1. Confidence score | Compute a 0–1 "gate score" from risk signals | "How worried should I be about this fix?" |
| 2. Fault-correctness | Regex check: does the fix match the fault? | "Does this fix even address the problem?" |
| 3. **DT Layer 1** (graph sim) | networkx connectivity check | The *quick sketch* model |
| 4. **DT Layer 2** (containerlab) | Real FRR routers in containers | The *realistic miniature* model |

The gate score routes the fix: **below 0.55** → skip simulation, send straight to a human; **0.55–0.85** → run the quick Layer-1 sketch; **0.85 and above** → run the heavy, realistic Layer-2 twin. Truly dangerous commands (`write erase`, `reload`, `format`, `erase startup`) are **blocked instantly** before any of this.

> **Precise:** `validate_node.py:157` orchestrates the passes. Gate score = `clamp(1.0 − objective·0.6 + llm_confidence·0.4)` with weights change_type_risk 0.30 / blast_radius 0.25 / historical_accuracy 0.25 / context_clarity 0.10 / symptom_match 0.10 (`confidence_scorer.py:99`). Bands + `BLOCKED_PATTERNS` at `confidence_scorer.py:166` & `:251`.

### Layer 1 — the quick sketch (graph_simulator.py)
1. It pulls the list of healthy OSPF neighbor links from the KG and **draws a connect-the-dots graph**: each router is a dot, each *FULL* OSPF adjacency is a line (`_build_graph`).
2. It **applies the proposed change to the sketch**: if the command is `shutdown` or `no router ospf`, it *erases the whole router* from the graph. (`no shutdown` and everything else leave the sketch unchanged.)
3. It asks one question: **"is the network still all in one piece?"** (`nx.is_connected`), and computes shortest paths between every pair of routers.
4. Separately, it gives the command a **risk score** from a pattern table, and declares: `passed = still-connected AND risk < 0.75`.

> **Precise:** `graph_simulator.py:93` (`simulate`), graph build `:220`, change applied `:249`, connectivity `:207`. Risk table: `write erase/reload`=0.95, `no router ospf/bgp`=0.85, `no ip route`=0.75, `shutdown`=0.70, `no shutdown`=0.20, default 0.40. Pass formula `:126`.

**Honest limitations of Layer 1:**
- It's **coarse**: a `shutdown` on one port erases the *entire router* from the model, even if only one link is affected — a deliberate worst-case, but not realistic.
- If the KG has **no neighbor data** (true for ~30–60s after boot, or always in some mock setups), it returns *"reachability OK"* **vacuously** — it passes without actually simulating (`graph_simulator.py:193`). This is the single most important caveat for a demo.

### Layer 2 — the realistic miniature (containerlab_manager.py)
Only runs for high-confidence fixes (≥0.85). It builds a **real tiny network**: three FRR router containers wired in a triangle, then:
1. Translates the IOS fix to FRR syntax (`ios_to_frr.translate()` — the unit suite we already built).
2. `containerlab deploy` the 3-router topology → **wait 12s** for OSPF to converge.
3. Apply the fix to the affected router via `docker exec … vtysh` → **wait 6s** to re-converge.
4. Run three real checks: OSPF neighbors are up, an OSPF route exists, and pings succeed across the triangle.
5. Tear the whole thing down.

> **Precise:** `containerlab_manager.py:123` (`validate`), waits `_CONVERGENCE_WAIT=12` / `_RECONVERGENCE_WAIT=6` (`:69`), 3 containers `frrouting/frr:10.1.1` (`containerlab/topology.yml`), checks at `:213/:219/:225`, teardown in `finally :254`.

**Honest limitations of Layer 2:**
- It takes **~25–35 seconds per run** and uses real Docker. On the demo host (no `containerlab` binary installed) it **self-skips** entirely — so in the mock demo, Layer 2 never actually runs.
- The topology is **hard-coded to exactly 3 routers**; it can't model the real fix's surroundings.

### The most important DT caveat of all
**The DT is advisory — it does not block approval.** The result is shown on the approval card, but the Approve button is never disabled based on it (`ui/app.py:869`). The architecture *looks* like a safety gate; the UI does not *enforce* one.

---

## Part B — How the Knowledge Graph works

### The idea, in simple terms
The KG is a **graph database (Neo4j)** holding a structured picture of the network. Think of it as a wiki where every **router, port, neighbor, and route is a page**, and the **links between pages are real network relationships** you can traverse. Agents read it to get "context" about a device before reasoning, and the DT reads it to know the topology.

### What's stored (the schema)
Four kinds of "dots" (nodes) and three kinds of "lines" (relationships):

| Node | Key (how duplicates are prevented) | Holds |
|------|-----------------------------------|-------|
| `Device` | `device_id` (unique) | hostname, mgmt_ip, ospf_router_id, config_hash, … |
| `Interface` | `(device_id, name)` | ip, admin/oper status, errors, mtu, … |
| `OSPFNeighbor` | `(local_device, neighbor_id)` | state (FULL/…), area, cost, timers |
| `Route` | `(device_id, prefix, protocol)` | next_hop, metric, admin_distance |

Lines: `(Device)-[:HAS_INTERFACE]->(Interface)`, `(Device)-[:OSPF_NEIGHBOR]->(OSPFNeighbor)`, `(Device)-[:HAS_ROUTE]->(Route)`.

> **Precise:** schema/constraints `neo4j-kg/sync.py:117` and `kg_client.py:229`. Indexed: `Device.device_id` (unique), `Device.mgmt_ip`. **`Route.prefix` is NOT indexed** (matters for scale).

### How the KG gets filled (neo4j-sync)
There is exactly **one writer**: the `neo4j-sync` container.
- **On first boot:** if the graph is empty, it **seeds a hard-coded 3-router triangle** (router-1/2/3 = 1.1.1.1 / 2.2.2.2 / 3.3.3.3, each with interfaces, two FULL OSPF neighbors, three routes).
- **In mock mode (`USE_REAL_DEVICES=false`, the demo default):** after seeding, it does **nothing but print a heartbeat every 30s** — the picture never changes.
- **In real mode:** every 30s it loops the device list **serially**, calling two MCP tools per device (`ospf_parser`, `interface_parser`) and updating those nodes. **Routes are never refreshed** after seeding, and the device list is a **hard-coded array** (`STATIC_DEVICES`) — adding a router means editing code.

> **Precise:** seed `sync.py:32` & `:305`, mock heartbeat `:329`, real-mode serial poll `:323`, MERGE upserts `:139/:157/:173/:193`.

### How the KG is read (kg_client.py)
- **`get_device_context(device_id)`** — the main one. Grabs a device plus its interfaces, neighbors, and routes in one query. This is what each specialist calls to "learn about the device."
- **`get_all_ospf_neighbors()`** — feeds the DT's Layer-1 sketch.
- **`get_blast_radius(prefix)`** — *despite the name, this is a flat one-hop count*: "how many devices have a route to this exact prefix?" It does **not** walk the topology or propagate impact. (`kg_client.py:113`)
- **`get_topology()` / `get_topology_summary()`** — **dead code**: defined but never called anywhere.

### Freshness model (important)
The KG is a **derived, eventually-consistent copy** of device state — not the source of truth (the devices/MCP are). In real mode it lags by up to ~30s; in mock mode it's a frozen snapshot. Every read **fails soft**: if Neo4j is down or empty, the reader gets a safe default (e.g. blast radius → `{affected_devices:1}`) instead of an error. That's why the system stays up — but also why missing data is silent.

> **Note:** each specialist's context step opens its **own** short-lived `KGClient` and never closes it (`*/workflow/context_node.py`), leaking driver connections — fine at demo scale, a leak at volume.

---

## Part C — How scalable is this, really?

**Honest verdict:** a **demo-scale prototype**. It is coherent and impressive for **3 devices, one incident at a time, one user**. Every dimension below has a hard ceiling, and two of them are reachable *at the workshop* with no growth at all.

### Ranked bottlenecks (first to break → last)

| # | What breaks | Bites at | Fix effort |
|---|-------------|----------|-----------|
| 1 | **Containerlab twin corrupts itself if two run at once.** One shared `topology.yml` + config dir, **no lock**. Two simultaneous Layer-2 validations race deploy-vs-teardown. | **2 concurrent incidents** | Architectural |
| 2 | **Team-leader handles one incident at a time** + drops repeats. Single `brpop` loop; a slow LLM call (up to 120s, or 60s cold-start) stalls intake; a 60s per-device dedup window sends repeats to the dead-letter queue. | **~3 incidents/min burst** | Moderate |
| 3 | **KG sync is strictly serial.** Polls each real device one-by-one every 30s; with real SSH (~2s/device) the cycle exceeds 30s and falls behind. | **~7 real devices** | One-liner (`asyncio.gather`) |
| 4 | **KG queries lack guards.** `get_topology` pulls the whole graph (no LIMIT); `Route.prefix` is unindexed → full scan per blast-radius lookup. | **~500 devices** | Moderate (add index + LIMIT) |
| 5 | **Specialists are serial + block the event loop.** One message at a time; the **synchronous** OpenAI SDK is called inside async code, freezing that agent during each call. | **2 concurrent/specialist** | Moderate |
| 6 | **Layer-1 sketch recomputes all-pairs shortest paths** every validation, in the event loop. | **~100 devices** (CPU stall) | Moderate |
| 7 | **Can't run replicas.** Redis dedup keys, shared containerlab files, and a hard-coded device dict are all singletons — a second copy of any agent collides. | **first replica** | Architectural |
| 8 | **Cold-start freeze.** A cold HF endpoint makes the agent sleep up to 60s inline, consuming no work meanwhile. | **first cold deploy** | Moderate |

### Why two of these matter for June 9 specifically
- **#1 (containerlab):** if a demo scenario triggers two incidents that both escalate to the realistic twin, they corrupt each other. *Mitigation today:* the demo host has no containerlab installed, so Layer-2 self-skips — the risk is latent, not active, in the mock demo.
- **#2 (team-leader):** firing **3+ syslogs from one router within 60s** means only the first is processed and the rest are dropped. This is observable at workshop scale. *This is actually the intended "cascade suppression" feature* — worth narrating as a feature, but know its sharp edge.

### What "production-scale" would take (grouped)
- **Config / quick:** parallelize the sync poller (`asyncio.gather`); add a `Route.prefix` index; warm LLM endpoints at startup.
- **Moderate:** make agents process concurrently and use the **async** OpenAI client; bound/limit KG queries; replace the O(N²) reachability with targeted `has_path`.
- **Architectural:** per-validation containerlab namespaces (or an asyncio lock) so the twin can run safely/concurrently; a distributed lock for dedup; externalize the device list to the KG so replicas can scale out.

---

## Bottom line
- **KG = the network's memory** (a Neo4j picture of devices/links/routes), filled by one poller, read by agents and the DT, eventually-consistent, fails soft.
- **DT = a pre-approval rehearsal** with a fast graph sketch and a slow real-container twin, gated by a confidence score — but **advisory only**, and **largely inert in the mock demo** (Layer-2 skipped, Layer-1 often passes vacuously without KG topology).
- **Scale = demo-grade.** Beautiful for a small, sequential, single-user demo; the realistic twin and the single-consumer orchestrator are the first things to fall over, and both can be touched at workshop scale. The fixes are known and mostly moderate; only true concurrency and replica scale-out are architectural.
