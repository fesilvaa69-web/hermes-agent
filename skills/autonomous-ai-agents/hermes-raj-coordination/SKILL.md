---
name: hermes-raj-coordination
description: Coordinate Hermes Agent with OpenClaw Raj — dual-agent workflow, message passing, and SIAS integration between the two systems
version: 1.0.0
author: Hermes
tags: [openclaw, raj, multi-agent, coordination, sias, telegram]
category: autonomous-ai-agents
created: 2026-04-20
---

# Hermes ↔ Raj (OpenClaw) Coordination

Dual-agent setup where:
- **Raj** = OpenClaw orchestrator (planner/scheduler) 
- **Hermes** = executor/analyst (this agent)
- Communication via Telegram topics

---

## Architecture

```
[Felipe] → [Telegram] → [Raj (OpenClaw gateway)]
                            ↓
                      [Hermes Agent]
```

- Raj: `~/.openclaw/workspace/` — `openclaw-gateway` daemon + `raj_learning_loop.py`
- Hermes: `~/.hermes/` — separate agent instance
- Both connected to same Telegram bot (bot ID: 853334...)

---

## Key Paths

```bash
OpenClaw CLI:      ~/.npm-global/bin/openclaw
Raj workspace:     ~/.openclaw/workspace/
Raj learning loop: ~/.openclaw/workspace/raj_learning_loop.py
SIAS scheduler:    ~/.openclaw/workspace/sias/
Cortex docs:       ~/.openclaw/workspace/cortex/references/
OpenClaw logs:     ~/.openclaw/logs/
Commands log:      ~/.openclaw/logs/commands.log
Raj learning log:  ~/.openclaw/logs/raj_learning.log
Hermes cron out:   ~/.hermes/cron/output/
```

---

## CLI Reference

### OpenClaw Commands (full path required)

```bash
# Check OpenClaw version
~/.npm-global/bin/openclaw --version

# List agents
~/.npm-global/bin/openclaw agents list

# Send message to Raj (TIMEOUT if agent is busy — use Telegram instead)
~/.npm-global/bin/openclaw agent --agent main --message "text" --deliver --reply-channel telegram --reply-account default --reply-to 5556338087

# Trigger Raj learning loop manually
python3 ~/.openclaw/workspace/raj_learning_loop.py --trigger-daily
python3 ~/.openclaw/workspace/raj_learning_loop.py --trigger-insight
python3 ~/.openclaw/workspace/raj_learning_loop.py --list
```

### Process Monitoring

```bash
# Check if Raj is running
ps aux | grep -E "openclaw|raj" | grep -v grep

# Watch Raj learning log
tail -f ~/.openclaw/logs/raj_learning.log

# Check commands received
tail ~/.openclaw/logs/commands.log
```

---

## Message Delivery (Preferred Method)

When `openclaw agent` command times out, use **Telegram topic messaging** instead:

```python
# Via Hermes send_message tool
send_message(
    message="📋 Status report for @Raj_AIgentBot",
    target="telegram:Teste / topic 1862"  # Raj's Telegram topic
)
```

Find Raj's topic ID from `~/.openclaw/telegram/update-offset-*.json` or commands.log.

---

## SIAS Integration Workflow

Docs referenced:
- `WORKFLOW_CICLO_EVOLUTION.md` — macro cycle (Phase A-F)
- `SIAS_APPLIED_EXAMPLES.md` — tool archetypes and patterns
- `HERMES_TESTING_PROTOCOL.md` — testing methodology

### Execute SIAS Cycle

1. **Phase A (Exploration)**: Canary scan via delegate_task
2. **Phase B (Diagnosis)**: ICE scoring framework
3. **Phase C (Research)**: External sources per SIAS docs
4. **Phase D (Testing)**: 4-layer system (Smoke → Unit → Integration → Stress)
5. **Phase E (Integration)**: Monitoring and DIAGRAM update

```bash
# Output location for SIAS artifacts
mkdir -p ~/.hermes/cron/output/
```

### SIAS Priority Findings (2026-04-20)

| Dimension | Score | Gap |
|-----------|-------|-----|
| Testes | 3.8% | **66.2pp** ← CRITICAL |
| SIAS | 60% | 20pp |
| Observabilidade | 60% | 15pp |

---

## Raj Agent Delegation

Raj can delegate to 6 specialized agents:

| Agent | ID | Telegram Account |
|-------|-----|------------------|
| DATA | data_agent | data |
| BACKTEST | backtest_agent | backtest |
| CORE | core_agent | core |
| ML | ml_agent | ml |
| RADAR | radar_agent | radar |
| GUARDIAN | guardian_agent | guardian |

Delegation command (avoid — use Telegram instead):
```bash
~/.npm-global/bin/openclaw agent --agent data_agent --message "task" --deliver \
  --reply-channel telegram --reply-account data --reply-to 5556338087
```

---

## Pitfalls

1. **`openclaw: command not found`** — Use full path `~/.npm-global/bin/openclaw`
2. **Agent message times out** — Raj is processing; use Telegram topic instead
3. **Gateway not responding** — Check `ps aux | grep openclaw-gateway` and restart if needed
4. **SIAS scheduler not running** — Verify with `tail ~/.openclaw/logs/sias_scheduler.log`

---

## Verification

```bash
# Check Raj status
ps aux | grep raj_learning | grep -v grep
# Expected: python3 ...raj_learning_loop.py --start

# Check OpenClaw gateway
ps aux | grep openclaw-gateway | grep -v grep  
# Expected: openclaw-gateway running

# Verify Telegram connection
tail ~/.openclaw/logs/commands.log | grep "agent:main"
# Should show recent message receipts
```

---

## RADAR Agent Known Issues (2026-04-20)

Critical bugs found and fixed in `radar_telegram_scheduler.py`:

| Bug | Symptom | Fix |
|-----|---------|-----|
| Bare `except:` clauses | Silent failures | Replace with specific exception handlers |
| `send_telegram_message_with_retry` typo | NameError on any send | Correct name: `send_telegram_with_retry` |
| Invalid briefing types (`abertura`, `teste`) | Silent rejection by `briefing_v2.py` | Added whitelist + fallback to `"update"` |
| Import path issues (`trading_agent` module) | `No module named` errors | Use `importlib.util.spec_from_file_location()` |
| No job-level exception protection | One crash kills scheduler | Added `@job_wrapper` decorator to all jobs |

**Verification:**
```bash
tail -30 ~/.openclaw/logs/radar_scheduler.log
```

---

## Raj ↔ Hermes Skills Sync

Bidirectional skill sync established (2026-04-20):

| Direction | Skills Synced |
|-----------|---------------|
| Raj → Hermes | hermes-token-debug, port-security-check, telegram-guard-rails, menu-telegram |
| Hermes → Raj | systematic-debugging, test-driven-development, sias-cycle-execution |

**Files created:**
- `cortex/skills_sync.py` — bidirectional sync script with hash-based change detection
- `cortex/synapse/RAJ_HERMES_SKILLS_GAP_ANALYSIS.md` — full gap analysis
- `cortex/synapse/skill_sync_state.json` — sync state tracking

**Usage:**
```bash
# Dry-run first
python3 ~/.openclaw/workspace/cortex/skills_sync.py --dry-run

# Execute sync
python3 ~/.openclaw/workspace/cortex/skills_sync.py

# Status check
python3 ~/.openclaw/workspace/cortex/skills_sync.py --status
```

---

## Parallel Execution Pattern

When Felipe says "both should do X" or mentions @Raj_AIgentBot together:
1. **Start your portion immediately** — don't wait for Raj to respond
2. Send coordination message to Raj via Telegram topic
3. Execute your phase in parallel
4. Compare/synthesize results when both complete

Raj and Hermes are independent workstreams. Coordination messages are for alignment, not sequencing.

## SIAS Coordination (2026-04-20)

**Raj's Telegram topic:** `-1003872232486:3394`

**SIAS Database:** `~/.openclaw/workspace/sias/improvement_db.sqlite`

Key queries for health check:
```python
# Get agent coverage scores
cursor.execute("""
    SELECT agent_id, coverage_score, health_status, snapshot_date
    FROM agent_health_snapshots ORDER BY snapshot_date DESC
""")

# Get running cycles
cursor.execute("""
    SELECT id, agent_id, focus_area, cycle_number, status
    FROM improvement_cycles WHERE status = 'running'
""")

# Get blocked/low-score proposals
cursor.execute("""
    SELECT id, agent_id, focus_area, quality_score, blocked_by_layer
    FROM improvement_proposals
    WHERE quality_score < 0.5 OR blocked_by_layer > 0
    ORDER BY quality_score ASC LIMIT 10
""")
```

**Latest SIAS State (2026-04-20):**
| Agent | Coverage | Status |
|-------|----------|--------|
| ML | 36.3% | needs_attention |
| RADAR | 36.3% | needs_attention (cycle #20 running) |
| CORE | 56.0% | needs_attention |
| GUARDIAN | 53.0% | needs_attention |
| DATA | 75.7% | needs_attention |
| BACKTEST | 92.3% | needs_attention |

Cycles: 86 completed, 1 running.

## When to Use This Skill

- User mentions "Raj", "OpenClaw", "Raj_AIgentBot"
- Multi-agent orchestration with OpenClaw
- SIAS cycle execution (parallel with Raj when requested)
- Delegation to specialized agents (DATA, CORE, ML, etc.)
- Troubleshooting Raj/Hermes communication
- RADAR agent health issues (check coverage_score)
- Cross-agent skill inheritance setup
