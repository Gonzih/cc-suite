# gitkb Integration

gitkb (git-kb) is the sole persistent memory system for the cc-suite. It provides 49 MCP tools for creating, searching, linking, and versioning knowledge documents. All agents across all workspaces share the money-brain KB.

CLI: `/opt/homebrew/bin/git-kb`  
MCP transport: stdio (`git-kb mcp`)  
Version: 0.2.12

---

## KB Location

Authoritative shared KB: `/Users/feral/cc-discord-workspace/money-brain/.kb/`

Each meta-agent workspace also gets a local `.kb/` initialized on first message by `ensureWorkspace()` in cc-discord. These are per-workspace KBs — separate from the shared one.

To access the shared money-brain KB from any directory:
```bash
export GITKB_ROOT=/Users/feral/cc-discord-workspace/money-brain
```

---

## How Agents Get gitkb Access

**Meta-agents (cc-discord):**
`injectMcp()` writes `.mcp.json` into each workspace with gitkb as first server.

**Job agents (cc-agent):**
`injectMcpConfig()` in `src/mcp-inject.ts` writes `.mcp.json` with gitkb + `GITKB_ROOT` pointing at the shared money-brain KB. Every spawned job can read and write the shared KB.

---

## Core MCP Tools

| Tool | Use |
|------|-----|
| `kb_list` | Browse documents |
| `kb_show` | Read a document by slug |
| `kb_create` | Create new document |
| `kb_update` | Update content or metadata |
| `kb_search` | Full-text search |
| `kb_semantic` | Semantic similarity search |
| `kb_commit` | Commit staged changes |
| `kb_status` | Show uncommitted changes |
| `kb_log` | Document version history |
| `kb_link` | Link documents (graph edges) |
| `kb_graph` | View relationship graph |

---

## Slug Conventions

```
feedback/{rule-name}     — operating rules and lessons learned
project/{initiative}     — current project state and goals
reference/{resource}     — deployment, credentials, external systems
```

---

## CLI Quick Reference

```bash
git-kb list                       # list all docs
git-kb show feedback/no-amai      # read a doc
git-kb commit --all -m "msg"      # commit all staged changes
git-kb search "redis keys"        # search content
```
