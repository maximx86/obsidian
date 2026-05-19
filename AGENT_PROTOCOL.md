# AGENT_PROTOCOL.md — Autonomous Agent Operational Protocol

**Version:** 2.0.0
**Scope:** Operational rules for all autonomous agents writing to or reading from the vault
**Authority:** Human founder
**Applies to:** Brainstormer, Planner, Builder, Validator, and any future agent roles

---

## Agent Roles

| Role | Primary responsibility | Typical output |
|---|---|---|
| Brainstormer | Ideation, exploration, hypothesis generation | `06_AI_Outputs/Brainstormer/` — raw ideas, concepts |
| Planner | Architecture, strategy, planning | `06_AI_Outputs/Planner/` — plans, ADR drafts, SOPs |
| Builder | Implementation, code, execution | `06_AI_Outputs/Builder/` — code, reports, artifacts |
| Validator | Fact-checking, cross-referencing, review | `06_AI_Outputs/Validator/` — validation reports, conflict flags |
| Curator | Vault governance, promotion, indexing | All canonical zones (exclusive write) |

The Curator role is distinct — it is the only role that promotes notes to canonical status. The Curator may be a human, an automated pipeline, or a dedicated agent, but always operates under elevated scrutiny.

---

## Onboarding Sequence

Every agent, at every session start, must complete this sequence before doing any work.

### Step 1 — Read System Context

```
Read: 09_System/SCHEMA.md
Read: last 20 lines of 09_System/audit-log.md
```

Purpose: Know current conventions and what happened recently.

### Step 2 — Read Project Brief

```
Read: 02_Projects/{PROJECT}/00_Dashboard/
Read: 02_Projects/{PROJECT}/01_Context/
```

Purpose: Understand project goals, current sprint, and active constraints.

### Step 3 — Load Canonical Context (Layer 1)

Run retrieval query:

```
layers: [1]
canonical: true
canonicality: >= 3
project: {PROJECT_NAME}
limit: 15
order_by: relevance_score DESC
```

Load the top 15 results. These are the most established facts about your project.

### Step 4 — Load Operational State (Layer 2)

```
Read: 08_Memory/Operational/ (project-filtered, TTL not expired)
Read: 02_Projects/{PROJECT}/08_Memory/Operational/
Read: most recent 02_Projects/{PROJECT}/10_Agent_Protocols/handoffs/
```

Purpose: Know current assumptions, blockers, and what the previous agent was doing.

### Step 5 — Check Health

```
Read: 01_Dashboard/vault-health.md
```

Note any flagged issues, pending conflicts, or stale notes relevant to your work.

### Step 6 — Confirm Orientation

Before beginning work, confirm:
- [ ] I know what schema version I am operating under
- [ ] I know the current sprint goal
- [ ] I know the most recent canonical decisions affecting my work
- [ ] I know if any contested notes exist in my domain
- [ ] I know what the previous agent was doing and why they stopped

If any of these cannot be confirmed, flag to human before proceeding.

---

## Allowed Write Zones

Agents may write ONLY to these paths:

```
02_Projects/{PROJECT}/06_AI_Outputs/{agent_name}/
00_Inbox/Staging/
08_Memory/Operational/                          (with TTL required)
02_Projects/{PROJECT}/08_Memory/Operational/   (with TTL required)
02_Projects/{PROJECT}/10_Agent_Protocols/handoffs/  (structured format required)
```

### Write rules within allowed zones

1. Every file written must include the minimum frontmatter: `title`, `summary`, `type`, `project`, `status`, `agent`, `canonicality`, `confidence`, `created`
2. `canonicality` must reflect honest assessment — never inflate to force promotion
3. Operational memory notes must include `ttl: YYYY-MM-DD`
4. Never create files with `canonical: true`
5. Never assign an `id:` field — leave blank and let the Curator assign it

---

## Forbidden Write Zones

Agents must never write to:

```
00_Inbox/Raw/
04_Knowledge/
06_Operations/SOPs/
08_Memory/Institutional/
09_System/SCHEMA.md
09_System/index.md
09_System/audit-log.md
09_System/id-counter.txt
02_Projects/{P}/03_Strategy/Decisions/  (Curator-only)
main branch (Git)
```

If an agent finds itself needing to write to a forbidden zone, it must stop, document why in its output file, and escalate to the Curator or human.

---

## Memory Loading Policy

When loading context for a task, follow this retrieval cascade strictly:

```
1. Layer 1 (canonical) — always load first
2. Layer 2 (operational) — load if Layer 1 is insufficient
3. Layer 3 (execution artifacts) — load only for task continuity
4. Layer 0 (raw archive) — NEVER load unless human explicitly says "load raw archive"
```

**Token budget guidance:**
- Maximum context from Layer 1: 15 notes (~3000–4000 tokens)
- Maximum context from Layer 2: 5 notes (~1000 tokens)
- Maximum context from Layer 3: 3 notes (~600 tokens)
- Raw archive: 0 tokens (excluded)

If the top 15 canonical notes are insufficient, inform the human — do not pad with raw transcripts or low-quality operational notes.

---

## Output File Conventions

All agent outputs must follow this structure:

### File path

```
02_Projects/{PROJECT}/06_AI_Outputs/{agent_name}/YYYY-MM-DD-HHMM-{task-slug}.md
```

### Required frontmatter

```yaml
---
title: "Descriptive task title"
summary: "One sentence: what this output contains and its conclusion."
type: agent_output
subtype: plan | research | report | analysis | distillation | handoff
project: FounderOS
status: raw
agent: Builder
canonicality: 2
confidence: high | medium | low
created: YYYY-MM-DD
task_ref: optional task ID or sprint reference
---
```

### Required body structure

```markdown
## Task Context
What was requested. What problem this addresses.

## Output
The actual content — detailed and complete.

## Key Insights
Bullet list. Only genuine insights, not restatements of the task.

## Promotion Candidates
List any insights here that you believe warrant canonical promotion, with a brief justification.
- Candidate: [description] — Reason: [why canonical] — Suggested type: [concept/adr/sop]

## Blockers and Open Questions
- [ ] Question or blocker — owner: [human/agent] — priority: [high/medium/low]

## Dependencies
Links to related canonical notes or execution artifacts this output depends on.
```

---

## Commit Policy

### Before every commit

```bash
git fetch origin
python3 scripts/validate_schema.py --staged
python3 scripts/check_id_immutability.py --staged
python3 scripts/scan_secrets.py --staged
```

### Commit format

```
type(scope): summary

Types: feat | promote | ingest | refactor | fix | archive | lint | conflict | snapshot
Scope: project/{name} | knowledge | content | system | vault
```

### Commit rules

1. Commit only to your own agent branch: `agent/{name}/{task-slug}`
2. Never push to `main` directly
3. Commit atomically — one logical change per commit
4. Include the task reference in commit message when available
5. Never commit files outside your allowed write zones
6. If pre-commit hook fails, fix the issue before committing — never bypass hooks

### When to commit

- After completing a discrete task
- Before starting a new task (close out the previous one)
- At session end (even if work is incomplete)
- Before requesting human review

---

## Promotion Workflow (agents → canonical)

Agents do not promote their own work. The promotion process is:

1. **Agent writes output** to `06_AI_Outputs/{agent}/` with complete frontmatter
2. **Agent flags promotion candidates** in the output file under "## Promotion Candidates"
3. **Agent commits** to their branch
4. **Curator reviews** the output (may be automated for lower-canonicality items)
5. **Curator applies quality gates** (see DISTILLATION_WORKFLOW.md)
6. **Curator assigns ID** if promoting
7. **Curator promotes** to appropriate canonical path
8. **Curator updates index** and commits to `promote/` branch
9. **Curator merges** promote branch to `main`
10. **Curator logs** to audit-log.md

Agents should not expect immediate promotion. The promotion queue may have a delay.

---

## Handoff Procedures

When an agent's session ends or is interrupted, it must write a handoff snapshot.

### Handoff file path

```
02_Projects/{PROJECT}/10_Agent_Protocols/handoffs/YYYY-MM-DD-HHMM-{agent}-handoff.md
```

### Handoff frontmatter

```yaml
---
title: "Handoff — {task-slug} — {agent} — {YYYY-MM-DD}"
type: agent_output
subtype: handoff_snapshot
project: FounderOS
status: raw
agent: Builder
canonicality: 1
confidence: high
created: YYYY-MM-DD
ttl: YYYY-MM-DD  # +90 days default
---
```

### Required handoff body

```markdown
## Current Operational State
- Task: [what was being worked on]
- Progress: [percentage or last completed milestone]
- Last action completed: [specific action and outcome]

## Active Assumptions
List of assumptions currently in play. A successor agent must validate these.
- Assumption: [description] — source: [note ID or transcript]
- Assumption: [description] — validated: [yes/no]

## Blockers
- [ ] Blocker description — owner: [human/agent] — status: [pending/escalated]

## Context to Load (in order)
Successor agent must read these before resuming:
1. [[link to canonical brief or ADR]]
2. [[link to strategy doc]]
3. [[link to most recent execution report]]

## Immediate Next Action
[Single, specific action the successor should take first. Be concrete.]

## What Not to Do
- [Approach that was tried and failed, with reason]
- [Constraint or known pitfall]

## Escalation Triggers
If any of the following happen, stop work and escalate to human:
- [Condition 1]
- [Condition 2]
```

---

## Escalation Rules

Agents must escalate to the human when:

1. A task requires writing to a forbidden zone
2. A Tier 3+ conflict is detected with an existing canonical note
3. An assumption is found to be invalid and it affects active project direction
4. A blocker has been unresolved for more than 48 hours
5. Confidence on a required output cannot be raised above `low`
6. The canonical context is insufficient and raw archive access is needed
7. Two different sources in Layer 1 directly contradict each other
8. A structural vault change is needed

**Escalation format:**

Write an escalation note to `00_Inbox/Staging/escalation-YYYY-MM-DD-{slug}.md`:

```yaml
---
title: "Escalation — {issue-slug} — {YYYY-MM-DD}"
type: agent_output
subtype: escalation
project: {PROJECT}
status: raw
agent: {agent_name}
priority: high | medium | low
created: YYYY-MM-DD
---

## Issue
[Clear description of what requires human decision]

## Context
[Relevant canonical notes, what was attempted]

## Options
[2–3 options with brief analysis]

## Recommended option
[Agent recommendation, with honest confidence level]

## Blocking
[What work is blocked pending this decision]
```

---

## Contradiction Handling

When an agent encounters a claim that conflicts with an existing canonical note:

1. **Do not silently proceed** as if the conflict doesn't exist
2. Note the conflict in your output under a `## Contradictions Detected` section
3. Include both IDs (or note paths if not yet promoted)
4. State which claim you believe is correct and why
5. Flag `contested: true` in your own output note
6. Do not attempt to resolve Tier 3+ contradictions — escalate

**Format for flagging in output:**

```markdown
## Contradictions Detected

Conflict with: [[knw-20260513-00031]]
My claim: [your claim]
Existing canonical claim: [their claim]
My assessment: [which is more likely correct and why]
Confidence in my assessment: high | medium | low
Recommended resolution: [approach]
```

---

## Confidence Handling

Every output must include a `confidence` frontmatter field. Rules:

| Confidence | Meaning | Allowed actions |
|---|---|---|
| high | Strong evidence, verified against multiple sources | Can recommend for canonical promotion |
| medium | Reasonable evidence, single source or partial validation | Can be used as operational context |
| low | Uncertain, speculative, or unverified | Must not be promoted; flag for human review |

If a required output can only be produced at `low` confidence:
1. State this explicitly
2. Explain what would raise confidence
3. Do not present it as established fact
4. Escalate if the decision is blocking

---

## Hallucination Prevention Rules

1. Never assert institutional facts without citing a source from Layer 1 retrieval
2. Never invent canonical note IDs — look them up or leave blank
3. Never claim a decision was made without linking to the ADR or canonical note
4. When uncertain, write "Based on available context..." or "This requires validation against..."
5. Never generate SOPs without explicit human or Curator review
6. Do not extrapolate from partial context — ask for more context instead
7. If your response contradicts something you retrieved, stop and recheck

---

## Rollback Procedures

If an agent's commit caused a problem:

1. Do NOT attempt to fix it in a new commit that adds more changes
2. Create a minimal rollback commit: `git revert HEAD~N`
3. Log the rollback in `09_System/audit-log.md`
4. Write an escalation note explaining what happened
5. Wait for Curator/human review before resuming

If a file was accidentally written to a forbidden zone:
1. Do not modify it further
2. Move it to `07_Archive/Quarantine/` via the Curator (escalate immediately)
3. Log the incident

---

## Session Shutdown Checklist

Before ending any session:

1. [ ] All work committed to agent branch
2. [ ] Handoff snapshot written and committed
3. [ ] All open blockers logged
4. [ ] All promotion candidates flagged in output files
5. [ ] No uncommitted files in working directory (`git status` is clean)
6. [ ] No files accidentally written outside allowed zones
7. [ ] Audit log updated if Curator actions were taken

---

## What Agents Can Do Autonomously

- Read any file in the vault (except secrets)
- Write to allowed zones with proper frontmatter
- Flag contradictions and escalate
- Create promotion candidates (not promotions)
- Write handoff snapshots
- Commit to own branch
- Run validation scripts
- Process files in `00_Inbox/Staging/`
- Write escalation notes

## What Requires Human Approval

- Promoting any note with `canonicality: 4` or `5`
- Resolving Tier 3+ conflicts
- Creating new top-level vault folders
- Modifying `09_System/SCHEMA.md`
- Archiving active project notes
- Any vault structural changes
- Running raw archive queries
- Extending operational memory TTL beyond 90 days

## What Must Never Happen

- Agent writes to `00_Inbox/Raw/` after initial save
- Agent writes to `04_Knowledge/` directly
- Agent sets `canonical: true` on own output
- Agent pushes to `main` branch
- Agent changes an existing `id:` field
- Agent hard-deletes any file
- Agent bypasses pre-commit hooks
- Agent auto-resolves Tier 3+ conflicts
- Agent commits secrets or API keys
- Agent includes raw transcripts in RAG queries
