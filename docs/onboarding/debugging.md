# Debugging

Common failure modes + how to recover. Updated 2026-05-30.

## "Agent can't reach MCP"

```bash
docker compose logs mcp-server
curl http://localhost:8001/health     # expect 200
```

If MCP is down: `docker compose restart mcp-server`. If the issue persists, check Redis DNS — the recurring DNS gotcha is the most common cause:

```bash
docker compose exec mcp-server python -c "
import os, redis
r = redis.from_url(os.getenv('REDIS_URL','redis://redis:6379'))
print('ping:', r.ping())
"
```

If `ping` raises NXDOMAIN: `docker compose restart mcp-server` (the boot-reconcile heals this).

## "UI shows no approvals"

```bash
docker compose exec redis redis-cli LLEN approval_queue
docker compose exec redis redis-cli LLEN a2a_team_leader_inbox
```

If `approval_queue` is 0 and `a2a_team_leader_inbox` is non-zero: Team Leader isn't draining. Logs:

```bash
docker compose logs team-leader | tail -40
```

Common cause: LLM endpoint is cold. The architect ReAct loop sometimes blocks the team leader event loop for 30-60s on a cold start.

## "Pass 4 returns skip_reason"

Each skip_reason is named:

| skip_reason | Fix |
|---|---|
| `containerlab_not_installed` | `which containerlab` on the EC2 host; if missing, install. Override mounts it at `/usr/bin/containerlab:/usr/bin/containerlab:ro`. |
| `fabric_yaml_not_found:<paths>` | Bind-mount `./fabric.yaml:/app/fabric.yaml:ro` in `docker-compose.override.yml`. |
| `templates_dir_missing:<path>` | Bind-mount `./containerlab:/app/containerlab` in override. |
| `device_not_in_fabric:<id>` | Add the device to `fabric.yaml`. |
| `evpn_config_generation_failed` | `pip install jinja2` in the agent container, or check templates exist. |
| `evpn_deploy_failed` | Check `deploy_stderr_tail` in the evidence dict — usually docker network conflict or image not pulled. |

## "LLM endpoint returns errors"

```bash
docker compose logs stability-agent | grep -iE "llm|endpoint|fallback"
```

If you see `LLM cold start, falling back to OpenAI`: that's the auto-fallback working. Expected on first request after instance start.

If you see `LLM endpoint unreachable, no fallback`: check the endpoint URL in `.env`:

```bash
echo $STABILITY_ENDPOINT
curl -fsS "$STABILITY_ENDPOINT/health" || echo "endpoint down"
```

## "Architect ReAct hangs"

Architect uses 10 read-only tools. If one tool blocks (e.g., MCP doesn't respond), the ReAct loop hangs.

```bash
docker compose logs virtual-architect | grep -iE "tool_call|thought|error" | tail -20
```

If a specific tool is stuck: restart MCP server and retry.

## "Tests pass locally but fail on EC2"

The common cause: the running Python process has the OLD code in its import cache. After deploy, you need a container restart:

```bash
sudo docker compose restart stability-agent
sleep 25                    # wait for the process to come back up
```

NOT enough for env changes — force-recreate (see [deploy.md](deploy.md)).

## "Containerlab deploy fails with bridge race"

```
Failed to lookup link "br-XXXXX": Link not found.
```

Known containerlab 0.74.x netlink race. Either:

- Upgrade containerlab to 0.75+ on the host
- Pre-create the docker network with the same name before validate_evpn runs

The runtime injects a per-call unique mgmt-network name (see `containerlab_manager.py::validate_evpn`) so concurrent runs don't collide.

## "Containerlab deploy fails with subnet overlap"

```
Requested subnet 172.20.20.0/24 overlap an existing Docker network
```

The runtime injects mgmt subnet `172.30.0.0/24` by default. If your stack uses that range:

```bash
export CONTAINERLAB_MGMT_SUBNET=172.40.0.0/24
docker compose up -d --no-deps --force-recreate stability-agent
```

## "Deploy doesn't reflect new code"

Three causes, in order of likelihood:

1. **Bucket-delete race.** If your deploy script does `aws s3 cp ...` in foreground and `aws s3 rb ... &` in background, the bucket can disappear before SSM downloads. Move the `aws s3 rb` AFTER the SSM completes.
2. **macOS AppleDouble files.** `tar` includes `._*` files; on Linux these become "broken" siblings. Always: `tar --exclude='._*' -czf ...`.
3. **Container env stale.** `restart` keeps old env. Use `up -d --no-deps --force-recreate` after compose changes.

## "How do I tell what's actually running on EC2?"

```bash
aws ssm send-command \
  --instance-ids <INSTANCE_ID> \
  --document-name AWS-RunShellScript \
  --region eu-central-1 \
  --parameters 'commands=[
    "cd /home/ubuntu/agentic-netops-mvp",
    "git log --oneline -1",
    "git rev-parse HEAD",
    "md5sum agents/shared/validate_node.py agents/shared/containerlab_manager.py",
    "sudo docker ps --format \"{{.Names}} {{.Status}}\""
  ]'
```

That gives you commit, files-on-disk hash, container status. Cross-reference with what's on GitHub.
