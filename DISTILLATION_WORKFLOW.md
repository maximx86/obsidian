# DISTILLATION_WORKFLOW.md — Transcript Distillation Operating Manual

**Version:** 2.0.0
**Scope:** Complete workflow for converting raw AI reasoning transcripts into canonical vault knowledge
**Authority:** Human founder + Vault Manager

---

## Core Principle

A raw AI reasoning transcript is not knowledge. It is:

- A trace of a thinking process
- A record of alternatives considered and rejected
- A scaffold that led to a conclusion
- A recovery artifact

**The conclusion is the knowledge. The scaffold is the archive.**

Distillation is the process of extracting the 5–10% signal from a transcript and discarding the 90–95% scaffolding — without losing provenance.

---

## What Gets Distilled

| Extract | Why |
|---|---|
| Strategic decisions with clear rationale | Drive future work and prevent re-debating settled questions |
| Architectural choices affecting ≥2 future agents | Ensure new agents don't inadvertently reverse key decisions |
| Reusable frameworks or mental models | Institutional capital that compounds over time |
| SOPs that will be executed ≥2 times | Procedural knowledge should not live only in one person's head |
| Validated facts about entities | Competitor analysis, tool evaluations, external research |
| Operational state summaries | Enable agent continuity without re-reading the transcript |

## What Does NOT Get Distilled

| Skip | Why |
|---|---|
| Reasoning traces that led nowhere | Scaffolding with no residual value |
| Discarded alternatives | Preserved implicitly in archive if recovery needed |
| Model hedging and caveats | Not knowledge — editorial noise |
| Speculative outputs without decisions | Premature; distill only when acted upon |
| Low-confidence claims with no corroboration | Noise; would pollute canonical retrieval |
| Implementation details that will be rewritten | Too ephemeral to canonicalise |

---

## Distillation Pipeline

### Phase 1 — Triage

Before distilling anything, every transcript must be scored.

**Triage scoring:**

| Score | Label | Criteria |
|---|---|---|
| 5 | Critical | Contains foundational architectural decisions, unique strategic insights, or irreplaceable reasoning |
| 4 | High | Contains validated conclusions, concrete plans, or frameworks applicable beyond one sprint |
| 3 | Medium | Contains some useful concepts mixed with significant scaffolding |
| 2 | Low | Mostly exploratory; may contain one or two isolated insights |
| 1 | Archive | Scaffolding, brainstorming, discarded alternatives — no extractable value |

**Triage process:**

1. Read the first 20% and last 20% of the transcript
2. Scan for section headers or conclusion markers
3. Ask: "If I lost this transcript today, would I lose knowledge I don't have elsewhere?"
4. If yes → score 3–5. If no → score 1–2.
5. Log score in `09_System/ingestion-manifest.csv`

**Routing by score:**

- Score 5: Full distillation + human review gate
- Score 4: Full distillation + automated quality gate
- Score 3: Selective distillation (concepts only; skip reasoning traces)
- Score 2: Spot-check only; archive unless agent finds explicit signal
- Score 1: Archive to `00_Inbox/Raw/` with no distillation

---

### Phase 2 — Archive

Regardless of distillation priority, every transcript is first archived.

```bash
# Copy to immutable raw archive
cp transcript.md 00_Inbox/Raw/YYYY-MM-DD-{slug}.md

# Add minimal metadata header to the raw copy
# (do not modify the content below)
```

Raw archive frontmatter (minimal — do not over-engineer):

```yaml
---
title: "Raw transcript — {slug}"
type: raw_transcript
status: archived
created: YYYY-MM-DD
source: claude | grok | sonnet | other
project: {PROJECT or 'unknown'}
triage_score: 1-5
distillation_status: pending | in-progress | complete | archive-only
---
```

This file is now immutable. It will never be modified. It serves only as provenance.

---

### Phase 3 — Segmentation

For transcripts scoring 3–5, segment the transcript into typed chunks before distilling.

**Segment types:**

- `decision` — A clear choice was made between alternatives
- `concept` — A framework, mental model, or principle was articulated
- `sop_candidate` — A repeatable process was described
- `plan` — An implementation plan or task breakdown was produced
- `research` — External facts, benchmarks, or entity analysis
- `operational` — Current state, blockers, assumptions relevant to active work
- `noise` — Everything else (reasoning scaffolding, hedging, discarded alternatives)

**Segmentation approach:**

1. Read the full transcript
2. Mark boundaries using comments or a separate segmentation document
3. Assign each segment a type and an estimated quality score (high/medium/low)
4. Discard all `noise` segments — do not carry them into the distillation run
5. Proceed with distillation on the remaining typed segments only

---

### Phase 4 — Distillation Run

Apply the distillation template to the segmented transcript. One distillation output file per transcript.

**Output path:**

```
00_Inbox/Staging/distillation-YYYY-MM-DD-{slug}.md
```

**Distillation agent instructions (include in system prompt):**

```
You are processing a segmented reasoning transcript for canonical distillation.

Rules:
1. Extract ONLY conclusions, decisions, and durable knowledge.
2. Do NOT include reasoning scaffolding, discarded alternatives, or hedging.
3. Every extracted note must stand alone without the transcript.
4. Assign confidence: high only if the claim is validated or repeatedly confirmed.
5. Assign confidence: medium if reasonable but single-source.
6. Assign confidence: low if speculative or uncertain — flag for quarantine.
7. Flag any claims that contradict existing canonical notes you are aware of.
8. Use only tags from the SCHEMA.md taxonomy.
9. Operational Memory Snapshot TTL = today + 14 days unless the work is ongoing.
10. Do not pad knowledge notes. 3–8 sentences maximum per extracted note.
11. Do not invent canonical IDs — leave the id field blank.
```

---

## Distillation Template

Use this template for every distillation run. The output of this template is a staging file — not a canonical note yet.

```markdown
---
# DISTILLATION OUTPUT — NOT a canonical note
# Curator assigns IDs during promotion
distillation_of: "00_Inbox/Raw/YYYY-MM-DD-slug.md"
distilled_by: "{agent_name}"
distilled_at: YYYY-MM-DD
project: {PROJECT}
confidence_overall: high | medium | low
triage_score: 3 | 4 | 5
---

## 1. Strategic Summary

<!-- 3–5 sentences. Conclusions only. No reasoning trail. No hedging.
     Answer: what was decided, what changed, what must be remembered forever. -->

## 2. Key Decisions (ADR format)

<!-- One block per decision. Skip if no clear decision was made. -->

### Decision: {Title}

- **Context:** Why was this decision necessary?
- **Options considered:** (2–3 options, brief)
- **Decision made:** {exact choice}
- **Rationale:** Why this option over alternatives
- **Consequences:** Known tradeoffs or downstream effects
- **Status:** proposed | accepted | deprecated
- **Confidence:** high | medium | low
- **Promote to:** `02_Projects/{P}/03_Strategy/Decisions/adr-{NNN}-{slug}.md`

---

## 3. Extracted Knowledge Notes

<!-- One block per extractable knowledge item.
     Each becomes one canonical note after promotion.
     3–8 sentences per note. No padding. No scaffolding. -->

### KN-{slug}

- **Suggested type:** concept | framework | entity | research
- **Title:** {Concise title}
- **Body:** {Distilled knowledge. Self-contained. No references to the transcript.}
- **Tags:** {from SCHEMA.md taxonomy}
- **Confidence:** high | medium | low
- **Canonicality candidate:** 3 | 4
- **Promote to:** `04_Knowledge/{Concepts|Entities|Internal|External}/`

---

## 4. SOP Candidates

<!-- Only if a repeatable process was explicitly defined or clearly implied.
     Skip if the process is one-off or project-specific. -->

### SOP: {Title}

- **Trigger:** When does this process start?
- **Steps:** (numbered list — concrete and actionable)
  1. Step one
  2. Step two
- **Owner:** {agent role or human}
- **Cadence:** one-time | recurring | on-trigger
- **Confidence:** high | medium | low
- **Promote to:** `06_Operations/SOPs/sop-{domain}-{slug}.md`

---

## 5. Open Questions

<!-- Items requiring future research, validation, or decisions.
     These do NOT get promoted to canonical. They go to project task lists. -->

- [ ] {Question} — owner: {human/agent} — deadline: {date or milestone}
- [ ] {Question} — owner: {human/agent} — deadline: {date or milestone}

---

## 6. Risks and Assumptions

<!-- Explicit assumptions embedded in the reasoning that, if wrong, would
     invalidate one or more extracted knowledge notes. -->

| Assumption | Risk if wrong | Confidence |
|---|---|---|
| {assumption 1} | {consequence} | high/medium/low |

---

## 7. Execution Artifacts

<!-- If the transcript produced actionable tasks or plans, note them here.
     Do not duplicate them — just reference or route them. -->

- **Related project:** {project name}
- **Tasks to create:** {brief list or link to task file}
- **Implementation notes:** {any notes specifically for the Builder agent}

---

## 8. Operational Memory Snapshot

<!-- What must an agent know to resume this work RIGHT NOW.
     This section becomes a Layer 2 operational memory note. -->

- **Current state:** {one sentence — where the project stands}
- **Active assumptions:** {list the assumptions being made now}
- **Blockers:** {list any current blockers}
- **Next action:** {single specific next action}
- **TTL:** {YYYY-MM-DD — today + 14 days default}

---

## 9. Source Provenance

- **Raw transcript:** `[[00_Inbox/Raw/YYYY-MM-DD-slug.md]]`
- **Key lines/sections:** {line ranges or section headers containing the key material}
- **Date of conversation:** YYYY-MM-DD
- **Model(s) used:** Claude Sonnet / Grok / other
- **Other related transcripts:** {links if known}
- **Distillation confidence note:** {anything the Curator should know about the quality of this extraction}
```

---

## Quality Gates

### Quality Gate ① — Automated

Applied immediately after distillation, before human review.

**Auto-quarantine if:**
- `confidence_overall: low` AND no section contains a clear strategic decision
- Strategic Summary is empty or only restates the transcript topic
- All extracted knowledge notes have `confidence: low`
- Operational Memory Snapshot has no `next action`

**Auto-pass if:**
- At least one section has `confidence: high`
- Strategic Summary contains at least one concrete conclusion
- At least one Knowledge Note or ADR is extractable

**Quarantine path:** `07_Archive/Quarantine/distillation-YYYY-MM-DD-{slug}-quarantined.md`
**Reason field added:** `quarantine_reason: "gate1-insufficient-signal"`

---

### Quality Gate ② — Human Checkpoint

Required for:
- Any extracted note proposed for `canonicality: 4` or `5`
- Any ADR that proposes reversing or modifying an existing canonical decision
- Any extraction where a contradiction with existing canonical notes is flagged
- All distillations from `triage_score: 5` transcripts (full human review)

**Hold path:** `00_Inbox/Staging/review/{slug}.md`
**Human action required:** approve, reject, or modify each extracted item

---

## Deduplication Process

Before any knowledge note is promoted, it must be checked for near-duplication.

**Process:**

1. Run `scripts/dedup_check.py` against the proposed note
2. If similarity score > 0.70 against an existing canonical note:
   - If older note is better → quarantine new, add provenance pointer to old
   - If new note is better → promote new, archive old with `superseded_by`
   - If approximately equal → merge: keep older ID, update body, update `updated` date
3. If no match found → proceed to promotion

**Dedup threshold:** 0.70 cosine similarity (adjustable in `scripts/dedup_check.py`)

---

## Promotion Process

After both quality gates:

1. Curator assigns `id: knw-YYYYMMDD-NNNNN` to each note
2. Curator creates the canonical note file in the appropriate path
3. Curator adds cross-links (`[[wikilinks]]` to related canonical notes)
4. Curator sets `canonical: true`, `status: promoted`
5. Curator updates `09_System/index.md`
6. Curator commits on `promote/{date}-{slug}` branch
7. Curator merges to `main`
8. Curator appends to `09_System/audit-log.md`
9. Curator updates `09_System/ingestion-manifest.csv`

---

## Promotion Destinations

| Extracted item | Destination |
|---|---|
| Strategic decision → ADR | `02_Projects/{P}/03_Strategy/Decisions/adr-{NNN}-{slug}.md` |
| Reusable concept | `04_Knowledge/Concepts/{slug}.md` |
| Entity (person, org, tool) | `04_Knowledge/Entities/{slug}.md` |
| Proprietary insight/framework | `04_Knowledge/Internal/{slug}.md` |
| External research/benchmarks | `04_Knowledge/External/{slug}.md` |
| SOP | `06_Operations/SOPs/sop-{domain}-{slug}.md` |
| Institutional truth | `08_Memory/Institutional/{slug}.md` |
| Operational snapshot | `08_Memory/Operational/{slug}.md` (with TTL) |

---

## Bad Extraction Examples

These are examples of what NOT to produce. The Curator rejects these.

### Bad: Transcript scaffolding preserved

```markdown
### KN-reasoning-about-models
- **Body:** "We explored several options. Claude is good at reasoning tasks but we weren't sure
  about Grok. After a long discussion we tentatively concluded that Claude might be better
  for most cases but it depends on the situation and we need more testing."
```

**Problem:** This is the reasoning trace, not the conclusion. It has no stable knowledge value. Rejected.

**Correct version:**

```markdown
### KN-model-selection-for-reasoning
- **Body:** "Claude Sonnet is the preferred model for reasoning-heavy tasks in this system.
  Grok is evaluated for real-time web retrieval tasks only. This decision is revisable
  after benchmark testing in Q3 2026."
- **Confidence:** medium
```

---

### Bad: Over-padded knowledge note

```markdown
### KN-git-branches
- **Body:** "Git branches are a feature of Git version control systems that allow parallel
  development. In this system, we have decided that agents should use branches because
  concurrent writes to main can cause conflicts. Therefore agents use branches. The branch
  strategy we are using involves agents creating their own branches. This is important because
  of concurrency. Git was created by Linus Torvalds."
```

**Problem:** 70% is filler. The actual knowledge is one sentence. Rejected.

**Correct version:**

```markdown
### KN-git-agent-isolation
- **Body:** "Autonomous agents work exclusively on `agent/{name}/{task-slug}` branches.
  No agent pushes directly to `main`. All merges to `main` are serialized through the
  Curator. This prevents concurrent canonical write corruption."
- **Confidence:** high
```

---

### Bad: SOP without concrete steps

```markdown
### SOP: Handle new transcripts
- **Steps:**
  1. Look at the transcript
  2. Decide what to do with it
  3. Process it appropriately
```

**Problem:** Steps are so vague they provide no operational value. Rejected.

---

### Bad: ADR without a real decision

```markdown
### Decision: Thinking about the memory architecture
- **Decision made:** We discussed various options and may decide later
```

**Problem:** No decision was made. This is a topic, not an ADR. Rejected. Move to Open Questions.

---

## Anti-Patterns

1. **Distilling priority 1–2 transcripts** without explicit human request
2. **Preserving the reasoning trail** instead of the conclusion
3. **Over-promoting operational context** as canonical knowledge
4. **Creating SOPs from single mentions** of a process
5. **Skipping the archive step** — always archive raw first
6. **Distilling a transcript the same day** it was created, before the thinking has settled
7. **Using the transcript author's words** verbatim in knowledge notes (rewrite to extract the idea)
8. **Promoting knowledge notes with confidence: low** — quarantine them instead
9. **Creating one giant knowledge note** per transcript instead of multiple atomic notes
10. **Losing provenance** — every promoted note must link back to its source transcript
