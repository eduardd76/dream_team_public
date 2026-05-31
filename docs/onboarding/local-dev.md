# Local dev

## Prereqs

- Docker + Docker Compose
- Python 3.11
- `gh` CLI (for repo work)
- AWS CLI (for SSM deploys)

## Bring up the stack

```bash
cd agentic-netops-mvp
cp .env.example .env       # edit if needed; mock-mode defaults work
docker compose up -d
```

Health check (wait ~30s after up):

```bash
curl -sf http://localhost:8001/health     # MCP server
curl -sf http://localhost:8002/acp/health # Team Leader ACP
```

Open the UI: <http://localhost:8501>

## Service ports

| Service | Port |
|---|---|
| Streamlit UI | 8501 |
| Team Leader ACP | 8002 |
| MCP Server (external) | 8001 |
| MCP Server (Docker-internal) | 8000 |
| Redis | 6379 |
| Neo4j Browser | 7474 |
| Syslog | 5140/udp |
| Verifiability / transcripts UI | 8505 |

## Run the docs site locally

```bash
make docs-serve
```

Visits <http://127.0.0.1:8000>. Live-reloads on doc edits.

## Trigger an incident

```bash
# Built-in dev trigger (enum-limited; supports ospf_neighbor_down / interface_down / interface_errors / connectivity_failure)
curl -X POST http://localhost:8001/incident/trigger \
  -H "X-MCP-API-Key: <MCP_API_KEY>"

# Raw syslog injection (anything goes through triage)
docker compose exec redis redis-cli LPUSH incident_queue \
  '{"incident_id":"dev-1","device_id":"leaf-nx","raw_message":"%BGP-5-ADJCHANGE: neighbor 10.0.0.11 Down L2VPN EVPN session admin shut by operator","timestamp":"2026-05-30T19:00:00Z"}'
```

## Apply code changes without rebuild

Agent code is mounted as volumes — just restart the affected container:

```bash
docker compose restart stability-agent
```

If you changed `environment:` or `env_file:` in compose, you need force-recreate:

```bash
docker compose up -d --no-deps --force-recreate stability-agent
```

## Run tests

```bash
cd tests
pip install -r requirements.txt

# Phase 3 surface
/tmp/utestvenv/bin/python -m pytest test_validate_evpn_phase3.py -v
/tmp/utestvenv/bin/python -m pytest test_evpn_fault_rules.py -v
/tmp/utestvenv/bin/python -m pytest test_real_vrnetlab_backend.py -v

# Full suite
/tmp/utestvenv/bin/python -m pytest -v
```

## Reset clean state (between dry runs)

```bash
for q in approval_queue approved_history rejected_history dead_letter_queue \
         incident_queue a2a_stability_specialist_inbox a2a_team_leader_inbox; do
  docker compose exec redis redis-cli DEL $q
done
docker compose exec redis redis-cli --scan --pattern 'incident_active:*' | \
  xargs -r docker compose exec redis redis-cli DEL
```

## Verify mocks are wired

```bash
docker compose exec mcp-server python -c "
import os
print('USE_REAL_DEVICES:', os.getenv('USE_REAL_DEVICES'))
print('USE_FALLBACK:', os.getenv('USE_FALLBACK_ON_LLM_ERROR'))
"
```

Expect: `USE_REAL_DEVICES=false`. Anything else means you're hitting real device gear — confirm intent.
