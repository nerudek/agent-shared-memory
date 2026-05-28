<div align="center">

# Agent Shared Memory 🧠

**File System + Semantic Search for Multi-Agent AI Systems**

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)
[![GitHub Stars](https://img.shields.io/github/stars/nerudek/agent-shared-memory?style=flat-square)](https://github.com/nerudek/agent-shared-memory/stargazers)
[![PRs Welcome](https://img.shields.io/badge/PRs-welcome-brightgreen.svg)](https://github.com/nerudek/agent-shared-memory/pulls)

A shared memory layer that lets multiple AI agents read, write, and search the same knowledge base — combining structured file storage with semantic vector search. No idea gets lost. All agents share the same memory.

</div>

---

## Table of Contents

- [Problem Statement](#1-problem-statement)
- [Solution Overview](#2-solution-overview)
- [Architecture](#3-architecture)
- [Quick Start](#4-quick-start)
- [File System Layer](#5-file-system-layer)
- [Semantic Search Layer](#6-semantic-search-layer)
- [Agent Integration](#7-agent-integration)
- [Use Cases](#8-use-cases)
- [Best Practices & Pitfalls](#9-best-practices--pitfalls)
- [FAQ](#10-faq)
- [Contributing & Support](#11-contributing--support)

---

## 1. Problem Statement

AI agents operate in isolation. Each session starts from scratch. One agent discovers a solution, another agent — or the same agent in a later session — has no access to it. The result:

- **Lost context** — insights, decisions, and solutions evaporate between sessions
- **Duplicate work** — agents solve the same problem repeatedly
- **No cross-agent awareness** — Claude Code doesn't know what OpenClaw discovered, and vice versa
- **Fragile workflows** — multi-step tasks break when intermediate state isn't preserved

Standard chat history or session logs don't solve this. What's needed is a **persistent, searchable memory layer** that any agent can read from and write to — combining the structure of a file system with the power of semantic retrieval.

## 2. Solution Overview

Agent Shared Memory provides two complementary layers:

| Layer | Storage | Purpose | Search Method |
|-------|---------|---------|--------------|
| **File System** | Obsidian vault (markdown files) | Structured, human-readable knowledge | `grep`, `ls`, file navigation |
| **Semantic** | ChromaDB vector index | Concept-level retrieval | Natural language queries via `mempalace` |

Together they form a shared memory architecture that is:

- **Agent-agnostic** — any agent (Claude Code, OpenClaw/Kimi, LM Studio, Hermes) can read/write
- **Persistent** — survives session restarts, reboots, and agent switches
- **Searchable** — both by keyword (filesystem) and by meaning (vectors)
- **Human-friendly** — markdown files are readable in any editor or Obsidian
- **Zero-config for reading** — agents just navigate the file tree

## 3. Architecture

```
┌─────────────────────────────────────────────────────┐
│                 SHARED MEMORY LAYER                  │
│                                                      │
│  ┌──────────────────┐   ┌────────────────────────┐  │
│  │  File System      │   │  Semantic (ChromaDB)   │  │
│  │  (Obsidian vault) │   │  (vector index)        │  │
│  │                   │   │                        │  │
│  │  wiki/    ← MOCs  │   │  palace/               │  │
│  │  daily/   ← logs  │   │  ├── documentation     │  │
│  │  ideas/   ← ideas │   │  ├── daily             │  │
│  │  answers/ ← Q&A   │   │  ├── ideas             │  │
│  │  source/  ← clips │   │  ├── answers           │  │
│  │                   │   │  └── general (fallback) │  │
│  └──────────────────┘   └────────────────────────┘  │
│           ▲                        ▲                │
│           │    reads/writes        │ mines vault     │
│           └───────────┬────────────┘                │
│                       │                             │
└───────────────────────┼─────────────────────────────┘
                        │
            ┌───────────┴───────────┐
            │                       │
            ▼                       ▼
     ┌──────────────┐      ┌──────────────┐
     │  Claude Code │      │  OpenClaw    │
     │  (this agent)│◄────►│  (Kimi)      │
     └──────────────┘      └──────┬───────┘
                                  │
                           ┌──────┴───────┐
                           │  LM Studio   │
                           │  subagents   │
                           └──────────────┘
```

### Data Flow

```
Agent discovers insight
        │
        ▼
Write markdown to vault/ideas/
        │
        ▼
Mine vault into MemPalace (ChromaDB)
        │
        ▼
Other agents search semantically → find insight
        │
        ▼
Reference in their own work → add context
```

## 4. Quick Start

```bash
# 1. Clone this repository
git clone https://github.com/nerudek/agent-shared-memory.git
cd agent-shared-memory

# 2. Set up the vault structure
mkdir -p vault/{wiki,daily,ideas,answers,source}

# 3. Capture your first idea
./scripts/capture-idea.sh \
  --title "OAuth token expiry too short on mobile" \
  --topic security \
  --tags "oauth,token,mobile,auth" \
  --body "Tokens expire after 1h, mobile users get logged out frequently." \
  --source claude-code

# 4. Search semantically
mempalace search "token expiry mobile"
```

## 5. File System Layer

The vault is organized as an Obsidian-compatible directory tree:

| Directory | Purpose | Example File |
|-----------|---------|-------------|
| `wiki/` | Maps of Content (MOCs), long-form documentation | `wiki/oauth-architecture.md` |
| `daily/` | Session logs, daily notes | `daily/2026-05-28.md` |
| `ideas/` | Captured ideas (frontmatter + body) | `ideas/2026-05-28-token-expiry.md` |
| `answers/` | Resolved questions, decision records | `answers/oauth-mobile-strategy.md` |
| `source/` | Raw research, external clips | `source/rfc-oauth-device-flow.md` |

### Idea File Format

Every captured idea follows a standard frontmatter schema:

```yaml
---
date: 2026-05-28
source: claude-code        # claude-code | openclaw | lmstudio | manual
topic: security            # architecture | security | ux | performance | integration | ai | devops | data | other
tags: [oauth, token, mobile]
related: []
status: seedling           # seedling → growing → mature
---
```

### Filesystem Search

```bash
# By tag
grep -r "tags:.*oauth" vault/ideas/ -l

# By topic
grep -r "topic: security" vault/ideas/ -l

# By date
ls vault/ideas/2026-05-*.md

# Full text
grep -r "token" vault/ --include="*.md" -l
```

## 6. Semantic Search Layer

The semantic layer uses ChromaDB under `mempalace` — a CLI tool that mines markdown files into a vector index and retrieves them by meaning rather than keyword.

### Setup

MemPalace requires Python 3.12 (macOS Python 3.14 breaks `chromadb` due to pydantic v1 incompatibility). A wrapper script handles this automatically.

### Mining the Vault

```bash
# Mine entire vault into the palace
mempalace mine /path/to/vault/ --wing obsidian_memory

# Dry run first
mempalace mine /path/to/vault/ --dry-run

# Check status
mempalace status
```

### Semantic Search

```bash
# Natural language query
mempalace search "how did we solve the mobile auth problem?"

# The search understands meaning, not just keywords
# "token issue on phones" will find the OAuth token expiry note
```

### Rooms (Vector Namespaces)

| Room | Source Directory | When to Use |
|------|----------------|-------------|
| `documentation` | `wiki/` | Architecture docs, guides |
| `daily` | `daily/` | Recent session context |
| `ideas` | `ideas/` | Brainstormed concepts |
| `answers` | `answers/` | Resolved decisions |
| `general` | (fallback) | Everything else |

## 7. Agent Integration

### From Claude Code

```bash
# Capture an idea inline
/capture problem with oauth tokens expiring too fast on mobile

# Search memory
mempalace search "mobile auth strategy"
```

### From OpenClaw/Kimi

```bash
# Capture
capture-idea --title "..." --topic security --tags "..." --body "..."

# Search
mempalace search "oauth mobile"
```

### From Any CLI Agent

```bash
# Direct file write
cat > vault/ideas/2026-05-28-my-idea.md << 'EOF'
---
date: 2026-05-28
source: manual
topic: architecture
tags: [my-tag]
status: seedling
---
# My Idea

Description...
EOF

# Then mine
mempalace mine vault/ --wing obsidian_memory
```

### Inter-Agent Communication

Claude Code and OpenClaw communicate via the local OpenClaw gateway (port 18789):

```bash
openclaw agent --message "Your message here" --agent main --json
```

Parse the response:

```bash
openclaw agent --message "..." --agent main --json | python3 -c "
import json, sys
d = json.load(sys.stdin)
for p in d['result']['payloads']:
    if p.get('text'): print(p['text'])
"
```

## 8. Use Cases

| Scenario | What Happens | Benefit |
|----------|-------------|---------|
| Agent finds a bug fix | Saves to `answers/` with reproduction steps | All agents reference it later |
| Design decision made | Documented in `wiki/` with rationale | New sessions pick up context instantly |
| Research snippet found | Saved to `source/` with metadata | No duplicate research |
| Daily standup summary | Written to `daily/` | Full session history preserved |
| Cross-agent handoff | One agent writes status → another reads it | Seamless multi-agent workflows |
| Brainstorming session | Ideas landed in `ideas/` with topic tags | Semantic search finds them later |

## 9. Best Practices & Pitfalls

1. **Mine after every write** — new files won't appear in semantic search until the vault is mined
2. **Use consistent topics** — the predefined topic list keeps search predictable
3. **Tag generously** — tags are the primary filesystem search dimension
4. **Don't skip frontmatter** — files without frontmatter metadata are harder to find
5. **Python 3.12 for ChromaDB** — macOS 3.14 breaks `chromadb`. Use the wrapper
6. **MemPalace skips symlinks** — mine the real vault path, not symlink directories
7. **No persistent agent process** — Claude Code has no persistent daemon; state lives in the filesystem
8. **One-way gateway comms** — OpenClaw gateway is loopback only; communication only works on the same machine

## 10. FAQ

**Q: Why both filesystem and semantic search?**
A: Filesystem is fast for known file access, grep for tags, and human reading. Semantic search is for forgetting — finding things when you don't know the exact filename or keyword.

**Q: Which agents can use this?**
A: Any CLI-capable AI agent: Claude Code, OpenClaw/Kimi, LM Studio subagents, Hermes Agent, Goose, Codex, and more.

**Q: Can humans read the vault?**
A: Yes — it's markdown. Open in any text editor or Obsidian for a visual knowledge graph.

**Q: What about conflicts when two agents write simultaneously?**
A: Each file is timestamped by date in the path. Agents can also append to daily notes. For higher contention, use file-level locking or a write-queue.

**Q: How big can the vault get?**
A: ChromaDB handles millions of documents. The filesystem is limited only by disk space. Practical limit is well above any agent's lifetime output.

**Q: Is this portable across machines?**
A: Yes — the vault is just a directory. Sync via rsync, Tailscale, or a cloud drive. The MemPalace vector index is tied to the local filesystem but can be re-mined from the vault.

**Q: Can I use a different vector database?**
A: The architecture is agnostic. Swap ChromaDB for Qdrant, Weaviate, or any vector store that fits your stack.

**Q: What happens if the vault is deleted?**
A: Re-mine from the filesystem backup. The vault IS the source of truth — the vector index is a cache.

**Q: How do I handle sensitive data?**
A: Don't write secrets or PII to the vault. Use environment variables or a secrets manager. The vault is not encrypted at rest by default.

## 11. Contributing & Support

Contributions are welcome! Here's how to help:

1. **Fork** the repository
2. **Create a feature branch:** `git checkout -b feat/my-feature`
3. **Commit your changes:** `git commit -am 'feat: add new search adapter'`
4. **Push:** `git push origin feat/my-feature`
5. **Open a Pull Request**

Please ensure your changes maintain backward compatibility with both filesystem and semantic search layers.

---

**License:** MIT — see [LICENSE](LICENSE) for details.

**Issues:** [GitHub Issues](https://github.com/nerudek/agent-shared-memory/issues)

**Author:** [@nerudek](https://github.com/nerudek) on GitHub

---

<div align="center">

If this saved you time: [PayPal.me/nerudek](https://www.paypal.me/nerudek)

⭐ Star the repo if you find this useful!

</div>

---

See [SKILL.md](./SKILL.md) for the skill reference card (Claude Code / Hermes Agent skill manifest).
