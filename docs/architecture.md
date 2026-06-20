# cc-suite Architecture

Current state: 2026-06-19

## Component Overview

cc-wire is the core shared library. All other packages are consumers of it.

| Package | Version | Role |
|---------|---------|------|
| cc-wire | 0.4.0 | Shared Redis key builders, types, primitives |
| cc-agent | 0.16.15 | MCP server — job lifecycle, wikis, swarms |
| cc-discord | 0.2.25 | Discord bot — per-channel meta-agent orchestration, cron engine |
| cc-agent-ui | 0.5.37 | Web UI — WebSocket + Redis pub/sub monitoring |
| gitkb | 0.2.12 | Distributed KB — 49 MCP tools, git-like versioning |

cc-tg (Telegram) is **deprecated and fully removed**.

---

## Per-Channel Meta-Agent Pattern

Each Discord channel maps to a namespace. cc-discord maintains one autonomous `claude --continue` session per namespace. No coordinator. No routing layer.

```
Discord message
  → cc-discord enqueues to Redis: cca:discord:meta:{ns}:input
  → MetaAgentManager polls (3s interval)
  → dequeue message
  → ensureWorkspace(ns, repoUrl)  — git clone if missing, git-kb init
  → injectMcp(ns, wsPath, token) — writes .mcp.json with gitkb + cc-agent
  → spawnSession(ns, wsPath, msg) — claude --continue -p "{msg}" --output-format stream-json
  → Claude session runs in ~/cc-discord-workspace/{ns}/
  → JSONL events stream to cca:meta:{ns}:stream (pub/sub) + cca:meta:{ns}:log (list, capped 2000)
```

Claude sessions are persistent (`--continue`) — they accumulate context across messages within the same channel. Output format is `stream-json` so tool usage events are structured and visible in cc-agent-ui.

---

## Slash Commands (cc-discord 0.2.24+)

| Command | Effect |
|---------|--------|
| `/restart` | Graceful process.exit(0) — launchd respawns clean. Canonical restart method. |
| `/clear` | Deletes JSONL session file in workspace — next message starts fresh session. |
| `/compact` | Runs `claude --continue -p "/compact"` in the namespace workspace. |
| `/cron add` | Add a new cron: schedule, task, repo_url, compact_every |
| `/cron list` | List all active crons |
| `/cron pause` | Pause a cron by ID |
| `/cron resume` | Resume a paused cron |
| `/cron delete` | Delete a cron by ID |

Commands are per-namespace (channel they're typed in). Emoji reaction ❌ or 🛑 on a cron message also kills that cron.

---

## Workspace Layout

Each meta-agent workspace: `~/cc-discord-workspace/{namespace}/`

On first message:
1. Repo is cloned (from channel's registered repoUrl)
2. `git-kb init` initializes a local `.kb/` in that workspace
3. `.mcp.json` is written with gitkb + cc-agent servers

`.mcp.json` template injected by cc-discord:
```json
{
  "mcpServers": {
    "gitkb": {
      "command": "/opt/homebrew/bin/git-kb",
      "args": ["mcp"]
    },
    "cc-agent": {
      "command": "/opt/homebrew/bin/npx",
      "args": ["-y", "--prefer-online", "@gonzih/cc-agent"],
      "env": { "CC_AGENT_NAMESPACE": "{ns}", "..." : "..." }
    }
  }
}
```

---

## cc-agent MCP Injection into Spawned Agents

When cc-agent spawns a job agent, it calls `injectMcpConfig(workDir, ns)` in `src/mcp-inject.ts`. This writes a `.mcp.json` with gitkb as the first entry and `GITKB_ROOT` pointing at the shared money-brain KB:

```json
{
  "mcpServers": {
    "gitkb": {
      "command": "/opt/homebrew/bin/git-kb",
      "args": ["mcp"],
      "env": {
        "GITKB_ROOT": "/Users/feral/cc-discord-workspace/money-brain"
      }
    }
  }
}
```

All spawned agents read/write the shared KB regardless of where they're cloned.

---

## Cron Engine (cc-discord 0.2.24+)

CronEngine lives inside cc-discord. Cron fires are **not agent spawns** — they are RPUSH messages into the target namespace's meta-agent input queue. This means cron is a loop mechanism, not a job spawner.

```
CronEngine fires
  → RPUSH cca:discord:meta:{ns}:input "{task prompt}"
  → MetaAgentManager picks it up (same as a Discord message)
  → Claude --continue session processes it with accumulated context
  → auto-compact fires every N fires (compact_every field)
```

Emoji kill: cc-discord watches for ❌/🛑 reactions on cron messages → pauses the cron.

See [Self-Healing Loops](self-healing-loops.md) for the loop pattern built on top of this.

---

## Key Redis Structures (cc-wire 0.4.0)

### Job lifecycle
- `cca:job:{id}` — hash: status, output, namespace, repoUrl, created_at
- `cca:job:done:{id}` — pub/sub, fires on terminal state
- `cca:job:list` — sorted set of all job IDs

### Meta-agent messaging
- `cca:discord:meta:{ns}:input` — list (RPUSH/BLPOP) — message queue per namespace
- `cca:discord:meta:{ns}:status` — hash: last_seen, session_pid, workspace_path
- `cca:discord:channels:index` — set of registered Discord channel IDs
- `cca:discord:channel:{channel_id}` — hash: namespace, **repoUrl** (camelCase — NOT `repo_url`)

### Meta-agent streaming
- `cca:meta:{ns}:stream` — pub/sub, live stream-json JSONL events `{ ts, ns, type, data }`
- `cca:meta:{ns}:log` — list, event history (capped at 2000, LPUSH + LTRIM)
- `cca:meta:{ns}:heartbeat` — last seen timestamp

### Process singleton
- `cca:discord:instance` — UUID of the live cc-discord process (30s TTL, refreshed every 10s)
  - On startup: write new UUID. On each poll: verify UUID matches → exit if not (stale process)
  - Prevents overlap window when launchd respawns

### Notifications and dedup
- `cca:notify:{ns}` — pub/sub for job completion notifications
- `cca:chat:log:{ns}` — list: full conversation log (bridged to cc-agent-ui)
- `cca:discord:sent:{ns}` — dedup set (120s TTL): job IDs already forwarded to Discord

### Cron
- `cca:discord:cron:list` — sorted set of cron job IDs
- `cca:discord:cron:{id}` — hash: schedule, task, namespace, repoUrl, last_run, fire_count, compact_every, status
- `cca:discord:cron-message:{messageId}` — set: cron IDs watching this Discord message (for emoji kill)

> **Critical:** channel hash field is `repoUrl` (camelCase), NOT `repo_url`. cc-wire reads camelCase. Wrong field name = channel registered but repo never found.

---

## Integration Tests

Each package has integration tests using a mock Claude binary:

| Package | Tests | Coverage |
|---------|-------|---------|
| cc-discord | 17 | spawnSession Redis streaming, CronEngine, CLAUDE_BIN injection |
| cc-agent | 28 | job lifecycle, mock claude binary |
| cc-wire | 69 | all key builders, types, deprecated markers |
| cc-suite-tests | 68 | health, key schema, message flow, cron, log analysis |

**Mock Claude binary:** `CLAUDE_BIN` env var overrides `resolveClaude()` in both cc-discord and cc-agent. Test fixture: `test/fixtures/mock-claude.js` (configurable via `MOCK_CLAUDE_RESPONSE`, `MOCK_CLAUDE_EXIT_CODE`, `MOCK_CLAUDE_DELAY_MS`).

**Redis isolation:** Integration tests MUST use Redis DB 1 (`redis://localhost:6379/1`), never DB 0 (production).

---

## cc-agent-ui Tab Structure (0.5.37+)

1. **Meta Agents** — Discord-like two-panel view. Left: namespace list. Right: structured stream-json events from `cca:meta:{ns}:log`. Renders tool_use, tool_result, assistant, text, result events — same visual fidelity as Jobs tab.
2. **Jobs** — job list and output
3. **Crons** — cron schedules
