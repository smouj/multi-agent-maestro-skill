---
name: multi-agent-maestro
version: 1.2.0
description: Coordinates heterogeneous AI agents with dynamic role assignment and state synchronization across distributed workspaces
author: SMOUJBOT Core Team
tags: [orchestration, multi-agent, automation, ai, coordination]
maintainer: ops@smouj.dev
homepage: https://github.com/smouj/openclaw/blob/main/docs/skills/multi-agent-maestro.md
dependency: [openclaw-core >= 2.1.0, docker-compose >= 2.20.0, redis >= 7.0]
conflicts: [single-agent-mode, legacy-workflows]
enabled: true
---

# Multi-Agent Maestro

Orchestrates heterogeneous AI agents with dynamic role assignment, state synchronization, and fault tolerance across distributed workspaces. Manages agent lifecycle, task delegation, and cross-agent data flow for complex multi-system workflows.

## Purpose

Real use cases:
- Coordinate RPGCLAW-OPS and FLICKCLAW-OPS agents for simultaneous repository and infrastructure updates
- Assign IMG-OPS and THREEJS-OPS agents to create 3D assets with generated textures in pipeline
- Execute VPS-OPS security hardening while RPGCLAW-OPS rotates secrets, synchronizing rollback points
- Run parallel code review (code-reviewer), testing (test-engineer), and frontend validation (frontend-specialist) on PRs
- Delegate FlickClaw development tasks to appropriate Kilocode modes based on codebase analysis
- Manage agent failover when workspace isolation is breached or resource limits exceeded

## Scope

### Commands

`maestro agent assign <agent-type> --to-workspace <path> [--role <role>] [--priority <1-5>]`
Assigns an agent instance to a workspace with specific role and priority.

`maestro agent list [--status <active|idle|failed|all>] [--format <json|table>]`
Lists all registered agents with current state and assignments.

`maestro task dispatch <prompt> --to <agent1,agent2,...> [--sync|--async] [--timeout <seconds>] [--depends-on <task-id>]`
Dispatches a task to multiple agents with dependency graph support.

`maestro workflow create <name> --definition <yaml-path> [--vars <json-path>]`
Creates a reusable workflow from YAML definition with variable substitution.

`maestro workflow run <workflow-id> [--param key=value ...] [--trace] [--resume-from <step-id>]`
Executes a workflow with parameter injection and trace logging.

`maestro state snapshot [--workspace <path>] [--include-logs] [--output <tar.gz>]`
Captures complete state of all agents and workspaces for rollback.

`maestro state restore <snapshot-path> [--skip-failed] [--dry-run]`
Restores agents and workspaces to snapshot state.

`maestro monitor start [--interval <seconds>] [--alert-webhook <url>] [--metrics-port <port>]`
Starts real-time monitoring with health checks and alerting.

`maestro agent health <agent-id> [--detail] [--log-lines <n>]`
Checks health status of specific agent with optional log tail.

`maestro queue status [--agent <id>] [--task <id>] [--waiting|--processing|--completed]`
Shows task queue state across all agents.

### Environment Variables

`MAESTRO_WORKSPACE_ROOT` - Root directory for all isolated workspaces (default: /tmp/maestro-workspaces)
`MAESTRO_REDIS_URL` - Redis connection for state sharing (default: redis://localhost:6379)
`MAESTRO_LOG_LEVEL` - Logging verbosity (DEBUG, INFO, WARN, ERROR)
`MAESTRO_MAX_CONCURRENT` - Maximum parallel agent executions (default: 4)
`MAESTRO_SNAPSHOT_RETENTION` - Days to keep automatic snapshots (default: 7)
`MAESTRO_KILOCODE_BIN` - Path to Kilocode CLI binary (default: /usr/local/bin/kilo)
`MAESTRO_AGENT_TIMEOUT` - Default agent task timeout in seconds (default: 1800)
`MAESTRO_FAILSAFE_MODE` - Enable aggressive rollback on any agent failure (true/false)

## Work Process

1. **Agent Discovery & Registration**
   - Maestro scans configured agent types from `/etc/maestro/agents.d/*.yaml`
   - Each agent definition includes: slug, mode, workspace_template, resource_limits, capabilities
   - Agents register via `maestro agent register` or auto-register on first use
   - Validation: verify Kilocode mode exists, workspace template is accessible, resources available

2. **Workspace Isolation**
   - For each agent task, Maestro creates isolated workspace under `MAESTRO_WORKSPACE_ROOT/<agent-id>/<task-id>/`
   - Copies workspace_template contents if template path exists
   - Injects agent-specific environment from `agents.d/<agent>.yaml` env block
   - mounts: workspace dir (rw), shared cache dir (ro), secrets directory (ro)
   - Network: isolated via Docker network `maestro-<workspace-hash>` unless `shared_network: true`

3. **Task Dispatching**
   - `maestro task dispatch` validates target agents are registered and healthy
   - Creates task record in Redis with unique ID, status=queued, created_at timestamp
   - If `--sync`: blocks until all agents complete, aggregates results, exits with first non-zero code if any fail
   - If `--async`: returns immediately with task-tracker-id, user polls `maestro task status <id>`
   - Dependencies: checks `--depends-on` tasks completed before dispatch

4. **Execution & Monitoring**
   - Maestro spawns agent process: `kilo <mode> "<prompt>" <workspace-dir>`
   - Captures stdout/stderr to `workspace/agent.log`
   - Updates Redis every 5 seconds with heartbeat, memory/CPU usage (from /proc if available)
   - If `--trace`: wraps prompt with "TRACE: [timestamp] step: " prefixes for each stdout line

5. **Health Checking**
   - Every 30 seconds (configurable via `--interval`):
     - Check process exists and responds to SIGTERM
     - Verify workspace dir still accessible
     - Ping agent via Redis key `agent:<id>:last_heartbeat` (must be < 60s old)
     - If failed: mark agent unhealthy, trigger rollback if `FAILSAFE_MODE=true`

6. **State Synchronization**
   - Agents can share data via `maestro state set <key> <value>` (stored in Redis, TTL 24h by default)
   - Workflows can read previous step output: `{{ state.<key> }}` in YAML definitions
   - On successful task completion, Maestro auto-sets `task:<id>:result` in state

7. **Snapshot & Rollback**
   - `maestro state snapshot` creates tar.gz with:
     - All workspace directories for active agents
     - Redis state dump (RDB)
     - Agent registry YAML
     - Container configs if Docker used
   - `maestro state restore` stops all agents, restores workspaces from tar, loads Redis RDB, restarts agents to previous assignments
   - **Rollback safety**: before restore, Maestro validates snapshot integrity (checks SHA256 manifest inside tar)

8. **Shutdown & Cleanup**
   - `maestro shutdown [--grace-period <seconds>]` sends SIGTERM to all agents, waits, then SIGKILL
   - Cleans workspace directories older than `SNAPSHOT_RETENTION` days (unless `--keep` specified)
   - Preserves last successful snapshot as `latest` symlink

## Golden Rules

1. **Never** run Maestro itself inside a workspace it manages (avoid nested isolation). Maestro runs on host system only.
2. **Always** assign agents with explicit `--to-workspace` pointing outside Maestro's own root to prevent recursive workspace creation.
3. **Check agent health** before dispatching critical tasks: `maestro agent health <id>` must show `status: healthy`.
4. **Use `--sync` for dependent tasks**, `--async` for fire-and-forget. Never assume async task completes successfully without explicit status check.
5. **Never modify workspace contents** manually while agent is running. All changes must go through agent's Kilocode process to maintain state consistency.
6. **Always set `MAESTRO_WORKSPACE_ROOT`** to a disk with sufficient space (≥ 10GB per concurrent agent). Default /tmp may be small.
7. **For production workflows, use `maestro workflow create/run`** instead of manual `maestro task dispatch` to capture reproducible definitions.
8. **Verify Redis is running** before any Maestro command that involves state: `redis-cli ping` must return PONG.
9. **Never restore snapshot** without first checking `maestro agent list` to ensure no critical agents are active (unless intentional rollback during incident).
10. **Always include `--trace`** in debugging multi-agent failures; the trace log shows inter-agent message timing and payloads.

## Examples

**Example 1: Coordinate code review and testing on a PR**
```bash
# Assign code-reviewer and test-engineer agents to fresh workspaces
maestro agent assign code-reviewer --to-workspace /tmp/pr-123-review --priority 1
maestro agent assign test-engineer --to-workspace /tmp/pr-123-tests --priority 2

# Dispatch review task to code-reviewer, then run tests only if review passes (sync dependency)
maestro task dispatch "Review PR #123, check for security issues and code style" \
  --to code-reviewer --sync

# If review succeeded (exit code 0), dispatch tests
maestro task dispatch "Run unit and integration tests for PR #123" \
  --to test-engineer --depends-on $(maestro task list --last | jq -r '.id') --sync

# Capture final state for audit
maestro state snapshot --output /backups/pr-123-snapshot-$(date +%s).tar.gz
```

**Example 2: FlickClaw feature with parallel frontend and backend development**
```bash
export MAESTRO_WORKSPACE_ROOT=/opt/flickclaw-workspaces
maestro agent assign flickclaw-bot --to-workspace /opt/flickclaw-workspaces/backend --role backend-dev
maestro agent assign frontend-specialist --to-workspace /opt/flickclaw-workspaces/frontend --role ui-dev

# Run both agents async, they coordinate via shared state
maestro task dispatch "Implement user authentication API endpoint with JWT" \
  --to flickclaw-bot --async

maestro task dispatch "Create login form component with validation and error handling" \
  --to frontend-specialist --async --timeout 3600

# Poll until both tasks complete
while true; do
  backend_status=$(maestro task status --last flickclaw-bot | jq -r '.status')
  frontend_status=$(maestro task status --last frontend-specialist | jq -r '.status')
  [[ "$backend_status" == "completed" && "$frontend_status" == "completed" ]] && break
  sleep 30
done
```

**Example 3: Incident rollback with snapshot restore**
```bash
# Emergency: agents diverged, restore last known good state
maestro agent list  # see active agents
maestro shutdown --grace-period 10  # stop all agents gracefully

# Find latest snapshot
latest=$(ls -t /backups/maestro-snapshots/*.tar.gz | head -1)
maestro state restore "$latest" --dry-run  # verify what will change
maestro state restore "$latest"  # perform restore

# Restart agents to previous assignments
maestro agent assign --from-snapshot "$latest"
```

**Example 4: Create reusable workflow for RPGCLAW deployment**
`workflows/deploy-rpgclaw.yaml`:
```yaml
name: "rpgclaw-deploy"
version: "1.0"
steps:
  - id: "git-pull"
    agent: "rpgclaw-ops"
    prompt: "Pull latest changes from main, run git status, note any migrations"
    timeout: 300
    on_success: "build"
  - id: "build"
    agent: "rpgclaw-ops"
    prompt: "Run npm run build, capture output, fail on error"
    timeout: 600
    on_success: "health-check"
  - id: "health-check"
    agent: "vps-ops"
    prompt: "Verify RPGCLAW responds on :3000, check logs for errors"
    timeout: 120
vars:
  branch: "main"
  environment: "production"
```

Run:
```bash
maestro workflow create rpgclaw-deploy --definition workflows/deploy-rpgclaw.yaml
maestro workflow run rpgclaw-deploy --param branch=main --trace
```

## Dependencies & Requirements

- **openclaw-core >= 2.1.0**: Provides base agent lifecycle management and workspace templates
- **docker-compose >= 2.20.0**: Required when agents use Docker isolation (`isolate_with: docker`)
- **redis >= 7.0**: State database with RedisJSON module for structured state storage
- **kilo CLI**: Kilocode binary in PATH or `MAESTRO_KILOCODE_BIN` (must support modes: flickclaw-bot, code-reviewer, test-engineer, frontend-specialist, flick-devops, img-ops, threejs-ops)
- **Filesystem**: `MAESTRO_WORKSPACE_ROOT` must be on ext4/xfs with at least 10GB free per concurrent agent
- **Network**: Agents with `shared_network: true` require host network access; ensure firewall allows agent outbound connections
- **Kernel**: Linux kernel >= 5.4 for cgroup v2 isolation (if using Docker)

Verify installation:
```bash
maestro --version        # Should show 1.2.0+
kilo --list-modes        # Should list all expected agent modes
redis-cli INFO server    # redis_version:7.x
docker compose version   # Docker Compix version v2.20+
df -h $MAESTRO_WORKSPACE_ROOT  # Sufficient free space
```

## Rollback Commands

`maestro agent stop <agent-id> [--force]` - Stop specific agent without affecting others.

`maestro agent reset <agent-id>` - Stop agent, clear its workspace, re-register from template.

`maestro state rollback <snapshot-path> [--skip-verify]` - Full system rollback to snapshot (alias for `state restore`).

`maestro task cancel <task-id> [--reason "<text>"]` - Cancel queued/running task, optionally provide reason in audit log.

`maestro workflow stop <workflow-id>` - Stop all running steps of a workflow, marks workflow as cancelled.

`maestro config reset --to-default` - Reset Maestro configuration to defaults (does NOT touch workspaces or Redis).

`maestro workspace clean <agent-id>` - Delete workspace directory for agent, stops agent first if running.

`maestro redis flush` - **DANGER**: Clears all state from Redis. Use only when state corrupted and snapshots unavailable.

**Rollback procedure for failed multi-agent deployment:**
1. `maestro agent list` → identify failed agents
2. `maestro task status <failed-task-id>` → get failure details
3. `maestro state snapshot --output /tmp/pre-rollback-$(date +%s).tar.gz` (if still possible)
4. `maestro shutdown --grace-period 5`
5. `maestro state restore /backups/latest-good-snapshot.tar.gz`
6. `maestro agent assign --from-snapshot /backups/latest-good-snapshot.tar.gz`
7. `maestro monitor start --alert-webhook https://alerts.smouj.dev/failover` to confirm recovery

## Troubleshooting

### Agent fails to start with "mode not found"
- Check `kilo --list-modes` includes required mode
- Verify `MAESTRO_KILOCODE_BIN` points to correct Kilocode binary
- Ensure Kilocode version matches agent expectations (`kilo --version`)

### Workspace isolation breach (agents seeing each other's files)
- Maestro does NOT use Docker by default. Set `isolate_with: docker` in agent YAML definition
- Verify Docker network `maestro-*` exists and has no containers from other projects
- Check workspace permissions: `ls -ld $MAESTRO_WORKSPACE_ROOT/*` should be 700

### Redis connection errors
- Start Redis: `sudo systemctl start redis` or `docker run -p 6379:6379 redis:7-alpine`
- Set `MAESTRO_REDIS_URL=redis://localhost:6379` if not default
- Check firewall: `redis-cli ping` from Maestro host must succeed

### Tasks stuck in "queued" forever
- Check agent health: `maestro agent health <agent-id>`
- Verify `MAESTRO_MAX_CONCURRENT` not exceeded: `maestro agent list | grep -c active`
- Inspect Redis queue: `redis-cli LRANGE maestro:queue:pending 0 -1`
- If agent died, restart: `maestro agent restart <agent-id>`

### Snapshot restore fails with "workspace missing"
- Ensure `MAESTRO_WORKSPACE_ROOT` points to same path as when snapshot created
- Check tar integrity: `tar -tzf snapshot.tar.gz | head`
- Use `--skip-failed` to continue restore even if some workspace dirs missing

### High memory usage with many agents
- Each agent gets isolated workspace; aggregate memory = sum of agent processes + Kilocode overhead
- Set `MAESTRO_MAX_CONCURRENT` to limit parallel agents
- Use Docker memory limits in agent definition: `resources: { memory: "512m" }`

### Debugging multi-agent race conditions
- Run workflow with `--trace` to capture inter-agent timing
- Check Redis state keys: `redis-cli KEYS "state:*" | xargs -I{} redis-cli GET {}`
- Enable DEBUG logging: `export MAESTRO_LOG_LEVEL=DEBUG` before running
- Inspect agent logs: `cat $MAESTRO_WORKSPACE_ROOT/<agent-id>/<task-id>/agent.log`

### Agent reports "permission denied" on workspace
- Maestro creates workspace with 700 permissions. Ensure Maestro process runs as same user that owns `MAESTRO_WORKSPACE_ROOT`
- If using Docker isolation, check Docker daemon has read/write access to workspace dir (`docker info | grep RootDir`)

### Workflow step skipped unexpectedly
- Check `on_success` field in workflow YAML: it must match the next step's `id` exactly
- Verify step status: `maestro task status <task-id>` shows `status: failed` if previous step failed
- Use `--resume-from <step-id>` to restart workflow from specific step
```