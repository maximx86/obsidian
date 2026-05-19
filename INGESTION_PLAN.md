# INGESTION_PLAN.md — Historical Transcript Migration Plan

**Version:** 2.0.0
**Scope:** Import, triage, distillation, and promotion plan for 200+ existing AI reasoning exports
**Status:** Pre-execution

---

## Overview

You have 200+ AI reasoning conversation exports from a multi-LLM distillation workflow. These range from 3,000 to 8,000+ lines and contain a mix of:

- Critical architectural decisions
- Strategic reasoning
- Exploratory scaffolding
- Discarded alternatives
- One-off implementations
- Plain cognitive noise

The goal is not to ingest all of them. The goal is to:

1. Archive all of them safely (zero loss)
2. Extract the 5–10% that contains durable institutional knowledge
3. Promote that knowledge to canonical memory
4. Ignore or quarantine the rest without guilt

**Estimated breakdown (before triage):**
- Priority 5 (critical): ~10–20 transcripts
- Priority 4 (high value): ~40–60 transcripts
- Priority 3 (mixed): ~60–80 transcripts
- Priority 2 (mostly noise): ~40–60 transcripts
- Priority 1 (archive only): ~20–40 transcripts

---

## Phase 0 — Setup (before any ingestion)

Complete these steps before touching any transcript:

1. Confirm vault structure is deployed (see VAULT_STRUCTURE.md)
2. Confirm `09_System/id-counter.txt` exists and contains `0`
3. Confirm `09_System/ingestion-manifest.csv` exists with header row
4. Confirm all pre-commit hooks are installed and passing
5. Confirm `09_System/SCHEMA.md` is finalised
6. Create Git remote and confirm `main` branch is clean

**Manifest header:**

```csv
filename,original_date,project,approx_lines,triage_score,type,distillation_status,notes,canonical_ids
```

---

## Phase 1 — Safe Archival (Week 1)

**Goal:** Get everything into Git safely. Zero distillation. Zero promotion. Zero risk.

### Process

For each transcript:

1. Name the file: `YYYY-MM-DD-{slug}.md` (use conversation date, not today)
2. Add the minimal archive frontmatter (see below)
3. Copy to `00_Inbox/Raw/`
4. Add a row to `09_System/ingestion-manifest.csv`
5. Do NOT modify the content

Batch in groups of 20–30 per Git commit:

```bash
git add 00_Inbox/Raw/
git add 09_System/ingestion-manifest.csv
git commit -m "ingest(vault): archive 25 transcripts batch-01"
```

### Archive frontmatter template

```yaml
---
title: "Raw transcript — {slug}"
type: raw_transcript
status: archived
created: YYYY-MM-DD
source: claude | grok | sonnet | mixed | unknown
project: FounderOS | CinematicOS | IAMBrain | SlowEncodes | SocialMedia | global | unknown
approx_lines: 4200
triage_score: 0
distillation_status: pending
---
```

Set `triage_score: 0` — it will be assigned in Phase 2.

### Success criteria for Phase 1

- [ ] All 200+ transcripts in `00_Inbox/Raw/`
- [ ] All rows added to `09_System/ingestion-manifest.csv`
- [ ] All commits pushed to remote
- [ ] No transcript modified after archiving
- [ ] Git log shows clean ingest commits

**Time estimate:** 2–4 hours (mostly copy + rename + header work)

---

## Phase 2 — Triage Scoring (Week 1–2)

**Goal:** Score every transcript 1–5 without distilling anything.

### Process

For each transcript in `00_Inbox/Raw/`:

1. Read first 15% and last 15% of the transcript
2. Scan for section headers, explicit conclusions, decision markers
3. Ask: "Does this contain knowledge I cannot reconstruct from what's already canonical or from memory?"
4. Assign score 1–5
5. Assign primary project and type classification
6. Update `ingestion-manifest.csv`

**Triage agent prompt:**

```
You are triaging a raw AI reasoning transcript for distillation priority.

Read the transcript and output a JSON object with:
{
  "triage_score": 1-5,
  "project": "FounderOS | CinematicOS | ... | unknown",
  "type": "architecture | strategy | research | planning | brainstorm | mixed",
  "has_decisions": true/false,
  "has_frameworks": true/false,
  "has_sops": true/false,
  "distillation_estimate": "full | selective | archive-only",
  "reason": "one sentence explaining the score"
}

Scoring guide:
5 = Contains foundational decisions or unique insights not available elsewhere
4 = Contains validated conclusions or concrete frameworks
3 = Mixed — some useful concepts, significant scaffolding
2 = Mostly exploratory — possible isolated insight
1 = Archive only — no extractable durable value
```

### Triage batching

Score 20–30 transcripts per agent session. Update manifest after each batch. Commit:

```bash
git add 09_System/ingestion-manifest.csv
git commit -m "ingest(vault): triage scores batch-02 25 transcripts"
```

### Human review of triage scores

After all transcripts are scored, the human reviews:
- All score-5 transcripts (confirm they are truly critical)
- All score-1 transcripts (confirm they are truly archive-only)
- Any "unknown" project assignments

Adjust scores as needed before proceeding to Phase 3.

### Success criteria for Phase 2

- [ ] All 200+ transcripts have a triage score in the manifest
- [ ] All scores reviewed and confirmed by human
- [ ] Manifest committed to Git
- [ ] Estimated counts per score level confirmed

---

## Phase 3 — Critical Distillation (Week 2–3)

**Goal:** Distill all priority-5 transcripts. Full human review gate on all extractions.

### Process

1. Select all transcripts where `triage_score = 5`
2. For each: run full distillation using `DISTILLATION_WORKFLOW.md` template
3. Output to `00_Inbox/Staging/distillation-{date}-{slug}.md`
4. Curator applies quality gate ① (automated)
5. Human reviews ALL extractions from priority-5 transcripts (no exceptions)
6. Human approves, rejects, or modifies each ADR and knowledge note
7. Curator promotes approved notes with assigned IDs
8. Update manifest: `distillation_status = complete`

### Why priority 5 gets full human review

These are the transcripts most likely to contain:
- Foundational decisions that drove weeks of subsequent work
- Frameworks that are now embedded in live systems
- Strategic reasoning that should be explicitly documented

A bad extraction here creates canonical noise that actively misleads future agents. The cost of human review is low; the cost of bad canonical knowledge is high.

### Time estimate

- 10–20 transcripts × ~45 min distillation each = 8–15 hours agent time
- Human review: ~2–3 hours spread over the week
- Curator promotion: ~2 hours

---

## Phase 4 — High-Value Distillation (Week 3–6)

**Goal:** Distill all priority-4 transcripts with automated quality gate.

### Process

1. Select all transcripts where `triage_score = 4`
2. Batch into groups of 10 transcripts per distillation run
3. Run full distillation using template
4. Apply quality gate ①: auto-quarantine low-confidence extractions
5. Apply quality gate ②: human reviews only ADRs and canonicality-4+ candidates
6. Curator promotes passing notes
7. Update manifest weekly

### Batching strategy

Run 2–3 batches per week. Do not attempt to distill the entire priority-4 pool in one session — distillation quality degrades with fatigue (agent or human).

### Deduplication

After each batch, run:

```bash
python3 scripts/dedup_check.py \
  --batch "00_Inbox/Staging/" \
  --canonical "09_System/vault.db" \
  --threshold 0.70
```

Review the dedup report. Merge near-duplicates before promoting.

---

## Phase 5 — Selective Distillation (Week 6–8)

**Goal:** Spot-check priority-3 transcripts; archive priority-1 and priority-2.

### Priority 3 — Selective approach

1. Have the triage agent re-read each priority-3 transcript looking specifically for:
   - An explicit decision with rationale
   - A reusable framework stated concisely
   - An SOP that was defined and would be used again
2. If found: extract that section only and distill it
3. If not found: mark as `distillation_status = archive-only`
4. Do not distill reasoning scaffolding even if it's interesting

### Priority 2 — Archive only

All priority-2 transcripts: set `distillation_status = archive-only`. Done.

### Priority 1 — Archive only

All priority-1 transcripts: set `distillation_status = archive-only`. Done.

**Archive-only does not mean worthless.** It means:
- The raw transcript is preserved in `00_Inbox/Raw/` indefinitely
- If a future agent or human needs to recover reasoning from it, it is available
- It simply doesn't contaminate active canonical retrieval

---

## Deduplication Strategy

### Why dedup matters at this scale

200+ transcripts covering overlapping topics across months will produce many near-duplicate knowledge extractions. Without dedup, canonical memory becomes noisy and self-contradicting.

### Dedup process

1. After each promotion batch, run similarity comparison against existing canonical notes
2. Threshold: cosine similarity > 0.70 = potential duplicate
3. Review pairs with a simple rule:
   - Same concept, older note is better: quarantine new, add provenance note to old
   - Same concept, new note is better: promote new as superseding the old, archive old
   - Complementary (same topic, different angle): keep both, cross-link them

### Cross-project deduplication

Many transcripts likely discuss the same underlying concepts across different projects. Watch for:
- Vault/memory architecture concepts appearing in multiple FounderOS + infrastructure transcripts
- Agent design patterns appearing in multiple sessions
- Git workflow concepts discussed repeatedly

When detected: create one global canonical note in `04_Knowledge/` and link project-specific notes to it.

---

## Quality Control

### What gets promoted

A note is promoted if it passes all of:

1. Quality gate ① (automated): confidence ≥ medium, at least one strategic signal
2. Dedup check: no near-duplicate already canonical
3. Quality gate ② (human): required for canonicality 4–5, ADRs, and P5 transcript extractions
4. At least one wikilink to a related canonical note (added by Curator)

### What gets quarantined

A note is quarantined (not deleted, not promoted) if:

- Confidence = low overall
- Strategic Summary is empty or purely descriptive
- All extracted knowledge notes are confidence = low
- The extraction is a restatement of content already canonical
- The output is reasoning trace, not conclusion

Quarantine path: `07_Archive/Quarantine/distillation-{date}-{slug}-quarantined.md`
Quarantine reason added to frontmatter: `quarantine_reason: "low-signal | duplicate | low-confidence"`

### What gets discarded (treated as archive-only)

- Priority 1–2 transcripts without explicit strategic signal
- Distillation outputs that fail quality gate ① and have no promotable items

"Discard" means: the raw transcript stays in `00_Inbox/Raw/` forever. No distillation output is created. It simply isn't processed.

---

## Risk Management

| Risk | Likelihood | Mitigation |
|---|---|---|
| Canonical noise from low-quality extractions | High | Quality gate ① auto-quarantine + human gate ② |
| Duplicate concepts from overlapping transcripts | High | Dedup check before every promotion |
| Missing critical knowledge in low-scored transcripts | Medium | Human reviews triage scores before Phase 3 |
| Agent hallucination during distillation | Medium | Distillation agent rules + Curator validation |
| Slow progress causing abandoned ingestion | Medium | Phased approach — value delivered incrementally per phase |
| Transcript archive corruption | Low | Git immutability + pre-commit hooks |

---

## Success Metrics

After complete ingestion, measure:

| Metric | Target |
|---|---|
| % of transcripts archived | 100% |
| % of priority-5 transcripts fully distilled | 100% |
| % of priority-4 transcripts fully distilled | ≥ 80% |
| Canonical notes created total | ≥ 50 (estimate) |
| Quarantine rate (distillations quarantined vs attempted) | < 30% |
| Dedup merge rate (near-duplicates found) | < 20% of promoted notes |
| Human time per week during ingestion | ≤ 3 hours |

---

## Manifest Schema

`09_System/ingestion-manifest.csv` — full column specification:

```csv
filename,original_date,project,approx_lines,triage_score,type,distillation_status,quarantine_reason,canonical_ids,notes

# distillation_status values:
#   pending         — not yet triaged
#   archive-only    — triaged, not worth distilling
#   queued          — triaged, queued for distillation
#   in-progress     — distillation run started
#   gate1-fail      — failed quality gate 1, quarantined
#   gate2-hold      — awaiting human review (quality gate 2)
#   complete        — fully distilled and promoted
#   partial         — some items promoted, some quarantined
```

---

## Weekly Review Cadence

During the ingestion period (estimated 6–8 weeks):

| Day | Task |
|---|---|
| Monday | Review gate-2 queue; approve/reject pending ADRs |
| Wednesday | Curator runs promotion batch for approved items |
| Friday | Review dedup reports; resolve near-duplicates |
| Friday | Update manifest; check progress vs targets |

After ingestion is complete, the manifest remains as a permanent provenance record.
