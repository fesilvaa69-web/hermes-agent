---
name: audit-convergence
description: Cross-validate internal documentation against external audit data — compare, identify gaps, create missing structures, update diagrams. Used when surface scans differ from deep audits.
triggers:
  - inventory vs audit discrepancy
  - new diagram created but audit exists
  - migration or cross-validation between agents
---

# Audit Convergence — Raj + Hermes Joint Protocol

## Trigger Conditions
- When a new inventory/diagram is created but an external audit exists
- When there are discrepancies between "surface scan" and "deep scan" data
- When migrating or cross-validating documentation between agents

## Purpose
Cross-validate internal documentation against external audit data, identify gaps, and create missing structures in a systematic way.

## Phases (4-Phase Protocol)

### PHASE 1: COMPARAÇÃO
**Executor:** Hermes
1. List what OUR diagram says exists
2. List what AUDIT says exists
3. Create `audit_comparison.json`

**Output:** `vm_state/audit_comparison.json`

### PHASE 2: IDENTIFICAÇÃO
**Executor:** Raj + Hermes (joint)
1. List all gaps with severity
2. Identify false information in existing docs
3. Prioritize by impact

**Output:** Gap list with priority ranking

### PHASE 3: CORREÇÃO
**Executor:** Hermes (on Raj approval)

Create missing structures:
1. `progress_tracker.json` — 6-phase status + scores
2. `quarantine_registry.json` — all quarantined files
3. `sias_registry.json` — all SIAS cycles
4. Update diagram to new version (vX.Y)

### PHASE 4: VALIDAÇÃO
**Executor:** Raj + Hermes → @Fe
Present validated result to human.

## Critical Learnings from AUDIT-CONJUNTA-001

| Learning | Impact |
|----------|--------|
| Surface scan (18 tools) ≠ Deep scan (80 files) | Always check for full audit before surface inventory |
| Quarantine in multiple dirs | Check ALL locations |
| SIAS cycles may exist undocumented | Cross-check with audit |
| knowledge_base CAN exist even if diagram says doesn't | Verify filesystem |

## Files Created

```
vm_state/
├── audit_comparison.json      # PHASE 1
├── progress_tracker.json      # PHASE 3
├── quarantine_registry.json    # PHASE 3
└── vm_diagram.md → vX.Y       # PHASE 3

sias_state/
└── sias_registry.json         # PHASE 3
```

## Verification Commands

```bash
# Find all quarantine directories
find ~/.openclaw -type d -name "*quarantine*" 2>/dev/null

# Count files in quarantine
find ~/.openclaw -path "*quarantine*" -type f 2>/dev/null | wc -l

# Check for existing deep scan inventories
find ~/.openclaw -name "inventario*.json" -o -name "*audit*.json" 2>/dev/null
```

## Version Bump Rule
- v1.0: Initial real data
- v1.1: Roadmap + quality baseline
- v1.2: Audit convergence (corrections applied)
- Bump minor for fixes, major for structural changes

## Pitfalls
- **DON'T** assume surface inventory is complete
- **DON'T** mark things non-existent without verifying filesystem
- **DON'T** proceed PHASE 3 without formal task approval
- **DO** use exact filesystem paths, not assumptions
