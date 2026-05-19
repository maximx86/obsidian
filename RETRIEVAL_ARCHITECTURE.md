# RETRIEVAL_ARCHITECTURE.md — RAG and Retrieval Architecture

**Version:** 2.0.0
**Scope:** How agents query, rank, and consume vault knowledge
**Optimised for:** Precision over recall, low noise, canonical authority

---

## Design Philosophy

This retrieval architecture is built around one constraint:

**A bad canonical context is worse than no context.**

An agent operating with noisy, contradicting, or speculative context will produce worse outputs than an agent that simply acknowledges it doesn't have enough information and asks.

Therefore:
- Precision always beats recall
- Raw transcripts never enter agent context by default
- Low-canonicality content is filtered before being served
- Contested knowledge is flagged, not silently included
- Short, high-quality context windows beat long, noisy ones

---

## Retrieval Priority (the cascade)

When an agent needs context, it follows this strict cascade. Each layer is queried only if the previous layer did not return sufficient results.

### Layer 1 — Canonical Knowledge (always first)

```
Filter: canonical = true
Filter: canonicality >= 3
Filter: status = promoted
Filter: status != archived
Paths: [
  04_Knowledge/Concepts/,
  04_Knowledge/Entities/,
  04_Knowledge/Internal/,
  04_Knowledge/External/,
  06_Operations/SOPs/,
  08_Memory/Institutional/,
  02_Projects/{P}/03_Strategy/Decisions/
]
Limit: 15 notes
Order by: relevance_score DESC, updated DESC
```

**Sufficient** = 5+ notes clearly relevant to the query. If Layer 1 returns 5+ relevant notes, stop here.

### Layer 2 — Operational Memory (second)

```
Filter: type in [operational_memory, handoff_snapshot]
Filter: project = {PROJECT}
Filter: ttl > today
Paths: [
  08_Memory/Operational/,
  02_Projects/{P}/08_Memory/Operational/,
  02_Projects/{P}/10_Agent_Protocols/handoffs/
]
Limit: 5 notes
Order by: created DESC
```

Used when Layer 1 doesn't cover current operational state (agent handoffs, active assumptions, sprint context).

### Layer 3 — Execution Artifacts (third)

```
Filter: project = {PROJECT}
Filter: status != archived
Filter: created > (today - 30d)
Paths: [
  02_Projects/{P}/06_AI_Outputs/,
  02_Projects/{P}/04_Execution/Reports/
]
Limit: 3 notes
Order by: created DESC
```

Used only when reconstructing recent work context that hasn't been promoted yet.

### Layer 0 — Raw Archive (never by default)

```
EXCLUDED from all default queries.
Only accessible via explicit human instruction:
  "Search the raw archive for transcripts about {topic}"
```

If an agent needs raw archive access, it must escalate. It cannot self-authorise this query.

---

## Index Topology

### Primary Index (SQLite)

**File:** `09_System/vault.db`
**Contents:** All canonical notes with full frontmatter as structured records
**Rebuilt:** After every promotion batch via `scripts/index_builder.py`
**Queries:** Exact match on id, project, type, status, canonicality, tags

```sql
CREATE TABLE canonical_notes (
    id TEXT PRIMARY KEY,
    title TEXT NOT NULL,
    summary TEXT NOT NULL,
    type TEXT NOT NULL,
    project TEXT NOT NULL,
    status TEXT NOT NULL,
    canonical INTEGER NOT NULL,
    canonicality INTEGER NOT NULL,
    confidence TEXT NOT NULL,
    source TEXT,
    agent TEXT,
    created DATE,
    updated DATE,
    review_after DATE,
    ttl DATE,
    contested INTEGER DEFAULT 0,
    superseded_by TEXT,
    file_path TEXT NOT NULL
);
```

### Vector Index (semantic search)

**File:** `09_System/vectors.db` (sqlite-vec) or `09_System/chroma/` (Chroma)
**Contents:** Embeddings of `summary` + first 500 chars of body for all Layer 1 notes
**Rebuilt:** After every promotion batch
**Queries:** Cosine similarity search on natural language queries

**Embedding strategy:**
- Embed: `{title}. {summary}. {body_excerpt}`
- Model: `all-MiniLM-L6-v2` (local, fast) or `text-embedding-3-small` (OpenAI, higher quality)
- Dimension: 384 (MiniLM) or 1536 (OpenAI)
- Do NOT embed raw transcripts

### Operational Index (in-memory or lightweight SQLite)

**File:** `09_System/operational.db` (optional, rebuilt nightly)
**Contents:** Layer 2 operational notes — lighter index, expires with TTL
**Rebuilt:** Nightly cron
**Queries:** Project filter + TTL check

---

## Chunking Strategy

Chunking decisions affect retrieval quality more than almost any other factor.

### Rule 1: Canonical notes are atomic

**Never chunk canonical notes by default.** Each canonical note was written to be a single retrievable unit. Chunking breaks this.

Exception: If a canonical note exceeds ~2000 tokens (approximately 1500 words), split at H2/H3 section boundaries, preserving the note's `id`, `title`, and `summary` in each chunk's metadata.

### Rule 2: Chunk at semantic boundaries

If a note must be chunked:

```
Boundary: H2 or H3 headers
Max chunk size: 800 tokens
Overlap: 50 tokens (include section header in next chunk)
Metadata on each chunk: parent note id, title, summary, section heading
```

### Rule 3: Embed the summary, retrieve the body

For semantic search, embed the `summary` field (concise, retrieval-optimised). When a note is retrieved, load the full body for the agent context window. This separates "what to find" (summary embedding) from "what to read" (full body).

### Rule 4: Operational and execution notes get minimal chunks

Operational memory notes and execution artifacts are short by design (< 500 tokens). Embed and retrieve whole. No chunking needed.

### Chunk size reference

| Note type | Embed | Max chunk | Overlap |
|---|---|---|---|
| Canonical knowledge (<2000 tokens) | Full summary + body excerpt | Full note | None |
| Canonical knowledge (>2000 tokens) | Summary | 800 tokens | 50 tokens |
| Operational memory | Summary + full body | Full note | None |
| Execution artifacts | Summary | 600 tokens | 100 tokens |
| Raw transcripts | NOT INDEXED | — | — |

---

## Query Routing

### Routing logic

```python
def route_query(query: str, context: dict) -> list[RetrievalScope]:
    scopes = []

    # Always start with canonical
    scopes.append(RetrievalScope(
        layer=1,
        filter={"canonical": True, "canonicality__gte": 3},
        limit=15,
        paths=CANONICAL_PATHS
    ))

    # Add operational if task context needed
    if context.get("needs_operational"):
        scopes.append(RetrievalScope(
            layer=2,
            filter={"project": context["project"], "ttl__gte": today()},
            limit=5,
            paths=OPERATIONAL_PATHS
        ))

    # Add execution if recent work context needed
    if context.get("needs_execution"):
        scopes.append(RetrievalScope(
            layer=3,
            filter={"project": context["project"], "created__gte": days_ago(30)},
            limit=3,
            paths=EXECUTION_PATHS
        ))

    # Raw archive: NEVER added automatically
    return scopes
```

### Common query patterns

**Agent starting a new task on FounderOS:**

```python
scopes = [
    L1(project="FounderOS", limit=10),
    L2(project="FounderOS", limit=5),  # most recent handoff + active assumptions
]
```

**Agent looking up a concept:**

```python
scopes = [
    L1(type="concept", limit=8),
    L1(type="adr", project="FounderOS", limit=5),
]
```

**Agent generating an SOP:**

```python
scopes = [
    L1(type="sop", limit=10),
    L1(domain="operations", limit=5),
]
```

**Agent doing research (has recent execution context):**

```python
scopes = [
    L1(project=PROJECT, limit=10),
    L3(project=PROJECT, type="research", limit=3),
]
```

---

## Canonical Filtering

These filters are applied as hard pre-filters before semantic ranking:

```python
CANONICAL_HARD_FILTERS = {
    "canonical": True,
    "status__in": ["promoted"],
    "canonicality__gte": 3,
    "status__not": "archived",
}
```

Notes failing any hard filter are NEVER returned in canonical queries, regardless of semantic similarity.

### Soft filters (ranking adjustments)

These don't exclude — they adjust rank:

- `project match`: +20% relevance boost
- `canonicality == 5`: +15% boost
- `canonicality == 4`: +10% boost
- `updated > 90d ago`: -10% penalty
- `review_after` in the past: -15% penalty (stale)
- `contested == true`: -25% penalty + flag in context

---

## Contradiction Detection

When retrieving notes, run a contradiction check:

```python
def check_contradictions(retrieved_notes: list) -> list[ConflictWarning]:
    warnings = []
    for note in retrieved_notes:
        if note.contested:
            for conflicting_id in note.contradictions:
                conflicting_note = lookup(conflicting_id)
                if conflicting_note in retrieved_notes:
                    warnings.append(ConflictWarning(
                        note_a=note.id,
                        note_b=conflicting_id,
                        description=note.conflict_note
                    ))
    return warnings
```

If contradictions are detected, prepend this to the agent context:

```
⚠️ CONFLICT NOTICE:
Note {id_a} ({title_a}) and Note {id_b} ({title_b}) make contradictory claims.
Conflict description: {conflict_note}
Both are included below. Do not treat either as definitive. Escalate if this blocks your task.
```

---

## Freshness Handling

### Stale detection

A note is considered stale when:
- `review_after` date has passed
- OR `updated` date is > 180 days ago AND the note is `type: operational_memory` or `type: sop`

Stale notes are included in retrieval results but flagged:

```
📅 NOTE: This note ({id}) has not been reviewed since {last_updated}.
It may contain outdated information. Verify before acting on it.
```

### TTL enforcement for operational memory

Operational memory notes (Layer 2) must have a `ttl` date. The daily lint script (`scripts/ttl_check.py`) flags all notes where `ttl < today`.

Flagged notes appear in `01_Dashboard/vault-health.md` under "Expired Operational Memory."

Expired notes are NOT automatically deleted. They are flagged for human review. The human either:
- Extends TTL (if still relevant)
- Promotes the note to institutional memory (if it became durable)
- Archives it (if it's no longer needed)

---

## Retrieval Anti-Patterns

Do not do the following:

### Anti-pattern 1: Including raw transcripts in context

```python
# WRONG
scopes = [L1(...), L2(...), L0(archive=True)]
```

Raw transcripts are 3,000–8,000 lines of reasoning scaffolding. Including them in agent context:
- Floods the context window with noise
- Reduces precision of relevant context
- May include superseded or rejected conclusions

### Anti-pattern 2: Using high recall to compensate for bad queries

```python
# WRONG
scopes = [L1(limit=50, canonicality__gte=1)]
```

50 results at canonicality 1–5 includes speculative output and early-stage brainstorms. Use fewer, higher-quality results.

### Anti-pattern 3: Querying without project filter on project-specific tasks

```python
# WRONG — for a FounderOS task
scopes = [L1(limit=15)]  # no project filter

# RIGHT
scopes = [L1(project="FounderOS", limit=10), L1(project="global", limit=5)]
```

Without a project filter, global concepts dominate and project-specific ADRs get buried.

### Anti-pattern 4: Skipping the cascade

```python
# WRONG — skipping to execution artifacts because canonical feels slow
scopes = [L3(project="FounderOS", limit=20)]
```

Execution artifacts contain unvalidated agent outputs. Starting here bypasses all quality gates.

### Anti-pattern 5: Treating contested notes as authoritative

If `contested: true` notes are included in context, they must be flagged. Never use a contested note as the sole basis for a decision.

---

## Recommended Tooling

| Component | Tool | Why |
|---|---|---|
| Metadata index | SQLite (built-in Python) | Zero dependency, Git-trackable schema, fast queries |
| Vector store (simple) | sqlite-vec | SQLite extension, no external service, sufficient for <50k notes |
| Vector store (scale) | Chroma (local) | Easy migration from sqlite-vec when >50k notes |
| Embeddings (local) | `sentence-transformers/all-MiniLM-L6-v2` | No API cost, runs on CPU, 384-dim, fast |
| Embeddings (quality) | OpenAI `text-embedding-3-small` | Higher quality, costs money, 1536-dim |
| Query interface | Python script or simple API | Keep it simple; no need for a full search server |

### Recommendation

Start with `all-MiniLM-L6-v2` + `sqlite-vec`. Migrate to Chroma + OpenAI embeddings if retrieval quality becomes a bottleneck at >20k canonical notes.

---

## Index Rebuild Script

```python
# scripts/index_builder.py
"""
Rebuilds vault.db (SQLite metadata) and vectors.db (embeddings)
from all canonical notes.

Run after every promotion batch.
Run on vault restore after infrastructure loss.
"""
import pathlib, sqlite3, yaml, re
from sentence_transformers import SentenceTransformer

CANONICAL_PATHS = [
    "04_Knowledge/",
    "06_Operations/SOPs/",
    "08_Memory/Institutional/",
]

def parse_frontmatter(filepath):
    text = pathlib.Path(filepath).read_text()
    match = re.match(r'^---\s*\n(.*?)\n---\s*\n(.*)', text, re.DOTALL)
    if not match:
        return {}, text
    fm = yaml.safe_load(match.group(1)) or {}
    body = match.group(2)
    return fm, body

def rebuild_sqlite(conn, vault_root):
    conn.execute("DROP TABLE IF EXISTS canonical_notes")
    conn.execute("""CREATE TABLE canonical_notes (
        id TEXT PRIMARY KEY, title TEXT, summary TEXT, type TEXT,
        project TEXT, status TEXT, canonical INTEGER, canonicality INTEGER,
        confidence TEXT, source TEXT, agent TEXT, created TEXT, updated TEXT,
        review_after TEXT, ttl TEXT, contested INTEGER, superseded_by TEXT,
        file_path TEXT
    )""")
    for base_path in CANONICAL_PATHS:
        for md_file in pathlib.Path(vault_root / base_path).rglob("*.md"):
            fm, _ = parse_frontmatter(md_file)
            if not fm.get("canonical"):
                continue
            conn.execute("INSERT OR REPLACE INTO canonical_notes VALUES (?,?,?,?,?,?,?,?,?,?,?,?,?,?,?,?,?,?)", (
                fm.get("id",""), fm.get("title",""), fm.get("summary",""),
                fm.get("type",""), fm.get("project",""), fm.get("status",""),
                int(bool(fm.get("canonical"))), fm.get("canonicality",0),
                fm.get("confidence",""), fm.get("source",""), fm.get("agent",""),
                str(fm.get("created","")), str(fm.get("updated","")),
                str(fm.get("review_after","")), str(fm.get("ttl","")),
                int(bool(fm.get("contested"))), fm.get("superseded_by",""),
                str(md_file)
            ))
    conn.commit()

if __name__ == "__main__":
    vault_root = pathlib.Path("/home/ubuntu/vaults/business")
    conn = sqlite3.connect(str(vault_root / "09_System/vault.db"))
    rebuild_sqlite(conn, vault_root)
    print(f"Index rebuilt: {conn.execute('SELECT COUNT(*) FROM canonical_notes').fetchone()[0]} canonical notes")
```
