# AUTOMATION_ROADMAP.md — Staged Automation Roadmap

**Version:** 2.0.0
**Scope:** What to automate, when, and how — for a solo founder plus agents
**Philosophy:** Automate the routine. Protect the judgment calls. Build incrementally.

---

## Governing Principle

**Automate only what is already working manually.**

Do not automate a process you haven't done manually at least 5 times. Premature automation of an unstable process creates fragile infrastructure and masks feedback.

The correct sequence is always:
1. Do it manually
2. Document it (that documentation becomes the automation spec)
3. Automate it
4. Verify the automation produces the same result as the manual process

---

## What Should Never Be Automated

These decisions require human judgment and should always stay manual:

| Decision | Why it must stay manual |
|---|---|
| Canonicality 4–5 promotion | Foundational knowledge — wrong calls compound |
| Tier 3+ conflict resolution | Resolving contradictions between validated facts |
| Triage scoring of transcripts | Human understanding of context and project history |
| Structural vault changes | Adding top-level folders, major reorganisation |
| Schema evolution | Every change has downstream consequences |
| ADR approval | Architecture decisions drive future agent behaviour |
| Archiving active project notes | Could destroy ongoing operational context |

---

## What Must Not Be Automated Yet

These are valid future automations but should not be built until their manual equivalents are stable:

| Automation | Prerequisite |
|---|---|
| Auto-promotion of canonicality-3 notes | Must have 50+ manually promoted notes first |
| Agent-triggered distillation runs | Must have completed Phase 3–4 ingestion manually first |
| Automated conflict detection | Must have a stable canonical index first |
| RAG query caching | Must have stable retrieval patterns first |
| Agent memory pruning | Must have manual TTL review process running for 30d |

---

## Stage 1 — Immediate (implement in Week 1)

These automations are simple, high-value, and low-risk. Implement before running any agents.

### 1.1 Pre-commit hooks

**What:** Validate schema, check ID immutability, scan for secrets, protect raw archive.
**Why immediate:** Prevents data corruption from day one.
**Implementation:** See GIT_WORKFLOW.md — Pre-commit Hooks section.
**Effort:** 2 hours

```bash
chmod +x .git/hooks/pre-commit
# Hook content in GIT_WORKFLOW.md
```

### 1.2 Daily snapshot commit

**What:** Commit vault state every night even if nothing changed.
**Why immediate:** Provides clean restore points and ensures remote always has fresh state.
**Implementation:**

```bash
# scripts/daily_snapshot.sh
#!/bin/bash
cd /home/ubuntu/vaults/business
git fetch origin
python3 scripts/vault_lint.py --output 01_Dashboard/vault-health.md
DATE=$(date +%Y-%m-%d)
git add 01_Dashboard/vault-health.md
git commit --allow-empty -m "snapshot(vault): daily snapshot $DATE"
git push origin main
```

**Cron entry:**

```cron
59 23 * * * /bin/bash /home/ubuntu/vaults/business/scripts/daily_snapshot.sh >> /var/log/vault-snapshot.log 2>&1
```

**Effort:** 1 hour

### 1.3 ID counter script

**What:** `scripts/next_id.py` generates the next available canonical ID.
**Why immediate:** Required for any promotion activity.
**Implementation:** See SCHEMA.md — Counter Script section.
**Effort:** 30 minutes

### 1.4 Ingestion manifest

**What:** `09_System/ingestion-manifest.csv` with standard header and template row.
**Why immediate:** Required before Phase 1 of ingestion begins.
**Implementation:** Create the CSV file with the columns defined in INGESTION_PLAN.md.
**Effort:** 15 minutes

---

## Stage 2 — Near-term (implement in Week 2–4)

These require the vault to have some content and the manual workflows to be understood.

### 2.1 Schema validation script

**What:** `scripts/validate_schema.py` — validate all canonical notes against SCHEMA.md rules.
**Why:** Catches missing fields, bad summaries, wrong canonicality values before they reach main.
**Runs:** Pre-commit (on staged files) + daily lint (on all canonical notes).
**Implementation:** See GIT_WORKFLOW.md — scripts/validate_schema.py.
**Effort:** 3 hours

### 2.2 TTL expiry checker

**What:** `scripts/ttl_check.py` — finds all operational memory notes past their TTL.
**Why:** Without this, expired operational notes silently linger in L2 and contaminate agent context.
**Runs:** Daily cron, appends flagged notes to `01_Dashboard/vault-health.md`.

```python
# scripts/ttl_check.py
import pathlib, yaml, re, datetime, sys

OPERATIONAL_PATHS = [
    "08_Memory/Operational/",
    "02_Projects/FounderOS/08_Memory/Operational/",
    "02_Projects/CinematicOS/08_Memory/Operational/",
    # add other projects
]

def parse_frontmatter(filepath):
    text = pathlib.Path(filepath).read_text()
    match = re.match(r'^---\s*\n(.*?)\n---', text, re.DOTALL)
    return yaml.safe_load(match.group(1)) if match else {}

today = datetime.date.today()
expired = []
vault_root = pathlib.Path("/home/ubuntu/vaults/business")

for path in OPERATIONAL_PATHS:
    for f in (vault_root / path).glob("*.md"):
        fm = parse_frontmatter(f)
        ttl_str = fm.get("ttl")
        if ttl_str:
            try:
                ttl = datetime.date.fromisoformat(str(ttl_str))
                if ttl < today:
                    expired.append({"file": str(f), "ttl": ttl, "title": fm.get("title", "")})
            except ValueError:
                pass

if expired:
    print(f"\n## Expired Operational Memory ({today})\n")
    for item in expired:
        print(f"- [{item['title']}]({item['file']}) — expired: {item['ttl']}")
else:
    print(f"No expired operational memory as of {today}.")
```

**Effort:** 2 hours

### 2.3 SQLite index builder

**What:** `scripts/index_builder.py` — builds `vault.db` from all canonical notes.
**Why:** Required before agents can do structured retrieval.
**Runs:** After every promotion batch + on vault restore.
**Implementation:** See RETRIEVAL_ARCHITECTURE.md — Index Rebuild Script.
**Effort:** 3 hours

### 2.4 Vault lint

**What:** `scripts/vault_lint.py` — comprehensive daily health check.
**Why:** Catches orphans, broken links, schema violations, stale notes.
**Runs:** Daily cron, outputs to `01_Dashboard/vault-health.md`.

**Checks to implement:**

```python
# scripts/vault_lint.py
CHECKS = [
    # Schema compliance
    "canonical_notes_have_required_fields",
    "promoted_notes_have_canonical_true",
    "summaries_under_150_chars",
    "canonicality_3_to_5_only_on_canonical",

    # Structural integrity
    "orphan_notes_no_incoming_links",
    "broken_wikilinks",
    "id_uniqueness_no_duplicates",

    # Lifecycle hygiene
    "expired_ttl_notes",
    "notes_past_review_after_date",
    "archived_notes_not_in_active_indexes",

    # Contamination prevention
    "raw_transcripts_not_in_canonical_paths",
    "agent_outputs_not_marked_canonical",
]
```

**Effort:** 4 hours

### 2.5 Broken wikilink detector

**What:** Part of vault lint — finds `[[links]]` that resolve to no file.
**Why:** Broken links silently degrade navigation and cross-reference integrity.

```python
def find_broken_wikilinks(vault_root):
    import re
    broken = []
    all_files = {f.stem.lower() for f in pathlib.Path(vault_root).rglob("*.md")}
    for md_file in pathlib.Path(vault_root).rglob("*.md"):
        content = md_file.read_text()
        links = re.findall(r'\[\[([^\]|]+)(?:\|[^\]]+)?\]\]', content)
        for link in links:
            link_name = link.split('#')[0].strip().lower()
            if link_name and link_name not in all_files:
                broken.append({"file": str(md_file), "broken_link": link})
    return broken
```

**Effort:** 1 hour (part of vault lint)

---

## Stage 3 — Near-term (implement in Week 4–8, during ingestion)

Build these alongside the ingestion pipeline.

### 3.1 Deduplication checker

**What:** `scripts/dedup_check.py` — semantic similarity check before promotion.
**Why:** Prevents duplicate canonical knowledge from polluting retrieval.
**Runs:** Before every promotion batch (manual trigger).
**Implementation:** See SKILL.md — vault hygiene, and DISTILLATION_WORKFLOW.md.

```python
# scripts/dedup_check.py
from sentence_transformers import SentenceTransformer, util
import pathlib, yaml, re, sys

THRESHOLD = float(sys.argv[1]) if len(sys.argv) > 1 else 0.70

model = SentenceTransformer('sentence-transformers/all-MiniLM-L6-v2')
vault_root = pathlib.Path("/home/ubuntu/vaults/business")

def get_canonical_notes():
    notes = []
    for path in ["04_Knowledge/", "06_Operations/SOPs/", "08_Memory/Institutional/"]:
        for f in (vault_root / path).rglob("*.md"):
            text = f.read_text()
            match = re.match(r'^---\s*\n(.*?)\n---\s*\n(.*)', text, re.DOTALL)
            if not match:
                continue
            fm = yaml.safe_load(match.group(1)) or {}
            if fm.get("canonical"):
                body_excerpt = match.group(2)[:300]
                embed_text = f"{fm.get('title','')}. {fm.get('summary','')}. {body_excerpt}"
                notes.append({"id": fm.get("id"), "title": fm.get("title"), "text": embed_text, "file": str(f)})
    return notes

def check_new_note(new_text, canonical_notes):
    canonical_texts = [n["text"] for n in canonical_notes]
    new_emb = model.encode([new_text])
    can_embs = model.encode(canonical_texts)
    scores = util.cos_sim(new_emb, can_embs)[0]
    results = sorted(zip(scores.tolist(), canonical_notes), reverse=True)
    return [(score, note) for score, note in results if score >= THRESHOLD]
```

**Effort:** 3 hours

### 3.2 Vector index builder

**What:** `scripts/vector_index_builder.py` — builds embedding index for semantic search.
**Why:** Enables semantic retrieval beyond exact keyword match.
**Runs:** After every promotion batch + on restore.

```python
# scripts/vector_index_builder.py
"""
Builds semantic embeddings for all canonical notes.
Stores in 09_System/vectors.db using sqlite-vec.
"""
# (see RETRIEVAL_ARCHITECTURE.md for full implementation guidance)
# Minimal version: use a JSON file for embeddings at small scale
import json, pathlib
from sentence_transformers import SentenceTransformer

model = SentenceTransformer('sentence-transformers/all-MiniLM-L6-v2')
vault_root = pathlib.Path("/home/ubuntu/vaults/business")

# Load all canonical notes from vault.db
# Embed: title + summary + body[:300]
# Store as JSON: {id: embedding_list}
# At >5k notes: migrate to sqlite-vec or Chroma
```

**Effort:** 4 hours

### 3.3 Ingestion manifest reporter

**What:** Script that reads `ingestion-manifest.csv` and generates a progress summary.
**Why:** Tracks ingestion progress without manually counting CSV rows.

```bash
# scripts/ingestion_report.sh
python3 -c "
import csv
counts = {}
with open('09_System/ingestion-manifest.csv') as f:
    for row in csv.DictReader(f):
        s = row.get('distillation_status', 'unknown')
        counts[s] = counts.get(s, 0) + 1
print('Ingestion Progress:')
for k,v in sorted(counts.items()):
    print(f'  {k}: {v}')
"
```

**Effort:** 30 minutes

---

## Stage 4 — Future (implement after stable baseline, 3–6 months out)

Do not build these yet. Build them when manual equivalents are stable and the need is clear.

### 4.1 Automated stale note reviewer

Notify human (via Telegram or email) when notes pass their `review_after` date. Do not auto-archive.

**Prerequisite:** 90+ days of `review_after` usage in production.

### 4.2 Automated conflict detection

Run semantic similarity checks between all new promotions and existing canonical notes to flag potential contradictions before Curator review.

**Prerequisite:** Stable vector index with >100 canonical notes.

### 4.3 Context package builder

Pre-build retrieval bundles for common agent tasks (e.g., "start FounderOS sprint", "write a new ADR", "onboard new Builder agent") and cache them in `09_System/context-packages/`.

**Prerequisite:** Stable retrieval patterns identified after 50+ agent sessions.

### 4.4 Auto-promotion pipeline for canonicality-3

Automatically promote distillation outputs that pass all quality gates and score canonicality-3 with high confidence, without requiring human approval for each one.

**Prerequisite:** Quality gate ① and ② have false positive rate < 5% over 30 promotions.

### 4.5 Telegram integration

Receive vault health alerts, escalation notifications, and daily briefing summary via Telegram bot.

**Prerequisite:** All Stage 1–2 automations stable for 30+ days.

---

## Automation Infrastructure Requirements

### Dependencies

```
# scripts/requirements.txt
pyyaml>=6.0
sentence-transformers>=2.2
torch>=2.0          # required by sentence-transformers
sqlite-vec>=0.1     # or chromadb>=0.4
```

### Installation

```bash
pip install -r scripts/requirements.txt
# or: pip install pyyaml sentence-transformers
```

### Cron schedule summary (Stage 1–2)

```cron
# Daily vault lint + health report
00 06 * * * python3 /home/ubuntu/vaults/business/scripts/vault_lint.py --output /home/ubuntu/vaults/business/01_Dashboard/vault-health.md

# TTL expiry check (appends to vault-health.md)
05 06 * * * python3 /home/ubuntu/vaults/business/scripts/ttl_check.py --append --output /home/ubuntu/vaults/business/01_Dashboard/vault-health.md

# Daily snapshot
59 23 * * * /bin/bash /home/ubuntu/vaults/business/scripts/daily_snapshot.sh

# Weekly index rebuild (in case of drift)
00 04 * * 0 python3 /home/ubuntu/vaults/business/scripts/index_builder.py
```

---

## Monitoring and Vault Health Metrics

### Metrics to track (all automated by vault lint)

| Metric | Target | Alert threshold |
|---|---|---|
| Schema compliance rate | 100% | Any failure |
| Orphaned notes | 0 | > 5 |
| Broken wikilinks | 0 | > 10 |
| Expired operational notes | 0 | > 3 |
| Notes past `review_after` | < 10% | > 20% |
| Contested canonical notes | < 5% | > 10% |
| Promotional queue depth | < 20 items | > 50 items |

### Vault health report format

`01_Dashboard/vault-health.md` is regenerated daily:

```markdown
# Vault Health Report — YYYY-MM-DD

## Summary
- Total canonical notes: N
- Schema compliance: N/N (100%)
- Orphaned notes: N
- Broken wikilinks: N
- Expired operational memory: N
- Notes past review_after: N

## Action Required
- [ ] {specific item needing human attention}

## Expired Operational Memory
- [note title](path) — expired YYYY-MM-DD

## Pending Conflicts
- [{id_a} vs {id_b}](09_System/conflicts/) — flagged YYYY-MM-DD

## Promotional Queue
- N items awaiting Curator review in 00_Inbox/Staging/
```

---

## Summary: Automation Timeline

| Week | Automation |
|---|---|
| Week 1 | Pre-commit hooks, daily snapshot, ID counter, ingestion manifest |
| Week 2–3 | Schema validation, TTL checker, vault lint skeleton |
| Week 3–4 | SQLite index builder, broken wikilink detector |
| Week 4–6 | Dedup checker, vector index builder, ingestion reporter |
| Month 3+ | Stale notifier, conflict detection, context packages |
| Month 6+ | Auto-promotion pipeline, Telegram integration |
