# VAULT_STRUCTURE.md — Final Vault Architecture

**Version:** 2.0.0
**Scope:** Canonical folder structure for the business cognitive vault
**Status:** Production — approved structure

---

## Final Structure

```text
/home/ubuntu/vaults/business/
├── 00_Inbox/
│   ├── Staging/              # Unprocessed drops awaiting triage
│   │   └── review/           # Items awaiting human review (quality gate 2)
│   ├── Raw/                  # IMMUTABLE — all raw transcripts and imports
│   └── Telegram/             # Telegram capture queue
│
├── 01_Dashboard/
│   ├── daily-briefing.md
│   ├── vault-health.md
│   ├── active-projects.md
│   └── content-calendar.md
│
├── 02_Projects/
│   ├── CinematicOS/
│   ├── FounderOS/
│   ├── IAMBrain/
│   ├── SlowEncodes/
│   └── SocialMedia/
│
├── 03_Areas/                 # Ongoing responsibilities (Finance, Health, Branding)
│
├── 04_Knowledge/             # CANONICAL — global, project-agnostic
│   ├── Concepts/             # Timeless conceptual understanding
│   ├── Entities/             # People, orgs, products, tools, models
│   ├── Internal/             # Proprietary frameworks, lessons, insights
│   └── External/             # Third-party research, benchmarks, analysis
│
├── 05_Content/               # Content production pipeline
│   ├── Ideas/
│   ├── Drafts/
│   ├── Published/
│   ├── Repurposing/
│   └── X-Captures/
│
├── 06_Operations/            # CANONICAL — SOPs and operational procedures
│   └── SOPs/                 # All standard operating procedures
│
├── 07_Archive/
│   ├── Projects/             # Closed project snapshots
│   ├── Knowledge/            # Superseded canonical notes
│   └── Quarantine/           # 14-day hold before permanent archive
│
├── 08_Memory/
│   ├── Operational/          # Active assumptions, TTL-bound
│   ├── Institutional/        # CANONICAL — validated long-term truths
│   └── Syntheses/            # Cross-project synthesis notes
│
├── 08_Templates/             # Note templates
│   ├── project-scaffold.md
│   ├── research-note.md
│   ├── decision-record.md
│   ├── meeting-note.md
│   ├── agent-handoff.md
│   ├── content-idea.md
│   ├── daily-briefing.md
│   ├── entity-page.md
│   ├── concept-page.md
│   └── default.md
│
├── 09_System/
│   ├── SCHEMA.md             # Frontmatter standards (canonical)
│   ├── SKILL.md              # Master operating manual (canonical)
│   ├── AGENT_PROTOCOL.md     # Agent rules (canonical)
│   ├── index.md              # Master vault catalog
│   ├── audit-log.md          # Append-only action log
│   ├── id-counter.txt        # Monotonic ID sequence counter
│   ├── ingestion-manifest.csv # Historical transcript processing status
│   ├── conflicts/            # Unresolved semantic contradictions
│   ├── agent-protocols/      # Role-specific agent instructions
│   └── context-packages/     # Pre-built retrieval bundles for common tasks
│
├── scripts/                  # Automation scripts
│   ├── vault_lint.py
│   ├── index_builder.py
│   ├── dedup_check.py
│   ├── next_id.py
│   ├── validate_schema.py
│   ├── check_id_immutability.py
│   ├── scan_secrets.py
│   └── ttl_check.py
│
├── _assets/                  # Media files (images, attachments)
└── .git/
```

---

## Per-Project Structure

Each project under `02_Projects/{Project}/` follows this structure:

```text
{Project}/
├── 00_Dashboard/             # Project health, sprint status, agent assignments
├── 01_Context/               # Brief, goals, constraints, background
├── 02_Research/
│   ├── Raw/                  # Unprocessed research inputs
│   ├── Refined/              # Cleaned, metadata-tagged
│   └── Promoted/             # Promoted to 04_Knowledge/ (symlink or copy)
├── 03_Strategy/
│   └── Decisions/            # ADRs for this project
├── 04_Execution/
│   ├── Kanban/               # Current board state
│   ├── Tasks/                # Individual task files
│   ├── Sprints/              # Sprint definitions and retrospectives
│   └── Reports/              # Completion reports, delivery artifacts
├── 05_Content/               # Project-specific content (not global 05_Content/)
├── 06_AI_Outputs/            # AGENT WRITE ZONE
│   ├── Brainstormer/
│   ├── Planner/
│   ├── Builder/
│   └── Validator/
├── 07_Assets/                # Project-specific media
├── 08_Memory/
│   ├── Operational/          # Project operational memory (TTL-bound)
│   └── Institutional/        # Project-specific institutional knowledge
├── 09_Meetings/              # Meeting notes
├── 10_Agent_Protocols/       # Agent handoffs and role-specific instructions
│   └── handoffs/             # Agent handoff snapshots
├── 11_Infrastructure/        # Config, architecture diagrams, infra notes
├── 12_Archive/               # Closed sprints, deprecated notes, transcripts
│   └── transcripts/          # Raw transcripts specific to this project
└── README.md                 # Project brief (human-readable)
```

---

## Changes from Previous Version

### What Changed

#### Added `scripts/` as a top-level directory

**Why:** Automation scripts need a stable, version-controlled home. Placing them inside `09_System/` would mix documentation with executable code. A dedicated `scripts/` folder makes them discoverable and easier to invoke from Git hooks.

#### Added `09_System/id-counter.txt`

**Why:** The previous schema described an ID counter without specifying where it lives. This explicit file ensures the Curator can atomically read, increment, and commit the counter in a single Git transaction.

#### Added `09_System/ingestion-manifest.csv`

**Why:** With 200+ historical transcripts, a structured manifest is essential for tracking what has been archived, triaged, distilled, and promoted. A CSV is machine-readable and Git-trackable without needing a database.

#### Added `00_Inbox/Staging/review/` subfolder

**Why:** Quality Gate ② items need a dedicated holding area. Without it, unreviewed items mix with unprocessed drops in `Staging/`, creating ambiguity.

#### Added `06_Operations/SOPs/` explicitly

**Why:** SOPs were listed as canonical knowledge but had no clear folder home. `06_Operations/SOPs/` makes them a first-class canonical zone with a clear retrieval path.

#### Added `SKILL.md`, `AGENT_PROTOCOL.md` to `09_System/`

**Why:** The master operating manual and agent protocol should live in the system directory alongside SCHEMA.md. They are canonical system documents, not content.

#### Added `10_Agent_Protocols/handoffs/` to per-project structure

**Why:** Handoff snapshots need a structured, predictable location. Agents and recovery processes must be able to find the most recent handoff without guessing.

---

### What Was Removed

#### Removed `08_Memory/Syntheses/` from the canonical structure

**Why:** "Syntheses" was an ambiguous category that overlapped with both Institutional memory and canonical knowledge notes. Notes that emerged from synthesis belong in `04_Knowledge/Internal/` (if global) or `08_Memory/Institutional/` (if project-specific). The `Syntheses/` folder created a third option that served no distinct purpose. The folder can remain if content currently lives there, but no new notes should be created in it.

#### Removed `09_System/agent-protocols/` as a subfolder (merged into AGENT_PROTOCOL.md)

**Why:** Having a subfolder of agent protocol files plus a top-level `AGENT_PROTOCOL.md` was redundant. The single canonical document is sufficient. Role-specific extensions can live as sections within `AGENT_PROTOCOL.md` or as separate files in the project-level `10_Agent_Protocols/` folders.

---

### What Should Remain As Placeholders

These folders exist but should contain only placeholder `README.md` files until actual content is promoted:

- `03_Areas/` — Ongoing responsibility areas (Finance, Health, Branding) are defined here but not yet populated
- `04_Knowledge/External/` — External research notes awaiting historical transcript distillation
- `08_Memory/Institutional/` — Will populate as transcripts are distilled

Placeholder format:

```markdown
# {Folder Name}

This folder is reserved for {description}.
No content yet. Populated via the promotion workflow.

See: DISTILLATION_WORKFLOW.md for how content arrives here.
```

---

### What Should Not Exist Yet

Do not create the following until genuine need arises:

- Subfolders within `04_Knowledge/` beyond the four defined (Concepts, Entities, Internal, External)
- Additional agent role folders within `06_AI_Outputs/` beyond the four defined
- Any vector database or SQLite file committed to Git (these are gitignored, not committed)
- A `10_Metrics/` or similar analytics folder — not warranted at current scale
- Project-specific `06_Operations/` folders — all SOPs live in the global `06_Operations/SOPs/`
- A `00_Inbox/Processed/` folder — processed items are routed directly to their destination

---

## Layer-to-Folder Mapping

Quick reference for the four-layer memory architecture mapped to vault paths:

| Layer | Label | Primary paths |
|---|---|---|
| Layer 0 | Raw Archive | `00_Inbox/Raw/`, `02_Projects/{P}/12_Archive/transcripts/` |
| Layer 1 | Canonical Knowledge | `04_Knowledge/`, `06_Operations/SOPs/`, `08_Memory/Institutional/`, `02_Projects/{P}/03_Strategy/Decisions/` |
| Layer 2 | Operational Memory | `08_Memory/Operational/`, `02_Projects/{P}/08_Memory/Operational/`, `02_Projects/{P}/10_Agent_Protocols/handoffs/` |
| Layer 3 | Execution Artifacts | `02_Projects/{P}/06_AI_Outputs/`, `02_Projects/{P}/04_Execution/`, `00_Inbox/Staging/` |

---

## Gitignore Recommendations

```gitignore
# Derived artifacts — always rebuildable
vault.db
vectors.db
*.embeddings
.chroma/

# Python
__pycache__/
*.pyc
.venv/

# OS
.DS_Store
Thumbs.db

# Secrets (should never exist in vault, but belt-and-suspenders)
*.env
.env.*
*_secret*
*_key.txt

# Obsidian workspace
.obsidian/workspace.json
.obsidian/cache
```

Everything else — including all `.md` notes, the CSV manifest, the `id-counter.txt`, and all system files — is committed to Git.
