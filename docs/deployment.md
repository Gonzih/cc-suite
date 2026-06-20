# cc-suite Deployment

## Service Map

| Service | Runtime | How to restart |
|---------|---------|---------------|
| cc-discord | launchd (KeepAlive) | `/restart` slash command (canonical), or PID-based kill |
| cc-agent | MCP server via npx | No persistent process; restarted with cc-discord |
| cc-agent-ui | Docker container | `docker restart <container>` |

---

## cc-discord

**Command:** `npx --prefer-online @gonzih/cc-discord@latest`

**Restart (canonical):** `/restart` slash command in Discord — cleanest, no race conditions.

**Restart (manual — macOS PID kill):**
```bash
ps aux | grep cc-discord | grep -v grep | awk '{print $2}' | while read pid; do kill "$pid"; done
```

> `pkill -f cc-discord` does **NOT** work on macOS — the launchd process name doesn't match the filter. Always use PID-based kill.

**NEVER:** `launchctl unload` — removes KeepAlive, process won't come back.

**Check version before kill:** `npm show @gonzih/cc-discord version`

**npm cache:** `/Users/feral/.npm-cache-money-brain/` (custom cache dir, avoids conflicts)

When new version published: kill → launchd respawns with `--prefer-online` pulling fresh.

---

## cc-agent-ui (Docker)

```dockerfile
FROM node:22-slim
ENV PORT=7701
EXPOSE 7701
CMD ["/bin/sh", "-c", "exec npx --prefer-online -y @gonzih/cc-agent-ui@latest"]
```

---

## Deploy Cycle

```
agent builds code
→ npm run build
→ npm version patch
→ npm publish --access public
→ restart service (/restart in Discord for cc-discord, docker restart for ui)
→ npx --prefer-online pulls latest on respawn
```

---

## Process Tree

```
launchd (KeepAlive: true)
  └── npx @gonzih/cc-discord@latest
        └── node cc-discord/dist/index.js
              ├── CronEngine — fires RPUSH to meta input queues
              └── MetaAgentManager
                    └── claude --continue ... ← Moni (money-brain meta-agent)
                          └── cc-agent MCP
                                └── claude ... ← per-job agents (each has gitkb)
```

---

## Workspace Locations

- money-brain meta-agent: `~/cc-discord-workspace/money-brain/`
- cron meta-agent: `~/cc-discord-workspace/cron/`
- cc-suite-tests meta-agent: `~/cc-discord-workspace/cc-suite-tests/`
- Per-job agents: cloned into temp dirs by cc-agent, gitkb injected pointing at money-brain KB

---

## launchd Plist (`com.feral.cc-discord`)

Key env vars:
- `DISCORD_BOT_TOKEN` — bot token
- `DISCORD_GUILD_IDS` — guild ID
- `CC_AGENT_TRUSTED_OWNERS` — `gonzih,ecoclaw,simorgh-app`
- `DISCORD_DEFAULT_CATEGORY_ID` — default channel category
- `CLAUDE_CODE_OAUTH_TOKEN` / `CLAUDE_TOKENS` — Claude oauth token
- `REDIS_URL` — `redis://localhost:6379`
- `npm_config_cache` — `/Users/feral/.npm-cache-money-brain`

---

## gitkb Binary

Path: `/opt/homebrew/bin/git-kb`  
Shared KB: `/Users/feral/cc-discord-workspace/money-brain/.kb/`  
Remote access env var: `GITKB_ROOT=/Users/feral/cc-discord-workspace/money-brain`

---

## Stopped Services

- `com.feral.cc-tg.plist` — renamed to `.plist.disabled` (cc-tg deprecated)
- `com.feral.cc-tg-simorgh.plist` — renamed to `.plist.disabled`
