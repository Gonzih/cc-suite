# cc-suite

Meta repository for the cc-suite ecosystem — architecture, deployment, and development guides.

## Packages

| Package | npm | Role |
|---------|-----|------|
| [cc-wire](https://github.com/gonzih/cc-wire) | `@gonzih/cc-wire` | Shared Redis key builders, types, primitives |
| [cc-agent](https://github.com/gonzih/cc-agent) | `@gonzih/cc-agent` | MCP server — job lifecycle, wikis, swarms |
| [cc-discord](https://github.com/gonzih/cc-discord) | `@gonzih/cc-discord` | Discord bot — per-channel meta-agent orchestration, cron engine |
| [cc-agent-ui](https://github.com/gonzih/cc-agent-ui) | `@gonzih/cc-agent-ui` | Web UI — WebSocket + Redis pub/sub monitoring |
| [cc-suite-tests](https://github.com/gonzih/cc-suite-tests) | — | Integration test harness (private) |

cc-wire is the foundation. All other packages are consumers.

## Docs

- [Architecture](docs/architecture.md) — system design, Redis key schema, cron engine, loop pattern
- [Deployment](docs/deployment.md) — installation, restart procedures, launchd, Docker
- [Self-Healing Loops](docs/self-healing-loops.md) — cron as stateful AI loop
- [gitkb Integration](docs/gitkb-integration.md) — shared knowledge base across all agents
- [Migration History](docs/migration.md) — why cc-tg was removed, key decisions

## Quick Start

cc-discord is the entry point. It runs as a launchd daemon, spawns per-channel meta-agent Claude sessions, manages crons, and exposes slash commands.

```bash
# Install
npx --prefer-online @gonzih/cc-discord@latest

# Restart (canonical)
# Use /restart slash command in Discord

# Restart (manual, macOS)
ps aux | grep cc-discord | grep -v grep | awk '{print $2}' | while read pid; do kill "$pid"; done
```

## Architecture in One Paragraph

Each Discord channel maps to a namespace. cc-discord maintains one persistent `claude --continue` session per namespace in `~/cc-discord-workspace/{ns}/`. Messages from Discord are RPUSH'd to a Redis queue; the session picks them up and streams structured output (tool calls, results, text) back to Redis for cc-agent-ui to display. Crons live inside cc-discord — they fire by RPUSH'ing a message into the same queue, making cron = a scheduled loop, not a job spawner. cc-agent is a pure MCP server that job agents call for spawning sub-jobs, wikis, and swarms. cc-wire is the single source of truth for all Redis key names.
