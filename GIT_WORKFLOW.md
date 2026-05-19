# GIT_WORKFLOW.md — Git Operating Model

**Version:** 2.0.0
**Scope:** Branch strategy, commit conventions, promotion workflow, and disaster recovery for the vault
**Authority:** Human founder

---

## Design Goals

1. **Canonical main is always clean.** Nothing reaches `main` without validation.
2. **Agents are isolated.** An agent can never corrupt canonical state directly.
3. **Promotion is serialized.** No concurrent writes to `main` from multiple promotion batches.
4. **History is complete.** Every action is traceable in Git log + audit log.
5. **Recovery is always possible.** The vault can be fully restored from Git alone.

---

## Branch Strategy

### Branch Types

| Branch pattern | Purpose | Creator | Lifetime |
|---|---|---|---|
| `main` | Canonical vault state. Always valid. | System | Permanent |
| `agent/{name}/{task-slug}` | Agent working branches. All agent writes go here. | Agent | Per task |
| `promote/{YYYY-MM-DD}-{slug}` | Curator promotion batches. One at a time. | Curator | Per promotion |
| `ingest/{YYYY-MM-DD}-batch-N` | Bulk historical transcript ingestion. | Curator | Per batch |
| `hotfix/{slug}` | Urgent canonical corrections (typos, broken links, wrong IDs). | Human | Per fix |

### Branch Rules

1. No one pushes directly to `main` — only fast-forward merges from `promote/` or `hotfix/` branches
2. Agents never merge their own branches — the Curator always merges after review
3. One active `promote/` branch at a time — serial promotion prevents index conflicts
4. Agent branches are deleted after merge (or after 30 days if abandoned)
5. `main` is the only branch that matters for recovery — all other branches are ephemeral

---

## Commit Conventions

### Format

```
type(scope): summary under 72 chars

Optional body: what changed and why (not how)

Refs: task-ref or transcript-id if applicable
```

### Allowed Types

| Type | When to use |
|---|---|
| `feat` | New content added (research, plans, drafts) |
| `promote` | Canonical promotion — a note moved from staging to canonical |
| `ingest` | Archiving raw transcripts or import batches |
| `refactor` | Reorganising files without changing content |
| `fix` | Correcting a factual error, broken link, or schema violation |
| `archive` | Moving notes to `07_Archive/` |
| `lint` | Vault health fixes (orphans, broken links, stale TTLs) |
| `conflict` | Flagging or resolving a semantic conflict |
| `snapshot` | Daily vault snapshot (no content change) |
| `chore` | System maintenance (hook updates, schema evolution, script changes) |

### Allowed Scopes

```
project/FounderOS
project/CinematicOS
project/IAMBrain
project/SlowEncodes
project/SocialMedia
knowledge
content
operations
system
vault
```

### Examples

```bash
promote(project/FounderOS): add ADR-001 git branch strategy from 2026-05-13 distillation
feat(project/FounderOS): add Builder output — vault architecture plan sprint 4
ingest(vault): archive 12 transcripts batch-02 triage complete
lint(vault): fix 3 broken wikilinks, flag 1 expired TTL
conflict(knowledge): flag contradiction knw-20260510-00031 vs knw-20260519-00047
snapshot(vault): daily snapshot 2026-05-19
chore(system): schema migration 2.0 → 2.1 add domain field
```

---

## Agent Commit Workflow

Every agent follows this sequence for every commit:

```bash
# 1. Fetch latest state
git fetch origin

# 2. Ensure on correct agent branch
git checkout agent/{name}/{task-slug}
# or create it
git checkout -b agent/{name}/{task-slug}

# 3. Stage only files in allowed write zones
git add 02_Projects/{PROJECT}/06_AI_Outputs/{agent}/
git add 00_Inbox/Staging/
# git add 08_Memory/Operational/ (if operational notes were written)

# 4. Pre-commit checks run automatically via hook
# (see Pre-commit Hooks section)

# 5. Commit
git commit -m "feat(project/FounderOS): builder output — vault architecture sprint 4"

# 6. Push agent branch
git push origin agent/{name}/{task-slug}
```

**Never run `git add .` — always stage explicitly by path.**

---

## Curator Promotion Workflow

The Curator uses `promote/` branches for all canonical changes:

```bash
# 1. Fetch latest
git fetch origin
git checkout main
git pull --ff-only

# 2. Create promotion branch
git checkout -b promote/2026-05-19-founderos-adr001

# 3. Stage promotion — copy/move from staging to canonical path
# (done by Curator manually or via script)

# 4. Validate
python3 scripts/validate_schema.py --staged
python3 scripts/check_id_immutability.py --staged
python3 scripts/scan_secrets.py --staged

# 5. Stage and commit
git add 04_Knowledge/ 08_Memory/Institutional/ 09_System/index.md 09_System/audit-log.md 09_System/id-counter.txt
git commit -m "promote(project/FounderOS): add 3 canonical notes from 2026-05-13 distillation"

# 6. Merge to main (fast-forward only)
git checkout main
git merge --ff-only promote/2026-05-19-founderos-adr001

# 7. Push
git push origin main

# 8. Delete promotion branch
git branch -d promote/2026-05-19-founderos-adr001
git push origin --delete promote/2026-05-19-founderos-adr001
```

---

## Ingestion Workflow

For archiving batches of raw transcripts:

```bash
# 1. Create ingestion branch
git checkout -b ingest/2026-05-19-batch-02

# 2. Copy transcripts to 00_Inbox/Raw/ (immutable after this step)
cp -r ~/exports/batch-02/*.md 00_Inbox/Raw/
# Add minimal metadata headers (see DISTILLATION_WORKFLOW.md)

# 3. Update ingestion manifest
# (append rows to 09_System/ingestion-manifest.csv)

# 4. Commit
git add 00_Inbox/Raw/ 09_System/ingestion-manifest.csv
git commit -m "ingest(vault): archive 15 transcripts batch-02 all triage scored"

# 5. Merge to main
git checkout main
git merge --ff-only ingest/2026-05-19-batch-02
git push origin main

# 6. Cleanup
git branch -d ingest/2026-05-19-batch-02
```

---

## Pre-commit Hooks

Save this as `.git/hooks/pre-commit` and make it executable (`chmod +x .git/hooks/pre-commit`):

```bash
#!/bin/bash
set -e

echo "=== Pre-commit validation ==="

# 1. Schema validation — check required frontmatter fields on staged canonical notes
python3 scripts/validate_schema.py --staged
if [ $? -ne 0 ]; then
    echo "ERROR: Schema validation failed. Fix frontmatter before committing."
    exit 1
fi

# 2. ID immutability — reject if any existing id: field was changed
python3 scripts/check_id_immutability.py --staged
if [ $? -ne 0 ]; then
    echo "ERROR: Immutable ID was modified. Revert the id: field change."
    exit 1
fi

# 3. Secret scan — reject if any API keys, tokens, or passwords detected
python3 scripts/scan_secrets.py --staged
if [ $? -ne 0 ]; then
    echo "ERROR: Potential secret detected in staged files. Remove before committing."
    exit 1
fi

# 4. Raw archive protection — reject writes to 00_Inbox/Raw/
STAGED_RAW=$(git diff --cached --name-only | grep "^00_Inbox/Raw/" | grep -v "^00_Inbox/Raw/\.gitkeep$" || true)
if [ -n "$STAGED_RAW" ]; then
    # Allow new files (ingest commits), block modifications
    for f in $STAGED_RAW; do
        if git ls-files --error-unmatch "$f" 2>/dev/null; then
            echo "ERROR: Modifying existing file in Raw archive is forbidden: $f"
            exit 1
        fi
    done
fi

# 5. Canonical zone protection — agents must not commit to canonical paths
CURRENT_BRANCH=$(git rev-parse --abbrev-ref HEAD)
if [[ "$CURRENT_BRANCH" == agent/* ]]; then
    FORBIDDEN=$(git diff --cached --name-only | grep -E "^(04_Knowledge/|06_Operations/SOPs/|08_Memory/Institutional/|09_System/SCHEMA\.md|09_System/index\.md|09_System/audit-log\.md)" || true)
    if [ -n "$FORBIDDEN" ]; then
        echo "ERROR: Agent branch cannot commit to canonical zones:"
        echo "$FORBIDDEN"
        exit 1
    fi
fi

echo "=== Pre-commit validation passed ==="
exit 0
```

---

## Validation Scripts

### scripts/validate_schema.py

```python
#!/usr/bin/env python3
"""Validates frontmatter on staged or all canonical notes."""
import subprocess, yaml, pathlib, sys, re

REQUIRED_CANONICAL = ['id', 'title', 'summary', 'type', 'project', 'status',
                       'canonical', 'created', 'updated', 'canonicality',
                       'confidence', 'source', 'schema_version']
CANONICAL_PATHS = ['04_Knowledge/', '06_Operations/SOPs/', '08_Memory/Institutional/']

def get_staged_files():
    result = subprocess.run(['git', 'diff', '--cached', '--name-only'], capture_output=True, text=True)
    return [f for f in result.stdout.strip().split('\n') if f.endswith('.md') and f]

def parse_frontmatter(filepath):
    text = pathlib.Path(filepath).read_text()
    match = re.match(r'^---\s*\n(.*?)\n---', text, re.DOTALL)
    if not match:
        return {}
    try:
        return yaml.safe_load(match.group(1)) or {}
    except yaml.YAMLError:
        return None

def validate_file(filepath):
    errors = []
    fm = parse_frontmatter(filepath)
    if fm is None:
        return [f"Invalid YAML frontmatter in {filepath}"]
    if not any(filepath.startswith(p) for p in CANONICAL_PATHS):
        return []  # Only enforce on canonical zone files
    if not fm.get('canonical'):
        return []  # Only enforce on canonical:true notes
    for field in REQUIRED_CANONICAL:
        if field not in fm:
            errors.append(f"Missing required field '{field}' in {filepath}")
    if 'summary' in fm and len(str(fm['summary'])) > 150:
        errors.append(f"Summary exceeds 150 chars in {filepath}")
    if 'canonicality' in fm and fm['canonicality'] not in [3, 4, 5]:
        errors.append(f"Canonical note has canonicality < 3 in {filepath}")
    return errors

staged = '--staged' in sys.argv
files = get_staged_files() if staged else [str(p) for p in pathlib.Path('.').rglob('*.md')]
all_errors = []
for f in files:
    if pathlib.Path(f).exists():
        all_errors.extend(validate_file(f))
if all_errors:
    for e in all_errors:
        print(f"SCHEMA ERROR: {e}")
    sys.exit(1)
sys.exit(0)
```

### scripts/check_id_immutability.py

```python
#!/usr/bin/env python3
"""Rejects commits that modify existing id: fields."""
import subprocess, re, sys

def get_staged_diffs():
    result = subprocess.run(['git', 'diff', '--cached', '-U0'], capture_output=True, text=True)
    return result.stdout

diffs = get_staged_diffs()
violations = []
current_file = None
for line in diffs.split('\n'):
    if line.startswith('+++ b/'):
        current_file = line[6:]
    # A removed id: line (old value) paired with added id: line (new value) = modification
    if re.match(r'^-id:\s+knw-', line) and current_file:
        violations.append(f"Immutable ID modified in: {current_file}")

if violations:
    for v in violations:
        print(f"ID IMMUTABILITY ERROR: {v}")
    sys.exit(1)
sys.exit(0)
```

### scripts/scan_secrets.py

```python
#!/usr/bin/env python3
"""Detects potential secrets in staged files."""
import subprocess, re, sys

PATTERNS = [
    (r'sk-[a-zA-Z0-9]{32,}', 'OpenAI/Anthropic API key'),
    (r'[a-zA-Z0-9_-]{32,}:[a-zA-Z0-9_-]{32,}', 'Token pair'),
    (r'(?i)(password|passwd|secret|token|api_key)\s*[:=]\s*["\']?[a-zA-Z0-9!@#$%^&*()_+-]{8,}', 'Credential assignment'),
    (r'Bearer [a-zA-Z0-9_-]{20,}', 'Bearer token'),
]

def get_staged_content():
    result = subprocess.run(['git', 'diff', '--cached', '-U0'], capture_output=True, text=True)
    return result.stdout

content = get_staged_content()
violations = []
for line_num, line in enumerate(content.split('\n'), 1):
    if not line.startswith('+'):
        continue
    for pattern, label in PATTERNS:
        if re.search(pattern, line):
            violations.append(f"Line {line_num}: Possible {label}: {line[:80]}...")

if violations:
    for v in violations:
        print(f"SECRET SCAN WARNING: {v}")
    sys.exit(1)
sys.exit(0)
```

---

## Conflict Resolution

### Git merge conflicts (file-level)

```bash
# For non-canonical files (agent outputs, staging) — prefer incoming
git checkout --theirs {file}
git add {file}

# For canonical notes — STOP and escalate to human
git merge --abort
# File an escalation note, then resolve manually
```

### Semantic conflicts (knowledge-level)

These are handled outside Git — see SCHEMA.md conflict section and SKILL.md conflict handling.

---

## Daily Snapshot Strategy

Run automatically via cron at 23:59 local time:

```bash
#!/bin/bash
# scripts/daily_snapshot.sh
cd /home/ubuntu/vaults/business
git fetch origin

# Run daily lint
python3 scripts/vault_lint.py --output 01_Dashboard/vault-health.md
python3 scripts/ttl_check.py --output 01_Dashboard/vault-health.md --append

DATE=$(date +%Y-%m-%d)

# Stage health dashboard changes
git add 01_Dashboard/vault-health.md

# Snapshot commit (empty if nothing changed, using --allow-empty)
git commit --allow-empty -m "snapshot(vault): daily snapshot $DATE"
git push origin main
```

---

## Rollback Procedures

### Roll back the last commit

```bash
git revert HEAD --no-edit
git push origin main
# Log the revert in audit-log.md
```

### Roll back multiple commits

```bash
# Identify the good commit
git log --oneline | head -20

# Revert to it (creates new revert commits — preserves history)
git revert HEAD~3..HEAD --no-edit
git push origin main
```

### Emergency hard reset (use only after full backup confirmation)

```bash
# DANGEROUS — only if revert is impossible and corruption is severe
git fetch origin
git reset --hard origin/main~N  # N = number of commits to reverse
# This rewrites history — confirm with human before running
```

### Restore a deleted or overwritten file

```bash
# Find the last commit that had the file
git log --all -- path/to/file.md

# Restore it
git checkout {commit-hash} -- path/to/file.md
git commit -m "fix(vault): restore accidentally modified file"
```

---

## Disaster Recovery

### Scenario: Local machine lost

```bash
# On new machine
git clone {remote-url} /home/ubuntu/vaults/business
cd /home/ubuntu/vaults/business
pip install -r scripts/requirements.txt
python3 scripts/index_builder.py
python3 scripts/vector_index_builder.py
# Vault is operational
```

### Scenario: Remote repository lost

If the remote is lost and only the local clone exists:
1. Create new remote repository
2. `git remote set-url origin {new-remote-url}`
3. `git push --all origin`
4. `git push --tags origin`

### Scenario: Vault corrupted by agent writes to wrong zones

1. `git log --oneline` — identify the corruption commit
2. `git revert {commit-hash}` — revert it
3. Run `python3 scripts/vault_lint.py` to confirm clean state
4. Add pre-commit hook protection if the corruption vector was not already blocked
5. Log the incident in `09_System/audit-log.md`

---

## Audit Logging

Every Curator action must be appended to `09_System/audit-log.md`:

```markdown
## {YYYY-MM-DD HH:MM}

- **Action:** promote | ingest | archive | conflict | lint | rollback
- **Actor:** Curator | human | {agent_name}
- **Scope:** {project or global}
- **Summary:** {what happened in one sentence}
- **Notes promoted:** {list of IDs if applicable}
- **Git commit:** {commit hash}
```

The audit log is append-only. Never edit past entries. Never delete entries.

---

## Merge Policy Summary

| Branch type | Merge strategy | Who merges | To |
|---|---|---|---|
| `agent/*` | `--no-ff` (preserve agent branch history) | Curator | `main` via `promote/` |
| `promote/*` | `--ff-only` (clean linear history) | Curator | `main` |
| `ingest/*` | `--ff-only` | Curator | `main` |
| `hotfix/*` | `--ff-only` | Human | `main` |

`--ff-only` on `main` ensures the main branch history is always a clean linear chain. If `--ff-only` fails, the branch is out of date — rebase it onto `main` first.
