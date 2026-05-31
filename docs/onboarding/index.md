# For future operators + new engineers

You walked in cold. Here's the system mental model in 10 minutes.

## The model

1. **Layer 5 (devices) → Layer 4 (MCP server)**: every read is an MCP HTTP call with an idempotency_key. Mock mode is the dev default; flip to live with `USE_REAL_DEVICES=true`.
2. **Layer 4 → Layer 3 (specialists)**: agents pull device state via MCP, never SSH directly.
3. **Layer 3 → Layer 2 (Team Leader)**: A2A messages over Redis inboxes.
4. **Layer 2 → Layer 1 (UI)**: ACP REST + Streamlit.
5. **The whole pipeline produces an approval card**. Humans approve. `change-executor` applies.

## Where to read next

| Topic | Read this |
|---|---|
| How files are organized | [codebase-map.md](codebase-map.md) |
| Run it on your laptop | [local-dev.md](local-dev.md) |
| Deploy to EC2 + use SSM | [deploy.md](deploy.md) |
| What breaks and how to fix | [debugging.md](debugging.md) |

## The fastest 60-second tour

```bash
# Bring up the stack (mock mode)
docker compose up -d

# Trigger one incident
curl -X POST http://localhost:8001/incident/trigger \
  -H "X-MCP-API-Key: <MCP_API_KEY>"

# Watch the approval card land
open http://localhost:8501
```

If you can run those 3 commands and see an approval card, you've understood the platform at the level the customer cares about. Everything else is depth.
