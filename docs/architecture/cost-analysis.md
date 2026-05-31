# Cost analysis

Honest dollar math for three deployment tiers. AWS list prices, us-east-1, on-demand. Discount for committed use is your conversation with AWS.

## Tier 1 — Workshop / pilot stack (the AutoCon5 demo)

**Hardware footprint:** one `t3.xlarge` or `m6i.2xlarge` for the agent + UI stack; the LLM endpoints run on separate GPU boxes (`g5.xlarge` x 3 for the specialist tier + `g5.2xlarge` for the architect).

| Component | Instance | $/hr | Notes |
|---|---|---:|---|
| App box (agents + MCP + UI + Redis + Neo4j) | `m6i.2xlarge` | $0.384 | 8 vCPU, 32 GB RAM |
| Stability LLM endpoint | `g5.xlarge` | $1.006 | Qwen2.5-7B in vLLM 4-bit |
| Troubleshooting LLM endpoint | `g5.xlarge` | $1.006 | CodeLlama-7B in vLLM 4-bit |
| Security LLM endpoint | `g5.xlarge` | $1.006 | Mistral-7B in vLLM 4-bit |
| Architect LLM endpoint | `g5.2xlarge` | $1.624 | Qwen2.5-7B + va-v2 LoRA, bf16 |
| **Total / hr** | | **$5.026** | |
| **Total / day at 14hr Berlin-business uptime** | | **$70.36** | autocon5 schedule stops at 22:00 |
| **Total / month at workshop schedule** | | **~$2,100** | (excludes EBS, transfer, idle hrs) |

Sufficient for: workshop demos, ~3-fabric pilot in dry-run mode, internal evaluation.

## Tier 2 — Production single-fabric

Replaces the GPU boxes with self-hosted vLLM endpoints (or third-party APIs); adds the V3 persistent shadow box.

| Component | Instance | $/hr | Notes |
|---|---|---:|---|
| App box | `m6i.4xlarge` | $0.768 | Bigger for production load |
| 4 LLM endpoints (4x g5.xlarge) | various | $4.030 | Same as Tier 1 |
| **V3 persistent shadow host** | `m6i.8xlarge` | $1.536 | 32 vCPU, 128 GB RAM, hosts vrnetlab NX9K + cEOS + SRL twins |
| Neo4j + TimescaleDB (managed) | RDS-equivalent | $0.300 | Or self-hosted on app box |
| **Total / hr** | | **$6.634** | |
| **Total / day at 24×7** | | **$159.22** | |
| **Total / month** | | **~$4,840** | |

Per-incident validation cost: ~$0.006 at 10s avg validate time (see [scalability-v3.md](scalability-v3.md)).

## Tier 3 — Multi-fabric / multi-tenant

Multiple app boxes (one per region) + shared V3 shadow + centralized KG.

| Component | Instance | $/hr | Notes |
|---|---|---:|---|
| App boxes (3 regions) | `m6i.4xlarge` × 3 | $2.304 | |
| LLM endpoints (shared) | 4× g5.xlarge | $4.030 | One pool, all regions |
| V3 shadow (1, shared) | `m6i.8xlarge` | $1.536 | |
| Or V3 dedicated per region | × 3 | $4.608 | Trade $/hr for latency |
| KG cluster | RDS managed | $0.800 | Multi-AZ Neo4j |
| **Single-region 1 V3** | | **$8.670** | $6,242/mo |
| **3-region 3 V3** | | **$11.742** | $8,454/mo |

## What customers actually pay

The pilot is **$50K-$120K fixed-price** (see [pilot-program.md](../pitch/pilot-program.md)) — covers all AWS costs + engineering time during the 8-week pilot window. Customers do not see an AWS bill until production go-live (Tier 2 / Tier 3 above).

## Where the cost comes from

- ~62% LLM endpoints (predictable; scales with fabric size + incident rate)
- ~24% V3 persistent shadow host (fixed; covers any fabric size up to ~50 nodes per host)
- ~6% app box (low; mostly idle)
- ~8% managed KG (low; only relevant at Tier 2+)

## What can be cut

- **LLM endpoints → third-party APIs.** Cut ~$4/hr; trade for per-token cost. Break-even ~50 incidents/day.
- **V3 shadow → on-demand**. Spin up only when needed; ~$0 idle but +5-10 min cold-start. Trade $1.50/hr fixed for per-incident bursts.
- **Multi-AZ KG → single-AZ.** Cut $0.50/hr; trade for ~5 min recovery time after AZ event.

## Trust math

Customers can run the numbers in our docs against their AWS console. The architecture (single-fabric Tier 2) is reproducible without us — by design. Lock-in comes from the platform's accumulated KG state + customer policies, not from infrastructure complexity.
