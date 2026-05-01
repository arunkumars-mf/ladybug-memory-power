# Graph Query Examples for Ladybug Memory

Use `memory_traverse` to run read-only Cypher queries against the memory graph. This is where LadybugDB's graph capabilities shine — finding patterns and connections that flat search can't.

## Basic Queries

### List all topics with memory counts
```cypher
MATCH (t:Topic)<-[:ABOUT]-(m:Memory)
RETURN t.name AS topic, COUNT(m) AS memories
ORDER BY memories DESC;
```

### Find all memories about a specific topic
```cypher
MATCH (m:Memory)-[:ABOUT]->(t:Topic {name: 'python'})
RETURN m.id, m.content, m.importance
ORDER BY m.importance DESC;
```

### Find memories with multiple topics in common
```cypher
MATCH (m:Memory)-[:ABOUT]->(t1:Topic {name: 'architecture'})
MATCH (m)-[:ABOUT]->(t2:Topic {name: 'ladybugdb'})
RETURN m.id, m.content, m.category;
```

## Relationship Queries

### Find all related memories
```cypher
MATCH (m:Memory {id: 5})-[:RELATED_TO]->(related:Memory)
RETURN related.id, related.content, related.category;
```

### Find what superseded a decision
```cypher
MATCH (new:Memory)-[:SUPERSEDES]->(old:Memory)
RETURN new.id, new.content AS current_decision,
       old.id, old.content AS superseded_decision;
```

### Find chains of superseded decisions
```cypher
MATCH path = (latest:Memory)-[:SUPERSEDES*1..5]->(oldest:Memory)
WHERE NOT ()-[:SUPERSEDES]->(latest)
RETURN latest.content AS current,
       [n IN nodes(path) | n.content] AS history;
```

## Analysis Queries

### Most connected memories (by relationship count)
```cypher
MATCH (m:Memory)-[r]-()
RETURN m.id, m.content, COUNT(r) AS connections
ORDER BY connections DESC
LIMIT 10;
```

### Topics that bridge different categories
```cypher
MATCH (m1:Memory)-[:ABOUT]->(t:Topic)<-[:ABOUT]-(m2:Memory)
WHERE m1.category <> m2.category
RETURN t.name AS topic,
       COLLECT(DISTINCT m1.category) AS categories,
       COUNT(DISTINCT m1) + COUNT(DISTINCT m2) AS total_memories
ORDER BY total_memories DESC;
```

### Memories by category and importance
```cypher
MATCH (m:Memory)
RETURN m.category, m.importance, COUNT(*) AS count
ORDER BY m.category, m.importance;
```

### Least accessed memories (candidates for cleanup)
```cypher
MATCH (m:Memory)
WHERE m.access_count = 0
RETURN m.id, m.content, m.category, m.importance, m.created_at
ORDER BY m.created_at ASC
LIMIT 20;
```

### Most accessed memories (your core knowledge)
```cypher
MATCH (m:Memory)
RETURN m.id, m.content, m.category, m.access_count
ORDER BY m.access_count DESC
LIMIT 10;
```

## Code Analysis Queries

These become powerful when storing code analysis results:

### Find all memories about a specific program
```cypher
MATCH (m:Memory)-[:ABOUT]->(t:Topic {name: 'COTRN02C'})
RETURN m.content, m.category
ORDER BY m.importance DESC;
```

### Find programs that are related through memories
```cypher
MATCH (m1:Memory)-[:ABOUT]->(t1:Topic)
MATCH (m1)-[:RELATED_TO]->(m2:Memory)-[:ABOUT]->(t2:Topic)
WHERE t1.name <> t2.name
RETURN t1.name AS source, t2.name AS related_to, m1.content
LIMIT 20;
```

### Find all decisions about a topic
```cypher
MATCH (m:Memory)-[:ABOUT]->(t:Topic {name: 'architecture'})
WHERE m.category = 'decision'
RETURN m.id, m.content, m.importance
ORDER BY m.updated_at DESC;
```

## Tips

- `memory_traverse` blocks write operations (CREATE, DELETE, SET, MERGE) for safety
- Use `memory_relate` to create relationships, not Cypher writes
- Cypher queries are case-sensitive for property values
- Use `LIMIT` to avoid large result sets consuming context window
- Combine with `memory_get` to fetch full content of interesting results
