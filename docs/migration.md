# Migration History

## cc-tg — Fully Deprecated

cc-tg (Telegram coordinator) is deprecated and removed. Do not reference it.

**Why:** cc-discord became the primary communication layer. The coordinator pattern proved problematic — a single session routing everything dies from context overload.

**What replaced it:** Per-channel meta-agents in cc-discord. Each Discord channel gets its own autonomous Claude session.

---

## Key Architectural Decisions

**No coordinator.** Coordinator = single session that routes everything. Dies from context overload. Per-channel sessions are isolated and auto-recover.

**gitkb as sole memory system.** File-based memory is Claude Code's native memory. gitkb is the authoritative cross-agent KB. `GITKB_ROOT` env var makes it accessible from any directory.

**cc-agent responsibility boundary.** cc-agent spawns and manages job agents. It is NOT responsible for meta-agent lifecycle — that is cc-discord's job.

**Loop-as-threads removed.** Loop implementation using threads (loop-manager.ts) was removed. Loops are now cron + `--continue` session. See [self-healing-loops.md](self-healing-loops.md).

**Cron moved into cc-discord.** Previously cron fired agent spawns. Now cron fires RPUSH to meta input queue. Same path as a Discord message. Simpler, more transparent, loops for free.

---

## Version History

| Date | Package | Change |
|------|---------|--------|
| 2026-06-18 | cc-wire 0.4.0 | Added cron/meta keys, deprecated coordinator/TG exports |
| 2026-06-18 | cc-agent 0.16.15 | Removed coordinator spawn, added gitkb injection, mock Claude binary |
| 2026-06-18 | cc-discord 0.2.24 | CronEngine, stream-json output, /cron slash commands, dedup |
| 2026-06-18 | cc-discord 0.2.25 | Integration tests isolated to Redis DB 1 |
| 2026-06-18 | cc-agent-ui 0.5.37 | Agents tab shows full tool usage (stream-json events) |
