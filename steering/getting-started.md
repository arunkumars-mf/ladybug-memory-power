# Getting Started with Ladybug Memory

## Quick Setup (5 minutes)

### Step 1: Install Dependencies

```bash
# Create a virtual environment
python3 -m venv ladybug-memory-mcp/venv

# Install packages
ladybug-memory-mcp/venv/bin/pip install mcp fastembed real-ladybug
```

### Step 2: Configure MCP Server

Add to `.kiro/settings/mcp.json`:

```json
{
  "mcpServers": {
    "ladybug-memory": {
      "command": "/absolute/path/to/ladybug-memory-mcp/venv/bin/python",
      "args": ["/absolute/path/to/ladybug-memory-mcp/server.py"],
      "env": {
        "MEMORY_DB_PATH": "/absolute/path/to/.agent-memory/memory.lbug",
        "MEMORY_DEDUP_THRESHOLD": "0.92",
        "MEMORY_EMBEDDING_MODEL": "BAAI/bge-small-en-v1.5"
      },
      "disabled": false,
      "autoApprove": [
        "memory_store", "memory_search", "memory_get", "memory_update",
        "memory_delete", "memory_relate", "memory_traverse",
        "memory_list", "memory_stats", "memory_consolidate"
      ]
    }
  }
}
```

**Important:** Use absolute paths for `command` and `args`. Relative paths cause ENOENT errors on reconnect.

### Step 3: Create Hooks

Create three hook files in `.kiro/hooks/`:

**`.kiro/hooks/persist-memory.kiro.hook`:**
```json
{
  "enabled": true,
  "name": "Persist Learnings to Memory",
  "version": "1",
  "when": { "type": "agentStop" },
  "then": {
    "type": "askAgent",
    "prompt": "Review this conversation for any new learnings, user preferences, decisions, or patterns worth remembering. For each one, call the memory_store tool with appropriate content, category (learning/preference/decision/pattern), tags, and importance (1-5). Skip anything trivial or already stored. Be selective — only store things that would be useful in future conversations."
  }
}
```

**`.kiro/hooks/recall-memory.kiro.hook`:**
```json
{
  "enabled": true,
  "name": "Recall Relevant Memories",
  "version": "1",
  "when": { "type": "promptSubmit" },
  "then": {
    "type": "askAgent",
    "prompt": "Before responding to the user's request, recall relevant context from memory. If the user's message contains a clear topic or question, call memory_search with a query derived from it. If the message is vague, a greeting, or lacks a searchable topic, call memory_list(count=5) instead to load general context. Only call one — not both. If relevant memories are found, incorporate them naturally into your approach. Do not mention the memory system to the user unless they ask about it."
  }
}
```

**`.kiro/hooks/consolidate-memory.kiro.hook`:**
```json
{
  "enabled": true,
  "name": "Consolidate Agent Memory",
  "version": "1",
  "when": { "type": "userTriggered" },
  "then": {
    "type": "askAgent",
    "prompt": "Run memory consolidation: 1) Call memory_consolidate(similarity_threshold=0.80) to find clusters of similar memories. 2) For each cluster, use your judgment to decide whether the memories should be merged — some related memories are intentionally distinct (different concerns, different contexts). 3) For clusters that should merge, write a single comprehensive memory using memory_store that captures all the important details from the cluster members. 4) Call memory_delete on the old individual memories that were merged. 5) Report what was merged and what was kept separate, with brief reasoning."
  }
}
```

### Step 4: Verify

Ask Kiro: "what do you remember about me?"

If the hooks are working, Kiro will search memory and respond with any stored context. On a fresh install, it will say it doesn't have any memories yet — that's correct.

### Step 5: First Memory

Store something manually to verify:

Ask Kiro: "Remember that I prefer Python over JavaScript for backend development"

The agentStop hook will fire and store this as a preference. Next session, the promptSubmit hook will recall it.

## How It Works

```
New session starts
  → promptSubmit hook fires
  → Kiro calls memory_search or memory_list
  → Relevant past memories loaded into context
  → Kiro responds with full context awareness

During conversation
  → Kiro works normally, no extra memory calls
  → (unless you ask about something specific)

Session ends
  → agentStop hook fires
  → Kiro reviews conversation for new learnings
  → Calls memory_store for each new insight
  → Auto-dedup prevents duplicates
  → Topics auto-linked for graph traversal

Periodically (manual)
  → Click "Consolidate Agent Memory" hook
  → Kiro finds similar memories
  → Reviews clusters with judgment
  → Merges where appropriate
  → Cleans up old entries
```

## Testing with In-Memory Mode

For testing without affecting your real memory:

```json
"env": {
  "MEMORY_DB_PATH": ":memory:"
}
```

All data is lost when the server restarts. Useful for testing hooks and tools.
