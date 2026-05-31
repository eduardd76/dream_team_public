# Deploy (EC2 + SSM)

## The deploy targets

| Plane | Where | Role |
|---|---|---|
| **Local** | your laptop | rapid iteration |
| **GitHub** | `eduardd76/dream_team_original` | source of truth |
| **EC2 production** | `AI_Dream_Team` (instance `<INSTANCE_ID>`) at `<APP_HOST_IP>` | live demo + pilot stack |

All three planes must stay in sync. The pattern: commit locally → push to GitHub → deploy to EC2 via SSM.

## Why SSM, not SSH

The SSH keys (`mcp-server/<JUMP_HOST_KEY>.pem`) were rotated 2026-05-19 and no longer authenticate. Use AWS SSM Run Command instead:

```bash
aws ssm send-command \
  --instance-ids <INSTANCE_ID> \
  --document-name AWS-RunShellScript \
  --region eu-central-1 \
  --parameters 'commands=["sudo docker ps"]'
```

The `ed` IAM user (account `<AWS_ACCOUNT_ID>`) has SSM-Session access. The SSM agent is online on the box.

## Standard deploy pattern (single file change)

```bash
# 1. Stage in a tmp S3 bucket
BUCKET=ed-netops-deploy-$(date +%s)
aws s3api create-bucket --bucket "$BUCKET" --region eu-central-1 \
  --create-bucket-configuration LocationConstraint=eu-central-1
aws s3 cp agents/shared/validate_node.py "s3://$BUCKET/validate_node.py" --region eu-central-1

# 2. Run the deploy command via SSM
aws ssm send-command \
  --instance-ids <INSTANCE_ID> \
  --document-name AWS-RunShellScript \
  --region eu-central-1 \
  --parameters "commands=[
    \"aws s3 cp s3://$BUCKET/validate_node.py /home/ubuntu/agentic-netops-mvp/agents/shared/validate_node.py --region eu-central-1\",
    \"sudo chown ubuntu:ubuntu /home/ubuntu/agentic-netops-mvp/agents/shared/validate_node.py\",
    \"cd /home/ubuntu/agentic-netops-mvp && sudo docker compose restart stability-agent\"
  ]"

# 3. Clean up the bucket
aws s3 rb "s3://$BUCKET" --force --region eu-central-1
```

## Multi-file deploy (tarball pattern)

For 2+ files, tarball:

```bash
tar --exclude='._*' -czf /tmp/deploy.tar.gz \
    agents/shared/validate_node.py \
    agents/shared/containerlab_manager.py \
    containerlab/templates/

aws s3 cp /tmp/deploy.tar.gz "s3://$BUCKET/deploy.tar.gz" --region eu-central-1

aws ssm send-command \
  --instance-ids <INSTANCE_ID> \
  --document-name AWS-RunShellScript \
  --region eu-central-1 \
  --parameters "commands=[
    \"aws s3 cp s3://$BUCKET/deploy.tar.gz /tmp/deploy.tar.gz --region eu-central-1\",
    \"tar -xzf /tmp/deploy.tar.gz -C /home/ubuntu/agentic-netops-mvp\",
    \"sudo chown -R ubuntu:ubuntu /home/ubuntu/agentic-netops-mvp/agents /home/ubuntu/agentic-netops-mvp/containerlab\",
    \"cd /home/ubuntu/agentic-netops-mvp && sudo docker compose restart stability-agent\"
  ]"
```

!!! tip "AppleDouble files"
    On macOS, `tar` includes `._*` resource-fork files. They cause "file exists but not a file" errors on Linux. Always: `tar --exclude='._*' -czf ...`.

## Override → docker-compose changes

If you edit `docker-compose.yml` or `docker-compose.override.yml`:

```bash
# Force-recreate, not restart — restart preserves the old container env
cd /home/ubuntu/agentic-netops-mvp
sudo docker compose up -d --no-deps --force-recreate stability-agent
```

## EC2 instance scheduling

The `autocon5` schedule stops all 6 instances at 22:00 Berlin and starts them at 08:00. During the workshop demo window (08:00-22:00), the stack is up. Overnight, the stack is down — re-deploys must wait for morning.

## Boot reconcile

On every start, `/etc/systemd/system/netops-reconcile.service` runs:

1. Wait for Docker
2. `cd /home/ubuntu/agentic-netops-mvp && docker compose up -d`
3. Heal Redis DNS for `mcp-server`
4. Restart NetBox stack if needed

If a component is degraded after boot, check the reconcile logs:

```bash
sudo journalctl -u netops-reconcile -n 100
```

## Verify deploy landed

```bash
# Diff the deployed file against your local file
aws ssm send-command \
  --instance-ids <INSTANCE_ID> \
  --document-name AWS-RunShellScript \
  --region eu-central-1 \
  --parameters 'commands=["md5sum /home/ubuntu/agentic-netops-mvp/agents/shared/validate_node.py"]'

md5sum agents/shared/validate_node.py
```

Md5s must match.

## Run a live smoke

```bash
aws ssm send-command \
  --instance-ids <INSTANCE_ID> \
  --document-name AWS-RunShellScript \
  --region eu-central-1 \
  --parameters 'commands=[
    "sudo docker exec netops-redis redis-cli DEL approval_queue > /dev/null",
    "sudo docker exec netops-redis redis-cli LPUSH incident_queue %{\"incident_id\":\"smoke\",\"device_id\":\"leaf-nx\",\"raw_message\":\"%BGP-5-ADJCHANGE: neighbor 10.0.0.11 Down L2VPN EVPN session admin shut\"}%",
    "sleep 150",
    "sudo docker exec netops-redis redis-cli LINDEX approval_queue 0"
  ]'
```

Should return a JSON approval card with validation evidence.
