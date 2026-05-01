---
name: "ladybug-memory"
displayName: "Ladybug Memory"
description: "Persistent graph memory for AI agents using LadybugDB. Stores learnings, decisions, and preferences with semantic search, auto-dedup, and graph relationships between memories."
keywords: ["memory", "agent-memory", "persistent", "graph", "ladybugdb", "semantic-search", "recall", "learning"]
author: "arunkse"
---

# Ladybug Memory

## Overview

Ladybug Memory gives Kiro persistent memory across sessions. Every conversation starts with context from past sessions — your preferences, decisions, learnings, and patterns are automatically recalled and applied.

Built on LadybugDB, an embedded graph database with native vector search (HNSW) and full-text search. Unlike flat key-value memory stores, Ladybug Memory models knowledge as a graph with typed relationships between memories and topics.

**Key capabilities:**
- Automatic memory persistence after every task (via agentStop hook)
- Automatic context recall at the start of every prompt (via promptSubmit hook)
- Three-layer deduplication prevents duplicate memories
- Graph relationships between memories (RELATED_TO, SUPERSEDES)
- Topic nodes auto-linked from tags for graph traversal
- Hybrid search: semantic (vector) + keyword (FTS) combined
- LLM-driven consolidation to merge related memories intelligently

## Available Steering Files

- **getting-started** — Quick setup: install dependencies, configure MCP, create hooks
- **graph-queries** — Cypher query examples for memory traversal and graph exploration

## First-Time Setup (Agent Instructions)

**IMPORTANT: When this power is first activated, check if the required hooks exist. If they don't, offer to create them for the user.**

On first activation, do the following:

1. Check if these hook files exist in `.kiro/hooks/`:
   - `persist-memory.kiro.hook` (or any agentStop hook that calls memory_store)
   - `recall-memory.kiro.hook` (or any promptSubmit hook that calls memory_search)
   - `consolidate-memory.kiro.hook` (or any userTriggered hook that calls memory_consolidate)

2. If any are missing, tell the user:
   "Ladybug Memory works best with three hooks that automate memory persistence and recall. I can create them for you now — this is a one-time setup. Shall I proceed?"

3. If the user agrees, create the hooks using the `createHook` tool:

   **Hook 1: Persist Learnings**
   - id: `persist-memory`
   - name: `Persist Learnings to Memory`
   - eventType: `agentStop`
   - hookAction: `askAgent`
   - outputPrompt: `Review this conversation for any new learnings, user preferences, decisions, or patterns worth remembering. For each one, call the memory_store tool with appropriate content, category (learning/preference/decision/pattern), tags, and importance (1-5). Skip anything trivial or already stored. Be selective — only store things that would be useful in future conversations.`

   **Hook 2: Recall Context**
   - id: `recall-memory`
   - name: `Recall Relevant Memories`
   - eventType: `promptSubmit`
   - hookAction: `askAgent`
   - outputPrompt: `Before responding to the user's request, recall relevant context from memory. If the user's message contains a clear topic or question, call memory_search with a query derived from it. If the message is vague, a greeting, or lacks a searchable topic, call memory_list(count=5) instead to load general context. Only call one — not both. If relevant memories are found, incorporate them naturally into your approach. Do not mention the memory system to the user unless they ask about it.`

   **Hook 3: Consolidate Memory**
   - id: `consolidate-memory`
   - name: `Consolidate Agent Memory`
   - eventType: `userTriggered`
   - hookAction: `askAgent`
   - outputPrompt: `Run memory consolidation: 1) Call memory_consolidate(similarity_threshold=0.80) to find clusters of similar memories. 2) For each cluster, use your judgment to decide whether the memories should be merged — some related memories are intentionally distinct (different concerns, different contexts). 3) For clusters that should merge, write a single comprehensive memory using memory_store that captures all the important details from the cluster members. 4) Call memory_delete on the old individual memories that were merged. 5) Report what was merged and what was kept separate, with brief reasoning.`

4. After creating hooks, confirm: "Hooks created! Memory will now persist automatically after each task and recall context at the start of each prompt."

## MCP Server: ladybug-memory

### Tools

| Tool | Purpose |
|------|---------|
| `memory_store` | Store a memory with auto-dedup and auto-link to Topic nodes |
| `memory_search` | Hybrid semantic + keyword search, ranked by relevance |
| `memory_get` | Get full untruncated content of a memory by ID |
| `memory_update` | Update content, importance, or tags of a memory |
| `memory_delete` | Delete a memory and all its relationships |
| `memory_relate` | Create RELATED_TO or SUPERSEDES relationship between memories |
| `memory_traverse` | Run read-only Cypher queries against the memory graph |
| `memory_list` | List memories filtered by recency, category, topic, or importance |
| `memory_stats` | Database statistics: counts, categories, topics, relationships |
| `memory_consolidate` | Find clusters of similar memories for review and merging |

### Graph Schema

```
(:Memory) — content, embedding, category, tags, importance, access_count, timestamps
(:Topic)  — auto-created from tags, linked via ABOUT relationship

(:Memory)-[:ABOUT]->(:Topic)         — memory is about a topic
(:Memory)-[:RELATED_TO]->(:Memory)   — memories are related
(:Memory)-[:SUPERSEDES]->(:Memory)   — newer memory replaces older
```

### Tool Usage Examples

**Store a memory:**
```
memory_store(
  content="User prefers Python over Node.js for MCP servers",
  category="preference",
  tags=["python", "mcp", "language"],
  importance=4
)
```

**Search memories:**
```
memory_search(query="what language does the user prefer", top_k=5)
```

**Create a relationship:**
```
memory_relate(from_id=5, to_id=3, relationship="SUPERSEDES")
```

**Run a graph query:**
```
memory_traverse(cypher_query="MATCH (m:Memory)-[:ABOUT]->(t:Topic {name: 'python'}) RETURN m.content, m.importance")
```

**List by topic:**
```
memory_list(count=5, topic="architecture")
```

### Deduplication

Three layers of automatic dedup on every `memory_store` call:

1. **Exact hash** — SHA256 of normalized content. Identical content is rejected, importance bumped.
2. **Semantic similarity** — If cosine similarity > 0.92, merges into existing memory (keeps longer content, merges tags, bumps importance).
3. **Consolidation** — Manual via `memory_consolidate`. Finds clusters of related memories for LLM-driven review and merging.

### Categories

Memories are categorized for filtering:
- `learning` — Technical knowledge, facts, how things work
- `preference` — User preferences and choices
- `decision` — Architecture decisions, tool choices
- `pattern` — Recurring workflows, conventions
- `general` — Everything else

### Importance Scale

1 = trivial, 2 = minor, 3 = normal (default), 4 = important, 5 = critical

Higher importance memories get a relevance boost in search results.

## Recommended Hooks

For the best experience, create these three Kiro hooks:

### 1. Persist Learnings (agentStop)

Automatically extracts and stores learnings after every task:

```json
{
  "name": "Persist Learnings to Memory",
  "version": "1.0.0",
  "when": { "type": "agentStop" },
  "then": {
    "type": "askAgent",
    "prompt": "Review this conversation for any new learnings, user preferences, decisions, or patterns worth remembering. For each one, call the memory_store tool with appropriate content, category (learning/preference/decision/pattern), tags, and importance (1-5). Skip anything trivial or already stored. Be selective — only store things that would be useful in future conversations."
  }
}
```

### 2. Recall Context (promptSubmit)

Loads relevant context before responding to each prompt:

```json
{
  "name": "Recall Relevant Memories",
  "version": "1.0.0",
  "when": { "type": "promptSubmit" },
  "then": {
    "type": "askAgent",
    "prompt": "Before responding to the user's request, recall relevant context from memory. If the user's message contains a clear topic or question, call memory_search with a query derived from it. If the message is vague, a greeting, or lacks a searchable topic, call memory_list(count=5) instead to load general context. Only call one — not both. If relevant memories are found, incorporate them naturally into your approach. Do not mention the memory system to the user unless they ask about it."
  }
}
```

### 3. Consolidate Memory (userTriggered)

Manual button to clean up and merge similar memories:

```json
{
  "name": "Consolidate Agent Memory",
  "version": "1.0.0",
  "when": { "type": "userTriggered" },
  "then": {
    "type": "askAgent",
    "prompt": "Run memory consolidation: 1) Call memory_consolidate(similarity_threshold=0.80) to find clusters of similar memories. 2) For each cluster, use your judgment to decide whether the memories should be merged — some related memories are intentionally distinct (different concerns, different contexts). 3) For clusters that should merge, write a single comprehensive memory using memory_store that captures all the important details from the cluster members. 4) Call memory_delete on the old individual memories that were merged. 5) Report what was merged and what was kept separate, with brief reasoning."
  }
}
```

## Configuration

### Environment Variables

| Variable | Default | Description |
|----------|---------|-------------|
| `MEMORY_DB_PATH` | `~/.agent-memory/memory.lbug` | Path to LadybugDB database |
| `MEMORY_DEDUP_THRESHOLD` | `0.92` | Semantic similarity threshold for auto-dedup |
| `MEMORY_EMBEDDING_MODEL` | `BAAI/bge-small-en-v1.5` | FastEmbed model for embeddings |
| `MEMORY_EMBEDDING_DIM` | `384` | Embedding dimension (must match model) |
| `MEMORY_SEARCH_LIMIT` | `10` | Max results from memory_search |
| `MEMORY_LIST_LIMIT` | `20` | Max results from memory_list |
| `MEMORY_MAX_CONTENT` | `500` | Content truncation length in search/list results |
| `MEMORY_LATENCY_WARN_MS` | `50` | Log warning when operation exceeds this (ms) |

### Testing with In-Memory Mode

Set `MEMORY_DB_PATH=:memory:` for ephemeral in-memory storage (data lost on restart). Useful for testing hooks and tools without affecting your real memory.

## Troubleshooting

### Server won't start

**Error:** `ModuleNotFoundError: No module named 'real_ladybug'`
**Solution:** Install dependencies in the venv:
```bash
pip install real-ladybug fastembed mcp
```

### Vector index error on update

**Error:** `Cannot set property vec in table embeddings because it is used in one or more indexes`
**Solution:** This is handled automatically — `memory_update` uses delete+recreate when content changes. If you see this error, ensure you're using the latest server.py.

### Tags display incorrectly

**Symptom:** Tags show as `["[tag1,tag2]"]` instead of `["tag1", "tag2"]`
**Cause:** Old memories stored with JSON serialization. New memories use comma-separated format.
**Solution:** Re-store affected memories or run consolidation to refresh them.

### First run is slow

**Cause:** FastEmbed downloads the embedding model (~130MB) on first use.
**Solution:** Wait for the download to complete. Subsequent runs are instant.

## Best Practices

- **Be selective when storing** — only store things useful in future conversations
- **Use specific tags** — they become Topic nodes for graph traversal
- **Set importance thoughtfully** — it affects search ranking (5 = critical, 1 = trivial)
- **Run consolidation periodically** — click the Consolidate hook when memory grows large
- **Use memory_traverse for complex queries** — Cypher can find patterns that search can't
- **Use memory_relate for connections** — link related decisions, supersede outdated ones
