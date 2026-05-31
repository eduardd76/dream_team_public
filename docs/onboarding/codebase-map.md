# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Quick Reference

**Current Phase:** 11.4 (Architect ReAct UI + Streamlit SSE panel — visual E2E passed 2026-05-19, chain `a6c6fe08-72c9-4050-83f6-287275b910dd`)
**Stack:** Docker, Python 3.11, FastAPI, Streamlit, Redis, Neo4j, TimescaleDB (verifiability)
**LLM Support:** OpenAI, Gemini, Ollama, Hugging Face (fine-tuned, dedicated endpoints)
**Deployment:** Runs on AWS EC2 `AI_Dream_Team` (instance `<INSTANCE_ID>`, region `eu-central-1`) at `<APP_HOST_IP>`
**Working Directory:** All code lives in `agentic-netops-mvp/` (not the repo root)

## Architecture: 5-Layer with Strict Protocol Compliance

```
Layer 1: Streamlit UI         (ui/app.py)
           ↕ ACP  — REST/HTTP port 8002
Layer 2: Team Leader Agent    (agents/team_leader/agent_acp_a2a.py)
           ↕ A2A  — Redis queues  a2a_*_inbox
Layer 3: Specialist Agents    (agents/stability|troubleshooting|security|virtual_architect|noc/)
           ↕ MCP  — HTTP API port 8001
Layer 4: MCP Server           (mcp-server/main.py)
           ↕ SSH/NETCONF
Layer 5: Network Devices      (3× Cisco CSR1000v in AWS EVE-NG)
```

### Critical Architecture Rules

1. **Layer 3 agents NEVER SSH directly** — all device interaction goes through MCP tools (Layer 4)
2. **UI NEVER writes directly to Redis** — all human actions go through ACP (Layer 2)
3. **All MCP tool calls require `idempotency_key`** — prevents duplicate operations (24h TTL)
4. **Team Leader owns approval/history queues** — specialists only use their A2A inboxes
5. **Protocol assignment is fixed:** Layer 1↔2 = ACP, Layer 2↔3 = A2A, Layer 3↔4 = MCP

### Redis Queue Map

| Queue | Producer | Consumer |
|-------|----------|----------|
| `incident_queue` | Syslog Server | Team Leader |
| `a2a_stability_specialist_inbox` | Team Leader | Stability Agent |
| `a2a_troubleshooting_specialist_inbox` | Team Leader | Troubleshooting Agent |
| `a2a_security_specialist_inbox` | Team Leader | Security Agent |
| `a2a_virtual_architect_inbox` | Team Leader | Virtual Architect |
| `a2a_team_leader_inbox` | All Specialists | Team Leader |
| `approval_queue` | Team Leader | UI (read-only display) |
| `approved_history` / `rejected_history` | Team Leader | UI (read-only display) |

## Key Files

**Current Production Agents (Phase 2.1 — used in docker-compose):**
- `agents/team_leader/agent_acp_a2a.py` — ACP server (port 8002) + A2A delegation
- `agents/stability/agent_a2a.py` — OSPF/BGP/EIGRP specialist (Qwen2.5-7B)
- `agents/troubleshooting/agent_a2a.py` — Interface/connectivity specialist (CodeLlama-7B)
- `agents/security/agent_a2a.py` — ACL/authentication specialist (Mistral-7B)
- `agents/virtual_architect/agent_a2a.py` — Network design specialist (Llama3-8B fine-tuned)
- `agents/noc/agent_a2a.py` — NOC aggregation agent

**Protocol & Shared Utilities:**
- `agents/shared/acp_protocol.py` — IBM ACP (Layer 1↔2)
- `agents/shared/a2a_protocol.py` — Google A2A (Layer 2↔3), message types: `TASK_DELEGATION`, `TASK_RESPONSE`, `STATUS_UPDATE`, `ESCALATION`
- `agents/shared/mcp_client.py` — MCP HTTP client (Layer 3→4)
- `agents/shared/llm_client.py` — Multi-provider LLM client with automatic OpenAI fallback
- `agents/shared/kg_client.py` — Neo4j Knowledge Graph client
- `agents/shared/confidence_scorer.py` — Confidence scoring for recommendations

**MCP Server (Layer 4):**
- `mcp-server/main.py` — FastAPI with 13 tool endpoints + idempotency checking
- `mcp-server/ssh_tunnel.py` — SSH tunnel manager through AWS jump-host `ubuntu@<JUMP_HOST_IP>` (key: `mcp-server/<JUMP_HOST_KEY>.pem`)
- `mcp-server/tools/real_device.py` — Real Cisco connections via Scrapli
- `mcp-server/tools/mock_device.py` — Simulated Cisco router for testing

**Legacy/Experimental (not used in docker-compose):**
- `agents/*/agent.py` — Phase 1 legacy (architecture violations, deprecated)
- `agents/*/agent_adk.py` — Phase 2.0 Google ADK version (partial compliance)

## Common Commands

### Build and Run

```bash
# Start all services (first time or after Dockerfile changes)
docker-compose up --build

# Start without rebuild
docker-compose up -d

# Stop all (add -v to also clear Redis data)
docker-compose down

# View logs
docker-compose logs -f [service-name]   # e.g. stability-agent, team-leader, mcp-server
```

### Testing

```bash
cd tests
pip install -r requirements.txt

pytest test_mcp_tools.py -v                              # Unit tests (MCP tools)
pytest test_stability_agent.py -v                        # Integration tests
pytest test_end_to_end.py -v -s                          # E2E (requires running services)
pytest test_mcp_tools.py::test_ospf_parser_healthy -v    # Single test
```

### Manual Testing & Debugging

```bash
# Health checks
curl http://localhost:8001/health           # MCP Server
curl http://localhost:8002/acp/health       # Team Leader ACP

# Trigger test incident
curl -X POST http://localhost:8001/incident/trigger -H "X-MCP-API-Key: <MCP_API_KEY>"

# Call MCP tool directly
curl -X POST http://localhost:8001/tools/ospf_parser \
  -H "X-MCP-API-Key: <MCP_API_KEY>" -H "Content-Type: application/json" \
  -d '{"device_id": "router-1", "idempotency_key": "test-123"}'

# Send ACP approval
curl -X POST http://localhost:8002/acp/approval -H "Content-Type: application/json" \
  -d '{"action": "approve", "task_id": "task-123", "incident_id": "syslog-456", "decision_rationale": "OK"}'

# Check Redis queues
docker-compose exec redis redis-cli
> LLEN incident_queue
> LLEN approval_queue
> LLEN a2a_stability_specialist_inbox

# Test all 3 routers
python test_all_routers.py

# Test HF dedicated endpoints and fallback
python test_endpoints.py
```

### Applying Agent Code Changes

Code is mounted as volumes — no rebuild needed, only a restart:

```bash
docker compose restart stability-agent troubleshooting-agent security-agent virtual-architect
```

**If you changed `environment:` or `env_file:` in `docker-compose.yml`, `restart` is not enough** — the running container keeps its old env. Use force-recreate per service:

```bash
docker compose up -d --no-deps --force-recreate stability-agent troubleshooting-agent security-agent virtual-architect
```

Verify the new env actually landed: `docker inspect netops-stability-agent --format '{{range .Config.Env}}{{println .}}{{end}}' | grep TIMESCALEDB`.

### Running Agents Locally (faster iteration)

```bash
cd agents
pip install -r requirements.txt
python team_leader/agent_acp_a2a.py    # or any specialist agent_a2a.py
```

## Configuration

Copy `.env.example` (or `.env.production` for HF dedicated endpoints) to `.env` before starting.

**LLM Provider options** (set `LLM_PROVIDER` in `.env`):

| Option | Speed | Cost | Notes |
|--------|-------|------|-------|
| `huggingface_dedicated` | 2-5s warm / 30-60s cold start | ~$0.60/hr/endpoint | Best accuracy; auto-fallback to OpenAI |
| `openai` (gpt-4o-mini) | 3-5s | ~$0.15/1M tokens | Reliable default |
| `gemini` (gemini-2.0-flash-exp) | 1-3s | API key required | Fastest |
| `ollama` (mistral:7b) | 10-15s | Free | Local; requires `ollama serve` |

**EC2 deployment:**
- App runs on `<APP_HOST_IP>` (AI_Dream_Team EC2, instance `<INSTANCE_ID>`) — routers send syslog directly to this IP
- EVE-NG lab on `<JUMP_HOST_IP>` (eveng-prod EC2, eu-central-1) — MCP server tunnels here for SSH to routers
- `extra_hosts: host.docker.internal:host-gateway` in docker-compose resolves to EC2 host IP on Linux (needed for Ollama)
- **Canonical compose on EC2:** `/home/ubuntu/agentic-netops-mvp/docker-compose.yml` — not a git checkout, deployed as tarball/rsync. The path `/opt/AI_Dream_Team/docker-compose.yml` is a stale snapshot missing `drift-detector`, `change-executor`, `neo4j`, `neo4j-sync`; don't operate against it.
- **Operator access:** SSH key in `deploy.sh` (`~/.ssh/netops-eveng-key`) was rotated and no longer works as of 2026-05-19. Use AWS SSM Run Command against `<INSTANCE_ID>` instead — the SSM agent is online and the `ed` IAM user (account `<AWS_ACCOUNT_ID>`) has access. Example: `aws ssm send-command --instance-ids <INSTANCE_ID> --document-name AWS-RunShellScript --parameters 'commands=["sudo docker ps"]'`.
- **TIMESCALEDB env in specialists:** `docker-compose.yml` defines `TIMESCALEDB_HOST`, `TIMESCALEDB_PORT`, `TIMESCALEDB_DATABASE`, `TIMESCALEDB_USER`, `TIMESCALEDB_PASSWORD` for all 4 specialists. After editing compose env, you **must** `docker compose up -d --no-deps --force-recreate <svc>` — a plain `restart` keeps the old container env. On 2026-05-19 the stability/troubleshooting/security/virtual-architect containers were running with stale env (no TIMESCALEDB_* set in `docker inspect ... Config.Env`); the force-recreate eliminated the latent bug.

**Device mode** (set `USE_REAL_DEVICES` in `.env`):
- `false` — uses `mock_device.py` (default for testing)
- `true` — SSH through AWS jump-host `ubuntu@<JUMP_HOST_IP>` (key: `mcp-server/<JUMP_HOST_KEY>.pem`) to 192.168.255.10/20/30

**Service ports:**
- Streamlit UI: `8501`
- Team Leader ACP: `8002`
- MCP Server: `8001` (internal Docker: `8000`)
- Redis: `6379`
- Neo4j Browser: `7474` (user: neo4j, pass: <NEO4J_PASSWORD>)
- Syslog: `5140/udp`

## MCP Tool Development

All MCP tool calls require `X-MCP-API-Key` header and `idempotency_key` in the request body. The 24h idempotency cache is implemented via Redis in `mcp-server/main.py`.

To add a new tool:
1. Create `mcp-server/tools/your_tool.py`
2. Add Pydantic schemas to `mcp-server/models/schemas.py`
3. Register `@app.post("/tools/your_tool")` in `mcp-server/main.py` (follow the idempotency pattern)
4. `docker-compose restart mcp-server`

## Troubleshooting

**Agent can't reach MCP**: `curl http://localhost:8001/health`, then `docker-compose logs mcp-server`

**UI can't reach Team Leader**: `curl http://localhost:8002/acp/health`, then `docker-compose logs team-leader`

**No approvals appearing in UI**: Check `LLEN approval_queue` and `LLEN a2a_team_leader_inbox` in Redis; check LLM API key validity in logs

**Router connectivity**: `python test_all_routers.py` — common causes: AWS EC2 stopped, wrong device IPs (.10/.20/.30 not .11/.12), SSH not enabled on router

**HF endpoint cold start**: First request takes 30-60s; system automatically falls back to `gpt-4o-mini` — this is expected behavior

## Architecture Documentation

Detailed docs in the project root:
- `ARCHITECTURE_REVIEW_PHASE4.md` — full 5-layer architecture
- `ACP_A2A_IMPLEMENTATION.md` — protocol implementation details
- `QUICK_START_DEDICATED_ENDPOINTS.md` — HF endpoint setup
- `FINE_TUNING_GUIDE.md` — fine-tuning and uploading models
- `HOW_TESTS_ACTUALLY_WORK.md` — test design
- `DIGITAL_TWIN_ARCHITECTURE.md` — planned digital twin simulation
