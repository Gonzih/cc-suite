# Self-Healing Loops

Our implementation of AI agent loops. A cron fires on schedule, sends a structured message to a persistent `--continue` Claude meta-agent. The meta-agent accumulates context across every fire — it gets smarter with each iteration.

This is our alternative to thread-based loops. Simpler, stateful, transparent.

---

## Why It Works

- **Stateful**: `--continue` session persists context. Fire #10 knows everything from fires 1-9.
- **Scheduled**: the cron heartbeat is the loop clock. No runaway loops.
- **Auto-compact**: every N fires, `/compact` is injected before the message to prevent context overflow while preserving key learnings.
- **Emoji kill**: react ❌ or 🛑 to any cron notification in Discord to stop the loop.

---

## Loop Design Principles

Drawn from Anthropic agent patterns, LangGraph, and DSPy research:

1. **Explicit state summary** — Start each prompt with "Previous context: [last N problems, solutions tried, success rate]." Anchors the agent without token explosion.

2. **Structured reasoning with constraints** — ONE fix per fire. Format: Reason → Expected outcome → Rollback plan. Prevents hallucination loops.

3. **Meta-learning flag** — Tag fixes as `PATTERN_MATCH` (seen before, high confidence) vs `EXPLORATORY` (new, reason carefully). Steers toward high-confidence paths on repeat issues.

4. **Bounded depth** — Max 3 fix attempts per fire. If stuck: report blocker, stop. Never spiral.

5. **Clear success criteria** — Define done upfront. Stop when criteria met, not when model feels done.

---

## How Cron Becomes a Loop

```
CronEngine fires (every N minutes)
  → RPUSH cca:discord:meta:{ns}:input "{structured prompt}"
  → MetaAgentManager picks it up (same path as a Discord message)
  → Claude --continue session: accumulated context from all prior fires
  → processes prompt: OBSERVE → DIAGNOSE → ONE FIX → VERIFY → REPORT
  → every compact_every fires: auto-compact preserves learnings, clears tokens
```

The key insight: `--continue` + cron = stateful loop without managing state explicitly. The session IS the state.

---

## Current Loop: cc-suite Self-Healing

- **Cron ID**: `cc-suite-health-fe674cde`
- **Schedule**: every 30 minutes
- **Namespace**: `cron`
- **Goal**: 68/68 tests passing, cc-discord running, zero errors in recent logs
- **Compact every**: 5 fires
- **Test repo**: https://github.com/gonzih/cc-suite-tests
- **Discord channel**: `#cc-suite-tests`

---

## Creating a New Loop

Via slash command in any registered Discord channel:
```
/cron add */30 * * * * <your structured prompt here>
```

Via Redis directly:
```bash
ID="my-loop-$(openssl rand -hex 4)"
redis-cli HSET cca:discord:cron:$ID \
  namespace my-channel-namespace \
  schedule "*/30 * * * *" \
  message "OBSERVE: check X. DIAGNOSE: if broken, why. FIX: one thing. VERIFY: did it work. REPORT: status." \
  enabled true \
  fire_count 0 \
  compact_every 5 \
  created_at "$(date -u +%Y-%m-%dT%H:%M:%SZ)"
redis-cli SADD cca:discord:cron:list $ID
```

Then restart cc-discord so CronEngine picks up the new entry.

---

## Killing a Loop

- React with ❌ or 🛑 to any cron notification message in Discord
- `/cron delete <id>` slash command
- `redis-cli SREM cca:discord:cron:list <id>` + restart cc-discord
